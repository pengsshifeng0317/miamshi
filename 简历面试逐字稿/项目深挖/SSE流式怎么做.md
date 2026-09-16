# 模拟面试逐字稿：SSE 流式怎么做

> 简历产出：主导设计并实现寿险智能问答平台的 SSE 流式协议与断连治理 —— 双端事件契约、跨节点取消协议、限流拒绝的流内表达、两级首字延迟度量，并把协议从 RAG 六事件演进到 Agent 九事件。
> 覆盖模块：流式协议 / `SseEmitterSender` / `StreamTaskManager` / `AgentStreamEventBridge` / 双端流消费
> 熟练度：🔴

> **口径声明**：本篇按「主负责」档讲——流式协议与双端契约是我设计的。代码锚点均为 `ragent` 真实实现，配置数字以 `application.yaml` 为准。
> **两套协议**：`/rag/v3/chat`（workflow 档，6 个事件）是底座；`/agent/v1/chat`（agent 档，9 个事件）是它的演进。**Q1~Q9 讲底座，Q10 起讲演进。**
> **与旧稿的边界**：协议层之外（编排 / 工具 / 记忆 / 确认机制的新老对比）见《[Agent化升级（新老机制对比）.md](../项目深挖v2-升级清单/Agent化升级（新老机制对比）.md)》，本篇不重复。

---

## 🎤 开场总览（45 秒版，可直接背）

> 我先说**为什么做这件事**：大模型一轮回答要几十秒，同步等一个 `Result` 回来，用户面对的是几十秒白屏——这个体验不能上线。所以整条问答链路必须建立在流式协议上，而且这个协议要**同时被浏览器和评测脚本消费**，它是一份**对外契约**，不是内部实现细节。

具体四件事：

**第一，定事件契约。** 一条 `GET /rag/v3/chat` 建流，返回 `SseEmitter`，超时 5 分钟。事件是**两级**的：外层 `event:` 是协议语义，内层 body 的 `type` 是渲染语义。workflow 档六个事件，高频帧只有 `message` 一种。**埋一个点**：`finish` 和 `done` 是两个不同的终态帧，不是一个。

**第二，断连不能只关连接。** SSE 长连接随时会断，断了之后模型那条上游流还在烧 token，还占着限流信号量。所以做了 `StreamTaskManager`——**跨节点**取消协议：本地缓存 + Redis 取消标记 + Redis 属主 + Redisson 广播。**埋一个点**：taskId 是雪花 ID、时间有序可预测，它不能当访问凭证用。

**第三，拒绝也要走流内。** 被限流拒绝时不返回 429，而是在**已经建好的流上**发 `reject` 帧——因为 META 早发出去了，协议必须自洽。**埋一个点**：这件事我们做了一半，出错那条路还没统一进来。

**第四，协议跟着引擎演进。** 从 workflow 换到 agent 档时，六个事件不够用了——Agent 一轮里有**工具调用**和**中断在环确认**，所以演进到九个事件，新增 `block` / `tool` / `hint` / `confirm`，并且把「块的服务端起止」这种东西从客户端猜测改成了服务端声明。**埋一个点**：这不是加事件，是把协议从「文本流」升级成「过程流」。

**再补一句度量**：我埋了**两级** TTFT——`LLM_TTFT` 量模型首包，`USER_TTFT` 量用户感知首包。**埋一个点**：这两级的差值就是检索、意图、改写这些前置开销，A/B 之后我发现这笔账比我想的大。

一句话总结：**协议对外一致、连接对内可控、拒绝流内闭环、过程可被声明、性能可归因。**

---

## Q1（开场总览）介绍一下你的流式链路是怎么做的？从请求进来到第一个字推给前端，中间发生了什么？

**面试官追问**：从请求进来到第一个字推给前端，中间发生了什么？

**我**：按一次请求的生命周期看，分五段。

```
浏览器                网关             ragent 实例                          下游
  │                   │                   │                                │
  │ GET /rag/v3/chat?question=..&conversationId=..&deepThinking=..         │
  ├──────────────────►│                   │                                │
  │                   ├──────────────────►│ ① @IdempotentSubmit 幂等闸（按 userId）
  │                   │                   │ ② new SseEmitter(300_000ms)
  │                   │                   │ ③ 建 StreamChatEventHandler
  │                   │                   │    └─ 构造器里就发 META 帧 ────►│ 前端立刻拿到
  │                   │                   │       + taskManager.register   │ conversationId/taskId
  │                   │                   │ ④ chatQueueLimiter.enqueue     │
  │                   │                   │    ├─ 拿到并发许可 ─┐          │
  │                   │                   │    └─ 排队 15s 超时 ─► REJECT   │
  │                   │                   │ ⑤ pipeline.execute(ctx)        │
  │                   │                   │    ├─ 记忆 load                │
  │                   │                   │    ├─ 问题重写 / 意图识别       │
  │                   │                   │    ├─ 多路检索 → RRF → Rerank  │
  │                   │                   │    └─ LLM 流式 generate ──────►│ 模型
  │  event: meta      │◄──────────────────┤                                │
  │  event: message   │◄──────────────────┤ onContent / onThinking 回调    │
  │    {type:response}│                   │   └─ sendChunked 按码点切       │
  │  event: message   │◄──────────────────┤                                │
  │    {type:think}   │                   │                                │
  │  event: finish    │◄──────────────────┤ 先落库拿 messageId，再发        │
  │  event: done      │◄──────────────────┤ 字面量 "[DONE]"                │
  │                   │                   │  unregister + complete()       │
```

关键在于**顺序**，有三处是刻意排的：

**一，META 帧在限流入队之前就发了。** `StreamChatEventHandler` 的**构造器里**就 `sendEvent(META, new MetaPayload(conversationId, taskId))` 然后 `taskManager.register(taskId, userId, this::finishCancelledStream)`（`StreamChatEventHandler.java:90-93`）。为什么不等拿到许可再发？因为前端需要**立刻**拿到 `conversationId` 和 `taskId`——没有 `conversationId` 它没法把消息挂进会话列表，没有 `taskId` 它没法渲染那个「停止生成」按钮。用户排队 15 秒期间，界面不该是死的。

**代价是**：被限流拒绝时前端已经收到 META 了，所以拒绝链必须自己补齐 `META → REJECT → FINISH → DONE` 四条帧才自洽（`ChatQueueLimiter.java:145-156`）。这是用连贯性换来的成本，我认。

**二，`emitter` 是穿过限流器进去的，不是限流器外面包一层。** 看签名——`chatQueueLimiter.enqueue(question, conversationId, emitter, () -> traceRunner.run(...))`（`RAGChatServiceImpl.java:56-67`）。限流器拿到的是**已经建好的 emitter** 和一段"拿到许可后该干什么"的 supplier。所以「拒绝」不是我 return 一个错误码，而是**在这个 emitter 上走一套拒绝事件流**。这个结构直接决定了 Q9 的答案。

**三，业务逻辑跑在 `chatEntryExecutor` 上，不是 Tomcat 线程。** SSE 的 Servlet 线程必须在 `streamChat()` 返回后就能释放，否则并发上不去。pipeline 执行体被丢进 `chatEntryExecutor`，并用 `TtlRunnable.get(onAcquire)` 包了一层（`ChatQueueLimiter.java:77`）——执行线程跟接收请求的线程不是同一个，`ThreadLocal` 里的用户上下文、traceId 会丢，TransmittableThreadLocal 才能透传过去。

**超时是两档的**：`rag.default.sse-timeout-ms: 300000`（5 分钟）给 workflow 档，`agent.sse-timeout-ms: 900000`（15 分钟）给 agent 档。为什么差三倍——Agent 一轮里可能有多次工具调用 + 中间还要停下来等人确认，5 分钟不够。而且确认挂起时那个 `SseEmitter` 是**一直开着**的（Q12 讲）。

**埋钩子**：你可能注意到，我全程说「推事件」而不是「推 token」。因为我推给前端的粒度不是模型吐的粒度——这个在 Q5 讲。

---

## Q2（追问）为什么用 SSE？为什么不用 WebSocket，或者干脆长轮询？

**面试官追问**：SSE 听着最弱——单向、还有连接数限制，你为什么选它？

**我**：因为**我的场景恰好只需要单向**，SSE 的缺点我一个都没踩到，WebSocket 的成本我却要全付。

先摆需求：模型生成是**服务端→客户端的单向增量推送**；客户端往上行的只有一个完整问题，它是一条**新的 HTTP 请求**，不是同一条连接上的消息。也就是说这条长连接上**上行流量是零**。

| 维度 | 长轮询 | WebSocket | SSE |
|---|---|---|---|
| 传输 | 每轮一次 HTTP 建连 | 协议升级 `101 Switching Protocols` | HTTP/1.1 chunked，`text/event-stream` |
| 方向 | 单向 | **全双工** | 单向 |
| 增量粒度 | 每轮一个 HTTP 响应，粒度粗 | 帧级 | 事件级 |
| 断线重连 | 自己写 | 自己写（心跳 + 重连 + 状态恢复） | 协议内置（`retry:` / `Last-Event-ID`） |
| 自定义 header | 可 | 可（握手阶段） | **可（就是普通 HTTP 请求）** |
| 网关兼容 | 好 | 要网关支持升级 + 长连接时长 | 好（就是个不结束的 HTTP 响应） |
| 我的上行需求 | — | 零 | 零 |

**长轮询直接出局**：每轮建一次连接，模型一秒吐十几个 token 的场景下，轮询间隔要么长到看不出「流」，要么短到把连接数打爆。而且它延迟的下限就是轮询周期——你在流式体验上做的一切努力都被这个周期抹平。

**WebSocket 是被「用不上」淘汰的**。它贵在三件事：协议升级握手、自己实现心跳与重连、把「消息帧」语义全部自己定义一遍。我这条连接上行是零，全双工那部分付了钱用不上。而且 WS 在网关侧通常要单独配超时和空闲策略，运维面更大。

**SSE 的代价我很清楚，就三条**：

1. **单向**。我的上行本来就不是同一连接的消息，不构成限制。**反过来说**——如果哪天要做「生成中途插话让模型改方向」，就得换 WebSocket，这是选型边界。
2. **HTTP/1.1 同域并发连接数受限**（浏览器一般 6 个）。这是真限制：一个用户开 6 个标签页同时问就满了。**兜底是上 HTTP/2**——多路复用在一条 TCP 上跑，限制消失，SSE 反而更合适。
3. **无法用 `EventSource` 自定义请求。** 这条最实际，直接决定了前端实现方式。

**第三点展开讲，这是很多人答不上来的地方。** 原生 `EventSource` 有两个硬限制：**不能自定义请求头、不能取消**。而我们需要：

- 带 `Authorization` 头（sa-token 的 token，注意**不带 `Bearer` 前缀**），`EventSource` 加不了 header；
- 用户点「停止生成」时要能**主动中断这个流**。`EventSource` 只有 `close()`——那是关连接，跟「取消这一轮推理」不是一回事，而且拿不到取消的收尾语义。

所以前端**没走 `EventSource`，是手写 `fetch` + `ReadableStream`**：`fetch` 可以带 header、可以挂 `AbortSignal`（`useStreamResponse.ts:135-142`）。`createStreamResponse` 返回 `{ start, cancel }`，`cancel()` 就是 `controller.abort()`（`:171-174`）。

**代价**：`EventSource` 免费给的**流式解析**我得自己实现，协议里所有"跨 HTTP chunk 边界"的坑都得自己填。这是 Q14。

**埋钩子**：既然后端和前端都是自己写的解析，那协议里到底定义了哪些帧、字段结构怎么定，就变成我自己的责任了。

---

## Q3（追问）事件契约是怎么设计的？为什么是两级结构？

**面试官追问**：你说事件分两级，为什么要分？直接一级不够吗？

**我**：一级够，但会让高频路径变复杂。先给全貌。

**workflow 档（`/rag/v3/chat`）六个事件**（`SSEEventType.java:26-66`）：

| `event:` | 语义 | payload | 频率 |
|---|---|---|---|
| `meta` | 会话与任务元信息 | `MetaPayload(conversationId, taskId)` | 1 次 |
| `message` | 增量消息 | `MessageDelta(type, delta)` | **高频**（逐 token） |
| `finish` | 模型回复完成（**数据结算帧**） | `CompletionPayload(messageId, title, sources, messageStatus)` | 1 次 |
| `done` | 流结束（**哨兵帧**） | 字面量 `"[DONE]"` | 1 次 |
| `cancel` | 被取消 | `CompletionPayload` | 0/1 次 |
| `reject` | 被限流拒绝 | `MessageDelta("response", "系统繁忙，请稍后再试")` | 0/1 次 |

**两级是这样切的**：

- **外层 `event:` 是「协议语义」**——这一帧在**协议状态机**里是什么性质（元信息 / 数据 / 结算 / 终止 / 异常），决定前端走哪个分支、要不要改协议状态。
- **内层 body 的 `type` 是「渲染语义」**——这份数据该往**哪个 UI 容器**放（`response` 正文区 / `think` 推理折叠区）。

**为什么不给 `response` 和 `think` 各开一个 `event:` 名？** 因为 `message` 是高频帧，一秒十几到几十次。用事件名区分，等于让**协议层**每 token 做一次分支——前端 `switch (eventName)` 分支数翻倍。更关键的是语义耦合：「推理内容」和「答案内容」**是同一份模型输出的两个通道**，不是两种协议事件；把它们升格成事件名，等于把渲染细节泄漏进了协议层。

放进 body 之后，`message` 就稳定成「有增量来了」，前端在**一个分支**里做渲染分流（`useStreamResponse.ts:57-64`）：

```ts
case "message":
  {
    const messagePayload = payload as MessageDeltaPayload;
    if (messagePayload?.type === "think") {
      handlers.onThinking?.(messagePayload);
    }
    handlers.onMessage?.(messagePayload);
  }
```

**代价说清楚**：协议层省了分支，**渲染层多了一次判断**，而且这个判断在最热路径上。我在前端 store 层又收口了一道——`chatStore.ts` 里 `onMessage` 过滤 `type !== "response"`、`onThinking` 过滤 `type !== "think"`。等于**分流判断做两次**。

我认这个代价，因为两次判断的成本远低于「协议层分支翻倍 + 渲染语义泄漏进协议」。而且有个实际好处：**加一个新渲染通道不用改协议**。Agent 档后来加 `error` 这个 type 时（`AgentStreamEventBridge.java:79`），协议层**一个字都没改**。

**另外一个细节**：`CompletionPayload` 上有 `@JsonInclude(NON_NULL)`，`sources` 为空时字段整个不序列化。这样「命中知识库」和「没命中」在前端是「有 `sources` 键」vs「没有」，不用发一个 `null` 让前端再判空。

**埋钩子**：你会注意到上表里有两个"完成"帧——`finish` 和 `done`。这不是冗余，它们职责完全不同，而且这里面藏着我这篇里最不体面的一处不一致。

---

## Q4（追问）`finish` 和 `done` 为什么要两个？终态怎么保证？

**面试官追问**：两个完成事件是不是重复设计？最后只发一个不行吗？

**我**：不行，因为它们解决两个不同问题——**一个解决"数据要落库后才能发"，另一个解决"前端怎么知道流真的结束了"**。

**`finish` 是数据结算帧。** 看载荷 `CompletionPayload(messageId, title, sources, messageStatus)`——里面那个 `messageId` 是**先落库拿到**再回填的：

```java
// StreamChatEventHandler.java:214-232（onComplete，有删节）
String messageId = null;
try {
    ChatMessage message = ChatMessage.assistant(answer.toString(), thinkingContent, resolveThinkingDuration());
    message.setSources(sources);
    message.setRetrievedChunks(groundingChunks);
    message.setReplyToMessageId(replyToMessageId);
    message.setMessageStatus(ChatMessage.MessageStatus.NORMAL);
    messageId = memoryService.append(conversationId, userId, message);   // ← 先落库
} catch (Exception e) {
    log.error("对话完成时持久化消息失败，conversationId：{}", conversationId, e);
}
String title = resolveTitleForEvent();
sender.sendEvent(SSEEventType.FINISH.value(),
        new CompletionPayload(messageIdText, title, sources, ChatMessage.MessageStatus.NORMAL));  // ← 再发
sender.sendEvent(SSEEventType.DONE.value(), "[DONE]");
```

所以 `finish` 天然是**有副作用的、有条件的**——落库可能失败，`messageId` 可能是 `null`。它携带的是「这一轮的最终结算结果」。

**`done` 是纯哨兵帧**，永远是字面量 `"[DONE]"`（OpenAI 风格）。不带任何数据，只有一个含义：**这条流到此为止，是正常收尾，不是断了。**

**为什么必须有它**——因为 **SSE 协议本身没有「正常结束」这个信号**。HTTP chunked 的结束就是连接关闭，而客户端**分不清「服务端正常关连接」和「网络被掐断」**，两者在 `fetch` 层都是 `reader.read()` 返回 `done: true`。所以必须有一个**应用层**的结束标记。

前端把它当**硬约束**用了（`useAgentStream.ts:142-147`）：

```ts
// 后端每条出口都以 done 封尾 没收到它就是连接被掐断
// 当成功静默收场 这一轮会永远停在「等待响应」
// 主动取消由上层自己收尾 不算异常
if (!terminated && !signal?.aborted) {
  throw new Error("连接已中断，本轮回答未完成");
}
```

意图很清楚：**「有没有收到 done」是前端唯一能区分"正常结束"和"被掐断"的依据**。没收到又不是自己取消的，就是异常，必须抛——否则这一轮永远停在 loading 态。

### 现在讲不体面的地方——两套协议的终态行为不一致

| 出口 | Agent 档 | workflow 档 |
|---|---|---|
| 正常完成 | `finish(NORMAL)` + `done` | `finish(NORMAL)` + `done` |
| 被取消 | `cancel` + `done` | `cancel` + `done` |
| **出错** | `message(type=error)` + `finish(INTERRUPTED)` + `done` | **什么都不发**，直接 `completeWithError` |
| 等确认 | `confirm` + `done` | 不适用 |

Agent 档出错时**保证发满终态**（`AgentStreamEventBridge.java:206-231`），而 workflow 档只有一行：

```java
// StreamChatEventHandler.java:235-242
public void onError(Throwable t) {
    if (taskManager.isCancelled(taskId)) {
        return;
    }
    taskManager.unregister(taskId);
    sender.fail(t);   // → emitter.completeWithError(t)
}
```

**不发任何 SSE 事件，不保证 `done`。** 后果是：workflow 档前端那个 `onError` 处理器是**死代码**——后端从来不发 `error` 事件，分支永远走不到。前端只能靠「流断了且没收到 done」这条兜底路径（就是上面那段 throw）感知异常。

**workflow 档为什么这么写，我理解原因**：`SseEmitterSender.fail()` 的注释很直白——「**不再抛出异常，避免在流式响应已开始后触发全局异常处理器导致响应冲突**」。响应已经 committed（响应头和前面的 chunk 都发出去了），这时候让异常穿透到 Spring 全局异常处理器，它会试图写一个 JSON 错误体，而响应体已经开始了——只会撞出二次异常。所以只能关连接。

**但这是我的判断失误**：我把「不能写 JSON 错误体」直接推成了「干脆什么都不发」，而正确做法是 Agent 档那个——**在流内发一条 error 增量 + 一个 `finish(INTERRUPTED)` 终态**，一样不碰响应头，但前端能拿到明确的终态和文案。

Agent 档那两行注释把理由写透了（`AgentStreamEventBridge.java:213-214`）：

> 中断提示单独成 error 块：这是系统在说话，混进 answer 就跟模型的回答一个身份了
> 块和当场的增量都要发，只塞进 content 的话历史回放有块就不读 content，刷新后这句就没了

第一句是**身份问题**（系统提示不能伪装成模型的话），第二句是**持久化一致性问题**（落库的块和流出的帧必须对得上，否则刷新后少一句话）。这两条我在 workflow 档都没考虑到。

还有一处同类的：**幂等拦截走的是第三条路**。`RAGChatController.chat()` 上有 `@IdempotentSubmit`（`:47-50`），同一用户重复提交时返回 **HTTP 200 + JSON 体**，不是事件流。前端在 Agent 档里专门为这个加了判断（`useAgentStream.ts:175-180`）：

```ts
// @IdempotentSubmit 拦截时返回 200 + JSON 体而非事件流 文案取自 JSON 体
const contentType = response.headers.get("content-type") || "";
if (!contentType.includes("text/event-stream")) {
  const errBody = (await response.json().catch(() => null)) as { message?: string } | null;
  throw new Error(errBody?.message || "请求失败");
}
```

所以现在**三种「没给你正常答案」的情况走了三条不同路径**：限流拒绝走流内（`reject` 帧）、幂等拦截走流外 JSON、出错走连接关闭。**它们本该统一成流内终态**——这是我最想改的地方。

**埋钩子**：说回 `message` 帧。我刚才提过，我推给前端的粒度不是模型吐的粒度，而且那个粒度有一个很容易踩坏的边界。

---

## Q5（追问）`response` 和 `think` 两条流怎么并行推？分块粒度怎么定的？

**面试官追问**：推理内容和答案内容是分别两个 event 吗？分块大小怎么定的？

**我**：它们**共用一个 `message` event，靠 body 的 `type` 区分**——这是 Q3 那个两级结构最大的收益点。

**两条流从哪来**：不是我们切出来的，是**模型 SDK 的两个独立回调**。`StreamCallback` 接口定义七个回调（`infra-ai/.../chat/StreamCallback.java`），其中：

```java
void onContent(String content);                    // 答案正文
default void onThinking(String content) { }         // 推理，默认空实现
```

`onThinking` 是 `default` 空实现——**不支持思考链的模型，这个接口不用改任何东西**。两条流各自累积、各自分块、天然交错，实现侧不需要做任何合并或排序：

```java
// StreamChatEventHandler.java:194-207（有删节）
public void onThinking(String chunk) {
    if (taskManager.isCancelled(taskId)) return;
    if (StrUtil.isBlank(chunk)) return;
    if (thinkingStartMs == 0) {
        thinkingStartMs = System.currentTimeMillis();   // 只在第一个 think chunk 打点
    }
    thinking.append(chunk);
    sendChunked(TYPE_THINK, chunk);
}
```

**分块粒度，`messageChunkSize`**：代码默认值是 **5**（`resolveMessageChunkSize`，`AIModelProperties.Stream.messageChunkSize` 为空时取 5，`StreamChatEventHandler.java:98-102`）——累积 5 个字符再发一帧，用来减少帧数。

**但我要诚实说一句：这个值在 `application.yaml` 里被配成了 1**（`ai.stream.message-chunk-size: 1`）。也就是说**生产实际是逐 token 即发，聚合逻辑没有真正生效**。代码默认值与生产配置不一致——聚合的意义（压帧数、降 render 次数）在生产上等于没有。我的判断是：当时为了「首字更快出来」把 chunk size 压到 1，但没意识到这同时放弃了帧合并的收益，这两个目标是有冲突的（详见 Q13）。

**真正的技术点在切法上——`sendChunked` 是按码点切的，不是按 char 切的**（`StreamChatEventHandler.java:244-263`）：

```java
private void sendChunked(String type, String content) {
    int length = content.length();
    int idx = 0;
    int count = 0;
    StringBuilder buffer = new StringBuilder();
    while (idx < length) {
        int codePoint = content.codePointAt(idx);              // ← 取整个码点
        buffer.appendCodePoint(codePoint);
        idx += Character.charCount(codePoint);                  // ← 按码点宽度步进
        count++;
        if (count >= messageChunkSize) {
            sender.sendEvent(SSEEventType.MESSAGE.value(), new MessageDelta(type, buffer.toString()));
            buffer.setLength(0);
            count = 0;
        }
    }
    if (!buffer.isEmpty()) {
        sender.sendEvent(SSEEventType.MESSAGE.value(), new MessageDelta(type, buffer.toString()));
    }
}
```

`codePointAt` + `Character.charCount` 这一对，跟 `charAt` 的区别是**代理对**。

**Java 的 `String.length()` 是 UTF-16 code unit 数，不是字符数**。emoji、生僻字（比如「𠮷」）、部分少数民族文字，在 UTF-16 里占**两个** code unit。如果用 `content.charAt(i)` 逐字符取、按 `messageChunkSize` 切断，**切点刚好落在一个代理对中间**时，就把一个完整字符劈成两个孤立的「半个代理」。

后果很具体：孤立代理**不是合法 UTF-16 序列**，JSON 序列化会变成 `"\uD83D"` 这样的转义；运到前端 `JSON.parse` 出来是**替换字符或乱码**，而且**永远恢复不了**——另一半在下一个 chunk 里，前端已经把上一帧渲染出去了。

用户看到的就是「回答里某个 emoji 变成乱码方块」，而且是**概率性的**——只有切点正好落在那个字符上才出现（chunkSize=5、emoji 密集的回答，概率不低）。

**代价**：`codePointAt` 比 `charAt` 慢一点，`charCount` 多一次调用。这是**纯 CPU 换正确性**，而这个正确性是「不出坏字符」级别的，没有商量空间。而且这个开销跟模型推理时间相比可忽略。

**顺带一个细节**：thinking 耗时只在**第一个 think chunk 到达时**打起点，在**第一个 response chunk 到达时**结算（`:187-189`），并且 `Math.max(1, ...)` 兜底至少 1 秒。为什么不结算在 `onComplete`？因为「思考了多久」的语义到答案开始生成就结束了——用户看到的推理折叠区时长应该是这么算的，不是整轮耗时。

**埋钩子**：上面每个回调第一行都是 `if (taskManager.isCancelled(taskId)) return;`。那问题是——**谁**把 `cancelled` 置上的？客户端断连了，服务端怎么知道？这是我花时间最多的一块。

---

## Q6（追问）`SseEmitterSender` 封装了什么？为什么 `sendEvent` 失败要静默丢弃？

**面试官追问**：为什么发送失败不抛异常？静默丢弃会不会丢内容？

**我**：它封装的是 **`SseEmitter` 的生命周期竞态**。三个点：状态归一、失败静默、关闭幂等。

```java
// SseEmitterSender.java:32-53（有删节）
public class SseEmitterSender {
    private final SseEmitter emitter;
    private final AtomicBoolean closed = new AtomicBoolean(false);   // ① 状态归一

    public SseEmitterSender(SseEmitter emitter) {
        this.emitter = emitter;
        emitter.onCompletion(() -> closed.set(true));
        emitter.onTimeout(() -> closed.set(true));
        emitter.onError(e -> closed.set(true));
    }
```

**① 状态归一。** `SseEmitter` 给了三个回调——`onCompletion`（正常关）、`onTimeout`（超时）、`onError`（异常）——但**对发送方来说这三种情况是一样的：不能再发了**。所以三个回调统一归到一个 `AtomicBoolean closed`。业务代码只需要问一个问题：「还开着吗」。

**② 失败静默**（`:67-80`）：

```java
public void sendEvent(String eventName, Object data) {
    if (closed.get()) {
        return;                                    // ← 已关：直接 return，不抛
    }
    try {
        if (eventName == null) {
            emitter.send(data);
            return;
        }
        emitter.send(SseEmitter.event().name(eventName).data(data));
    } catch (Exception e) {
        fail(e);                                    // ← 发送失败：吞掉，只记日志
    }
}
```

**为什么静默？** 因为**调用方没有能力处理这个异常**。`sendEvent` 的调用方是 pipeline 里一堆业务 handler（`onContent`、`onThinking`、`onSources`…），它们唯一的合理反应就是「哦，发不出去了」，然后继续把这一轮跑完——**数据库还得落、限流信号量还得还、task 还得 unregister**。如果让异常穿透出去，这一轮的**收尾逻辑全会被跳过**：

- 消息不落库 → 用户刷新后这轮对话消失；
- `taskManager.unregister` 不执行 → Guava cache 里留一条 30 分钟的僵尸任务，Redis 里留两个 key；
- 限流信号量的 permit 不还 → 全局并发额度被永久吃掉一个。

**所以「静默丢弃」不是丢内容，是「不让发送失败这个次要故障，升级成资源泄漏这个主要故障」。** 至于"丢内容"——内容在 `answer` / `thinking` 两个 `StringBuilder` 里一直累着（`:57-58`），落库用的是它，不是发出去的东西。**发送是尽力而为，落库是必须成功**，两条路径分开。

**③ 关闭幂等**（`:88-93`、`:119-124`）：

```java
public void complete() {
    // 使用 CAS 原子操作，确保只关闭一次
    if (closed.compareAndSet(false, true)) {
        emitter.complete();
    }
}
```

`complete()` 和 `closeWithError()` 都用 `compareAndSet(false, true)`。为什么必须幂等——因为**终态从多个路径到达**：正常收尾会 `complete()`；超时回调可能同时触发；用户点停止走 `finishCancelledStream` 里的 `complete()`。并发到达时如果都去调 `emitter.complete()`，Spring 的 `SseEmitter` 会抛 `IllegalStateException`（重复完成）。

而 `fail()` 里有个**故意的设计**（`:95-109`）：

```java
public void fail(Throwable throwable) {
    closeWithError(throwable);
    log.warn("SSE send failed", throwable);
}
```

注释原文：**「不再抛出异常，避免在流式响应已开始后触发全局异常处理器导致响应冲突」**。这是 Q4 讲的那个理由——响应已 committed，全局异常处理器想写 JSON 错误体会撞。所以这里**只关连接、只记日志、不抛**。

**代价**：异常信息只进日志，前端拿不到任何错误详情。这就是 Q4 里 workflow 档「出错时前端只能靠断流感知」的**根源**。Agent 档后来在流内补了 error 增量和 `finish(INTERRUPTED)`，才算把这个信息补回给前端。

**埋钩子**：`closed` 解决的是「本地这个 emitter 还能不能发」。但断连要处理的远不止这个——**模型那条上游流还在跑**，它才是真正要掐的东西。

---

## Q7（追问）客户端断连了，服务端怎么知道？怎么把上游的模型调用也停掉？

**面试官追问**：这个取消是本地的还是要跨节点？

**我**：必须跨节点。这是我这块花时间最多的地方，分四层讲：**为什么难 → 结构 → 竞态怎么闭合 → 越权怎么防**。

### 一、为什么难，难在三个约束

**约束一：取消的触发点在容器线程上，没有用户上下文。** SSE 断开时是 Tomcat 的容器回调线程通知我们（`onCompletion` / `onTimeout` / `onError`），那个线程里 `UserContext.getUserId()` 拿不到。所以「谁有权取消」这件事，不能在取消发生的那一刻判断。

**约束二：停止请求可能落在别的节点上。** 用户点「停止生成」，这个 `POST /rag/v3/stop` 由网关负载均衡到哪台实例是不确定的——很可能不是正在跑这条流的那台。所以取消信号必须**跨节点广播**。

**约束三：taskId 不能当凭证用。** 它是 `IdUtil.getSnowflakeNextIdStr()` 生成的雪花 ID——**时间有序、可预测**，攻击者能猜到附近的 ID。如果停止接口只凭 taskId 就执行，任何人都能停掉别人的流。

### 二、结构：本地 + Redis 标记 + Redis 属主 + 广播

```
                    ┌──────────── 实例 A（跑着这条流）────────────┐
                    │  Guava Cache<taskId, StreamTaskInfo>        │
                    │    · cancelled  : AtomicBoolean             │
                    │    · ownerUserId: volatile String           │
                    │    · cancelAction / finalizer: Runnable     │
                    └─────────────────────────────────────────────┘
                                    ▲
                                    │ ① 本地命中就直接处理
                                    │
  ┌── 实例 B（收到停止请求）──┐     │
  │  POST /rag/v3/stop        │     │
  │   ① 读 Redis 属主并比对 ──┼─────┤
  │   ② 写 Redis 取消标记 ────┼──┐  │
  │   ③ publish 广播 ─────────┼──┼──┘ ③ 所有实例的监听器都收到
  └───────────────────────────┘  │
                                 ▼
        Redis: ragent:stream:cancel:{taskId}  ← 取消标记（TTL 30min）
        Redis: ragent:stream:owner:{taskId}   ← 属主
        Pub/Sub: ragent:stream:cancel  (Redisson RTopic)
```

为什么**四个都得有**，缺一个会怎样：

| 组件 | 作用 | 缺了会怎样 |
|---|---|---|
| 本地 Guava Cache | 存回调、存状态，取消时**执行**收尾 | 每条取消都查 Redis，且回调没处注册 |
| Redis 取消标记 | **标记先于注册到达**的兜底 | 广播到达时任务还没 register，信号就丢了 |
| Redis 属主 | 跨节点**鉴权** | 停止请求落在没跑这条流的节点上，无法校验 |
| Redisson 广播 | 把取消**即时**送达所有节点 | 只能靠每 ticket 各自的 200ms 轮询兜底（见限流篇） |

本地缓存是 `CacheBuilder.newBuilder().expireAfterWrite(Duration.ofMinutes(30)).maximumSize(10000)`（`StreamTaskManager.java:62-65`）——30 分钟跟 SSE 超时对齐，10000 上限是兜底（注释说"基本上不可能超出这个数量"）。

### 三、竞态怎么闭合——这是最精妙的一处

`register` 里有**两行赋值的顺序**，是刻意的（`:101-110`）：

```java
public void register(String taskId, String ownerUserId, Runnable onCancelFinalizer) {
    StreamTaskInfo taskInfo = getOrCreate(taskId);
    // 属主必须先于收尾回调落地：反过来的话，两条赋值之间到达的广播能拿本地回调杀掉一条无主的流
    taskInfo.ownerUserId = ownerUserId;
    taskInfo.finalizer = onCancelFinalizer;
    // 属主进 Redis 而非只留本地：停止请求可能落在没跑这条流的节点上
    if (StrUtil.isNotBlank(ownerUserId)) {
        RBucket<String> owner = redissonClient.getBucket(ownerKey(taskId));
        owner.set(ownerUserId, CANCEL_TTL);
    }
    if (isTaskCancelledInRedis(taskId, taskInfo)) {
        onCancelFinalizer.run();
    }
}
```

**注释那句就是关键**：「属主必须先于收尾回调落地」。

想象并发时序：广播线程持有 `taskInfo` 引用，读 `ownerUserId` 和 `finalizer` 决定要不要执行取消。如果 `finalizer` 先赋值、`ownerUserId` 后赋值，那么在**这两条赋值之间**，广播线程看到的是「finalizer 非空、ownerUserId 为空」——一条**无主流**。它的鉴权怎么办？放行，等于任何人埋的取消标记都能杀掉这条流；拒绝，又可能误杀一条合法的系统侧取消。

**解法是用赋值顺序建立可见性契约**：先写 `ownerUserId`，再写 `finalizer`。这样广播线程只要能看到 `finalizer` 非空，就**必然**能看到 `ownerUserId` 非空——两个字段**不可能同时处于「回调就绪但属主未知」的状态**。竞态窗口从根上不存在，而不是靠加锁去堵。

**还有一个防短路的细节**（`:203-219`）：

```java
private void cancelLocal(String taskId, String requester) {
    StreamTaskInfo taskInfo = tasks.getIfPresent(taskId);
    if (taskInfo == null) return;

    // 执行端复核发起方：taskId 时间有序可预测，越权取消喷得中就成
    // 不匹配时连 cancelled 都不置——置了会让 register 的复核短路，等于把标记复核那道门绕开
    if (!isRequesterAllowed(taskInfo, requester)) {
        log.warn("拒绝越权取消流式任务，taskId：{}，属主：{}，发起方：{}", taskId, taskInfo.ownerUserId, requester);
        return;
    }

    if (!taskInfo.cancelled.compareAndSet(false, true)) {   // CAS 只执行一次
        return;
    }
    if (taskInfo.cancelAction != null) taskInfo.cancelAction.run();
    if (taskInfo.finalizer != null)  taskInfo.finalizer.run();
}
```

注意「**不匹配时连 `cancelled` 都不置**」。为什么？因为 `register` 里还有一道复核：`isTaskCancelledInRedis(taskId, taskInfo)`——它会在注册那一刻去 Redis 查一次取消标记。如果 `cancelLocal` 在鉴权失败时**好心**把 `cancelled` 置上，那 `register` 里那个 `if (taskInfo.cancelled.get())` 检查就会直接短路返回 `true`，**第二道复核永远走不到**。

**这是一个「防重复」和「防绕过」冲突的典型案例**：置上 `cancelled` 能让取消更及时，但会让鉴权复核这道门形同虚设。这里选了**安全优先**——宁可让一次越权广播被两道门都拦住，也不能让它从后门溜进去。

`isRequesterAllowed` 的判定（`:196-201`）：系统侧（`__system__`）**无条件放行**，用户侧**只认精确属主，属主还没落地时一律不认**。前半句是因为系统侧回收（SSE 超时、客户端断连）本来就没有登录用户可比对；后半句正是"预埋标记要在 register 那一刻复核"的窗口——**本地放过去就等于绕开了复核**。

### 四、越权怎么防——两处，都在权限边界上

**第一处，停止接口不比 `taskId` 比属主**（`:146-157`）：

```java
public void cancelByUser(String taskId) {
    String requester = UserContext.requireUser().getUserId();
    RBucket<String> owner = redissonClient.getBucket(ownerKey(taskId));
    String ownerUserId = owner.get();
    if (StrUtil.isNotBlank(ownerUserId) && !ownerUserId.equals(requester)) {
        log.warn("拒绝越权停止流式任务，taskId：{}，属主：{}，发起方：{}", taskId, ownerUserId, requester);
        // 不区分「不存在」与「非属主」，免得停止接口变成他人任务的探测器
        throw new ClientException("任务不存在或已结束");
    }
    // 属主查不到多半是任务已结束，也可能是注册还没落地，故标记带上发起方交给注册那一刻复核
    publishCancel(taskId, requester);
}
```

注释那句「**不区分「不存在」与「非属主」**」是安全设计：两种情况返回不同错误，攻击者就能拿停止接口当**探测工具**——「这个 taskId 存在吗？」扫一遍就知道哪些 ID 活跃。统一成一个模糊错误，接口就不泄露信息。

**第二处，广播载荷带发起方，执行端复核**。载荷格式是 `taskId|requester`（`PAYLOAD_SEPARATOR = "|"`）。为什么带发起方——注释说「**只有发布端校验挡不住属主落地前的抢跑**」。`cancelByUser` 的校验在**发布端**，真正的执行在**消费端**，中间有时间差：如果属主还没写进 Redis（`register` 还没跑到），`cancelByUser` 里那个 `if` 会因为 `ownerUserId` 为空而**跳过校验**直接发布。所以必须让执行端拿到 `requester` 再复核一次。

分隔符选 `|` 是有理由的：`taskId` 和 `userId` 都是**雪花数字串**，不含竖线，所以 `indexOf("|")` 分割安全。系统侧占位用 `__system__`（用户 ID 是雪花数字串，撞不上）。

**还有滚动升级的兼容**（`:81-85`）：老版本节点广播的是**裸 taskId**（没分隔符）。解析时 `separator < 0` 就当系统侧收——因为「它在发布前已做过同样的属主校验」。这样新老节点混跑期间不会互相误判。

**最后，`bindHandle` 有个"幽灵任务"防护**（`:120-129`）：任务已经 `unregister` 之后不再创建记录，否则会在 cache 里留一条**永远不会被触发的任务**——占内存、30 分钟后才过期，而且如果有人拿它的 taskId 来取消，会得到"成功"的假象。

**触发链最后接上**：`ChatQueueLimiter.enqueue` 里通过 `cancelBinder` 把容器的三个回调绑到取消上（`ChatQueueLimiter.java:80-84`）：

```java
.cancelBinder(cancel -> {
    emitter.onCompletion(cancel);
    emitter.onTimeout(cancel);
    emitter.onError(e -> cancel.run());
})
```

这里又出现一次 Q6 的模式——**三个容器回调归一到同一个动作**。

**埋钩子**：取消信号打到了，`cancelled` 也置上了。但这时候 `answer` 那个 StringBuilder 里**已经攒了一大段内容**了。这段内容怎么办？

---

## Q8（追问）取消或者超时时，已经生成的内容会不会丢？

**面试官追问**：用户点了停止，那半截回答是扔掉还是保存？

**我**：**保存，而且落库必须发生在终止帧之前。** 这是我在这个问题上最坚持的一条。

### workflow 档：`finishCancelledStream`

`register` 时传进去的那个 `this::finishCancelledStream` 就是取消的收尾回调。它做两件事：**先把已累积内容落库**，再发 `cancel` + `done`。

```java
// StreamChatEventHandler.java（finishCancelledStream，有删节）
private void finishCancelledStream() {
    CompletionPayload payload = buildCompletionPayloadOnCancel();
    CompletionPayload actualPayload = payload == null ? new CompletionPayload(null, null) : payload;
    sender.sendEvent(SSEEventType.CANCEL.value(), actualPayload);
    sender.sendEvent(SSEEventType.DONE.value(), "[DONE]");
    sender.complete();
}
```

`buildCompletionPayloadOnCancel()` 里：`answer` 非空就构造 `ChatMessage.assistant(content, thinking, resolveThinkingDuration())`，带上 `sources` / `groundingChunks` / `replyToMessageId`，**`MessageStatus.INTERRUPTED`**，调 `memoryService.append` 拿 messageId。

**为什么先落库再发帧**：

1. **前端要把这条消息锚定到历史**——`cancel` 帧里带 `messageId`，前端才能把它渲染成列表里的一条，而不是一个临时气泡；
2. **刷新后要能看到半截回答**——落库了才是真的存在，否则用户刷新一次这轮就人间蒸发，体验上比"没回答"更糟；
3. **状态要能区分**——`INTERRUPTED` 跟 `NORMAL` 是两个状态，前端能标个"已中断"，也能让评测脚本把这类样本单独统计（不混进成功率）。

**落库失败怎么办？只 `log.error`，不阻断**（跟 Q9 的拒绝链同一条原则）。理由一样：**`DONE` 的优先级高于落库**。落库失败最多是这轮记录丢了，但如果因为落库异常导致 `DONE` 发不出去，前端会永远停在 loading——那是更严重的故障。

**每个增量入口都先查取消**：`onContent` / `onThinking` / `onSources` / `onGroundingChunks` 第一行都是 `if (taskManager.isCancelled(taskId)) return;`。取消之后不再往流里推任何东西，但**前面的内容在 StringBuilder 里没丢**——因为落库读的是 StringBuilder，不是"发出去过的内容"。这就是 Q6「发送是尽力而为，落库是必须成功」的另一面。

### agent 档多一层：取消前先看有没有待确认

Agent 档三个出口——`onComplete`、`onError`、`finishCancelledStream`——**都先判 `settleAwaitingConfirm()`**（`AgentStreamEventBridge.java`）。

语义是：如果这一轮正卡在「等用户确认写操作」，那么被取消 / 出错都**不该判为中断**，而应该走确认流程，把 pending 的 confirm 事件发出去，消息以 `AWAITING_CONFIRM` 落库。

**为什么**：写操作的确认态是有价值的。用户取消的是「这次流」——他不想再等这个回答往下生成；但他没取消「这个待确认动作」——那是一个**尚未执行的写操作**，应该还留着让他决定。而且工具与入参是从 `t_agent_state` 取当初那条原件（前端只传 `approved=true/false`），确认这条路本来就不依赖流还在不在。

**落库失败还有第二层降级——`settleUnpersistedConfirm()`**：拿不到 messageId 就无法结算确认态，那就发 `hint`（"系统提示"）+ `finish(INTERRUPTED)` + `done`，文案是「这条会话已无法继续，请新建会话重试」。

**要讲清楚的边界**：`AWAITING_CONFIRM` 是 Agent 消息状态里**唯一非终态**的状态，并且有 `hasPendingConfirm` 挡住新提问——不然用户在待确认状态里再问一句，会话就乱了。

**埋钩子**：上面一直在说「取消」和「出错」。还有一种"没给正常答案"的情况是**根本进不去**——被限流挡在门外。那条路我们没有 return 429。

## Q9（追问）限流拒绝为什么走 SSE，不返回 429？

**面试官追问**：都拒绝了，为什么还费劲发事件流，一个 429 不是更简单？

**我**：因为我们**已经把流建好了、META 也发出去了**——这时候再返回 429 就晚了，而且会自相矛盾。

回顾 Q1 的第一个顺序：`StreamChatEventHandler` 构造器里就发了 `META`。等限流器判定超时（默认排队 15 秒），前端**已经渲染出会话、拿到 taskId、显示着"排队中"**了。如果这时候 `ChatQueueLimiter` 决定"我不给你流"，它没法再返回一个 429——HTTP 状态码在响应头里，早发出去了。

所以拒绝只能在**流内**表达：

```java
// ChatQueueLimiter.java:145-156
private void sendRejectEvents(SseEmitter emitter, RejectedContext rejectedContext) {
    SseEmitterSender sender = new SseEmitterSender(emitter);
    if (rejectedContext != null) {
        sender.sendEvent(SSEEventType.META.value(), new MetaPayload(rejectedContext.conversationId, rejectedContext.taskId));
        sender.sendEvent(SSEEventType.REJECT.value(), new MessageDelta(RESPONSE_TYPE, REJECT_MESSAGE));
        sender.sendEvent(SSEEventType.FINISH.value(),
                new CompletionPayload(String.valueOf(rejectedContext.messageId), rejectedContext.title,
                        null, ChatMessage.MessageStatus.REJECTED));
    }
    sender.sendEvent(SSEEventType.DONE.value(), "[DONE]");
    sender.complete();
}
```

四帧：**先补一条 META**（因为这是新建的 sender，前端需要 conversationId/taskId 才能把这条挂进列表）→ `REJECT`（正文里显示"系统繁忙，请稍后再试"）→ `FINISH(REJECTED)` → `DONE`。注意 `REJECT_MESSAGE` 和 `RESPONSE_TYPE = "response"` 都是常量——**拒绝文案走的是正常的 `response` 通道**，前端用同一套渲染逻辑就能显示它，不用为"系统消息"再写一种样式。

选这条路的三个理由：

1. **协议统一**：前端只认一条流。异常态、拒绝态、正常态全在同一个状态机里，不用为"HTTP 错误"再写一套错误处理分支。
2. **可回溯**：拒绝对话也落库——用户的问题 + 一条 `REJECTED` 状态的 assistant 消息（新会话还要生成标题）。用户刷新后能看到"我问过、系统繁忙"，而不是消息凭空消失。
3. **前端复用**：能复用同一套渲染逻辑显示这句话。

### 代价和兜底

**一，落库不能阻塞收尾。** `recordRejectedConversation` 被 try-catch 包住，注释写着「**记录失败不能阻塞 emitter，否则前端永远收不到 DONE**」。跟 Q8 一条原则：**`DONE` 的优先级高于落库**。这是全篇反复出现的一条次序：**收尾帧 > 持久化 > 其他**。

**二，拒绝时的 conversationId 可能是新造的。** 入参没带就 `IdUtil.getSnowflakeNextIdStr()`，而且这里**跳过 existence 查询**——刚生成的雪花 ID 不可能命中已有会话，省一次 DB 查询。

**三，taskId 也是新造的**（拒绝没有真实任务），它只用于让前端把这条消息挂进列表。

**四，agent 档根本没有 `reject` 事件。** `resources/regression/agent-memory/AgentChatClient.java:24-25` 的注释写得很清楚：

> 与 RAG 的 `/rag/v3/chat` 协议不同：没有 reject 事件，限流在建流之前就以异常拒掉，多一个 tool 事件

也就是说 Agent 档的限流**在更早的位置**拦掉了（异常贯穿到全局异常处理器 → 返回 JSON 错误），所以它不需要 reject 帧；而 workflow 档因为 META 提前发，必须在流内收尾。**同一套限流组件，两档协议的拒绝路径不一样**——这也是 Q4 那三条不一致里的一条。

**埋钩子**：到这里 workflow 档的六个事件讲完了。接下来是重点——**换成 agent 档之后，这六个事件为什么不够用了**。

---

## Q10（追问）从 RAG 演进到 Agent，SSE 协议做了什么升级？

**面试官追问**：RAG 那六个事件不是能跑吗？换 Agent 为什么非要加事件？

**我**：因为**协议要表达的东西变了**。workflow 档的协议本质是「**一份文本的增量流**」——六个事件全在描述"文本怎么来、怎么结束"。但 Agent 一轮里除了文本还有两样东西：**工具调用**和**中断在环确认**。这两样没法用 `message` 表达，硬塞进去会让前端分不清"这句话是模型说的还是系统说的"、"这段是答案还是工具返回"。

所以从 6 个演进到 9 个。**先把对照表摆出来**：

| 维度 | workflow 档（6） | agent 档（9） | 加/没加的理由 |
|---|:--:|:--:|---|
| 元信息 | `meta` | `meta` | 不变。载荷仍是 `(conversationId, taskId)`，**没有加 traceId / 意图 / 子问题** |
| 增量文本 | `message`{response,think} | `message`{response,think,**error**} | **只加了 type，协议层一字未改**——Q3 那个两级结构在这里被验证了 |
| 文本块边界 | ❌ | **`block`** | 服务端声明块的起止与类型 |
| 工具过程 | ❌ | **`tool`** | Agent 有工具调用，RAG 没有 |
| 系统提示 | ❌ | **`hint`** | 迭代上限等系统级提示，**不落库** |
| 等确认 | ❌ | **`confirm`** | HITL，**与 `finish` 互斥** |
| 结算 | `finish` | `finish` | 不变 |
| 终止 | `done` | `done` | 不变 |
| 取消 | `cancel` | `cancel` | 不变 |
| 限流拒绝 | `reject` | ❌ **无** | 限流在建流之前就以异常拒掉 |

枚举定义在 `AgentSSEEventType.java:26-78`，注释第一行就写了「**Agent 模式 SSE 事件协议，与 workflow 协议两套分立**」——我们是**有意保留两套**的，不是遗留。

### 三条设计原则的演进

新增的四个事件，背后其实是三条原则变了：

**原则一：从「客户端猜」到「服务端声明」。** 这是 `block` 要解决的。

workflow 档里，前端只知道"来了很多 `message` 帧"。**块的边界（哪段是推理、哪段是答案）要前端自己按 `type` 变化推断，时间要前端自己打点**。当场渲染没问题，但有两个场景会坏：

- **刷新后重放历史**：历史消息里只有"块的类型"没有"块的起止"，前端重新打的时间跟当初真实生成时间对不上；
- **服务端知道而客户端不知道的信息**：比如某个工具块的**真实执行耗时**（来自工具体，见 Q11），前端根本无法推算。

所以 agent 档让服务端**主动发封口帧** `block`，载荷是 `AgentTextBlockSeal(kind, at, startedAt, endedAt, durationMs)`，`kind` 取 `reasoning` / `answer` / `error`。前端拿到就直接渲染，不需要推断。

**原则二：从「流文本」到「流过程」。** 这是 `tool` 和 `hint`。

workflow 档的一轮是「检索 → 生成 → 结束」，中间没有可展示的中间态，前端最多显示个"检索中"。agent 档一轮里会**反复调工具**，用户需要看到「正在查知识库」「正在算」「这个工具失败了」——这既是体验（否则长时间静默像卡死），也是**可信度**（用户能看到依据从哪来）。

`tool` 事件载荷是 `AgentToolProgress(toolCallId, name, displayName, status, result, ok, at, batchId, callIndex, startedAt, endedAt, durationMs, durationSource)`——注意它带 `displayName`（给人看的名字，不是代码里的 `search_knowledge`）、`batchId` + `callIndex`（一批并行工具的序号）、`durationSource`（耗时是来自工具体的真实测量还是整批共享）。

`hint` 是系统级提示，载荷 `AgentHintPayload(code, text)`，`code` 有 `AGENT_HINT` / `MAX_ITERATIONS`。**它不落库**——提示是运行时的，不是对话内容。比如达到 ReAct 迭代上限时发一条 hint，但**不判失败**（框架后续仍会生成总结与 `AgentResult`，只提示）。

**原则三：从「单向终点」到「可中断的挂起点」。** 这是 `confirm`。

workflow 档的流只有两种命运：走完，或被取消。agent 档多了一种：**走到一半停下来等用户裁决**。这一步的流语义很特殊——`SseEmitter` 得一直开着、`finish` 不能发（没结算完）、但也不能什么都不发（前端在等）。所以有了 `confirm`，并且明确它**与 `finish` 互斥**。这是 Q12 的主题。

### 一个反过来说明的点：`error` 是加 type，不是加事件

上表里最值得讲的是 `message` 那一行——**agent 档只往 body 的 `type` 里加了个 `error`，协议层一个字符没动**。

这就是 Q3 那个两级结构的回报。如果我当初按"每个渲染通道一个 event 名"设计，加 `error` 就要动协议（新增事件名、前端 switch 加分支、文档要改、评测脚本要改）。现在改的只是一行常量：

```java
// AgentStreamEventBridge.java:79 附近
private static final String DELTA_TYPE_ERROR = "error";
```

**代价回顾**（Q3 说过）：前端要在 `message` 分支里多做一次 type 判断。但跟"协议演进零成本"比，这笔账很划算——**协议是契约，契约的变更是最贵的**。

**埋钩子**：`block` 这个事件看上去只是"发个块边界"，但它服务端那一侧其实是最绕的一段——因为块的起止**不在桥里产生**，而在工具真正执行的地方产生。

---

## Q11（追问）`block`、`tool` 怎么保证前端看到的时间和状态是真的？

**面试官追问**：你说块的时间由服务端声明，那服务端自己不会算错吗？

**我**：会，如果它自己算的话。所以我的做法是——**桥不产生时间，只投影时间**。

### 单一事实源：`AgentToolExecutionFacts`

`AgentToolExecutionFacts` 的类注释写得很直接：「**批次号、序号与工具起止的唯一来源，桥只投影**」。

意思是：工具的**真实起止**在工具真正执行的地方（工具体）记录，桥（`AgentStreamEventBridge`）**不自己取 `System.currentTimeMillis()`**，而是从工具体那边投影过来。`applyExecutionTimes` 的注释：

> 投影工具体的真起止到块上，两端齐了才给耗时。没进过工具体的调用无时刻可投影

**为什么必须这样**：如果我允许桥自己打时间，就会出现**两套口径**——工具事件的时间来自工具体，块事件的时间来自桥，两者在中间有调度延迟，前端把同一个工具在"工具卡片"和"文本块"里显示成两个不同的耗时。用户会问「这俩为什么不一样」，而我答不上来。

**代价**：桥的逻辑变复杂了——它得维护"哪些调用已经有真时刻"，并且接受"有的调用永远拿不到真时刻"（没真执行过）。所以 `durationSource` 这个字段是必要的：它告诉前端这个耗时是**真测量**还是**整批共享**，前端可以选择不显示后者。

### `stateLock`：锁内改状态，锁外发帧

`onEvent(AgentEvent)` 是个大 switch，每个分支只改内存状态，**`flushSealedTextBlocks()` 统一在 switch 之后、锁外调用**。

```java
// AgentStreamEventBridge.java（结构示意，非逐字）
List<AgentBlock> sealedBuffer = ...;   // 待广播缓冲

onEvent(event) {
    synchronized (stateLock) {
        switch (event.getType()) {
            case ...: /* 只改状态，把要发的块 append 进 sealedBuffer */
        }
    }
    flushSealedTextBlocks();   // ← 锁外发 SSE
}
```

**为什么**：SSE 发送是 **IO**。持锁做 IO 会把整个事件处理串行化；更糟的是如果 `emitter.send` 阻塞（客户端不读、TCP 窗口满），**所有回调线程都会卡在这把锁上**。

**实现上的关键**：`sealedTextBlocks` 是个"待广播缓冲"，锁内只 append，锁外 `List.copyOf(...)` + clear 再逐个发。这样锁的持有时间只有"改几个字段 + 加进 list"，跟 IO 完全解耦。

### 一个容易被忽略的规则：只广播有起止的块

```java
// sealOpenTextBlock() 的核心判断（示意）
if (block.endedAt != null) {
    sealedTextBlocks.add(block);   // 只有起止都齐了才广播
}
```

注释写着：「**只广播有起止的，一次性补发的块发空帧会让前端认错待收口块**」。

意思是：有些块是**一次性补发**的（比如出错时系统插的那句提示，没有"开始生成"这个动作）。这种块只有 `at` 没有 `startedAt`/`endedAt`。如果给它发一个 `block` 封口帧，前端会以为"有一个块结束了"，于是把当前正在流式接收的那个块**错误地收口**——后面真正的增量就到了一个已经关闭的容器里，显示会错位。

**所以规则是：没有完整起止的块，不发声明的封口帧，只发它的内容增量。**

### `tool` 的状态机与三个补建场景

工具状态：`pending → running → awaiting → done / failed / denied / interrupted`。

前端靠 `tool` 事件的 `status` 字段驱动卡片状态。但服务端这边有个麻烦：**不是每个工具调用都会收到完整的开始和结束事件**。有三种情况我必须在结束时**补建**工具块：

| 场景 | 现象 | 处理 |
|---|---|---|
| 确认续跑 | 那一轮没有 `ToolCallStart`（工具是上一轮确认后接着跑的） | 在 `onToolExecutionStart` 补建 |
| 完全没有开头事件 | 框架只发了结果 | 在 `onToolEnd` 补建 + `log.warn` |
| 整批被拒 | 框架压根不发工具事件 | 在 `onAllToolsDenied` 补建，**不给批次号与起止** |

第三种是最微妙的：整批被拒的工具**从来没有执行过**，所以给它批次号和起止时间是**撒谎**。所以补建时**刻意留空**这两个字段——前端据此显示成"未执行"，而不是"执行了 0ms"。

**另外两个过滤/截断规则**：

- **内部工具不展示**：`ReActAgent.STRUCTURED_OUTPUT_TOOL_NAME` 是框架用结构化输出的伪工具，不是用户要看的，前端过滤掉。
- **工具结果截断**：`TOOL_RESULT_MAX_CHARS = 64_000`。工具返回可能很长（一整篇文档），全文塞进 SSE 既浪费带宽又拖慢渲染，截断后前端可以"展开查看"。

**埋钩子**：`confirm` 那个挂起态，我上面只说"与 finish 互斥"。但真实现起来，它是三个出口都要判的一个特殊分支。

---

## Q12（追问）等用户确认的时候，这条流是什么状态？

**面试官追问**：写操作要人工确认——那流是关掉还是挂着？

**我**：**挂着**。这是整个协议里状态最特殊的一处。

### 为什么不能关流

Agent 走 ReAct 循环，中间可能遇到需要人工裁决的写操作（比如"帮我把这个工单关掉"）。这时：

- **不能发 `finish`**——这一轮没结束，用户裁决后还要继续跑；
- **不能什么都不发**——前端在等，没有任何事件会以为卡死；
- **不能关连接**——关了用户点"同意"之后没地方接续。

所以 `confirm` 事件的语义是：**流还在，但主动权交给用户**。载荷 `AgentConfirmPayload(messageId, title, calls, durationMs)`，其中 `calls` 是 `AgentConfirmCall(toolCallId, name, displayName, fields, arguments)` 列表，`fields` 是 `AgentConfirmField(name, label, value)`——**前端渲染的是 `label` 和 `value`（给人看的），`arguments` 是原始参数**。

**消息状态落 `AWAITING_CONFIRM`**，这是 Agent 消息状态里**唯一非终态**的状态。并且有 `hasPendingConfirm` 挡住新提问——不然用户在待确认状态里再问一句，会话状态就乱了。

### 三个出口都要判 `settleAwaitingConfirm()`

`onComplete`、`onError`、`finishCancelledStream`——**三个出口第一件事都是判它**。

```java
// AgentStreamEventBridge.java（三出口共用的前置判断，示意）
if (settleAwaitingConfirm()) {
    return;   // 已按确认流程收尾，不再走本出口的默认逻辑
}
```

**为什么三个出口都要判**：因为"这一轮正卡在等确认"这个状态，跟"这一轮怎么结束的"是**正交**的。正常跑完到了确认点、出错了、被取消了——三种情况下，那个待确认的写操作**都还在那儿**，都该走确认流程，而不是被判成 `INTERRUPTED` 丢掉。

**这是我在这一块最重要的一条设计判断**：**用户取消的是「这次流」，不是「这个待确认动作」**。他不想再等这个回答往下生成，但那个尚未执行的写操作应该留着让他决定。

**一个批次级的细节**：`confirm` 时要**整批推 awaiting，不止卡片点名的那些**。因为同一批工具都没跑，如果只把点名的推成 `awaiting`、其余留在 `pending`，收尾时那批 `pending` 会被判成 `interrupted`——显示上就像"这些工具被中断了"，而实际上它们是被一起挂起了。**状态要整批一致。**

### 落库失败的第二层降级

如果确认态**落库失败**（拿不到 messageId），就没法结算确认——因为续跑时要靠 messageId 从 `t_agent_state` 取当初那条原件。这时走 `settleUnpersistedConfirm()`：

发 `hint`（系统提示）+ `finish(INTERRUPTED)` + `done`，文案是「**这条会话已无法继续，请新建会话重试**」。

**为什么选择判失败而不是继续挂着**：挂了也没意义——用户点"同意"之后服务端取不到原件参数，续跑必然失败。**早失败比晚失败好**，而且给出明确指引（新建会话）比留个点不动的按钮好。

### 前端的对应设计：确认请求必须走 POST + body

`useAgentStream.ts` 里有一行注释解释了这个协议选择：

```ts
// 带 body 即走 POST：确认裁决是有副作用的动作 不该塞进 query 让浏览器随手重发
body?: unknown;
```

裁决是**有副作用**的（可能触发一次真实的写操作），塞进 query string 的话浏览器、代理、日志都可能"顺手重发"或记录下来。放 POST body 里语义正确，也不会被缓存/预取。

**而且裁决请求不允许重试**——`useAgentStream` 只有 `streamOnce`，没有 `streamWithRetry`：

```ts
// 只发一次 失败就交给上层：Agent 一轮里可能已经执行过写操作
// 而客户端没有办法自证「这一轮在服务端没跑起来」——连一帧都没收到也可能只是回程断了
// 重发就意味着整轮重跑 写操作再执行一遍 与 HITL 的不重复提交直接冲突
```

这段是**整个前端流式实现里最重要的一段判断**。对比 workflow 档的 `streamWithRetry`（`retryCount ?? 2`、指数退避 `retryDelayMs * Math.pow(2, attempt)`、`signal.aborted` 直接抛出）——**RAG 可以重试，Agent 不可以**。

理由讲透了：重试的前提是"我知道上一次没成功"。RAG 是无副作用的查询，重试最多浪费一次检索；Agent 可能已经执行过写操作，而**客户端没有任何办法自证"这一轮在服务端没跑起来"**——连一帧都没收到也可能只是回程断了，服务端其实跑完了。所以重发就等于整轮重跑、写操作再执行一遍，跟 HITL 的不重复提交直接冲突。

**取舍**：放弃了自动重试，换来的是"绝不重复执行写操作"。对保险业务场景，这个方向的偏置是必须的。

**埋钩子**：协议讲完了，还有个问题——**首字延迟**。这是我在这条链路上唯一一处"优化做成了负收益"的地方，而且是 A/B 实测发现的。

---

## Q13（追问）首 token 延迟你们怎么度量的？优化过吗？

**面试官追问**：TTFT 你们怎么测的？优化效果如何？

**我**：我埋了**两级** TTFT，这个分层是本篇最有价值的度量设计；而 A/B 的结果是**一次诚实的回退**——我先把口径讲清楚，再讲数字。

### 为什么是两级

从用户按下回车到第一个字出现在屏幕上，中间有一大堆事情：记忆加载 → 问题重写 → 意图识别 → 多路检索 → RRF → Rerank → EvidenceGate → 拼 prompt → 调模型 → 等模型首包。**如果只埋一个 TTFT，这个数字变差了，你根本不知道是哪一段变差了。**

所以分两级：

| 埋点 | 名字 | 起 | 止 | 回答什么问题 |
|---|---|---|---|---|
| `LLM_TTFT` | `llm-first-packet` | 发起模型调用 | 模型返回第一个包 | 模型本身快不快 |
| `USER_TTFT` | `user-first-packet` | **pipeline 入口** | **推给前端第一个字** | 用户等多久 |

实现上，`USER_TTFT` 是包在回调外面的一层装饰（`StreamChatTraceRunner.java`）：

```java
private static final String USER_TTFT_NODE_NAME = "user-first-packet";
private static final String USER_TTFT_NODE_TYPE = "USER_TTFT";

StreamCallback traceAwareCallback = new ForwardingStreamCallback(callback) {
    @Override
    protected void onFirstContent() {
        recordUserTtft(traceId, runStartTime, startMillis);
    }
    @Override
    protected void onFinish(boolean success, Throwable error) {
        finishRun(traceId, success, error, startMillis);
    }
};
```

而 `LLM_TTFT` 是模型客户端上的一个 trace 节点（`LlmFirstPacketProbe.java:34`，`@RagTraceNode(name = "llm-first-packet", type = "LLM_TTFT")`）。

**两条注解注释把口径写死了**（可逐字引用）：

> 记录用户感知首包 TTFT：从 run 开始（pipeline 入口）到推给前端第一个字 / 反映完整链路前置开销（路由 / 改写 / 意图 / 检索 / LLM 首包等）

**关键价值在于差值**：`USER_TTFT − LLM_TTFT` ≈ **前置开销**（记忆 + 重写 + 意图 + 检索 + 拼接）。这个差值就是我后来做归因的依据，也是我发现"优化做成了负收益"的入口。

### 评测侧的口径（很重要，数字必须能复现）

评测脚本（`ragenteval` 的 `runner.py`）里的 TTFT 是**独立实现**的，不依赖服务端埋点：

```python
start = time.time()                                   # 发 requests.get 之前
...
if event_name == "message" and isinstance(payload, dict):
    delta_type = payload.get("type")
    content = payload.get("delta", "") or ""
    # 体感卡点是正式回答首字到达（type=response），不算 think 链路。
    if delta_type == "response":
        if state["first_token_ms"] is None and content:
            state["first_token_ms"] = int((time.time() - start) * 1000)
        state["response"] += content
```

**口径要点，逐条**：

1. **起点是发请求之前**（含建连、认证、排队）——不是"服务端开始处理"；
2. **只算 `type == "response"`**，`think` 的首包不算。注释给了理由：「体感卡点是正式回答首字到达，不算 think 链路」——这是**产品口径**，不是技术口径；
3. **必须 `content` 非空**——排除空增量帧；
4. **只记第一次**（`first_token_ms is None`）。
5. `final_status` 映射：`finish → success`、`reject → refused`、`cancel → cancelled`、`done → break`。

**为什么评测侧要独立实现而不复用服务端埋点**：因为要测的是**端到端用户感知**，服务端埋点看不见网络和客户端解析耗时。两套数字**有差值才是正常的**，差值就是网络 + 解析开销。

### A/B 实测：一次诚实的回退

**实验设计**（20 个样本）：

- **B 组（基线）**：`v1_20260530_161953` —— 无问题重写 + 全局检索
- **A 组（优化）**：`v1_20260530_163213` —— 问题重写 + 意图树检索

**延迟结果——A 组更慢**：

| 指标 | B（无优化） | A（优化） | 变化 |
|---|---:|---:|---:|
| `ttft_p50_ms` | 7131 | 8131 | **+1000ms（+14.0%）** |
| `ttft_mean_ms` | 7218 | 8303 | **+1085ms（+15.0%）** |
| `total_mean_ms` | 8541 | 9830 | **+1289ms（+15.1%）** |

**质量结果——A 组全面领先**：

| 指标 | 变化 |
|---|---|
| `hit@1` | **+28.5%** |
| `mrr@10` | +12.5% |
| `over_retrieval` | **100% → 0%** |
| `faithfulness` | +11.1% |
| `context_precision` | +23.0% |

**归因**：问题重写**额外多了一次 LLM 调用**，意图树检索**增加了路由开销**——两者叠加，延迟增加约 1 秒。

**我怎么看这笔账**：**这是拿 1 秒延迟换 28.5% 的 hit@1，我认为该换。** 理由是这个业务的失败模式不对称——保险问答答错（引用错的条款）的代价远高于多等 1 秒。所以这个回退我**认了，而且没回滚**。

**但我必须承认这里有一个我当时没做对的地方：优化是打包做的。** 我把"问题重写"和"意图树检索"当成一个整体 A/B，结果拿到 +1000ms 却没法拆出**哪一项贡献了多少**。事后看，意图树检索是"路由"，它的开销应该很小；真正贵的是问题重写那次额外 LLM 调用。如果当时**拆成两组单变量 A/B**，我很可能会得出"保留意图树检索、给问题重写加开关（比如只在多轮/指代场景才重写）"的结论——而不是现在这个"要么全要、要么全不要"的状态。

**这套子今天已经部分落地了**：`rag.query-rewrite.enabled` 和 `rag.citation.enabled` 都是独立开关，而且 `citation` 的注释直接写了「开启会动态追加引用规则并为资料注入编号，**增加首字延迟**」——这就是那次 A/B 之后学到的：**每个会增加首字延迟的特性，都要能单独关掉、并在配置里写明它的代价。**

**A/B 脚本的口径**（`eval/rag/report/diff.py`）：`compare()` 里 `LOWER_IS_BETTER` 包含所有 `ttft_*`；阈值是 **TTFT 类 500ms、`total_p95_ms` 1000ms**——超过阈值才报告为显著回退。也就是说 +1000ms 是**两倍于阈值**的回退，不是噪声。

**数据集口径要说清**：`eval_set_v1.jsonl` 是 **20 行**（A/B 用的就是它），`eval_set_v1_all.jsonl` 是 **150 行**。⚠️ **仓库文档里写的"150 在 v1.jsonl"与现状不一致**——这是我需要回去修正文档的地方。

**另一处口径要说清**：`--ragas-n` 默认 **1**（跑一次取结果），但文档建议 3 次取均值。**我报的数字是按默认值 1 跑的**，所以有单次波动风险——如果要报给别人，应该按 3 次均值重跑。

**最新的小样本录制**（5 样本，`eval/reports/v1_20260807_181028/_scores.json`）：`ttft_p50_ms = 5342.0`、`ttft_mean_ms = 4824.2`、`total_mean_ms = 10284.0`。⚠️ 这组跟 A/B 那组**不可直接比较**——样本量不同（5 vs 20）、数据集不同、而且 `p50(5342) > mean(4824)` 说明样本少且分布偏，只能当冒烟数据看，不能当趋势。

**埋钩子**：上面所有数字都来自评测脚本。那评测脚本是怎么"看"这个流的？它是另一个手写的 SSE 解析器。

---

## Q14（追问）客户端是怎么解析这个流的？自己写解析器有什么坑？

**面试官追问**：你说前端没用 `EventSource`，那解析是自己写的？

**我**：**两边都是自己写的**——前端（TypeScript）和评测脚本（Python）各一套。这不是我想要的，是 `EventSource` 和 `requests.iter_lines()` 都不够用。**而且这两套解析器有一个共同的、必须自己填的坑：事件跨 HTTP chunk 边界。**

### 坑在哪

SSE 的事件分隔符是**空行**（`\n\n` 或 `\r\n\r\n`）。但 HTTP chunked 传输**不保证**一个 chunk 里包含完整的事件——服务器发 `event: finish\ndata: {...}\n\n`，完全可能在 `\n\n` 之前被切成两个 TCP 包。

如果逐 chunk 解析、或者用"按行读"的现成方法，就会出现：

- **中间事件丢失**：`iter_lines()` 在分隔符跨 chunk 时会把那个空行吞掉（Python 评测脚本的注释原文就是这么写的）；
- **事件被拆坏**：`{"type":"response"` 在第一个 chunk，`"delta":"你好"}` 在第二个，逐 chunk `JSON.parse` 直接抛异常。

**解法两边一致：维护一个 buffer，按行切开，最后一段（可能不完整）留回 buffer。**

### 前端（TypeScript）

```ts
// useStreamResponse.ts / useAgentStream.ts（同构实现）
buffer += decoder.decode(value, { stream: true });
const lines = buffer.split(/\r?\n/);
buffer = lines.pop() ?? "";        // ← 关键：最后一段可能是半行，留回 buffer
for (const line of lines) {
  if (!line) { dispatchEvent(); continue; }          // 空行 = 事件结束
  if (line.startsWith(":")) { continue; }            // 注释/心跳行，忽略
  if (line.startsWith("event:")) { eventName = line.slice(6).trim(); continue; }
  if (line.startsWith("data:")) { dataLines.push(line.slice(5).trim()); }
}
```

四个要点：

1. **`buffer = lines.pop() ?? ""`**——这一行就是"跨 chunk 边界"的解药。`pop()` 拿走的最后一段**不保证是完整行**，必须留到下一轮。
2. **`decoder.decode(value, { stream: true })`**——多字节 UTF-8 字符也可能跨 chunk（跟 Q5 的代理对是**同一类问题在不同层面**：那里是 Java 侧切分，这里是网络侧切分）。`stream: true` 让 `TextDecoder` 把不完整的字节序列留在内部。
3. **`data:` 支持多行累积**（`dataLines.push` + `join("\n")`）——SSE 规范允许多个 `data:` 行构成一个事件。我们实际不发多行，但解析器按规范实现了。
4. **`:` 开头是注释行**（心跳 `:keep-alive` 常用这个），**必须忽略**，否则会被当成数据。

**一个我不知道但有意识保留的分支**：前端 `switch` 里有一个 `case "title"`——`useStreamResponse.ts` 为它留了 `onTitle` 处理器。**但后端从来不发 `title` 事件**（标题是随 `finish` 的 `CompletionPayload.title` 一起给的）。这是一处**死分支**，当时留的扩展位。**Agent 档里同样有死分支**：`onError` 处理器——Agent 档虽然会发 `error` **增量**，但那是 `message` 事件里的 `type`，不是独立的 `error` **事件**，所以 `case "error"` 也走不到。

**顺带一处我踩过的坑**：前端 `message` 分支是**先调 `onThinking`、再无条件调 `onMessage`**（Q3 那段代码）——也就是说 `think` 的增量会**被分发两次**。这不是 bug，是**有意的**：让 `onMessage` 能收到**完整**的增量序列（前端 store 可以据此重建顺序）。代价是各 store 必须自己过滤——`chatStore.ts` 过滤 `type !== "response"`、`agentChatStore.ts` 只接受 `type === "response" || type === "error"`。**分流责任下沉到了 store 层。**

### 评测侧（Python）

`eval/rag/pipeline/runner.py` 的 `parse_sse_stream` 是**独立实现**，而且注释把动机写得很清楚：

```python
def parse_sse_stream(byte_iter: Iterator[bytes]) -> Iterator[tuple[str, str]]:
    """从字节流里逐事件 yield (event_name, data_string)。

    自己切事件而不用 requests.iter_lines()：后者在事件分隔符 (`\n\n`)
    跨 HTTP chunk 边界时会把空行吞掉，导致中间事件丢失。
    """
    buffer = ""
    for chunk in byte_iter:
        if not chunk:
            continue
        buffer += chunk.decode("utf-8", errors="replace")
        while True:
            idx_crlf = buffer.find("\r\n\r\n")
            idx_lf = buffer.find("\n\n")
            if idx_crlf == -1 and idx_lf == -1:
                break
            if idx_crlf == -1:
                idx, sep_len = idx_lf, 2
            elif idx_lf == -1:
                idx, sep_len = idx_crlf, 4
            else:
                if idx_crlf < idx_lf:
                    idx, sep_len = idx_crlf, 4
                else:
                    idx, sep_len = idx_lf, 2
            event_block = buffer[:idx]
            buffer = buffer[idx + sep_len:]
            yield from _parse_event_block(event_block)

    if buffer.strip():
        yield from _parse_event_block(buffer)
```

**它比前端多做三件事，值得讲**：

1. **同时处理 `\r\n\r\n` 和 `\n\n`，取更靠前的那个**。前端用 `split(/\r?\n/)` 规避了这个问题（按行切，空行就是分隔符），Python 在字节层面处理，所以必须显式判两种。**取更靠前的**这个细节很重要——如果两种都出现，先出现的才是真分隔符。
2. **`while True` 循环**——一个 chunk 里可能包含**多个**完整事件，必须循环切到底，不能只切一个。
3. **收尾 `if buffer.strip()`**——最后一段没有分隔符收尾的残留也要解析（防止服务端最后一个事件没跟空行）。

**它的已知缺口（我必须主动说）**：

- **`id:` 和 `retry:` 不解析**。我们不用这两个字段（没有 `Last-Event-ID` 断点续传需求，重连是整轮重发），所以没实现。**但如果将来要支持断点续传，这里是必须补的。**
- **没有任何单元测试**。⚠️ 这是**评测侧最大的工程硬伤**——这个函数是"所有评测数字的入口"，它解析错一个事件，上面所有指标就全错，而且错得悄无声息（比如漏掉 `finish` 帧就会把成功样本判成失败）。按 80% 覆盖率的红线，它至少该有这几个用例：跨 chunk 边界、一个 chunk 多事件、`\r\n\r\n` 与 `\n\n` 混用、`:` 注释行、多行 `data:`、无尾分隔符。

### 评测侧的第二个硬伤：两次请求，两套检索结果

评测要同时拿**回答**和**检索证据**。但**生产 SSE 流里不带检索证据**——`META` 只有 `conversationId` + `taskId`，引用是折叠在 `finish` 帧的 `sources` 里的，而 `sources` 是**文档级引用**，不是 chunk 级，也没有 rerank 分数。

所以我们做了一个**评测旁路**：`GET /rag/eval`（需要 `app.eval.enabled=true`），**只跑检索、不调 LLM**，返回 `retrievedDocIds` / `retrievedChunkIds` / `retrievedContexts` / `retrievedContextDocIds` / `intentLeafIds` / `hasKb` / `hasMcp` / `traceId`。

**问题是：一次评测要打两次请求**——一次 SSE 拿回答，一次 `/rag/eval` 拿证据。

> 「两次请求是两次独立检索……两次的召回结果**不保证完全一致**……这是当前最大的设计妥协。」

**为什么不能合并**：让 SSE 流里带上 chunk 级证据，就等于把评测需求泄漏进生产协议——生产前端不需要 chunk 级证据和 rerank 分数，加进去就是**为一个非生产消费者改契约**。这是我拒绝的（Q3/Q10 反复强调：**协议是契约，为单方需求改契约是最贵的**）。

**代价我很清楚**：`/rag/eval` 那次检索的结果**原则上不能代表** SSE 那次实际用的证据。评测出来的 `context_precision` / `context_recall` 严格说是**对"同一次查询的另一次检索"的度量**，不是对"这次回答所用的证据"的度量。

**为什么暂时能接受**：检索链路在同样的输入下**高度确定**（同样的 query、同样的库、同样的 topK，向量检索是确定性的，只有 rerank 在有并列分时可能抖动），所以两次结果**绝大多数情况下一致**。但"绝大多数"不是"一定"，这是**统计噪音**。

**如果重做，我会选**：在 `finish` 帧的 `sources` 里**额外带上 chunk 级 id 列表**（不需要 contexts 全文，也不需要分数）——这是对生产协议**最小**的侵入（`sources` 本来就是引用信息），却能消掉评测侧的最大不确定性。这是一个"当时没想到、现在看很明显"的取舍。

**埋钩子**：还有一个我们代码库里**根本没有**的东西，但它一定会被问——网关和反向代理这一层。

---

## Q15（追问）网关和反向代理这一层有什么坑？

**面试官追问**：这个流要过 Nginx 吗？过的时候会不会被缓冲掉？

**我**：⚠️ **先说清楚：我们代码库里没有 Nginx / 网关配置**（没有 `*.conf`、没有 `Dockerfile`、没有 `docker-compose`），所以**这一段是我的通用知识 + 对自家架构的推断，不是我在这个项目里踩过的坑**。面试时我会明确这么说，而不是编一个事故。

这一层的核心矛盾是一句话：**SSE 的前提是"响应边产生边送达"，而网关的默认优化全是"攒够了再发"。**

### 坑一：响应缓冲（最常见、最致命）

Nginx 默认会对上游响应做缓冲。对普通接口这是好事（让后端尽快释放连接），但对 SSE 是**致命的**——它会等缓冲区满或上游结束才吐给客户端。

**现象**：后端日志显示逐帧发送正常，但前端**一次性收到全部内容**，或者卡很久然后突然全出来。看起来"流式没生效"。

**对策**：

```nginx
location /api/ragent/ {
    proxy_buffering off;              # 关代理缓冲
    proxy_cache off;                  # 关缓存
    proxy_set_header X-Accel-Buffering no;   # 显式告诉 Nginx 不要缓冲
    proxy_http_version 1.1;           # SSE 需要 HTTP/1.1
    proxy_set_header Connection "";   # 清掉 Connection 头，避免 keep-alive 被降级
    gzip off;                         # 见坑二
    chunked_transfer_encoding on;
}
```

`X-Accel-Buffering: no` 这个头**最好由后端应用自己发**——这样不依赖运维记得配（我们目前**没发**，这是一个应该补的点）。所以这是一个"要么改配置、要么改代码、两个都没做就会出问题"的典型。

### 坑二：压缩（gzip）破坏流式

`gzip` 需要攒够一个压缩块才输出，而且**缓冲**。开了 gzip，流式延迟会被拉回到"块大小"级别。

**更麻烦的是**：有些网关会自动对 `text/event-stream` 也启用 gzip（按大小或 MIME 规则匹配），这时候后端完全不知道。所以要**显式排除** `text/event-stream` 不压缩。

### 坑三：超时（三层，必须一层层对齐）

SSE 长连接会撞上**每一层**的空闲超时。任何一层短了，流就被掐断——而且表现为**前端"没收到 done"**（Q4 那段 throw 就是为这个准备的）。

| 层 | 参数 | 我们的值 | 关系 |
|---|---|---|---|
| 客户端 | `AbortSignal` / 浏览器 | 用户主动 | — |
| 网关 | `proxy_read_timeout` | ⚠️ 未知（无配置） | 必须 **>** 应用层 |
| 应用 | `sse-timeout-ms` | **300000**（workflow）/ **900000**（agent） | — |
| 模型 | `ai.chat.*.timeout-ms` | **120000**（standard）/ **180000**（deep） | 必须 **<** 应用层 |

**对齐原则**：模型超时 < 应用 SSE 超时 < 网关 `proxy_read_timeout`。任何一层反了，就会出现"上游还在跑但流已经被掐"或者"流还开着但模型早就断了"。

**为什么应用层要设 5 分钟 / 15 分钟这么大的值**：因为它是**兜底**，不是主控制手段。正常路径靠 `finish` 收尾，异常路径靠取消协议收尾；超时是"前面都失效了"时的最后一道。设太小会误杀正常的长回答。

**为什么 agent 档要 15 分钟**：Agent 一轮可能有多次工具调用 + **中间停下来等人确认**。而确认挂起时那个 `SseEmitter` 是**一直开着**的（Q12）——用户去开个会回来再点同意，如果超时设 5 分钟，回来时流已经死了。

### 坑四：负载均衡与连接粘性

SSE 是长连接，但**建立连接后中途不需要粘性**——因为一条流的全部事件都走同一条已建立的 TCP 连接。**但取消/停止请求需要跨节点**（Q7 那个 `StreamTaskManager` 就是为此而生）。

所以网关侧的隐含要求是：**`POST /rag/v3/stop` 不能假设落在原节点上**。这一点我们的设计已经处理了（属主进 Redis + 广播），但**如果将来加"会话粘性"来优化缓存命中，反而会掩盖这个问题**——所以粘性是优化，不是正确性的依赖，这个边界要守住。

### 坑五：HTTP/2 下的一个反直觉现象

HTTP/2 多路复用本来是 SSE 的救星（Q2 讲的连接数限制），但有些网关在 HTTP/2 到 HTTP/1.1 的**转换**过程中会重新引入缓冲。所以"上了 HTTP/2 就好了"不一定成立——**还是得关缓冲**。

### 面试时我怎么说这一段

我会这样起头：**"这一层我先声明——我们代码库里没有网关配置，所以这是我的通用认知，不是我们踩过的坑。如果让我上线，我会按这四件事检查：缓冲、压缩、三层超时对齐、以及停止请求的跨节点假设。"**

然后如果面试官追问"那你怎么验证没被缓冲"，答案是：**看帧到达的时间戳分布**。正常流式是"首帧早、后续帧均匀分布"；被缓冲是"首帧晚、然后一批同时到"。这个判据比看日志可靠——**因为后端日志永远是正常的**。

**埋钩子**（收尾）：整条链路的取舍，我最后归到一句话——**协议是契约，契约的变更是最贵的；所以我把协议做成了两级、做成了两套分立，而不是让它跟着渲染细节和单方需求变形。**

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里埋的钩子 | 面试官大概率会追 | 准备好了没 |
|---|---|---|
| Q1「我推给前端的粒度不是模型吐的粒度」 | 那粒度是怎么定的？ | ✅ Q5：`messageChunkSize`（代码默认 5 / 生产配 1）+ 按码点切 |
| Q1「META 在入队前就发了」 | 那被拒绝了前端不是收到一半？ | ✅ Q9：拒绝链补 `META→REJECT→FINISH→DONE` 四帧 |
| Q2「前端没用 `EventSource`」 | 那你怎么带 token、怎么取消？ | ✅ Q2 后半 + Q14：手写 `fetch` + `AbortSignal` + 自写解析 |
| Q2「SSE 的边界是中途插话」 | 那你们将来要做双向怎么办？ | ⚠️ 要答：换 WebSocket / 上 HTTP/2，并说清代价 |
| Q3「加渲染通道不用改协议」 | 真的没用改过？举个例子 | ✅ Q10：Agent 加 `error` 只改了一行 type 常量 |
| Q4「两套协议终态不一致」 | 那你为什么不统一？现在改了吗？ | ✅ 主动暴露：**未改**，说清三种拒绝走三条路 + 改造方案 |
| Q4「`done` 是硬约束」 | 没收到 `done` 前端会怎样？ | ✅ `useAgentStream.ts:145-147` 直接 throw |
| Q5「按码点切」 | 不按码点会怎样？ | ✅ 代理对被劈开 → 孤立代理 → JSON 变 `\uD83D` → 前端乱码且不可恢复 |
| Q5「chunkSize 生产配 1」 | 那聚合逻辑不是白写了？ | ✅ 诚实答：是，与首字延迟目标冲突，见 Q13 |
| Q6「发送失败静默丢弃」 | 那内容不就丢了？ | ✅ 内容在 `StringBuilder`，落库读它不读发出去的东西 |
| Q6「`fail` 故意不抛」 | 那前端怎么知道出错了？ | ✅ 这就是 workflow 档的洞，Agent 档在流内补了 error |
| Q7「taskId 不能当凭证」 | 那你怎么鉴权？ | ✅ 属主进 Redis + 停授口不区分"不存在/非属主"防探测 |
| Q7「属主必须先于回调落地」 | 反过来会怎样？ | ✅ 出现"无主流" → 越权广播能杀掉它 / 或误杀合法取消 |
| Q7「不匹配时连 `cancelled` 都不置」 | 少置一个标志有什么关系？ | ✅ 会让 `register` 的 Redis 复核短路 → **绕开鉴权那道门** |
| Q8「落库先于发帧」 | 落库失败呢？ | ✅ 只 `log.error` 不阻断：**`DONE` 优先级高于落库** |
| Q8「`AWAITING_CONFIRM` 是唯一非终态」 | 那它怎么收尾？ | ✅ Q12：三出口都判 `settleAwaitingConfirm` + `hasPendingConfirm` 挡新提问 |
| Q9「agent 档没有 reject」 | 那 agent 被限流怎么办？ | ✅ 建流前异常拒掉（返回 JSON），与 workflow 路径不同 |
| **Q10「有意保留两套协议」** | **为什么不合并成一套？** | ✅ **必答**：`confirm`/`tool` 对 RAG 无意义，合并会让 RAG 契约背不存在的状态；两套分立是**契约成本**换**各自最小** |
| Q10「只加 type 不加事件」 | 那 `block`/`tool` 为什么不能也塞进 type？ | ✅ 它们是**协议状态变化**（有起止、有状态机），不是渲染分流 |
| Q11「桥只投影时间」 | 桥自己打时间会怎样？ | ✅ 两套口径 → 同一工具在卡片和文本块里显示不同耗时 |
| Q11「锁内改状态锁外发帧」 | 持锁发帧会怎样？ | ✅ SSE 是 IO，`emitter.send` 阻塞会卡死所有回调线程 |
| Q11「整批推 awaiting」 | 只推点名的那几个不行吗？ | ✅ 其余留 `pending` 收尾会被判成 `interrupted`，显示像"被中断" |
| Q11「整批被拒不给我起止」 | 为什么不给？ | ✅ 从没执行过，给时间就是**撒谎**，前端应显示"未执行" |
| Q12「裁决走 POST + body」 | 为什么不能进 query？ | ✅ 有副作用，防浏览器/代理**随手重发** |
| Q12「Agent 不重试」 | RAG 为什么能重试？ | ✅ RAG 无副作用；Agent 可能已写过，且客户端**无法自证没跑起来** |
| Q13「+1000ms 是回退」 | 那你为什么不回滚？ | ✅ 拿 1 秒换 28.5% hit@1，失败模式不对称（答错 > 多等） |
| Q13「优化是打包做的」 | 那你现在会怎么做？ | ✅ 拆单变量 A/B；今天已落地 `query-rewrite.enabled` / `citation.enabled` 独立开关 |
| Q13「`--ragas-n` 默认 1」 | 那你的数字可信吗？ | ✅ 诚实：有单次波动，报给别人应按 3 次均值重跑 |
| Q14「自己写解析器的坑」 | 最大的坑是什么？ | ✅ 事件跨 HTTP chunk 边界；`buffer = lines.pop()` / `while True` |
| Q14「`id:` / `retry:` 解析了吗」 | 那断点续传呢？ | ✅ 没解析、没实现；要支持必须补 |
| Q14「解析器有测试吗」 | ⚠️ **主动暴露** | ✅ **没有单测**，是评测侧最大工程硬伤 + 该补的 6 个用例 |
| Q14「两次请求两套检索」 | 那指标还准吗？ | ✅ 主动暴露：是最大设计妥协；重做会把 chunk 级 id 加进 `finish.sources` |
| Q15「网关会不会缓冲」 | 你怎么知道没被缓冲？ | ✅ 先声明无配置；判据是**帧到达时间戳分布**（后端日志永远正常） |
| Q15「三层超时怎么配」 | 顺序反了会怎样？ | ✅ 模型 < 应用 SSE < 网关 `proxy_read_timeout` |

---

## 📋 面试建议（对真实面试的 3 条建议）

### 一、把自己定位成「协议设计者」而不是「用了 SSE 的人」

这一篇的区分度**不在**"我知道 SSE 是 `text/event-stream`"，而在**每一个取舍都是我做的、我知道代价在哪**。所以答题时**不要把 `finish`/`done` 讲成"框架规定的"**，而要讲成：

> "`done` 这个哨兵帧是我加的，因为 **SSE 协议本身没有'正常结束'信号**——连接关闭和网络掐断在客户端看来是一样的。没有它，前端无法区分这两种情况。"

同样，`block` 不是我"照着框架抄的"，而是我发现的："客户端自己推断块边界，在**刷新后重放**时会给出错误的时间"。**每讲一个设计，先讲它可以被替换掉的那个方案，以及它为什么不行。**

### 二、主动交三处不一致——这是这篇最值钱的动作

**面试官不会主动问你"你的协议里有哪些设计不一致"**，因为这需要他先读懂你的代码。你主动说，效果是：

> "这里我要主动说一个我们做得不好的地方：现在**三种'没给正常答案'的情况走了三条不同的协议路径**——限流拒绝走流内 `reject` 帧、幂等拦截走流外 JSON、出错直接关连接。**本该统一成流内终态**，Agent 档已经这么改了，workflow 档还没跟上。这是我最想改的地方。"

三处按优先级排（**主动说，不要等问**）：

1. **终态三条路**（Q4）——限流流内 / 幂等流外 / 出错关连接。**最该讲**，因为你连改造方案都想好了。
2. **评测侧无单测 + 双检索**（Q14）——"所有评测数字的入口没有一行测试"这句话本身就是工程师信号；配上下面的改造方向。
3. **协议两套分立**（Q10）——这条要**反向讲**：先说"我们**有意保留两套**"，再说清为什么合并更贵（RAG 契约不该背 `confirm`/`tool` 这些它永远不会有的状态）。**这个不是缺点，是你想清楚了的证据**——但如果你不主动说，面试官可能误以为是历史遗留。

**注意**：主动暴露的前提是**你同时给出了改造方案和当时的判断依据**。只说"我们这里做得不好"是减分；说"我当时因为 X 选了 A，代价是 B，正确做法是 C，因为 D"是加分。

### 三、数字必须带口径，尤其是那个「回退」的数字

这篇里唯一一组会被追问的硬数字是 A/B。**只要你报数字，第一句就报口径**：

> "20 个样本，A 组是问题重写 + 意图树检索，B 组两组都关；TTFT 从起点到 `type=response` 的第一个非空增量，**排除 think 链路**；用 `diff.py` 对比，TTFT 类显著性阈值 500ms。"

然后**主动加三句限定**（这三句是防守，也是信任来源）：

1. **"+1000ms 是两倍于显著性阈值的回退，不是噪声"**——防止对方以为你在报波动；
2. **"`--ragas-n` 默认跑 1 次，报给别人应该按 3 次均值重跑"**——主动交代方法学弱点；
3. **"这个数字是 20 样本小集，不是 150 全量"**——防止对方拿它当整体结论。

**最重要的一句**是归因和判断：

> "多出来的 1 秒，我判断是**问题重写那次额外 LLM 调用**，而不是意图树检索——因为路由的开销应该很小。但**这个归因是我事后推的，不是实测的**——当时 A/B 是打包做的，没有拆单变量。**这是我当时的方法失误**，我应该拆成两组单变量，那样很可能能得出'保留路由、给重写加场景开关'的结论。"

**为什么这句最值钱**：它同时证明了三件事——你知道怎么归因、你知道自己的实验设计有缺陷、你知道正确的做法是什么。**比"我优化了 TTFT"强得多**，因为后者站不住（你确实没优化成功）。

**最后一句收尾**（把回退变成判断）：

> "这笔账我认了，也没回滚——**保险问答答错的代价远高于多等 1 秒**。但我从这个项目学到的可复用的东西是：**每个会增加首字延迟的特性，都要能独立关掉，并在配置里写明它的代价**。今天 `rag.citation.enabled` 的注释就直接写了'增加首字延迟'——这就是那次教训的产物。"

---

> **相邻稿件**
> - 《[分布式限流怎么做.md](分布式限流怎么做.md)》——限流器怎么排队、怎么拒、以及为什么拒绝要走 `reject` 帧
> - 《[Agent化升级（新老机制对比）.md](../项目深挖v2-升级清单/Agent化升级（新老机制对比）.md)》——协议之外（编排 / 工具 / 记忆 / 确认机制）的新老对比
> - 《[RAGAS评测怎么做.md](RAGAS评测怎么做.md)》——评测指标口径；本篇 Q13/Q14 讲的是**流怎么被消费**，那篇讲**指标怎么算**


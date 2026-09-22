# 模拟面试逐字稿：SSE 流式怎么做（Agent 档 · RAG → Agentic）

> 简历产出：把 RAG 问答平台的流式协议从「6 事件文本流」演进到「9 事件过程流」，在 Agent 模式下把 AgentScope 的事件流桥接到 Servlet SSE，并配套块状态机 / 单一事实源计时 / 生命周期句柄 / 中断存盘式取消。
> 覆盖模块：`AgentChatController` / `AgentChatServiceImpl` / `AgentStreamEventBridge` / `AgentRunHandle` / `AgentRunGate` / `AgentToolExecutionFacts` / `AgentSSEEventType` / 前端 `useAgentStream.ts` + `agentTimeline.ts`
> 熟练度：🔴

> **口径声明**：本篇按「主负责」档讲——Agent 档的流式桥接与块/工具协议是我设计的。代码锚点均为 `ragent` 真实实现，配置数字以 `application.yaml` 为准。
> **本篇自包含**：Q1~Q4 会把 9 事件协议从头推导一遍（不依赖旧稿），Q5 起逐块深挖实现机制。旧稿《[SSE流式怎么做.md](../项目深挖/SSE流式怎么做.md)》讲的是 workflow 档（`/rag/v3/chat`，6 事件底座）与断连治理，本篇是它的 Agent 档演进姊妹篇；想看两套协议终态不一致的来龙去脉看旧稿 Q4。
> **两套协议分立**：`/rag/v3/chat`（workflow 档，6 事件）与 `/agent/v1/chat`（agent 档，9 事件）是**有意保留的两套契约**，不是历史遗留——为什么分立，见 Q4。

---

## 🎤 开场总览（45 秒版，可直接背）

> 先一句话说清**升级本质**：workflow 档的流是「一份文本的增量流」，Agent 档的流是「一次推理过程的投影」——协议要表达的东西变了，所以要加事件；而因为 Agent 有**状态要持久化**，取消也从「停发」升级成了「先让框架存盘、再断流」。

**我：**

RAG 那一版，一条流从进来到结束就是「检索 → 生成 → 结束」，六个事件全在描述文本怎么来、怎么结束，模型只在最后生成答案。Agent 化之后，一轮里多了**工具调用**和**中断在环确认**这两样东西，协议从 6 个事件演进到 9 个，新增 `block` / `tool` / `hint` / `confirm`，并且把「块的起止」从客户端猜测改成了服务端声明。

但协议只是表象，真正的实现升级有三件：

**第一，桥接。** AgentScope 内部是 Project Reactor 的 `Flux<AgentEvent>`，我们对外还是 Servlet 的 `SseEmitter`。做法是**命令式订阅**这个 Flux——`events.subscribe(bridge::onEvent, bridge::onError, bridge::onComplete)`，每个框架事件回调里写 SSE 帧。这不是 WebFlux，是一座「响应式事件流 → 命令式 SSE」的桥。**埋一个点**：为什么要桥而不是直接上 WebFlux。

**第二，过程状态机。** Agent 一轮里文本和工具是交错的，所以要维护一个块状态机——文本块开/封口、工具块 `pending→running→awaiting→done/denied/interrupted`，而且块的起止**桥自己不测**，是从工具体那边投影过来的。**埋一个点**：为什么时间只有一个事实源。

**第三，取消语义变了。** 老 RAG 取消就是「把 cancelled 置上、不再发帧」。Agent 不一样——AgentScope 有跨轮状态要存盘，直接断流会丢本轮工具执行结果。所以取消要先 `agent.interrupt()` 等框架存盘（最多 2 秒），超时才 `dispose()` 断流。**埋一个点**：这个 2 秒是拿什么换的。

一句话总结：**协议从文本流变过程流、桥接从回调变事件流、取消从停发变中断存盘**。

---

## Q1（开场总览）Agent 档一条流是怎么跑起来的？从请求进来到第一帧推给前端，中间发生了什么？

**面试官追问**：和 workflow 档比，多出来的那几步是什么？

**我**：分六段。先看时序，再讲顺序。

```
浏览器              网关               ragent 实例                                 下游
  │                 │                     │                                        │
  │ GET /agent/v1/chat?question=..&conversationId=..                              │
  ├────────────────►│                     │                                        │
  │                 ├───────────────────►│ ① new SseEmitter(900_000ms)            │
  │                 │                     │    streamChat()                        │
  │                 │                     │ ② runGate.acquire(userId, taskId)     │
  │                 │                     │    └─ Redis setIfAbsent 抢运行位       │
  │                 │                     │      抢不到 → 抛异常(未发帧)→JSON       │
  │                 │                     │ ③ touchConversation + ensureBaseline  │
  │                 │                     │    + addUserMessage(落库用户消息)      │
  │                 │                     │ ④ launchStream:                       │
  │                 │                     │    · new AgentToolExecutionFacts      │
  │                 │                     │    · new AgentRunHandle               │
  │                 │                     │    · sender.sendEvent(META) ─────────►│ 前端立刻拿到
  │                 │                     │    · taskManager.register             │ conversationId/taskId
  │                 │                     │    · agent.streamEvents(input, ctx)   │
  │                 │                     │        .doFinally(markUpstreamTerminated)
  │                 │                     │ ⑤ runHandle.start( 订阅这段 Flux )     │
  │                 │                     │    events.subscribe(                  │
  │                 │                     │      bridge::onEvent,   // 每个事件→SSE│
  │                 │                     │      bridge::onError,                 │
  │                 │                     │      bridge::onComplete)              │
  │  event: meta     │◄───────────────────┤                                        │
  │  event: message  │◄───────────────────┤ TEXT_BLOCK_DELTA → message{response}   │
  │  event: tool     │◄───────────────────┤ TOOL_CALL_START → tool{pending}        │
  │  event: block    │◄───────────────────┤ 封口块 → block 帧                      │
  │  event: confirm  │◄───────────────────┤ REQUIRE_USER_CONFIRM → confirm 帧      │
  │  event: finish   │◄───────────────────┤ 先落库拿 messageId，再发               │
  │  event: done     │◄───────────────────┤ 字面量 "[DONE]"                       │
```

**和 workflow 档比，真正多出来的是三处，都在入口这段：**

**一，闸门 `AgentRunGate` 取代了 `@IdempotentSubmit`。** workflow 档的幂等拦截是注解，覆盖的是「控制器返回 emitter 之前」的同步窗口；Agent 档换成 `AgentRunGate`——按用户抢一个 Redis 运行位，覆盖**整个流生命周期**（一条流跑多久，闸门占多久）。为什么必须这样：Agent 一轮可能几十秒、还停下来等人确认，如果只用同步窗口的幂等拦截，用户在等待确认期间再发一问就进去了，会话状态会乱。`runGate.acquire()` 抢不到直接抛异常，**这时 META 还没发**，所以全局异常处理器能干净地返回一个 JSON 错误，而不是像 workflow 档那样「流已经建了只能走流内 reject」。

**二，落库和建基线挪到了订阅之前。** `touchConversation`（建会话/取标题）→ `ensureExtractionBaseline`（建记忆抽取下界）→ `addUserMessage`（用户消息落库），三件都是**有副作用的落库**，必须赶在 `subscribe` 之前。尤其 `ensureExtractionBaseline`——注释写死「必须在 addUserMessage 之前建立基线，否则本轮消息会被划进历史、漏抽」。这是记忆抽取线的事，不展开，但说明 Agent 档的入口副作用比 workflow 档多。

**三，最核心的：订阅发生在 `runHandle.start()` 的锁里。** 看 `AgentChatServiceImpl.java:250-255`：

```java
runHandle.start(() -> {
    settleConfirmCard(scope);
    Disposable disposable = events.subscribe(bridge::onEvent, bridge::onError, bridge::onComplete);
    runHandle.bindStream(disposable, () -> agent.interrupt(userId, conversationId));
    taskManager.bindHandle(taskId, runHandle::interruptUpstream);
});
```

`start()` 的注释讲透了为什么**整段启动要放进锁里**：「整段启动放进锁里，否则取消可能先跑完收尾、这边随后才订阅，工具照样被执行一遍」。这是 Agent 档独有的竞态：workflow 档取消和启动之间没有「框架状态」要保护，Agent 档有——如果取消先跑完收尾、释放了闸门，紧接着 `subscribe` 才把流订阅上，那用户明明点了停止，这一轮的 ReAct 循环照样从头跑、写操作照样执行。所以「启动 + 订阅 + 绑定取消动作」是一个原子块，这个锁是 Q8 生命周期句柄的第一次登场。

**超时是两档**：workflow 档 `rag.default.sse-timeout-ms: 300000`（5 分钟），agent 档 `agent.sse-timeout-ms: 900000`（15 分钟，`AgentProperties.java:49`）。差三倍——Agent 一轮有多次工具调用 + 中间停下来等人确认，5 分钟不够，而且确认挂起时那个 `SseEmitter` 是**一直开着**的（Q11 讲）。

**埋钩子**：你可能注意到我全程说「推事件」而不是「推 token」。Agent 档这一点的分界比 workflow 档更硬——我推给前端的粒度，既不是模型吐的粒度，也不是框架吐的粒度，中间隔着一座桥。

---

## Q2（追问）为什么还是 `SseEmitter`，不上 WebFlux？那个 `Flux<AgentEvent>` 到底是什么角色？

**面试官追问**：你们用了 Project Reactor，为什么不干脆把接口做成 `Flux<ServerSentEvent>`？

**我**：因为那个 `Flux<AgentEvent>` **不是我们的 SSE 出口，是 AgentScope 的内部事件流**。把它跟 WebFlux 的响应式 SSE 混为一谈，是这个问题最常见的误解。

先把两个东西分开：

| | `Flux<AgentEvent>`（AgentScope） | WebFlux 的 `Flux<ServerSentEvent>` |
|---|---|---|
| 谁产生 | AgentScope 引擎（ReAct 循环吐事件） | 我们自己写 |
| 语义 | 「推理过程中发生了什么」 | 「要发给客户端的帧」 |
| 方向 | 引擎 → 桥 | 应用 → 浏览器 |
| 消费方 | 我们的桥 | WebFlux 框架 |

**所以正确的架构是：AgentScope 吐响应式事件流，我们用命令式订阅把它接到 Servlet 的 `SseEmitter` 上。** 三个回调恰好映射三条出口：

```java
events.subscribe(bridge::onEvent, bridge::onError, bridge::onComplete);
//                └─ 每个事件→SSE   └─ 出错→SSE   └─ 正常收尾→SSE
```

`onEvent` 就是 `AgentStreamEventBridge.onEvent()` 那个大 switch（Q3 讲）。响应式的那套背压、调度、线程切换，AgentScope 内部已经处理完了；桥拿到的是一个一个**已经成形的事件**，做的是翻译，不是编排。

**为什么不上 WebFlux？三个理由，一个技术一个经济一个风险：**

1. **双技术栈不划算。** 整个应用是 Spring MVC（Servlet）栈，认证、异常处理、拦截器、Sa-Token、`StreamTaskManager` 全在这条栈上。为一条接口引入 WebFlux，等于在同一个进程里维护两套 Web 基础设施——两套异常处理、两套拦截器、两套上下文透传。SSE 的语义（建流、写帧、关闭、超时回调）`SseEmitter` 已经表达得很干净，没有 WebFlux 才能做的事。

2. **我们不需要背压。** WebFlux 响应式 SSE 的核心收益是「从请求到响应全链路非阻塞 + 背压」。但我们的下游是「AgentScope 引擎 → 桥 → SseEmitter」——引擎那边是它自己的调度线程池，桥这边是命令式写帧，中间没有「上游快、下游慢」需要靠背压协调的场景（单个用户一条流，量级到不了）。

3. **风险在取消这端。** WebFlux 的取消是「订阅被 dispose」——`SseEmitter` 的取消则是「容器回调 + 我们手动打断框架」。我们后者的取消语义（Q9）已经和 `StreamTaskManager`、`AgentRunHandle` 深度绑定，换 WebFlux 等于把这块推倒重来。

**还有一个 `doFinally` 的细节值得讲**（`AgentChatServiceImpl.java:247-249`）：

```java
Flux<AgentEvent> events = agent.streamEvents(input, runtimeContext)
        .doFinally(signal -> runHandle.markUpstreamTerminated());
```

`doFinally` 在框架流**无论何种方式结束**（正常、出错、被取消）都会回调，它做的事是告诉句柄「上游已经死了」。这给 Q9 的取消提供了一个关键的判断依据：**打断之后，怎么知道框架到底存完盘没有**——就是靠这个信号 + 一个 `CountDownLatch` 去等。

**代价说清楚**：命令式订阅意味着桥的 `onEvent` 是**在引擎的线程上被调用的**，不是我们的线程。所以桥里的状态全部要自己保护（Q5 的 `stateLock`），不能让框架的事件线程和我们的收尾线程并发改同一块状态。

**埋钩子**：桥要翻译的「框架事件」到底有哪些、长什么样，是下一个问题。

---

## Q3（追问）AgentScope 的事件流长什么样？为什么需要一座桥？

**面试官追问**：不能把框架事件直接发给前端吗？

**我**：不能，因为**框架事件是引擎内部的语义，不是给前端的契约**。桥做的就是把前者翻译成后者。

AgentScope 一轮 ReAct 会吐**十一种**事件（`AgentStreamEventBridge.onEvent()` 的 switch，`AgentStreamEventBridge.java:151-172`）：

| 框架事件 | 含义 | 桥翻译成 |
|---|---|---|
| `TEXT_BLOCK_DELTA` | 答案文本增量 | `message{type:response}` |
| `THINKING_BLOCK_DELTA` | 推理增量 | `message{type:think}` |
| `TOOL_CALL_START` | 模型开始吐工具参数 | `tool{pending}` |
| `TOOL_RESULT_START` | 工具开始执行 | `tool{running}` |
| `TOOL_RESULT_TEXT_DELTA` | 工具结果增量 | （缓冲，不直接发） |
| `TOOL_RESULT_END` | 工具结束 | `tool{done/failed}` |
| `ALL_TOOLS_DENIED` | 整批工具被用户否决 | `tool{denied}` × N |
| `HINT_BLOCK` | 框架运行提示 | `hint` |
| `EXCEED_MAX_ITERS` | 到迭代上限 | `hint{MAX_ITERATIONS}` |
| `AGENT_RESULT` | 框架终答（非流式） | 兜底文本 |
| `REQUIRE_USER_CONFIRM` | 要求用户确认写操作 | `confirm` |

**为什么必须翻译，直接发不行吗？三个原因：**

**一，框架事件的名字和语义是引擎私有实现。** 比如 `TEXT_BLOCK_DELTA` 和 `THINKING_BLOCK_DELTA` 是两个事件，但对前端来说是「增量来了，`type` 是 `response` 还是 `think`」——这是 Q4 那个两级结构。如果前端直接依赖框架事件名，哪天 AgentScope 升级把事件改名、拆合，前端全崩。**协议是契约，契约的稳定不能押在第三方框架的事件命名上。**

**二，框架事件不是「一个事件一帧」的粒度。** `TOOL_RESULT_TEXT_DELTA` 是工具结果**逐段**吐的（一段文档可能吐几百次），如果每次都发一帧，前端帧数爆炸；所以桥把它**缓冲起来，只在 `TOOL_RESULT_END` 时一次性给**，还要做 64K 截断（Q6）。反过来，`AGENT_RESULT` 是非流式终答——它不产生任何增量，桥得拿它当「没有流式增量时的兜底」用（Q13）。

**三，有些框架事件根本不该透出。** 框架有一个 `STRUCTURED_OUTPUT_TOOL_NAME` 伪工具，是框架自己用结构化输出骗模型的，不是用户要看的，`isInternalTool()` 直接过滤（`AgentStreamEventBridge.java:538-540`）。

**所以桥的本质**：把「引擎的过程语义」翻译成「前端的过程契约」，翻译过程中**增删改**——缓冲、截断、过滤、补建（Q6）、投影时间（Q7）。

**埋钩子**：翻译的产物就是那 9 个事件。为什么是 9 个、为什么跟 workflow 档的 6 个刻意分成两套，是下一个问题。

---

## Q4（追问）协议从 6 到 9 事件，为什么两套分立而不合并？

**面试官追问**：`confirm` / `tool` 这些对 RAG 没意义，但你为什么不干脆把两套合并成一套、RAG 那边不用的事件就空着？

**我**：因为**合并会让 RAG 契约背上它永远不会有的状态**。契约的最小化本身是有价值的。先把对照表摆出来：

| 维度 | workflow 档（6） | agent 档（9） | 变化的理由 |
|---|:--:|:--:|---|
| 元信息 | `meta` | `meta` | 不变，载荷仍是 `(conversationId, taskId)` |
| 增量文本 | `message`{response,think} | `message`{response,think,**error**} | **只加 type，协议层一字未改** |
| 文本块边界 | ❌ | **`block`** | 服务端声明块的起止与类型 |
| 工具过程 | ❌ | **`tool`** | Agent 有工具调用，RAG 没有 |
| 系统提示 | ❌ | **`hint`** | 迭代上限等系统级提示，不落库 |
| 等确认 | ❌ | **`confirm`** | HITL，与 `finish` 互斥 |
| 结算 | `finish` | `finish` | 不变 |
| 终止 | `done` | `done` | 不变 |
| 取消 | `cancel` | `cancel` | 不变 |
| 限流拒绝 | `reject` | ❌ **无** | 限流在建流前以异常拒掉 |

枚举定义在 `AgentSSEEventType.java:26-78`，类注释第一行就写了「**Agent 模式 SSE 事件协议，与 workflow 协议两套分立**」。

**新增的四个事件，背后是三条原则变了：**

**原则一：从「客户端猜」到「服务端声明」——这是 `block`。** workflow 档里块的边界（哪段是推理、哪段是答案）要前端自己按 `type` 变化推断、时间自己打点。当场渲染没问题，但**刷新后重放历史**时，历史里只有块的类型没有块的起止，前端重打的时间跟真实生成时间对不上。所以 agent 档让服务端主动发 `block` 封口帧，载荷是 `AgentTextBlockSeal(kind, at, startedAt, endedAt, durationMs)`，前端直接渲染不推断。

**原则二：从「流文本」到「流过程」——这是 `tool` 和 `hint`。** workflow 档一轮中间没有可展示的中间态；agent 档会反复调工具，用户需要看到「正在查知识库」「这个工具失败了」。`tool` 载荷 `AgentToolProgress` 带 `displayName`（给人看的名字，不是代码里的 `search_knowledge`）、`batchId` + `callIndex`（一批并行工具的序号）、`durationSource`（耗时来自真测量还是整批共享）。`hint` 不落库——它是运行时提示，不是对话内容，比如到迭代上限发一条「正在生成当前执行结果的总结」，但**不判失败**（框架还会生成终答）。

**原则三：从「单向终点」到「可中断的挂起点」——这是 `confirm`。** workflow 档的流只有两种命运：走完或被取消。agent 档多了一种：走到一半停下来等用户裁决。这一步的流语义特殊——`SseEmitter` 得一直开着、`finish` 不能发（没结算完）、但也不能什么都不发（前端在等），所以有了 `confirm`，并且明确它**与 `finish` 互斥**（Q11）。

**为什么两套分立——这是个被反复问的点，我分两层答：**

1. **合并会让 RAG 契约背上不存在的状态。** 如果 RAG 也用 9 事件，那前端针对 RAG 这条流就得处理 `tool`/`confirm`/`block` 三种它**永远不会收到**的事件——协议状态机里多了三条死分支。而「RAG 一条流会不会突然冒出个 confirm」这个问题，靠的是「我们保证它不发」，而不是靠协议本身能表达——这是一种**契约层面的不诚实**。

2. **两套的变更风险彼此隔离。** RAG 档要加一个「拒绝」表达（`reject` 帧），不该碰 Agent 档；Agent 档要加一个「确认续跑」表达，也不该碰 RAG 档。合并成一套，任何一方的演进都要另一方陪跑回归测试。

**这里有个反过来说明最省事的点：`error` 是加 type，不是加事件。** 上表 `message` 那一行，agent 档只往 body 的 `type` 里加了个 `error`，协议层一个字符没动。这就是两级结构的回报——如果当初按「每个渲染通道一个 event 名」设计，加 `error` 就得动协议（加事件名、前端 switch 加分支、评测脚本改）。现在改的只是 `TextChannel.ERROR` 一个枚举（`AgentStreamEventBridge.java:683`）。

**代价回顾**：两级结构让前端在 `message` 分支里多做一次 type 判断。但跟「协议演进零成本」比，这笔账划算——**协议是契约，契约的变更是最贵的**。

**埋钩子**：`block` 这个事件服务端那一侧是最绕的——因为块的起止不在桥里产生，而且「什么时候封口」这件事，藏着一个很容易做错的边界。

---

## Q5（追问）`block` 是怎么产生的？块的边界怎么保证正确？

**面试官追问**：你怎么知道一段文本块什么时候结束、该发封口帧了？

**我**：**封口不是时间驱动的，是「事件驱动」的——下一个非增量事件到来时，封上一个文本块。** 这是理解整个块状态机的钥匙。

### 状态：一个「当前打开的文本块」+ 一个「待广播队列」

桥里维持四个关键状态（`AgentStreamEventBridge.java:126-133`）：

```java
private AgentBlock openTextBlock;      // 当前未封口的文本块
private StringBuilder openTextBuffer;  // 它的增量缓冲
private final List<AgentBlock> blocks = new ArrayList<>();      // 全部块（落库用）
private final List<AgentBlock> sealedTextBlocks = new ArrayList<>(); // 已封口待广播
```

**开块**（`appendTextBlock`）：增量到达时，如果当前没有同类块，就 `sealOpenTextBlock()` 封掉旧块、开一个新块（记 `at` 和 `startedAt`），然后 `openTextBuffer.append(delta)`。

**封口**（`sealOpenTextBlock`，`AgentStreamEventBridge.java:582-597`）：把 `openTextBuffer` 的文本回填进 `openTextBlock.setText(...)`，然后——**关键**——不是当场发，而是**扔进 `sealedTextBlocks` 待广播队列**。

**那什么触发封口？** 答案在 `onEvent` 的 switch 里：**每一个「不是文本增量」的事件，第一步都是 `sealOpenTextBlock()`**。看 `onToolCallStart`（`:381`）、`onToolExecutionStart`（`:401`）、`onToolEnd`（`:443`）、`onRequireUserConfirm`（`:293`）、`onAllToolsDenied`（`:467`）——全是。因为「模型吐到一半开始调工具了」，就意味着「上段文本结束了」。这就是块的边界：**它不需要计时器，需要的是「下一种事件」当哨兵。**

### 锁内改状态，锁外发帧

`onEvent` 的大 switch，每个分支只改内存状态，**`flushSealedTextBlocks()` 统一在 switch 之后、锁外调用**（`AgentStreamEventBridge.java:170-171`）：

```java
// onEvent 末尾
flushSealedTextBlocks();   // 锁外发 SSE
```

`flushSealedTextBlocks` 的逻辑（`:602-614`）：锁内 `List.copyOf(sealedTextBlocks)` 取快照 + `clear()`，锁外逐个 `sendEvent(BLOCK, ...)`。为什么必须锁外发——**SSE 发送是 IO**。持锁做 IO 会把整个事件处理串行化；更糟的是如果 `emitter.send` 阻塞（客户端不读、TCP 窗口满），所有回调线程都会卡在这把锁上。锁的持有时间只有「改几个字段 + 加进 list」，跟 IO 完全解耦。

### 一个最容易做错的边界：只广播有起止的块

`sealOpenTextBlock` 里那行判断（`:591-596`）：

```java
// 只广播有起止的，一次性补发的块发空帧会让前端认错待收口块
if (sealed.getStartedAt() != null) {
    long endedAt = facts.now();
    sealed.setEndedAt(endedAt);
    sealed.setDurationMs(endedAt - sealed.getStartedAt());
    sealedTextBlocks.add(sealed);
}
```

**有些块是没有「开始生成」这个过程的**——比如出错时系统插的那句中断提示（`onError` 里 `appendTextBlock(TextChannel.ERROR, NOTICE_INTERRUPTED, false)`，`streamed=false`），它是**一次性补发**的，没有 `startedAt`。如果给它发一个 `block` 封口帧，前端会以为「有一个块结束了」，于是把当前正在流式接收的那个块**错误地收口**——后面真正的增量就到了一个已经关闭的容器里，显示错位。所以规则是：**没有完整起止的块，不发声明的封口帧，只发它的内容增量。**

### 落库为什么还要一个 `blocks` 列表

`textOf(channel)`（`:546-559`）回答了「全文从哪来」：**按块序拼回**——封过口的块读 `block.getText()`，还没封口的那个块正文还没回填，读 `openTextBuffer`。落库写的是 `blocks`（JSONB），不是「发出去过的东西」。所以**发送是尽力而为，落库是必须成功**，两条路径分开——这一点跟 workflow 档是同一个原则，但 agent 档因为块结构，拼回全文多了一道「块序」的语义。

**埋钩子**：文本块的边界靠「下一种事件」当哨兵。那工具块呢？它的状态机更复杂，而且有个麻烦——不是每个工具调用都会收到完整的开始和结束事件。

---

## Q6（追问）`tool` 事件的状态机怎么保证前端看到的状态是真的？

**面试官追问**：工具状态那么多种，你怎么保证前端卡片显示的状态跟真实执行对得上？

**我**：靠一个明确的状态机 + 三个「补建」兜底 + 两个过滤/截断规则。先给状态机：

```
pending ──► running ──► done / failed
   │           │
   │           └──► interrupted（流中断时，settleBlocks 里判）
   ├──► awaiting（同批有人要确认，整批挂起）
   └──► denied（整批被用户否决）
```

状态在三个 `on*` 里推进：`onToolCallStart` 建块标 `pending`（`:374-389`）→ `onToolExecutionStart` 标 `running`（`:394-418`）→ `onToolEnd` 标终态（`:436-459`）。前端 `tool` 事件的 `status` 字段驱动卡片。

**麻烦在「不是每个调用都有完整的开始/结束事件」，所以有三个补建场景：**

| 场景 | 现象 | 处理（代码位置） |
|---|---|---|
| 确认续跑 | 那一轮没有 `ToolCallStart`（工具是上一轮确认后接着跑的） | `onToolExecutionStart` 里补建（`:404-408`） |
| 完全没有开头事件 | 框架只发了结果 | `onToolEnd` 里补建 + `log.warn`（`:447-451`） |
| 整批被拒 | 框架压根不发工具事件 | `onAllToolsDenied` 里补建（`:464-479`） |

**第三个最微妙**：整批被拒的工具**从来没有执行过**，所以给它批次号和起止时间是**撒谎**。`onAllToolsDenied` 补建时**刻意留空**这两个字段（`:473-474` 只 setStatus(DENIED)，不给 batchId 也不投影时间），前端据此显示成「未执行」，而不是「执行了 0ms」。

**两个过滤/截断规则：**

1. **内部工具不展示**：`isInternalTool()` 过滤 `STRUCTURED_OUTPUT_TOOL_NAME`（`:538-540`），框架用结构化输出的伪工具不是用户要看的。
2. **工具结果截断**：`TOOL_RESULT_MAX_CHARS = 64_000`（`:79`）。工具返回可能是一整篇文档，`onToolResultDelta` 里缓冲到 64K 就停（`:420-434`），超出部分既不落库也不展示——注意它是**在缓冲层截断**，不是发帧前截断，所以落库和展示天然一致（都只有 64K）。

**这里要讲一个「整批一致」的设计**（`onRequireUserConfirm` 里，`:294-299`）：

```java
// 整批推 awaiting，不止卡片点名的——同批都没跑，留在 pending 收尾会被判成 interrupted
for (AgentBlock opened : openToolBlocks.values()) {
    if (isOpen(opened.getStatus())) {
        opened.setStatus(AgentToolStatus.AWAITING.value());
    }
}
```

一批并行工具，如果只有点名要确认的那个推成 `awaiting`、其余留在 `pending`，那收尾时（`settleBlocks` 把开着的工具块改成 `interrupted`）这批 `pending` 会被判成「被中断」——显示上就像「这些工具被中断了」，而实际上它们是被一起挂起等用户裁决的。所以**同一批的状态要一起推**。前端 `agentTimeline.ts` 里 `batchWaiting` 字段对应的就是这个语义（`agentTimeline.ts:333-337`）：批里那些「随批等、但不是它自己要你点头」的工具，显示成「随批等待」而不是「待确认」。

**埋钩子**：工具块的 `startedAt`/`endedAt` 是从哪来的？你可能会想「桥自己 `System.currentTimeMillis()` 打一下不就行了」——这正是下一个问题要推翻的。

---

## Q7（追问）时间从哪来？为什么桥不自己打时间？

**面试官追问**：桥自己取个 `System.currentTimeMillis()` 不简单吗？为什么要绕一道？

**我**：因为**同一个事实如果有两个算法，迟早会算出两个值**。这是 `AgentToolExecutionFacts` 类注释的原话（`AgentToolExecutionFacts.java:35`）：

> trace、SSE、PG、前端一律读这里，不各自复算——同一个事实有两个算法，迟早会算出两个值

### 单一事实源：`AgentToolExecutionFacts`

这个类是一个 run 内「执行事实」的**唯一生成者**：run 起点、批次号、组内序号、单工具真起止、两条中断时刻，都只在这里产生一次。工具体（真正执行工具的地方）通过 `RuntimeContext` 拿到这个实例（`RUNTIME_CONTEXT_KEY = "ragent_tool_execution_facts"`），在执行前后调 `markStarted(toolCallId)` / `markEnded(toolCallId)`。

**桥的角色是投影，不是测量。** 看 `applyExecutionTimes`（`AgentStreamEventBridge.java:509-519`）：

```java
private void applyExecutionTimes(AgentBlock block) {
    ToolFact fact = facts.toolFact(block.getToolCallId());
    block.setStartedAt(fact.startedAt());
    block.setEndedAt(fact.endedAt());
    Long duration = fact.durationMs();
    if (duration == null) return;
    block.setDurationMs(duration);
    block.setDurationSource(AgentBlock.DURATION_SOURCE_TOOL);
}
```

桥从 `facts` 里**读**工具的真实起止，投影到块上。桥自己**没有一行 `System.currentTimeMillis()` 用来算工具耗时**（文本块的封口时间除外，那个是流式本身的语义）。

### 为什么必须这样——三个后果

1. **否则两套口径。** 工具事件的时间来自工具体，块事件的时间来自桥，两者中间有调度延迟，前端会把同一个工具在「工具卡片」和「文本块」里显示成两个不同的耗时。用户问「这俩为什么不一样」，答不上来。

2. **有的调用永远拿不到真时刻。** 没真执行过的调用（整批被拒、确认前挂起）没有起止可投影。所以 `durationMs` 的取值逻辑是**两端缺一返回 null**（`ToolFact.durationMs`，`:296-300`），`durationSource` 这个字段告诉前端「这个耗时是 `tool`（真测量）还是 `batch`（整批共享）」。前端据此决定「没测到的就不显示」，而不是显示个 0 误导人。

3. **重试和重复订阅不能改时刻。** `markStarted`/`markEnded` 都用 `compareAndSet(0L, now)` CAS 定格（`:162-179`）——重试、重复订阅都不会覆盖已经定格的起止。这个「CAS 一次」的模式贯穿整个类：`settleRun()`（`:97-100`）、`markInterrupted`/`markCancelled`（`:204-213`）都是。为什么用 CAS 而不是锁——因为这些调用散在工具体、中间件、桥、收尾四个地方，可能并发到达，CAS 比锁轻且语义就是「第一次说了算」。

### 中断时刻也是「两条路径互斥、取先写入的」

`terminationAt()`（`:226-236`）是给「断在半路的工具」补终点的：优雅中断和强制断流各有一个时刻（`interruptedAt` / `cancelledAt`），两条路径互斥，取先写入的那个。工具 `markTerminated`（`:185-190`）就用它——**断在半路的工具，终点不是「此刻」，而是「全局那个中断时刻」**，这样整条时间线里「谁在几点断的」是一致的。

**代价说清楚**：桥变复杂了——它得接受「有的调用永远拿不到真时刻」，并维护「哪些块有真时刻、哪些是补建的」。但这个复杂度换来了**时间线在 SSE / trace / PG / 前端四处的一致性**。前端的 `agentTimeline.ts` 把这条原则贯彻到了极致：文件头注释直接写「状态、批次、序号、执行耗时全部照抄服务端——**这里一个都不推断，也不碰 Date.now()**」。

**埋钩子**：时间的一致性解决了。但还有一条更狠的一致性问题——**三条收尾出口（正常 / 出错 / 取消）谁说了算**。这是下一个问题，也是这一篇最核心的并发设计。

---

## Q8（追问）`AgentRunHandle` 怎么保证三条出口只有一个能收尾？

**面试官追问**：正常完成、出错、被取消，三条路可能同时到，你怎么保证收尾只做一次？

**我**：靠一个「可重入锁 + settled 标志」的组合，把**整段收尾体**包成一个原子块。先看三条出口（`AgentRunHandle.java:223-242`）：

```java
public void complete(Runnable body) { settleAndClose(body); }
public void cancel(Runnable body)   { settleAndClose(body); }
public void fail(Runnable body)     { settleAndClose(() -> { stateSaveRequired = true; body.run(); }); }
```

三条出口都走 `settleAndClose` → `settle`：

```java
private boolean settle(Runnable body) {
    synchronized (lifecycleLock) {      // ← 可重入锁
        if (settled) return false;      // ← 已收尾，后来者直接退出
        settled = true;
        try { body.run(); }             // ← 收尾体只跑一次
        catch (Exception e) { log.error(...); }
        finally {
            taskManager.unregister(taskId);   // 注销任务
            runReleaseHooks();                // 释放钩子
        }
        return true;
    }
}
```

### 三个关键设计

**一，锁必须可重入。** 注释写死了原因（`:57`）：「必须可重入：同步结束的流会在订阅那一刻就回调收尾」。什么意思——AgentScope 有些场景会在 `subscribe` 的**同一个调用栈里**就同步触发 `onComplete`，这时候收尾发生在 `start()` 还持着 `lifecycleLock` 的时候。如果锁不可重入，同一个线程进 `settle` 会自己把自己锁死。

**二，`settled` 是「谁先抢到谁收尾」。** 三条出口无论谁先到，抢到锁的置 `settled=true`、跑收尾体；后来者看到 `settled` 直接返回 `false`。收尾体（发终态帧 + 落库）**恰好只跑一次**，不会出现「finish 和 cancel 都发了一遍」。

**三，释放钩子「每个只跑一次，收尾后再登记当场补跑」。** `onRelease`（`:146-157`）和 `runReleaseHooks`（`:280-285`）——钩子列表在收尾时遍历执行，之后 `clear()` 掉；如果收尾**之后**才 `onRelease` 登记一个新钩子，走「补跑」分支当场执行，不进列表。为什么要有这个语义：因为钩子的登记顺序就是释放顺序，而释放动作有先后依赖（Q9 讲「先补存盘、再放闸门、最后抽记忆」），不能因为「登记晚了」就跳过。

### 为什么 `start()` 也要放锁里——回到 Q1 那个竞态

`start(Runnable startup)`（`:126-141`）整段在锁里，且第一句就判 `isSettled() || isCancelled()`。这就是 Q1 说的：**取消可能先跑完收尾、这边随后才订阅，工具照样被执行一遍**。把「订阅 + 绑定取消动作」和「取消收尾」放进同一把锁，这个竞态窗口从根上不存在。

### 一个细节：`fail` 和 `complete/cancel` 的区别

`fail` 的收尾体多包了一层 `stateSaveRequired = true`（`:237-242`）。为什么——出错时框架**大概率没来得及存盘**，释放钩子里的「补存盘」逻辑（Q9）要看这个标志决定要不要补。而 `complete`/`cancel` 不置，因为正常完成框架已经存了，优雅取消也会先让框架存（Q9）。**这个标志决定了「错误路径和强制断流要不要在驱逐前补存一次」。**

**埋钩子**：三条出口的「谁先到」问题解决了。但「取消」这件事，Agent 档比 workflow 档多了一个完全不同的动作——**打断框架、等它存盘**。

---

## Q9（追问）取消怎么从「停发」演进成「中断存盘再断流」？

**面试官追问**：workflow 档取消就是停发，Agent 档为什么不能也这么干？

**我**：因为 **AgentScope 有跨轮状态要持久化**。workflow 档取消，把 `cancelled` 置上、不再发帧、把已累积内容落库就完了——因为那一轮没有「引擎状态」要保护。Agent 档不一样：这一轮里的工具执行结果、ReAct 的中间思考，都还在 AgentScope 的内存状态里，**直接断流会把这些全部丢掉，下一轮用户追问「你刚才查到了什么」时，模型失忆了**。

所以取消分两段：**先优雅打断、等框架存盘，超时才强制断流**。看 `interruptUpstream()`（`AgentRunHandle.java:169-204`）：

```java
public void interruptUpstream() {
    // 取 interruptAction = () -> agent.interrupt(userId, conversationId)
    // ① 先打断框架，让它走存盘分支
    facts.markInterrupted();
    interrupt.run();
    graceful = awaitUpstreamTermination();   // ② 等框架流结束，最多 2 秒
    if (!graceful) {
        stateSaveRequired = true;            // ③ 没等到：标记要补存盘
        facts.markCancelled();
    }
    if (current != null) current.dispose();  // ④ 最后才断流
}
```

`awaitUpstreamTermination` 就是 Q2 那个 `CountDownLatch`——`GRACEFUL_INTERRUPT_WAIT_MS = 2000L`（`:43`），`doFinally(markUpstreamTerminated)` 在框架流结束时 `countDown()`。**2 秒是拿什么换的**：打断后框架要执行完「存盘」这个收尾动作，我们等它最多 2 秒；2 秒没等到，说明框架卡住了，才 `dispose()` 强断流，并置 `stateSaveRequired=true` 让释放钩子兜底补存（`AgentChatServiceImpl.java:270-278` 里 `agent.saveAgentState`）。

**顺序不能反**——注释（`:167`）：「先打断框架、等它存盘，超时再 dispose 断流；顺序反了会丢掉本轮 Agent 状态」。如果先 `dispose`，流直接被掐，框架连「存盘」这个动作都来不及做。

### 取消信号的源头：三个容器回调归一到同一个动作

`bindEmitterLifecycle`（`AgentChatServiceImpl.java:367-379`）：

```java
AtomicBoolean recycled = new AtomicBoolean(false);
Runnable recycleUpstream = () -> {
    if (!runHandle.isSettled() && recycled.compareAndSet(false, true)) {
        taskManager.cancel(taskId);
    }
};
emitter.onTimeout(recycleUpstream);
emitter.onError(e -> recycleUpstream.run());
emitter.onCompletion(recycleUpstream);   // 关页走 completeWithError，只有 onCompletion 兜得住
```

三个容器回调（超时 / 出错 / 关闭）**归一到同一个动作**——这跟 workflow 档的 `SseEmitterSender`、`StreamTaskManager` 是同一个模式。`recycled` 这个 CAS 保证「关页 + 超时同时触发」时只取消一次。**关键在 `!runHandle.isSettled()`**：已经正常收尾的流，不能再取消，否则往 Redis 留死标记。

### 释放钩子的顺序是有依赖的

`bindReleaseHooks`（`:266-286`）登记了三件事，注释反复强调「换顺序会出问题」：

1. **补存盘**（若 `stateSaveRequired`）+ `clearStateCache`——「只清本次流实际使用的实例，避免重建后旧流误清新 Agent」；
2. **放闸门** `releaseGate`——「最后再放行同一用户的下一轮，避免新流加载状态后被本轮收尾清掉」；
3. **抽记忆** `scheduleMemoryExtraction`——「放在释放并发锁之后，确保记忆抽取时名额已归还」。

**顺序为什么重要**：如果先放闸门再清缓存，新流可能已经加载了状态、然后被旧流的 `clearStateCache` 清掉——一个用户「下一轮」抢到闸门后立刻拿到的是一份被清掉的状态。所以「清缓存」在「放闸门」**之前**，而「抽记忆」在最后（它是异步的，不阻塞主收尾）。

**埋钩子**：取消这条链路讲完了。但「取消」和「出错」之外，还有一个更早的拦截——**根本不让流建起来**的那个闸门。

---

## Q10（追问）`AgentRunGate` 按用户闸门为什么先于一切副作用？

**面试官追问**：workflow 档已经有 `@IdempotentSubmit` 了，Agent 档为什么还要再做一个闸门？

**我**：因为两者的**覆盖窗口**不一样，而 Agent 档必须覆盖整个流生命周期。先看 `@IdempotentSubmit` 和 `AgentRunGate` 的分工（`AgentRunGate.java:32-34` 的类注释）：

> 用户维度的 Agent 并发闸门：一个用户同一时刻只跑一条流。与 @IdempotentSubmit 的区别是覆盖整个流生命周期，而非控制器返回 emitter 前的同步窗口

### 实现：Redis 抢占运行位

`acquire`（`:53-60`）：

```java
String slotValue = taskId + "|" + conversationId;
RBucket<String> slot = redissonClient.getBucket("ragent:agent:running:" + userId);
if (!slot.setIfAbsent(slotValue, ttl())) {
    throw new ClientException("当前会话处理中，请稍后再发起新的对话");
}
return () -> release(userId, slotValue);
```

`setIfAbsent` 是**原子的**——多个节点上同一个用户的并发请求，只有一个能抢到。抢到的拿回一个 `release` 闭包（挂在收尾释放钩子上），抢不到的抛 `ClientException`。

**为什么「先于一切副作用」**：看 `streamChat` 的顺序（`AgentChatServiceImpl.java:85-98`）——`runGate.acquire()` 是**第一件事**，`touchConversation` / `ensureBaseline` / `addUserMessage` 这些落库都在它**之后**。所以被拒的请求**不留任何脏数据**（没有半建成的会话、没有落库的空消息）。这点跟 workflow 档「META 先发、拒绝只能走流内」形成对比：**Agent 档把拒绝提前到了建流之前，所以不需要 reject 帧**。

### 两个细节，都关于「别误伤」

**一，`release` 用 `compareAndSet(slotValue, null)` 而不是无条件删**（`:83-86`）：

```java
// 只放自己占的位：运行位若被 TTL 挤掉又被下一轮抢走，无条件删会把别人的闸门放掉
slot.compareAndSet(slotValue, null);
```

如果这条流跑得太久，TTL 到期把运行位挤掉了，紧接着同一用户的新一轮又 `setIfAbsent` 抢到了新的位——这时候旧流收尾，如果无条件 `delete`，就把**新流的闸门**放掉了，用户就能再并发一条流。`compareAndSet` 只在「值还是我当年写的那个」时才清，值对不上就是空操作。

**二，TTL 取 SSE 超时的两倍**（`:92-94`）：

```java
// 进程崩溃时没人来释放，TTL 是唯一出路
// 取 SSE 超时的两倍：长过任何一条活着的流，又不至于把用户挡到下个小时
Duration.ofMillis(agentProperties.getSseTimeoutMs() * 2);
```

进程崩溃时没有代码来调 `release`，TTL 是唯一兜底。取 2×（即 30 分钟）——长过任何一条活着的流（SSE 本身 15 分钟超时），又不会把用户挡到天荒地老。

### 还有一个配套：`runningTaskId` 认出「在跑的是不是这个会话」

`runningTaskId(userId, conversationId)`（`:66-77`）把运行位里的 `taskId|conversationId` 拆开比对——**删会话的时候要凭它认出在跑的是不是这一个**。运行位里存 `taskId|conversationId` 而不是只存 taskId，就是为了「删除会话」这个操作能判断「这条正在跑的流是不是属于要删的那个会话」。两段都是雪花数字串、不含竖线，所以 `|` 分割安全。

**埋钩子**：闸门挡住了「并发提问」。但还有另一种「这条流必须停下来」——不是被取消，是**停下来等用户裁决一个写操作**。

---

## Q11（追问）等用户确认的时候，这条流是什么状态？

**面试官追问**：写操作要人工确认，那这条 SSE 是关掉还是挂着？

**我**：**挂着**。这是整个协议里状态最特殊的一处。

### 为什么不能关流

Agent 走 ReAct 循环，中间可能遇到需要人工裁决的写操作（比如「帮我把这个工单关掉」）。这时：

- **不能发 `finish`**——这一轮没结束，用户裁决后还要继续跑；
- **不能什么都不发**——前端在等，没有任何事件会以为卡死；
- **不能关连接**——关了用户点「同意」之后没地方接续。

所以 `confirm` 事件的语义是：**流还在，但主动权交给用户**。载荷 `AgentConfirmPayload(messageId, title, calls, durationMs)`，其中 `calls` 是 `AgentConfirmCall(toolCallId, name, displayName, fields, arguments)`，`fields` 是 `AgentConfirmField(name, label, value)`——**前端渲染的是 `label` 和 `value`（给人看的），`arguments` 是原始参数**。

**消息状态落 `AWAITING_CONFIRM`**，这是 Agent 消息状态里**唯一非终态**的状态。并且有 `hasPendingConfirm` 挡住新提问——不然用户在待确认状态里再问一句，会话状态就乱了（这就是 Q1 入口那个「上一步操作还在等你确认」的检查，`AgentChatServiceImpl.java:94-96`）。

### 三个出口都要判 `settleAwaitingConfirm()`

这是这一块最重要的一条设计判断。`onComplete` / `onError` / `finishCancelledStream`——**三个出口第一件事都是判它**：

```java
// onComplete 里（AgentStreamEventBridge.java:180-183）
if (settleAwaitingConfirm()) {
    return;   // 已按确认流程收尾，不走 finish
}
```

**为什么三个出口都要判**：因为「这一轮正卡在等确认」这个状态，跟「这一轮怎么结束的」是**正交**的。正常跑完到了确认点、出错了、被取消了——三种情况下那个待确认的写操作**都还在那儿**，都该走确认流程，而不是被判成 `INTERRUPTED` 丢掉。

**一句话记住**：**用户取消的是「这次流」，不是「这个待确认动作」**。他不想再等这个回答往下生成，但那个尚未执行的写操作应该留着让他决定。而且工具与入参是从 `t_agent_state` 取当初那条原件（前端只传 `approved=true/false`），确认这条路本来就不依赖流还在不在。

### 落库失败的第二层降级

`settleAwaitingConfirm` 里如果 `settleAndPersistMessage` 拿到 null（落库失败），走 `settleUnpersistedConfirm()`（`:363-369`）：

发 `hint`（「系统繁忙，这一步没有执行；这条会话已无法继续，请新建会话重试」）+ `finish(INTERRUPTED)` + `done`。

**为什么选择判失败而不是继续挂着**：挂了也没意义——用户点「同意」之后服务端取不到原件参数，续跑必然失败。**早失败比晚失败好**，而且给出明确指引（新建会话）比留个点不动的按钮好。

**埋钩子**：确认之后怎么续跑？那条路的入口和首问**不是同一个**，而且有一个防篡改的设计藏在里面。

---

## Q12（追问）确认后怎么续跑？两条入口有什么区别？

**面试官追问**：用户点「同意」之后，是重新发一条提问请求吗？

**我**：不是重新提问，是**另一条入口** `POST /agent/v1/chat/confirm`。两条入口共享同一套 `launchStream`，但输入和前置处理不同。

### 两条入口的对照

| | 首问 `streamChat` | 续跑 `confirmPendingTool` |
|---|---|---|
| 入口 | `GET /agent/v1/chat` | `POST /agent/v1/chat/confirm` |
| 输入 | `question`（用户文本） | `conversationId + messageId + approved`（布尔） |
| 前置 | `touchConversation` + `addUserMessage` | `getPendingConfirm` + `resolveConfirmResultsOrExpire` |
| 给 Agent 的 Msg | `UserMessage(question)` | **空正文** `UserMessage` + metadata |

### 两个关键设计

**一，工具入参从 Agent 状态重取，前端只传 `approved`。** `resolveConfirmResultsOrExpire`（`AgentChatServiceImpl.java:189-200`）：

```java
List<ToolUseBlock> asking = askingToolCalls(agent.getAgentState(userId, conversationId).getContext());
if (asking.isEmpty()) {
    conversationService.expirePendingConfirm(...);   // 先结算卡片再报错
    throw new ClientException("待确认的操作已失效，请重新提问");
}
return asking.stream().map(toolCall -> new ConfirmResult(approved, toolCall)).toList();
```

**前端永远只传一个 `approved: true/false`，具体的工具名和参数一律从 Agent 的持久化状态里重取。** 为什么——如果前端回传完整参数，攻击者就能在确认卡片上**篡改**一个写操作的入参（把「关单 A」改成「关单 B」）。让参数从服务端状态取，前端就没有篡改的入口。这就是注释（`:187`）说的「工具入参从 Agent 状态取，前端只传同意/拒绝，防止篡改」。

**二，续跑的消息是「空正文 + metadata」的。** `startConfirmRun`（`:163-184`）：

```java
Msg resumeMsg = UserMessage.builder()
        .metadata(Map.of(Msg.METADATA_CONFIRM_RESULTS, confirmResults))
        .build();
```

注释（`:169`）：「空正文消息仅携带确认/拒绝结果，框架不会把它并进对话上下文」。**裁决结果不是一条「用户说了什么」，它是给框架的「你刚才要确认的那批工具，现在执行/否决」的指令**。用 metadata 而非正文承载，框架才知道「这不是一条要展示给模型看的用户消息」。

### 一个顺序上的微妙点：先结算卡片，再订阅

`settleConfirmCard` 是在 `runHandle.start()` 的锁内、`subscribe` **之前**调的（`AgentChatServiceImpl.java:250-251`）。注释（`:308-309`）：

> 只能在启动互斥区里做：先结算后订阅，取消才不会漏掉这次工具执行

如果反过来——先订阅（工具开始跑了）、后结算卡片（把「待确认」状态改掉）——那中间有个窗口：用户恰好在这一刻取消，取消逻辑看到的是「已经不在待确认态」，就会漏掉「这批工具到底执行了没」的结算。**先结算、后订阅**，保证「卡片状态」和「工具执行」之间没有缝隙。

### 还有一个过期兜底

`resolveConfirmResultsOrExpire` 里，如果状态里**已经没有待确认工具了**（用户点了两次、或者确认态被别处结算了），先 `expirePendingConfirm`（把卡片改成终态）再抛「待确认的操作已失效，请重新提问」——注释（`:195`）：「先结算卡片再报错，否则会话会一直卡住」。

**埋钩子**：讲完了正常、取消、确认三条路。最后一个是「半路出错」——那些已经生成的内容、已经执行过的工具，怎么保证落库和流出的是同一份？

---

## Q13（追问）出错/取消时，已经生成的内容怎么办？块和落库怎么对齐？

**面试官追问**：出了错，前面流了一半的内容是扔了还是保存？保存的话落库的和前端看到的一致吗？

**我**：**保存，而且落库的块、流出的帧、前端回放的块，三者必须是同一份。** 这是 Agent 档比 workflow 档多出来的一层一致性要求——因为落库的是 `blocks`（JSONB 结构），不是一段正文。

### 先看 onError：为什么 error 是「单独一个块」

`onError`（`AgentStreamEventBridge.java:206-231`）保证了「出错也发满终态」：

```java
// 中断提示单独成 error 块：这是系统在说话，混进 answer 就跟模型的回答一个身份了
// 块和当场的增量都要发，只塞进 content 的话历史回放有块就不读 content，刷新后这句就没了
appendTextBlock(TextChannel.ERROR, NOTICE_INTERRUPTED, false);
sender.sendEvent(MESSAGE, new AgentMessageDelta(TextChannel.ERROR.deltaType, NOTICE_INTERRUPTED));
String messageId = settleAndPersistMessage(content, AgentMessageStatus.INTERRUPTED);
sendTerminal(FINISH, new AgentCompletionPayload(messageId, title, "INTERRUPTED", facts.settleRun()));
```

两行注释把理由写透了。**第一句是身份问题**——系统提示不能伪装成模型的话，所以单开一个 `error` 通道。**第二句是持久化一致性问题**——前端回放历史时，`blocks` 里有块就按块渲染、**不读 content**；如果那句「回复到这里中断了」只塞进 content 而不落成块，刷新后这句就没了。所以「块」和「当场的增量」都要发。

### 收尾落库的统一入口：`settleAndPersistMessage`

三个出口（正常/出错/取消）最后都落到 `settleAndPersistMessage`（`:628-645`）：

```java
private String settleAndPersistMessage(String content, AgentMessageStatus status) {
    synchronized (stateLock) {
        thinking = textOf(REASONING);
        settled = settleBlocks();      // ① 封口 + 未完工具改 interrupted + 剔空块
    }
    flushSealedTextBlocks();           // ② 末段封口帧先于终态发出去
    try {
        return conversationService.addAssistantMessage(..., settled, ...);  // ③ 落库
    } catch (Exception e) {
        return null;                   // 拿不到 messageId，就不能发带 messageId 的终态
    }
}
```

**三件事的顺序是刻意的：**

1. **`settleBlocks()`（锁内）**：封口当前文本块、把还开着的工具块（`pending`/`running`）改成 `interrupted`、剔除空文本块。这一步让「落库的 blocks」定格成一个终态快照。
2. **`flushSealedTextBlocks()` 在落库之前**——注释（`:636`）「末段封口帧要赶在 finish/confirm/cancel 之前发出去」。为什么：终态帧（finish）里带的是落库后的 `messageId`，如果封口帧（block）落在 finish 之后，前端会先收到「这一轮结束了」再收到「最后一个块的封口」，时序错乱。**块封口 → 落库 → 终态**，这个次序不能乱。
3. **落库失败返回 null**——调用方拿到 null 就**不能再发带 messageId 的终态**（因为前端会拿这个 messageId 去回放，回放不到就空一条）。

### 一个容易漏的场景：非流式终答的补发

`onComplete` 里（`:184-199`）有个 `resend` 逻辑：

```java
String streamed = textOf(ANSWER);
content = StrUtil.isNotBlank(streamed) ? streamed : fallbackText;   // 优先流式增量
resend = streamed.isEmpty() && StrUtil.isNotBlank(content);         // 非流式没有增量
if (resend) {
    appendTextBlock(ANSWER, content, false);   // 一次性补发，不给起止
    sender.sendEvent(MESSAGE, new AgentMessageDelta("response", content));
}
```

有些模型/场景 AgentScope 不吐流式增量，只在最后给一个 `AGENT_RESULT` 终答（Q3 那个 `fallbackText`）。这时候前端**一个 `message` 帧都没收到过**，如果不补发，前端这一轮的正文就是空的。所以 `resend` 把整段终答**一次性补发**成 `message` 帧 + 一个无起止的块。注意 `appendTextBlock(..., false)` 的 `streamed=false`——这就是 Q5 说的「一次性补发的块没有起止、不发声明的封口帧」的来源。

**埋钩子**：服务端把一致性做到了「块 = 帧 = 回放」。那前端是怎么把这些帧还原成时间线的？它犯不犯错，直接决定这一整套设计有没有白做。

---

## Q14（追问）前端是怎么消费这个流的？和 workflow 档有什么区别？

**面试官追问**：前端也是手写解析吗？那么多事件它怎么还原成一条时间线？

**我**：**解析是手写的（和 workflow 档同构），但还原时间线多了一个「纯投影层」**。两件事分开讲。

### 解析层：`useAgentStream.ts`

和 workflow 档同一套「buffer 按行切、最后一段留回 buffer」的手写解析（`useAgentStream.ts:111-140`），外加两个 Agent 档特有的判断：

1. **`done` 是硬约束**（`:142-147`）：没收到 `done` 且不是自己取消的，直接 throw「连接已中断，本轮回答未完成」——否则这一轮永远停在「等待响应」。
2. **裁决走 POST、不允许重试**（`:150-152`）：

```ts
// 只发一次 失败就交给上层：Agent 一轮里可能已经执行过写操作
// 而客户端没有办法自证「这一轮在服务端没跑起来」——连一帧都没收到也可能只是回程断了
// 重发就意味着整轮重跑 写操作再执行一遍 与 HITL 的不重复提交直接冲突
async function streamOnce(...)
```

对比 workflow 档的 `streamWithRetry`（指数退避重试）——**RAG 可以重试，Agent 不可以**。理由讲透了：重试的前提是「我知道上次没成功」，而 Agent 可能已经执行过写操作，客户端**无法自证这一轮没跑起来**，重发就等于写操作再执行一遍。这个方向的偏置对保险业务是必须的。

### 投影层：`agentTimeline.ts`——「一个都不推断」

这是 Agent 档前端最值得讲的一层。文件头注释直接写：

> Agent 时间线的纯投影层：SSE 帧与落库块进来，轨迹行出去。状态、批次、序号、执行耗时全部照抄服务端——这里一个都不推断，也不碰 Date.now()

三个体现：

1. **`applyTextBlockSeal` 认领「最早那个同类且还没收口」的块**（`:161-179`）：服务端按封口先后发 block 帧，前端**依次认领**就与之一一对应，帧里不必另带块标识。认不到就丢弃——注释（`:159`）「无主的起止挂到别的块上，就是把一段文字的耗时说成另一段的」。

2. **`matchToolIndex` 优先按 `toolCallId` 配对**（`:96-114`）：同名并行工具不会串块；只有端点不回 id 的**旧数据**才按名字回落（降级后果是「同名并行且无 id 时两条结果可能张冠李戴，但调用次数与状态推进不会错」）。

3. **`batchWindow` 算批头时间是「工具体时间包络」**（`:275-292`）：起点取最早开工、终点取最晚收工——并行工具的区间本就重叠，逐条相加会把重叠段算两遍。而且注释（`:273`）明确这个「包络」**不等于**后端那个 `tool_batch`（后端从 acting 入口起算，含权限判定，比包络长几十毫秒），两个数不该互相对账。

**这套「纯投影」的价值**：它保证了「刷新前后是同一份」。因为落库的 `blocks` 和流出的 SSE 帧写的是**同一组字段**（`replayBlock` 和 `applyToolFrame` 都读 `startedAt/endedAt/durationMs/durationSource`），前端回放时不重新计算，照抄服务端——所以刷新不会「跳一下」。

**埋钩子**：前端消费得很完整。但有一个消费者**缺席了**——评测。这是这一篇里我最不体面的一处，也是最后一个问题。

---

## Q15（追问）评测侧怎么消费 Agent 档的流？

**面试官追问**：你们那个 RAGAS 评测，Agent 档也测了吗？

**我**：⚠️ **没有。这是我要主动说清楚的一件事——Agent 档的协议演进先于评测落地。**

看代码事实：`ragenteval` 里的 `eval/agent/` 目录**只有空脚手架**——`dataset/`、`metrics/`、`pipeline/`、`report/` 四个子目录里都只有 `__init__.py`，没有任何实现。现有的评测（`eval/rag/`）只覆盖 workflow 档：runner 打 `GET /rag/v3/chat`（SSE）+ `GET /rag/eval`（检索旁路）两次请求，拿回答和检索证据。

**这意味着什么，三条：**

1. **Agent 档的 9 事件协议，目前唯一的消费者是前端**。评测脚本的 `parse_sse_stream` 只认 workflow 档的 6 个事件（`finish`/`reject`/`cancel`/`message`），没有解析 `tool`/`block`/`confirm`。

2. **workflow 档的评测双接口模式，Agent 档没有对应物。** workflow 档靠 `/rag/eval` 拿检索证据来算 context_precision/recall；Agent 档要测「工具调用对不对、证据用没用」，需要的不是检索旁路，而是**工具调用轨迹**——而那个轨迹其实已经在落库的 `t_agent_message.blocks`（JSONB）里了，只是评测侧还没有读它的管道。

3. **所以如果我被追问「Agent 档质量怎么保证」，我的诚实答案是**：当前靠「前端可视化的过程流」做人工验收 + 复用 workflow 档检索质量的回归（因为 `search_knowledge` 内部跑的就是老管线）；**自动化的 Agent 评测（工具调用正确率、多轮一致性、确认流程）还没落地，这是我最想补的一块。**

**如果现在让我补，方向是**：既然 `blocks` 已经是「推理/工具/答案」的有序序列，评测其实不需要再打一次 SSE——**直接读落库的 `blocks` 就能重建完整轨迹**，比 workflow 档「双接口、两套检索」的妥协干净得多。workflow 档那个「答案和证据不同源」的老问题（打两次请求、两次独立检索），在 Agent 档根本不存在——因为块和证据本来就在一条落库记录里。

**埋钩子**（收尾）：整条链路的取舍，最后归到一句话——**workflow 档的流是「文本流」，Agent 档的流是「过程流」；过程流的前提是「过程能被声明」，而声明的可信度，靠的是单一事实源和「块=帧=回放」的一致性。协议是契约，契约的变更是最贵的，所以我把它做成了两级、做成了两套分立，而不是跟着渲染细节变形。**

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里埋的钩子 | 面试官大概率会追 | 准备好了没 |
|---|---|---|
| Q1「整段启动放进锁里」 | 不放会怎样？ | ✅ Q8：取消先跑完收尾、随后才订阅，工具照样执行一遍 |
| Q1「闸门取代幂等拦截」 | 两个有什么区别？ | ✅ Q10：同步窗口 vs 整个流生命周期 |
| Q2「命令式订阅 Flux」 | 为什么不上 WebFlux？ | ✅ Q2：双技术栈 / 不需背压 / 取消语义重做 |
| Q2「doFinally 标记上游终止」 | 这个信号干嘛用？ | ✅ Q9：等框架存盘的 CountDownLatch |
| Q3「框架事件不能直接透出」 | 为什么？ | ✅ 引擎私有语义 + 粒度不匹配 + 内部工具要过滤 |
| Q4「有意保留两套协议」 | 为什么不合并？ | ✅ 合并让 RAG 契约背不存在的状态；两套变更风险隔离 |
| Q4「error 只加 type 不加事件」 | 那 block/tool 为什么不能也塞 type？ | ✅ 它们是协议状态变化（有起止/状态机），不是渲染分流 |
| Q5「块边界靠下一种事件当哨兵」 | 那最后一个块谁封？ | ✅ 收尾时 `settleBlocks()` 统一封口 |
| Q5「只广播有起止的块」 | 没起止的块发了会怎样？ | ✅ 前端把正在接收的块错误收口，显示错位 |
| Q5「锁内改状态锁外发帧」 | 持锁发帧会怎样？ | ✅ SSE 是 IO，emitter.send 阻塞卡死所有回调线程 |
| Q6「整批推 awaiting」 | 只推点名的不行吗？ | ✅ 其余留 pending 收尾被判 interrupted，显示像被中断 |
| Q6「整批被拒不给我起止」 | 为什么不给？ | ✅ 从没执行过，给时间就是撒谎，应显示「未执行」 |
| Q7「桥只投影时间」 | 桥自己打会怎样？ | ✅ 两套口径，同一工具在卡片和文本块显示不同耗时 |
| Q7「CAS 定格时刻」 | 为什么 CAS 不是锁？ | ✅ 调用散在四处、可能并发到达，第一次说了算 |
| Q8「锁必须可重入」 | 不可重入会怎样？ | ✅ 同步结束的流在 subscribe 栈里回调收尾，自锁死 |
| Q8「fail 额外置 stateSaveRequired」 | 为什么要这个标志？ | ✅ 出错框架没存盘，释放钩子据此决定补存 |
| Q9「取消先 interrupt 再 dispose」 | 顺序反了会怎样？ | ✅ 直接断流，框架连存盘都来不及，丢本轮状态 |
| Q9「2 秒等待」 | 拿什么换的？ | ✅ 换框架优雅存盘的机会，超时才强断 + 兜底补存 |
| Q10「compareAndSet 释放」 | 无条件删会怎样？ | ✅ TTL 挤掉后被下一轮抢走，无条件删放掉别人的闸门 |
| Q11「三出口都判 settleAwaitingConfirm」 | 为什么不是只 onComplete 判？ | ✅ 挂起态与「怎么结束的」正交，取消/出错都不该丢待确认动作 |
| Q12「前端只传 approved」 | 传完整参数会怎样？ | ✅ 攻击者可在卡片上篡改写操作入参 |
| Q12「先结算后订阅」 | 反过来会怎样？ | ✅ 取消漏掉「这批工具执行了没」的结算 |
| Q13「error 单独成块」 | 只塞 content 会怎样？ | ✅ 回放有块不读 content，刷新后这句消失 |
| Q13「块封口先于终态」 | 反了会怎样？ | ✅ 前端先收到「结束」再收到「最后一个块封口」，时序错乱 |
| Q14「Agent 不重试」 | RAG 为什么能重试？ | ✅ RAG 无副作用；Agent 可能已写，且无法自证没跑起来 |
| Q14「纯投影不推断」 | 前端自己算会怎样？ | ✅ 刷新前后时间线跳变，与落库对不上 |
| Q15「Agent 档评测没落地」 | ⚠️ **主动暴露** | ✅ 诚实：空脚手架；但 blocks 已含完整轨迹，补评测比 workflow 干净 |

---

## 📋 面试建议（对真实面试的 3 条建议）

### 一、把自己定位成「协议 + 一致性设计者」，而不是「用了 AgentScope 的人」

这一篇的区分度**不在**「我知道 AgentScope 会吐事件流」，而在**每一个「一致性」都是我做出来的**。答题时把三件事反复钉住：

> 「Agent 档的流，本质是**一次推理过程的投影**。投影要可信，靠三条：**桥只投影不测量**（时间单一事实源）、**块=帧=回放**（落库和流出同一组字段）、**三出口互斥收尾**（生命周期句柄）。」

尤其 `AgentRunHandle` 和 `AgentToolExecutionFacts`，讲的时候要带出**「为什么」**——不是「我用了 CAS」，而是「三条收尾出口可能并发到达，我让第一次说了算」。

### 二、主动交两处「没做完」，这是这篇最值钱的动作

**面试官不会主动问「你哪里没做对」**，因为需要他先读懂你的代码。你主动说，效果是：

1. **Agent 档评测空脚手架**（Q15）——「协议演进先于评测落地，`eval/agent` 只有 `__init__.py`，Agent 档目前只有前端消费协议。但我能说清补救路径：`blocks` 已含完整轨迹，补评测不用走 workflow 档双接口的妥协。」这句同时证明了「我知道现状 + 我知道正确方向」。

2. **「两套协议分立」要反向讲**（Q4）——先说「我们**有意保留两套**」，再说清为什么合并更贵。**这不是缺点，是你想清楚了的证据**——但如果你不主动说，面试官可能误以为是历史遗留。

**注意**：主动暴露的前提是**同时给出判断依据和改造方向**。只说「这里做得不好」是减分；说「我因为 X 选了 A，代价 B，正确做法 C」是加分。

### 三、数字和「为什么」绑定，别只报值

这篇里会被追问的数字不多，但每一个都要带口径：

- **`sse-timeout-ms: 900000`（15 分钟）**——为什么是 workflow 档（5 分钟）的三倍：多次工具调用 + 确认挂起时 SseEmitter 一直开着。
- **`GRACEFUL_INTERRUPT_WAIT_MS = 2000L`（2 秒）**——这是「等框架存盘」的上限，不是随意取的；超时走「强断 + 兜底补存」。
- **`TOOL_RESULT_MAX_CHARS = 64_000`**——截断在缓冲层做，所以落库和展示天然一致。
- **闸门 TTL = SSE 超时 × 2（30 分钟）**——进程崩溃时唯一兜底，长过活流、又不把人挡到下小时。

**最重要的一句**是那条一致性原则（`AgentToolExecutionFacts.java:35` 类注释，可逐字引用）：

> 「同一个事实有两个算法，迟早会算出两个值。」

这句是你整篇的题眼——**从时间、到块、到收尾，你反复在做的是同一件事：消灭「第二个算法」。**

---

> **相邻稿件**
> - 《[SSE流式怎么做.md](../项目深挖/SSE流式怎么做.md)》——workflow 档 6 事件底座 + 断连治理（跨节点取消 / 限流流内拒绝 / 两级 TTFT），本篇是它的 Agent 档演进
> - 《[Agent化升级（新老机制对比）.md](Agent化升级（新老机制对比）.md)》——协议之外的新老对比（编排 / 工具 / 记忆 / 确认机制）；本篇只讲流式
> - 《[Ragent升级清单.md](Ragent升级清单.md)》——升级项总索引，第八节「追问入口」里的「② 事件桥接：AgentStreamEventBridge」即本篇

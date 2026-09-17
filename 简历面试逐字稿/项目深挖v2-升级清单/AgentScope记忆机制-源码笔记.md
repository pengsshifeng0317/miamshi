# AgentScope Java 记忆机制 · 源码笔记（完整版）

> **这份文件是什么**：`AgentScope记忆机制怎么做.md`（面试稿）的**配套源码笔记**。面试稿只留「结论 + 口径 + 每处 LLM 调用一个锚点」；代码块、文件行号、7 张图、数字速查表、修订记录**全在这里**。
>
> **什么时候翻它**：面试官追问到「那个类叫什么 / 那行大概怎么写的 / 你画一下」的时候。平时背只翻面试稿。
>
> **依据**：全部基于 **`2.0.2`** 源码逐文件逐行复核 —— `agentscope-core-2.0.2-sources.jar` 与 `agentscope-harness-2.0.2-sources.jar`（Maven Central 下载解包）。**不是 GitHub `main` 分支。**
>
> **⚠️ 这三条是这份笔记的立身之本，务必先读 6.0**：
> 1. `MemoryFlushMiddleware` 的**类注释写 `doOnComplete`，实现是 `concatWith`** —— 注释和代码语义相反，以代码为准。
> 2. `2.0.2` **没有** `MemoryBackgroundTasks` / `TranscriptMiddleware` / `LocalPeriodicGate` / `FlushQueue` / `drainFlushQueue` —— 这些是 `main` 才有的东西，讲了就翻车。
> 3. **Ragent 只依赖 `agentscope-core`，没有 `agentscope-harness`。** 三处 LLM 调用全在 harness 层。

---

## 目录

| 节 | 内容 | 对应面试稿 |
|---|---|---|
| 6.0 | 以 2.0.2 为准：五处口径修正 | 面试稿 6.0 —— **整表照搬** |
| 6.1 | 三张图：一次 flush 的全貌（图 1-3） | 面试稿无 —— **7 张图只在这里** |
| 6.2 | 五个可背诵的源码锚点 | 面试稿「锚点 A」= 这里的**锚点 1 + 锚点 5** |
| 6.3 | Consolidation 完整解读（图 4-5） | 面试稿「锚点 B」—— 关键是这里的 **6.3.4「没有算法」** |
| 6.4 | Compaction summary 完整解读（图 6-7） | 面试稿「锚点 C」—— 切点回挪在 **6.4.2** |
| 6.3.2 | 注入侧：`MEMORY.md` 怎么进 prompt | 面试稿「锚点 D」 |
| 附录 A | 关键数字速查（四张表） | 面试稿无 |
| 修订记录 | 2.0.2 去漂移总账 | 面试稿无 |

> **面试稿的第 6 层只有两块**：`6.0` 的口径修正表 + **四个锚点**（A=Flush / B=Consolidation / C=Compaction / D=注入侧，每处 LLM 调用一个）。锚点之外的代码块、行号、图、数字全在本文件。

---

> **⚠️ 版本与依赖口径（先说清，防被追问翻车）**
>
> - 本文档的**第 6 层**基于 **`2.0.2` 源码逐文件逐行复核**——`agentscope-core-2.0.2-sources.jar` 与 `agentscope-harness-2.0.2-sources.jar` 从 Maven Central 下载解包后读的。所以方法名、字段名、默认值**与 2.0.2 对齐，不是 GitHub `main` 分支**。
> - **Ragent 只依赖 `agentscope-core` + `agentscope-extensions-model-openai`（2.0.2），没有 `agentscope-harness`。** 而 Flush / Consolidation / Compaction summary 这三处调用**全都在 harness 层**（`io.agentscope.harness.agent.memory` / `...agent.middleware`）。
> - 所以被问到时，正确口径是：**「这套能力在 harness 层，我们项目用的是 core 层的裸 `ReActAgent`，本身不带。下面讲的是我读它源码的理解，不是我项目在跑的东西。」** 这句话把自己摘干净，同时证明你分得清层级——比含糊地说「我们没用它自带的记忆」强得多。
> - 第 6 层是**源码级**内容（文档里没有），其余五层是文档级内容。
> - ⚠️ **版本漂移警告**：v2 迭代很快，`main` 上已经有一批 2.0.2 没有的东西。**面试时绝对不要说 `main` 才有的符号名**——一旦面试官顺手翻 javadoc 就翻车。本文档第 6 层已按 2.0.2 清过一轮，具体清单见 **6.0**。

> ↑ 以上口径同样适用于面试稿。

---

## 第 6 层 · 源码级深挖（「我读过实现」的硬证据）

> **这一层的用法**：前五层是从**文档**能读出来的，这一层是从**源码**才能读出来的。面试官表现出「想验证你是不是真上手过」时，从 6.2 / 6.3 / 6.4 挑一两个锚点讲，比把第 1 层再背一遍有效得多。
>
> **别一次全倒**——挑一个锚点讲透，然后停，把追问权交回去。
>
> **这一层的分工**：6.0 是「先说清哪些不能讲」；6.1 是 flush 的三张图；6.2 是 flush 的五个锚点；**6.3 是 Consolidation**；**6.4 是 Compaction summary**。三处 LLM 调用各有一节。

### 6.0 以 2.0.2 为准：五处口径修正

> **面试前只来得及看一处的话，就看这里。** 下面五条是「按印象讲 / 按 `main` 分支讲会被戳穿」的地方，已逐条对照 `2.0.2` 源码核对，并给了 `文件:行号` 方便自己复核。

| # | 容易讲成的说法 | `2.0.2` 的实际情况 | 依据 |
|---|---|---|---|
| ① | flush 用 `doOnComplete` 挂在流上，「fire-and-forget，不拖慢 complete」 | **`onAgent` 用的是 `concatWith`** —— flush 是流的一部分，**会推迟 `onComplete`**；不占调用线程靠的是 `subscribeOn(boundedElastic())`。**而且该中间件的类注释里写的也正是 `doOnComplete`，与实现不符** —— 以代码为准 | `MemoryFlushMiddleware.java:44`（注释）vs `:145-155`（实现） |
| ② | 「会话日志 offload 归 `TranscriptMiddleware`，和 flush 是两个中间件」 | **`2.0.2` 没有 `TranscriptMiddleware`。** offload 就在 `MemoryFlushMiddleware#doFlush` 里，是 `flushMono.then(offloadMono)` 的第二步；而且**不受 `FlushTrigger` 门控**，每轮都跑 | `MemoryFlushMiddleware.java:190-203`；`MemoryFlushManager.java:212-292` |
| ③ | 「`HarnessAgent.close()` 会调 `MemoryBackgroundTasks.awaitQuiescence(5s)` 等在途 flush」 | **`2.0.2` 没有 `MemoryBackgroundTasks`，也没有任何为记忆写入做的排空机制。** core 层有 `GracefulShutdownManager`，但它的职责是**中断点检查**（`interruptIfShuttingDown`），不是等后台任务 | 全量 grep `MemoryBackgroundTasks` / `awaitQuiescence` → **0 命中**；`core/.../shutdown/GracefulShutdownMiddleware.java:63-91` |
| ④ | 「节流靠 `LocalPeriodicGate`，键是 `"memory-flush:" + scope + ":" + key`」 | **没有 `LocalPeriodicGate`。** 节流是中间件内的静态表 `SHARED_LAST_FLUSH_AT`（`ConcurrentHashMap<String, AtomicReference<Instant>>`）+ `AtomicReference.compareAndSet` 抢占；键是 **`isolationScope.name() + ":" + timerKeyFor(rc)`，没有 `memory-flush:` 前缀** | `MemoryFlushMiddleware.java:96-97, 217-235, 262-264` |
| ⑤ | 「同一会话同时只有一个 flush，后来的塞进 pending 槽覆盖旧的，`drainFlushQueue` 补跑」 | **没有 `FlushQueue` / pending 槽 / `drainFlushQueue` 这些东西。** 并发保护只有那一个 CAS 节流位；`ALWAYS` 模式下**没有并发合并**，谁拿到谁跑 | grep `FLUSH_QUEUES` / `FlushQueue` / `drainFlushQueue` → **0 命中** |

**这五条里 ④⑤ 最危险**——它们讲的是**根本不存在的机制**，面试官只要追问一句「那个类叫什么」，就露馅了。

**① 的性质不一样：它是「讲反了」，但反得很有价值。** 框架注释和实现不一致这件事，本身就是「我逐行读过」的最强证据——你能主动说出「注释里写的是 `doOnComplete`，但实现是 `concatWith`」，比背对一个字段名有说服力得多。

**保留成立、可以放心讲的三条**：

- **写盘语义没变**：`memory/YYYY-MM-DD.md` 用 `appendUtf8WorkspaceRelative` **追加**；`MEMORY.md` 用 `writeUtf8WorkspaceRelative` **整体覆盖**。
- **`NO_REPLY` 契约没变**：硬编码字符串相等判断，`Builder` 只校验 consolidation prompt 的两个 `%d`、**不校验** `flushPrompt`。
- **flush 不是增量抽取**：每次都把 `getContext()` 全量序列化发出去，去重纯靠 prompt 里那句 *"skip anything already covered"*。

**再补两条之前存疑、现已核实的（直接影响面试口径）：**

**① `agentscope-core` 的裸 `ReActAgent` 有没有挂载点？——五个全有。** `io.agentscope.core.middleware.MiddlewareBase` 就在 **core** 里（不在 harness），是一次性声明 5 个 hook 的接口：

| hook | 模式 | 包住什么 |
|---|---|---|
| `onAgent` | 洋葱 | **整次 agent 调用** |
| `onReasoning` | 洋葱 | **推理 / 模型调用阶段** |
| `onActing` | 洋葱 | **单次工具执行** |
| `onModelCall` | 洋葱 | **原始模型 API 调用** |
| `onSystemPrompt` | 管道 | 顺序变换 system prompt 字符串（多个中间件串行叠加） |

`io.agentscope.core.ReActAgent.Builder` 上就有 `middleware(...)`。但 **core 的 `middleware/` 目录里唯一的实现是 `TaskReminderMiddleware`——记忆相关的实现一个都没有，全在 harness 侧。**

所以 Ragent 的准确口径是：**「挂载点 core 全给了，但框架在 core 里不带任何记忆实现——带记忆的是 `agentscope-harness`（`HarnessAgent` + `MemoryFlushMiddleware` / `MemoryMaintenanceMiddleware` / `CompactionMiddleware`）。我们只依赖 core，所以那三条链得自己写，我们那三层记忆就是自己挂在这些钩子上的。」** 这比笼统说「core 没这能力」准确得多，也更显专业。

**② 洋葱顺序到底谁在外？——`order()` 数值大的在外，同值按注册顺序。** `order()` 默认返回 `1`，javadoc 原文：*"A larger number means a higher priority and places the middleware closer to the outside of the onion chain… Middlewares with the same order retain their builder registration order."* 而 harness 里**没有任何一个记忆中间件重写 `order()`** —— 所以 **6.1 图 1 里那个 `:2278 → :2296 → :2311` 的注册顺序就是真实的洋葱顺序**，可以放心讲（之前只敢说「注册顺序」，现在能说「因为都不重写 `order()`，所以注册顺序即洋葱顺序」）。

---

### 6.1 三张图：一次 flush 的全貌

#### 图 1 · 静态地图：这些组件落在哪一层

```text
┌──────────────────────────── agentscope-core ─────────────────────────────┐
│  ReActAgent（裸引擎，ragent 项目用的就是这个）                            │
│    AgentState ──► AgentStateStore                                        │
│      · getContext()  对话消息列表  ← flush 的输入来源                    │
│      · 压缩摘要 / 权限规则 / Plan Mode / todo（flush 不碰）              │
│  Middleware 接口（core 只给挂载点，不含记忆实现）                         │
│  shutdown/GracefulShutdownManager                                    │
│      · 职责是「停机时在中断点检查」，≠ 等待记忆写入（易混，见 6.0 ③）    │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │  ← ragent 项目到此为止
                                   │     （只依赖 agentscope-core 2.0.2）
┌──────────────────────────────────▼──────────── agentscope-harness ───────┐
│  HarnessAgent   （三个记忆中间件的注册顺序 :2278 → :2296 → :2311）        │
│    MemoryConfig     ── flushPrompt / flushTrigger / consolidation…       │
│    CompactionConfig ── summaryPrompt / flushBeforeCompact / keepTokens…  │
│                                                                          │
│  agent/middleware/                                                       │
│    MemoryFlushMiddleware ──── onAgent: concatWith(doFlush)               │
│        ├─ flushMemories    （受 FlushTrigger 门控）                      │
│        └─ offloadMessages  （无门控，每轮都跑 → 会话 jsonl）             │
│    MemoryMaintenanceMiddleware ─ onAgent: concatWith(maybeRunMaintenance)│
│        └─ 归档超期日流水账 → Consolidation → 清理超期会话日志            │
│    CompactionMiddleware ───── onReasoning: 压缩发生在「每次推理之前」     │
│    ToolResultEvictionMiddleware / WorkspaceContextMiddleware / …         │
│                                                                          │
│  agent/memory/                                                           │
│    MemoryFlushManager ─── 第 1 处 LLM：抽事实 → 追加日流水账 + 落 jsonl  │
│    MemoryConsolidator ─── 第 2 处 LLM：合并去重 → 覆盖 MEMORY.md         │
│    compaction/ConversationCompactor ─ 第 3 处 LLM：蒸馏前缀 → 注入上下文 │
│                                                                          │
│  Workspace（文件落点 + 唯一写者）                                         │
│    memory/YYYY-MM-DD.md        ← 只有 FlushManager 写（append）          │
│    MEMORY.md                   ← 只有 Consolidator 写（覆盖）            │
│    memory/.consolidation_state ← 只有 Consolidator 写（水位线，覆盖）    │
│    agents/<id>/sessions/<sid>.jsonl ← 只有 FlushManager 写（append+去重）│
│    memory/archive/             ← Maintenance 移动过去的超期流水账        │
└──────────────────────────────────────────────────────────────────────────┘
```

> **这张图最重要的一列是「唯一写者」。** 四个文件、四个写者、没有一把锁——`MEMORY.md` 靠「只有 Consolidator 写」避免和 flush 打架，日流水账靠「只有 FlushManager 写」避免和合并打架。这是**靠约定而不是靠同步原语**做的多写者隔离，代价在 6.3 里讲。

#### 图 2 · 流程图：一次 call() 结束后，flush 怎么走

```text
        ┌────────────────────────────────────────────┐
        │ next.apply(input) 源流发完                  │
        │ （SSE 的 token 事件已全部发出）              │
        └───────────────┬────────────────────────────┘
                        │ concatWith
                        │ ← 外层还没 complete，卡在这里等
                        ▼
        ┌────────────────────────────────────────────┐
        │ Mono.defer(() -> doFlush(agent, rc))        │
        │   .subscribeOn(boundedElastic())            │
        │     ← 只换执行线程，不脱离完成时序           │
        │   .then(Mono.<AgentEvent>empty())           │
        │     ← 不往流里补发任何事件                   │
        └───────────────┬────────────────────────────┘
                        ▼
        ┌────────────────────────────────────────────┐
        │ resolveAgentState(rc, agent) == null ? 结束 │
        │ messages = state.getContext()               │
        │ messages.isEmpty()             ? 结束       │
        │ new MemoryFlushManager(ws, model, prompt)    │
        │   ← 注意：每次调用都新建一个实例             │
        └───────────────┬────────────────────────────┘
                        ▼
        ┌────────────────────────────────────────────┐
        │ shouldFlushNow(rc)                         │
        │   ALWAYS    → true            （默认）      │
        │   NEVER     → false                        │
        │   THROTTLED → 静态表 SHARED_LAST_FLUSH_AT   │
        │       取 AtomicReference<Instant> last      │
        │       now - last < minGap ? → false         │
        │       else CAS(last → now) 成功才 true      │
        └──────┬──────────────────────────┬──────────┘
          false│                     true │
               ▼                          ▼
    ┌─────────────────────┐  ┌──────────────────────────────────────────┐
    │ flushMono = empty    │  │ flushMemories(rc, messages)              │
    │ （跳过抽取，但        │  │  1. 读 MEMORY.md          （只读参考）    │
    │   下面 offload 照跑）│  │  2. 读今日 memory/YYYY-MM-DD.md（已写）   │
    └──────────┬───────────┘  │  3. 拼 userPrompt：「别重复上面已有的」  │
               │              │     + 序列化后的对话                     │
               │              │  4. 只发 2 条：SYSTEM = flushPrompt      │
               │              │                USER   = userPrompt      │
               │              │  5. model.stream(2 条) ← 一次独立调用    │
               │              └───────────────────┬──────────────────────┘
               │                                  ▼
               │                  ┌───────────────────────────────┐
               │                  │ 结果 isBlank() 或              │
               │                  │ strip().equals("NO_REPLY") ?   │
               │                  └─────┬───────────────────┬─────┘
               │                     是 │                   │ 否
               │                        ▼                   ▼
               │            ┌────────────────────┐ ┌────────────────────────┐
               │            │ 丢弃，什么都不写    │ │ writeMemoryFiles:      │
               │            │ （空结果契约）      │ │ appendUtf8Workspace    │
               │            └────────────────────┘ │       Relative         │
               │                                   │ → memory/YYYY-MM-DD.md │
               │                                   │ 每段带 "## Memory      │
               │                                   │  Flush — <Instant>"    │
               │                                   └────────────────────────┘
               ▼
    ┌──────────────────────────────────────────────────────────────┐
    │ flushMono.then(offloadMono) ← offload 不受 FlushTrigger 门控   │
    │ offloadMessages(rc, messages, agentId, sessionId)              │
    │   → SessionTree: load() → syncFromRemote()                     │
    │       → 按 Msg.getId() 跳过已存在的 → append → flush()          │
    │   → updateSessionIndex(…, "conversation offloaded")            │
    └───────────────────────────────┬──────────────────────────────┘
                                    ▼
                    ┌────────────────────────────────┐
                    │ Mono.empty() 发出 → 源流 complete│
                    │ ← 调用方在这里才拿到 complete    │
                    └────────────────────────────────┘
```

**图 2 里四个值得单独讲的点**：

- **没有 pending 槽、没有队列、没有 drain** —— 2.0.2 的并发保护**只有 `shouldFlushNow` 里那一个 CAS 位，而且只在 `THROTTLED` 模式下生效**。默认的 `ALWAYS` 下，每次 `call()` 结束都会各跑各的 flush，同一用户的 N 个并发会话就是 N 次 LLM 调用。
- **`THROTTLED` 的 CAS 是「抢时间戳」，不是「合并任务」** —— 抢到的只是「允许发这次 LLM 调用」的权利；没抢到就**直接跳过、不排队、不补跑**。所以节流是**有损**的：被跳过那轮的对话不会单独抽取，只能等下一轮 flush 时把整份上下文再发一遍。这正好和「flush 本来就不是增量」相互成全。
- **`offload` 在 flush 之后、且完全没有门控** —— `flushMono.then(offloadMono)`。即使 flush 被跳过或抛错，offload 也一定跑：**会话 jsonl 的完整性优先级高于记忆抽取**。
- **`concatWith` 的代价是实打实的** —— 源流 complete 被推迟到整个 `doFlush` 跑完，其中包含一次完整的 LLM 调用。别把它说成 fire-and-forget。

#### 图 3 · 时序图：一次 call() 的完整时间轴

```text
用户  ReActAgent  AgentState  MemoryFlushMiddleware  MemoryFlushManager  模型  Workspace
 │        │            │              │                     │           │        │
 │─提问──►│            │              │                     │           │        │
 │        │─读 state─►│              │                     │           │        │
 │        │◄─context──│              │                     │           │        │
 │        │──ReAct 循环（推理 / 调工具）────────────────────┼───────────┼───────►│
 │◄SSE 流─│            │              │                     │           │        │
 │        │  ← token 事件不受影响：flush 不在这段链上          │           │        │
 │        │            │              │                     │           │        │
 │        │ 源流 onComplete 发出（但还没往下游传）             │           │        │
 │        │════════ concatWith：流的 complete 卡在这里等 ════════════════│        │
 │        │            │              │  Mono.defer → 订阅   │           │        │
 │        │            │              │  on boundedElastic   │           │        │
 │        │            │◄─resolveAgentState(rc, agent)         │           │        │
 │        │            │──getContext()►│                     │           │        │
 │        │            │              │──shouldFlushNow(rc)  │           │        │
 │        │            │              │   THROTTLED: 静态表+CAS│          │        │
 │        │            │      ┌───────┴───────┐              │           │        │
 │        │            │  true│        false  │              │           │        │
 │        │            │      ▼               ▼              │           │        │
 │        │            │  （false 跳过抽取，但 offload 照跑）  │           │        │
 │        │            │              │──flushMemories─────►│           │        │
 │        │            │              │                     │─读 MEMORY.md──────►│
 │        │            │              │                     │◄─只读参考──────────│
 │        │            │              │                     │─读今日流水账──────►│
 │        │            │              │                     │◄─已写部分──────────│
 │        │            │              │                     │─SYSTEM+USER 2 条─►│
 │        │            │              │                     │◄─结果 / NO_REPLY───│
 │        │            │              │                     │─append 日流水账───►│
 │        │            │              │◄────then────────────│           │        │
 │        │            │              │──offloadMessages───►│           │        │
 │        │            │              │   load+syncFromRemote│          │        │
 │        │            │              │   按 Msg.getId 去重  │           │        │
 │        │            │              │                     │─append jsonl─────►│
 │        │            │              │                     │─updateSessionIndex►│
 │        │◄─onComplete─│              │                     │           │        │
 │◄流关闭─│   ← 调用方在这里才拿到 complete（中间可能已过了一次 LLM 调用）  │        │
```

**读这张图的三个要点**：

1. **token 事件不受影响，但 complete 受影响** —— 上半段（用户能感知的）和 flush 在时间上不重叠，这是 `subscribeOn(boundedElastic())` 加 `concatWith` 的组合效果：**执行挪走了，完成时序还在**。所以「用户不会被拖慢」这句话只对了一半——内容不受影响，流的关闭时刻受影响。
2. **flush 只读 `AgentState`、只写 `Workspace`，不回写 `AgentState`** —— 记忆写入不碰会话状态机，这是它能安心异步的前提。也正因为这一点，某次 flush 失败不会让会话状态不自洽。
3. **整个 `doFlush` 就是一个 `Mono`，没有独立生命周期** —— 它不在任何线程池队列里，也不被任何全局注册表跟踪。**框架层面没有任何东西能在停机时找到它**；它之所以能被「等到」，纯粹是因为它是那条请求流的一部分。

---

### 6.2 五个可背诵的源码锚点

> **用法**：面试官说「你展开讲讲」时，从下面挑 **1 个**讲透，然后停。一次讲五个反而像背的。

**锚点 1｜框架的类注释和实现自相矛盾：注释写 `doOnComplete`，代码是 `concatWith`**

```java
// MemoryFlushMiddleware.java:44 —— 类注释原文
//   "Runs in {@link #onAgent}'s {@code doOnComplete} so long-term memories are
//    extracted and persisted after every call..."

// MemoryFlushMiddleware.java:145-155 —— 实际实现
return next.apply(input)
        .concatWith(
                Mono.defer(() -> doFlush(agent, rc))
                        .subscribeOn(Schedulers.boundedElastic())
                        .onErrorResume(e -> { log.warn("Memory flush failed: {}", e.getMessage()); return Mono.empty(); })
                        .then(Mono.<AgentEvent>empty()));
```

**注释说 `doOnComplete`——fire-and-forget、不推迟完成信号；代码写的是 `concatWith`——挂在流上、必然推迟完成信号。两者语义正好相反，以代码为准。**

准确口径是 **`concatWith`**：flush 是请求流的一部分，外层 `onComplete` 会一直等到整个 `doFlush` 跑完（其中含一次完整的 LLM 调用）才发出去。而 `subscribeOn(Schedulers.boundedElastic())` 只把**执行线程**挪出调用线程，**不改变完成时序**——「换线程」和「换完成顺序」是两件独立的事，这是最容易混为一谈的地方。

一句话版本：**「`MemoryFlushMiddleware` 的类注释说它跑在 `doOnComplete` 里，但 2.0.2 的实现是 `concatWith` + `subscribeOn(boundedElastic())`——注释和代码语义正好相反，以代码为准。所以准确说法是：flush 的执行线程不占调用线程（SSE 的 token 事件不受任何影响），但流的 complete 确实被推迟了一次 LLM 调用的时间。这个区别直接影响优雅停机的行为——因为它在流上，优雅停机等请求就会顺带等到它。」**

这个锚点的杀伤力在于**它只能逐行读源码得到**：javadoc 恰好是读代码的人最容易默认信任的东西，而它在这里是错的。

**锚点 2｜送进模型的只有两条消息**

```java
List<Msg> flushInput = new ArrayList<>();
flushInput.add(Msg.builder().role(MsgRole.SYSTEM)
        .content(TextBlock.builder().text(flushPrompt).build()).build());
flushInput.add(Msg.builder().role(MsgRole.USER)
        .content(TextBlock.builder().text(userPrompt.toString()).build()).build());
return model.stream(flushInput, null, null).reduce(new StringBuilder(), ...)
```

**注意这是「另起一次独立调用」，不是复用主对话。** 两条消息分别是：SYSTEM 装 `flushPrompt`，USER 装「`MEMORY.md` 现状 + 今日流水账现状 + 待抽取的对话」。这也解释了为什么它能换小模型——跟主推理完全解耦。

**锚点 3｜送进去的对话是「过滤 + 截断」过的**

`serializeMessages` 在序列化 `AgentState.getContext()` 时会做两件事：

| 动作 | 规则 |
|---|---|
| **过滤** | 丢掉 SYSTEM 消息；丢掉框架自己注入的内部上下文——带 `<session_context>` 的用户消息，以及 `ConversationCompactor.SUMMARY_MSG_NAME` 那条压缩摘要 |
| **截断** | 工具调用渲染成 `[tool_call: name(json)]`，入参 JSON **超 500 字符就截**；工具结果渲染成 `[tool_result: name] text`，正文**超 1000 字符就截并标 `...(truncated)`** |

一句话版本：**「它不是把上下文原样丢给模型——会话上下文和压缩摘要会被过滤掉，工具调用和结果按 500 / 1000 字符截断。这是成本控制的第一道闸，比 `truncateArgs` 还靠前。」**

**锚点 4｜`NO_REPLY` 是硬编码契约，而且 Builder 不校验它**

```java
String extracted = sb.toString();
if (extracted.isBlank() || extracted.strip().equals("NO_REPLY")) {
    log.debug("No memories to flush");
    return Mono.empty();   // ← 什么都不写
}
writeMemoryFiles(rc, extracted);
```

**纯字符串相等判断**，不是「包含」也不是正则。跟前面的 Builder 校验一对比，就构成一个很漂亮的风险观察：

| 定制入口 | Builder 是否校验 | 破坏后果 |
|---|---|---|
| `consolidationPrompt` | ✅ 构造期校验必须含**两个 `%d`** | 立刻抛异常，发现得了 |
| `flushPrompt` | ❌ **完全不校验** | 整份替换后模型回「无新增记忆」这类自然语言 → **不命中哨兵 → 每次都往流水账追加一段废话**，静默劣化 |

一句话版本：**「默认 flush prompt 里的 `NO_REPLY` 是硬编码的字符串契约，而 Builder 只校验 consolidation 的 `%d`、不校验 flush prompt。所以要定制只能在默认 prompt 后面追加，不能整份替换——不然会静默破坏空结果语义。」** 这是「我读过实现」的最强证据之一，因为它只能从源码读出来。

**锚点 5｜写盘是 append，而并发控制只有一个 CAS 位，且默认模式下不生效**

```java
// MemoryFlushManager#writeMemoryFiles —— 落盘侧
String dailyEntry = String.format("\n## Memory Flush — %s\n%s\n",
        Instant.now().toString(), content);
workspaceManager.appendUtf8WorkspaceRelative(
        rc, WorkspaceConstants.MEMORY_DIR + "/" + today + ".md", dailyEntry);

// MemoryFlushMiddleware#shouldFlushNow —— 并发侧，THROTTLED 分支
Instant now = Instant.now();
AtomicReference<Instant> ref = lastFlushAtFor(rc);       // 静态表
Instant last = ref.get();
if (Duration.between(last, now).compareTo(minGap) < 0) return false;
return ref.compareAndSet(last, now);                      // ← 唯一的并发原语
```

**落盘侧**：`appendUtf8WorkspaceRelative`——追加，永不覆盖。这是第一层「只追加、不判重」的物理保证，去重责任全部推给后面的 consolidation。

**并发侧**：2.0.2 **没有队列、没有 pending 槽、没有 drain**。全部并发保护就是上面那一个 `compareAndSet`，而且**只在 `THROTTLED` 模式下生效**——默认的 `ALWAYS` 直接 `return true`，完全不设防。CAS 的语义是「抢占时间戳」：抢到才发这次 LLM 调用，抢不到就**直接跳过、不排队、不补跑**，所以节流是**有损**的（这一点和「flush 本来就不是增量」正好相互成全）。

**节流表是 `static` 的，这是刻意设计**：`HarnessAgent.Builder.build()` 每次重建都会 new 一个新的中间件实例，表若挂在实例上，每次请求都会重置回 `Instant.EPOCH`，节流直接失效。键是 `isolationScope.name() + ":" + timerKeyFor(rc)`——**没有** `"memory-flush:"` 前缀，flush 和 maintenance 各自一张独立表。`STALE_ENTRY_MAX_AGE = 60min`，靠 `cleanupStaleEntries()` 扫掉冷 key，防止高 churn 下 map 无限涨。

一句话版本：**「写盘是 append、永不覆盖；并发只有 `shouldFlushNow` 里一个 `AtomicReference.compareAndSet`，而且默认的 `ALWAYS` 模式下根本不生效。表做成 static 是为了跨 `build()` 重建存活。但也正因为它是 static、只在单个 JVM 内，多副本部署时 N 个副本各计一份，实际频率最多是设定值的 N 倍。」**

---

### 6.3 Consolidation：「合并」是第二个独立 LLM 调用

**一句话定位**：flush 把新事实**追加**进 `memory/YYYY-MM-DD.md`；consolidation 把这些日流水账**合并**进 `MEMORY.md`——合并完是**整体覆盖**。

**挂载点**：`MemoryMaintenanceMiddleware#onAgent`（`HarnessAgent.java:2296`，注册在 `MemoryFlushMiddleware` 的 `:2278` 之后），同样是 `concatWith` + `subscribeOn(boundedElastic())`。有意思的是**这个类的注释是准确的**——它自己就写 "via `onAgent concatWith`"；反而是 `MemoryFlushMiddleware` 的注释写成了 `doOnComplete`（见 6.0 ①）。**同一个框架里，一个类的注释对、一个类的注释错**，这条本身就很值得说。

#### 6.3.1 触发链路与三重短路（图 4）

```text
     ┌──────────────────────────────────────────────────────────┐
     │ MemoryMaintenanceMiddleware#onAgent (:2296)              │
     │   next.apply(input).concatWith(                          │
     │       Mono.fromRunnable(() -> maybeRunMaintenance(rc))   │
     │           .subscribeOn(boundedElastic()))                │
     │   ← 同样挂在 concatWith 上：complete 也会被推迟           │
     └──────────────────────────┬───────────────────────────────┘
                                ▼
     ┌──────────────────────────────────────────────────────────┐
     │ maybeRunMaintenance(rc)                                  │
     │ ① now - last < minGap(默认 30min) ? → return  ← 短路一    │
     │ ② ref.compareAndSet(last, now) 失败 ? → return ← 短路二   │
     └──────────────────────────┬───────────────────────────────┘
                                ▼
     ┌──────────────────────────────────────────────────────────┐
     │ runMaintenance(rc)   —— 三步严格串行，无并行              │
     │  1. expireDailyFiles(rc)   超 90 天的 → memory/archive/   │
     │  2. consolidateMemory(rc)  ← 本图主角，里面是 .block()     │
     │  3. pruneOldSessions(rc)   超 180 天的 *.log.jsonl 删除    │
     └──────────────────────────┬───────────────────────────────┘
                                ▼
     ┌──────────────────────────────────────────────────────────┐
     │ MemoryConsolidator#consolidate(rc)                       │
     │   watermark     = readWatermark(rc)                      │
     │                   ← memory/.consolidation_state（ISO-8601）│
     │   runStart      = Instant.now()      ← 先取，后读盘       │
     │   currentMemory = readMemoryMd(rc)                       │
     │   dailyEntries  = readDailyEntries(rc, watermark)        │
     └──────────────────────────┬───────────────────────────────┘
                                ▼
                 ┌──────────────────────────────┐
                 │ dailyEntries.isBlank() ?      │
                 └──────┬────────────────┬──────┘
                   是   │                │ 否  ← 短路三
                        ▼                ▼
                 ┌────────────┐  ┌───────────────────────────────┐
                 │ Mono.empty │  │ maxChars = maxMemoryTokens * 4 │
                 │ ★ 一次 LLM │  │ systemPrompt = String.format(  │
                 │   调用都不发│  │   consolidationPrompt,         │
                 └────────────┘  │   maxMemoryTokens, maxChars)   │
                                 │ ← 两个 %d，Builder 构造期校验   │
                                 └───────────────┬───────────────┘
                                                 ▼
                                 ┌───────────────────────────────┐
                                 │ USER 消息只拼两段：            │
                                 │  "Current MEMORY.md:\n" + 现状 │
                                 │  "\n\nNew daily ledger entries │
                                 │   to merge (since <水位>):\n"  │
                                 │  + 日流水账原文                │
                                 │ ← 同样只发 2 条消息            │
                                 └───────────────┬───────────────┘
                                                 ▼
                                 ┌───────────────────────────────┐
                                 │ model.stream(2 条).reduce(...) │
                                 │  把 TextBlock 拼成一个字符串    │
                                 └───────────────┬───────────────┘
                                                 ▼
                                 ┌───────────────────────────────┐
                                 │ strip() 后 isBlank() ?         │
                                 └──────┬──────────────────┬─────┘
                                    是  │                  │ 否
                                        ▼                  ▼
                         ┌──────────────────────┐ ┌────────────────────────────┐
                         │ 跳过，水位不推进      │ │ writeUtf8WorkspaceRelative │
                         │ ★ 下一次还会重试      │ │   (rc,"MEMORY.md", 新内容) │
                         │   （自愈，不丢数据）  │ │   ← 整体覆盖，不是 append   │
                         └──────────────────────┘ └──────────────┬─────────────┘
                                                                 ▼
                                                 ┌────────────────────────────┐
                                                 │ writeWatermark(rc, runStart)│
                                                 │ ← 写「开始时刻」不是完成时刻 │
                                                 └────────────────────────────┘
```

**图 4 里五个值得单独讲的点**：

1. **三重短路，一层比一层省** —— ① 距上次不到 30 分钟，连 CAS 都不做就 return；② CAS 抢不到，说明别的线程正在跑，return；③ prompt 都拼好了，发现**没有新日流水账**，连这次 LLM 调用都不发。前两层省的是**时间**，第三层省的是**钱**。
2. **`consolidateMemory` 里是 `.block()`** —— `consolidator.consolidate(rc).block()`，在 `boundedElastic` 线程上把一条响应式链**同步阻塞**掉。也就是说 maintenance 线程会在这里死等一整次 LLM 调用。这在响应式代码里是个刻意的例外（整个 maintenance 流程本来就是同步的、一次性的），面试时可以主动点出来当**设计取舍**讨论，而不是等对方发现。
3. **`runMaintenance` 三步严格串行、没有并行** —— 归档 → 合并 → 清理会话日志。合并放中间是有道理的：先把该归档的挪进 `memory/archive/`，避免这次合并又读到过期文件。
4. **`MemoryConsolidator` 是构建期注入的单例，而 `MemoryFlushManager` 每次调用都 new 一个** —— 对比很有意思：consolidator 从 `MemoryMaintenanceMiddleware` 的构造参数进来，`maxMemoryTokens` 和 prompt 在**构建期就固定**；而 flush 那边 `doFlush` 里每次 `new MemoryFlushManager(workspaceManager, model, flushPrompt)`。所以 consolidation 的容量参数是**不可运行期变更**的，改上限要重建 Agent。
5. **水位写的是 `runStart`，不是写盘时刻** —— `runStart` 在**读盘之前**就取好了。这样在「读盘 → LLM 调用 → 写盘」整个窗口里，**任何新追加进来的日流水账都不会被跳过**。如果写成 `Instant.now()`（写完盘再取），那段窗口内 flush 追加的条目就会永远漏掉。这是整段代码里最精细的一个设计。

#### 6.3.2 容量控制：两个 `%d`，和一个无处不在的 `×4`

```java
int maxChars = maxMemoryTokens * 4;
String systemPrompt = String.format(consolidationPrompt, maxMemoryTokens, maxChars);
```

| 项 | 值 | 说明 |
|---|---|---|
| `maxMemoryTokens` 默认 | `4000` | `MemoryConsolidator` 构造参数，构建期固定 |
| `maxChars` | `maxMemoryTokens * 4` = `16000` | **硬编码估算**，不是真实 tokenizer |
| 两个 `%d` | `(maxMemoryTokens, maxChars)` | Builder **构造期校验**必须恰好两个，否则抛异常 |
| 这个上限怎么执行 | **只写在 prompt 里** | 代码层**不校验、不截断**模型的输出 |

**关键点：这个 4000 是纯「软约束」——它只出现在 prompt 文本里，代码完全不管模型有没有听话。** `writeConsolidatedMemory` 拿到什么写什么，一个字都不裁。

**那真实的硬闸在哪？在注入侧，而且比想象中弱。** 看 `WorkspaceContextMiddleware#buildWorkspaceSection`：

```java
int fixedTokens = estimateTokens(sessionContext) + estimateTokens(agentsContent)
                + estimateTokens(knowledgeBlock) + estimateTokens(additionalBlock);
if (includeMemoryContext) {
    int memoryTokens = estimateTokens(memoryContent);
    int available = maxContextTokens - fixedTokens;            // maxContextTokens 默认 8000
    if (available > 0 && memoryTokens > available) {
        memoryContent = truncateToTokenBudget(memoryContent, available);
    }
}
private static int estimateTokens(String text) { return text.length() / 4; }   // ← 又是 ×4
private String truncateToTokenBudget(String text, int maxTokens) {
    int maxChars = maxTokens * 4;
    if (text.length() <= maxChars) return text;
    return text.substring(0, maxChars) + TRUNCATION_NOTICE_...;   // ← 保留头部，砍尾部
}
```

三个结论：

1. **`×4` 是全框架统一的 chars-per-token 启发式** —— consolidator 用它算 `maxChars`，注入侧用它做 `estimateTokens`。它无处不在，而且从来不是真实 tokenizer。所以嘴上说「4000 token」的时候心里要清楚：**中文场景真实 token 数会明显高于 `length()/4` 的估算**，这个上限本身就是个估算值。
2. **8000 是共享预算，不是给 `MEMORY.md` 的专属额度** —— 先扣掉 session context、`AGENTS.md`、knowledge、additional 这几块的估算值，**剩下的才给记忆**。`AGENTS.md` 越长，留给记忆的越少。
3. **`available <= 0` 时那个 `if` 直接跳过，`MEMORY.md` 会整份注入** —— 这是读代码才看得出来的边界：预算被其他块吃光时，记忆不是被砍到 0，而是**完全不裁**。看起来是刻意的（总不能截成空串），但它意味着「上限」在最坏情况下失效。而且截断方向是**保留头部**（`substring(0, maxChars)`），尾部条目先丢——可 `MEMORY.md` 每次都是整体重写，**条目顺序由 LLM 决定**，所以「尾部有什么」本身是不确定的。

一句话版本：**「4000 这个上限只写在 prompt 里，代码不校验、不截断模型输出，是纯软约束。真正的硬截断在注入侧：`maxContextTokens` 默认 8000，但要先减去 session context 和 `AGENTS.md` 那几块的估算值，剩下多少才给 `MEMORY.md`——而且 `available <= 0` 时干脆不截。两边用的都是 `length()/4` 这个硬编码估算，不是真 tokenizer。」**

#### 6.3.3 写盘语义与「唯一写者」隔离

`consolidate` 落地只碰两个文件，**都是覆盖写**：

```java
private void writeConsolidatedMemory(RuntimeContext rc, String content) {
    workspaceManager.writeUtf8WorkspaceRelative(rc, "MEMORY.md", content);   // ← 整体覆盖
}
// 水位：STATE_REL_PATH = "memory/.consolidation_state"，写 ISO-8601 Instant
```

把它和 flush 侧放一起，就得到第 6 层那张图里「唯一写者」那一列：

| 文件 | 唯一写者 | 写语义 | 谁读 |
|---|---|---|---|
| `memory/YYYY-MM-DD.md` | `MemoryFlushManager` | **append** | `MemoryConsolidator`（读） |
| `MEMORY.md` | `MemoryConsolidator` | **整体覆盖** | `WorkspaceContextMiddleware`（注入 prompt）、flush（只读参考） |
| `memory/.consolidation_state` | `MemoryConsolidator` | **整体覆盖** | `MemoryConsolidator`（自己读水位） |
| `agents/<id>/sessions/<sid>.jsonl` | `MemoryFlushManager` | **append**（按 `Msg.getId()` 去重） | `SessionSearchTool`、会话恢复 |

**四个文件、四个写者、零把锁。** 这里没有 `synchronized`、没有文件锁、没有版本号、没有 CAS——纯靠**约定**：谁负责写哪个文件是写死在类结构里的，不是靠运行时仲裁。

于是能看出两个真实风险：

1. **`MEMORY.md` 的覆盖写没有任何并发保护。** 两个副本（或两个线程）同时 consolidation，就是经典 last-writer-wins——**后写的整份盖掉先写的，不是合并**。而 `SHARED_LAST_RUN_AT` 是 `static`、**只在单 JVM 内**，多副本下恰好是最容易同时触发的配置（30 分钟一到，所有副本的窗口同时到期）。**这个风险点只能从代码读出来，文档上一个字都没有。**
2. **`MEMORY.md` 是「可被 LLM 反复重写」的唯一真相源。** 日流水账只追加、永不丢；`MEMORY.md` 会被覆盖 N 次。所以**日流水账才是事实的持久层**，`MEMORY.md` 是它的一个**可重建的物化视图**——只要水位文件没坏，`MEMORY.md` 被写坏/写空是可以靠重跑 consolidation 救回来的。想清楚这一层，就能回答「记忆会不会被合并过程吃掉」这类追问。

> **补充一个「能救回来」的细节**：LLM 返回空时，水位**不推进**（图 4 里那条自愈路径）。所以下一次 maintenance 还会把**同一批**日流水账再发一遍。这是个 **at-least-once** 而非 at-most-once 的设计选择——宁可重复合并，也不静默丢。代价是：如果模型持续返回空，`MEMORY.md` 会一直停在旧版本，而水位一直是 30 分钟前。

#### 6.3.4 「合并算法」到底是什么？答案是：**没有算法**

这是整节最该背下来的一句。把 `consolidate` 读完，你会发现在「怎么合并」这件事上，代码**只做了一件事**：

```java
return model.stream(messages, null, null)   // 发一次 LLM 调用
        .reduce(new StringBuilder(), ...)   // 把流式输出拼成字符串
        .flatMap(sb -> { writeConsolidatedMemory(rc, sb.toString().strip()); ... });
```

**没有评分、没有权重、没有相似度去重、没有重要性排序、没有容量淘汰算法。** 输入是「`MEMORY.md` 现状 + 新增日流水账」，输出是「新的 `MEMORY.md`」，中间完全是黑盒。

`DEFAULT_CONSOLIDATION_PROMPT` 里那几条规则（大意）：保留长期有效的事实与偏好；去掉一次性/临时的内容；合并重复项；近期和频繁被引用的事实优先——**这些全部是 prompt 里的自然语言，没有一条在代码里有对应实现。** 尤其最后那条「近期和频繁被引用优先」，代码里**没有任何计数器、没有最后一次引用时间、没有访问频率统计**。这句话是一条**对模型的期望**，不是一个机制。

所以准确的回答是：**代码真正保证的只有「输入怎么选」（水位 + mtime），不保证「怎么合并」。**

再补一个对比，能把这条讲得更有杀伤力：

| | flush（写日流水账） | consolidation（写 MEMORY.md） |
|---|---|---|
| 喂进去的工具结果 | 截断到 **1000** 字符（参数 500） | **完全不截断** |
| 输入规模闸门 | 每次 call 都跑（默认 `ALWAYS`） | 30 分钟 `minGap` |
| 输出处理 | 判空 + `NO_REPLY` 哨兵 | 判空 |
| 容量控制 | 无（追加即可） | **只写在 prompt 里** |

**注意第二行和第四行**：consolidation 喂给模型的日流水账**一个字都不截**——`readDailyEntries` 把每个文件的全文 `content.strip()` 直接拼进去，只加了个 `### <文件名>` 前缀，然后按**文件名升序**排列。而它的输出上限又只是 prompt 里的一句话说。**输入不限长、输出靠自觉**——这就是为什么 `minGap = 30 分钟` 是这里唯一的成本闸门，也是为什么这个默认值是 30 分钟这么长。**这个闸门一旦设小，成本会随会话数线性上涨，而且涨的是最贵的那次全量调用。**

#### 图 5 · 时序图：一次 consolidation 的完整时间轴

```text
调用方  MemoryMaintenanceMiddleware  SHARED_LAST_RUN_AT  MemoryConsolidator  模型  Workspace
  │              │                          │                  │           │        │
  │─call()结束──►│                          │                  │           │        │
  │              │═══ concatWith：源流 complete 卡在这里等 ═══════════════════│        │
  │              │  (boundedElastic 线程) ──►│                  │           │        │
  │              │──lastRunAtFor(rc)────────►│                  │           │        │
  │              │◄─AtomicReference<Instant>─│                  │           │        │
  │              │  ① now-last < 30min ? ──► 是 → 直接返回      │           │        │
  │              │──compareAndSet(last,now)─►│                  │           │        │
  │              │◄─false → 直接返回（别的线程抢到了）            │           │        │
  │              │──expireDailyFiles(rc)────────────────────────────────────►│        │
  │              │   glob("*.md", memory/) → 解析文件名日期                  │        │
  │              │   → 早于 90 天前 → move 到 memory/archive/               │        │
  │              │──consolidate(rc)──────────┼─────────────────►│           │        │
  │              │                           │   runStart = now()           │        │
  │              │                           │   ─读 .consolidation_state─►│        │
  │              │                           │   ◄─水位 Instant────────────│        │
  │              │                           │   ─读 MEMORY.md────────────►│        │
  │              │                           │   ◄─全文────────────────────│        │
  │              │                           │   ─glob("*.md") 取 modifiedAt►│       │
  │              │                           │   ◄─候选文件列表────────────│        │
  │              │                           │   过滤 modifiedAt > 水位      │        │
  │              │                           │     （mtime 缺失/解析失败 → 保留，be safe）│
  │              │                           │   按文件名升序 → 逐个读全文──►│        │
  │              │                           │   ◄─日流水账原文（不截断）──│        │
  │              │                           │   拼 "### <文件名>\n<内容>"   │        │
  │              │                           │   ② 全空 → Mono.empty，返回   │        │
  │              │                           │   ─SYSTEM + USER 共 2 条───►│        │
  │              │                           │   ◄─合并后的 MEMORY.md─────│        │
  │              │                           │   ③ 输出 isBlank → 跳过写盘  │        │
  │              │                           │   ─写 MEMORY.md（覆盖）────►│        │
  │              │                           │   ─写 .consolidation_state─►│        │
  │              │                           │       = runStart（不是 now） │        │
  │              │◄──.block() 返回───────────┼─────────────────│           │        │
  │              │  ★ 阻塞点：整条链在这里同步等完一次 LLM 调用  │           │        │
  │              │──pruneOldSessions(rc)───────────────────────────────────►│        │
  │              │   glob("*.log.jsonl") → 早于 180 天 → delete              │        │
  │◄─onComplete──│                          │                  │           │        │
  │  ← 调用方在这里才拿到 complete                                             │        │
```

**读这张图的三个要点**：

1. **整个流程被一个 `.block()` 串成同步的** —— 从 `consolidate(rc)` 到 LLM 返回，maintenance 线程全程干等。这跟 flush 那条链（响应式、不 block）形成对比，也是为什么 maintenance 必须放在 `boundedElastic` 上：**它是会阻塞的**。
2. **「先取时间、后读盘」是全图最精细的一步** —— `runStart` 在读任何文件之前就定好了，最后写水位用的也是它。这样读盘到写盘之间**新追加的日流水账不会漏**。如果反过来用 `Instant.now()`，那段窗口的条目就永远丢了。
3. **失败方向全是不丢数据的方向** —— mtime 缺失/解析失败 → **保留该文件**（注释就是 `// be safe`）；LLM 输出为空 → **不推进水位**，下次重来；CAS 抢不到 → 跳过这轮，等下一个 30 分钟。**没有一处是「跳过以省事」**，全是「宁可多做一次也不漏」。

#### 6.3 收口（面试一句话）

> **「consolidation 本质上是『把 `MEMORY.md` 现状 + 水位之后的新日流水账，交给一次 LLM 调用重写整份 `MEMORY.md`』。代码真正保证的只有输入怎么选——水位 + mtime 过滤——合并逻辑本身完全在 prompt 里，没有评分、没有淘汰算法、连容量上限都只是 prompt 里的一句话。所以它是一条『可重建的物化视图』：日流水账只追加、永不丢，`MEMORY.md` 写坏了重跑就能救回来。三个细节值得记：水位写的是开始时刻而不是完成时刻，所以不丢中间窗口；输出为空时不推进水位，是 at-least-once；唯一的并发保护是 30 分钟里那个 CAS，多副本下各自算一份，同时触发时 `MEMORY.md` 是 last-writer-wins。」**

---

### 6.4 Compaction summary：第三个 LLM 调用（上下文压缩）

**一句话定位**：前两个 LLM 调用写的是**长期记忆**（落盘），这一个写的是**对话摘要**——它**不落盘为主**，产物是一条消息，**直接塞回上下文**顶替被压掉的那一段。

**挂载点**：`CompactionMiddleware#onReasoning`（`HarnessAgent.java:2311`，注册在**两个记忆中间件之后**）。**注意这个区别非常重要**：

| | 触发粒度 | 一次 `call()` 内可能跑几次 |
|---|---|---|
| `MemoryFlushMiddleware` / `MemoryMaintenanceMiddleware` | `onAgent` —— **每次调用**结束 | 1 次 |
| `CompactionMiddleware` | `onReasoning` —— **每次推理之前** | **N 次**（N = ReAct 循环的迭代数） |

所以压缩是**每轮推理前都检查一次**的，ReAct 循环跑 5 轮就检查 5 次。这也解释了为什么它必须便宜：它不是「会话结束才做一次」，而是**热路径上的每次迭代前哨**。

#### 6.4.1 「双阈值」怎么同时生效？——四条闸门、先到先得（图 6）

用户的直觉通常是「两个阈值怎么协调」。**答案是：不协调，是 `OR`。**

```java
private static boolean shouldCompact(List<Msg> messages, int totalTokens, CompactionConfig config) {
    if (config.getTriggerMessages() > 0 && messages.size() >= config.getTriggerMessages()) return true;
    if (config.getTriggerTokens()   > 0 && totalTokens    >= config.getTriggerTokens())   return true;
    return false;
}
```

两条独立判断，各自短路返回，**谁先到谁触发，没有优先级协商**。默认值 `triggerMessages = 50`、`triggerTokens = 0`。

**但这里有个必须提前说清的坑：`triggerTokens = 0` 不是「关」，是「动态模式的哨兵」。** `CompactionMiddleware#resolveEffectiveConfig` 会把它解出来：

```java
if (configTrigger == 0) {
    if (contextWindow > 0) {
        effectiveTrigger = contextWindow - config.getReserved();          // 默认 reserved = 20_000
        if (effectiveTrigger <= 0) {
            effectiveTrigger = Math.max(1, contextWindow / 2);            // ← 夹紧，防抖
            log.warn("Dynamic compaction trigger clamped: ...");
        }
    } else {
        effectiveTrigger = CompactionConfig.FALLBACK_TRIGGER_TOKENS;      // = 160_000
    }
}
```

所以默认配置下 **两个阈值都是活的**：条数 50 条，或者 token 到 `contextWindow - 20000`，**哪个先到就压**。

`resolveEffectiveConfig` 里还有一个对称的动态解析，管的是「压完之后留多少」：

```java
if (configKeep == -1) {                                                    // keepTokens 默认 -1
    if (contextWindow > 0) {
        int usable = contextWindow - config.getReserved();
        effectiveKeep = Math.min(config.getKeepTokensMax(),               // 8000
                        Math.max(config.getKeepTokensMin(),               // 2000
                                 (int) (usable * config.getKeepTokensRatio())));  // 0.25
    } else { effectiveKeep = 0; }
}
```

`-1` 同样**不是「关」**，是「按窗口算」：拿可用窗口的 25%，夹在 2000~8000 之间。

**「四条闸门」是这么排的（都是默认值）：**

```text
     ┌──────────────────────────────────────────────────────────────┐
     │ CompactionMiddleware#onReasoning（每次推理前）                │
     │   resolveEffectiveConfig(config, model)                      │
     │     triggerTokens == 0 → 动态 = contextWindow - reserved(20k) │
     │        夹紧：<=0 → max(1, contextWindow/2)                    │
     │        无 contextWindow → FALLBACK_TRIGGER_TOKENS = 160_000   │
     │     keepTokens == -1  → 动态 = min(8000, max(2000, usable*0.25))│
     └───────────────────────────┬──────────────────────────────────┘
                                 ▼
     ┌──────────────────────────────────────────────────────────────┐
     │ ConversationCompactor#compact                                │
     │  step 0  truncateArgs(…)      ← 默认 OFF（truncateArgsConfig=null）│
     │            25 条 / 40k token 才启动，超 2000 字符的字符串入参   │
     │            砍成「前 20 字符 + ...(argument truncated)」         │
     │  step 1  pruneToolResults(…)  ← 默认 ON（PruneConfig.defaults()）│
     │            保护最近 40k token 的工具输出，超 2000 字符的替换成  │
     │            「头 + ...(N chars pruned)... + 尾」预览；           │
     │            read_file / memory_search / memory_get /            │
     │            session_search 排除在外；可裁剪总量 ≥ 20k 才动手     │
     │  messages    = step1(step0(conversationMessages))             │
     │  totalTokens = TokenCounterUtil.calculateToken(messages)      │
     └───────────────────────────┬──────────────────────────────────┘
                                 ▼
     ┌──────────────────────────────────────────────────────────────┐
     │ shouldCompact(messages, totalTokens, config)                 │
     │   triggerMessages > 0 && size  >= 50            → true        │
     │   triggerTokens   > 0 && total >= 动态阈值       → true        │
     │   ★ 是 OR，不是 AND；谁先到谁触发，没有优先级协商               │
     └──────┬────────────────────────────────────────┬──────────────┘
       false│                                  true │
            ▼                                        ▼
    ┌────────────────┐   ┌────────────────────────────────────────────────┐
    │ Optional.empty │   │ cutoff = findSafeCutoffPoint(                  │
    │ 本轮不压缩      │   │              determineCutoffIndex(...))         │
    └────────────────┘   │  ← 可能要往回挪，见 6.4.2                        │
                         └───────────────────┬────────────────────────────┘
                                             ▼
                          ┌──────────────────────────────────────────────┐
                          │ cutoff <= 0 ? → 放弃压缩，返回 Optional.empty │
                          │ ★ 宁可这轮不压，也不产出空摘要                 │
                          └───────────────────┬──────────────────────────┘
                                              ▼
                          ┌──────────────────────────────────────────────┐
                          │ summaryInput = messages[0, cutoff)  ← 保留旧摘要│
                          │ flushInput   = filterSummaryMessages(summaryInput)│
                          │                ← 剔除旧摘要，见 6.4.3        │
                          │ tail         = messages[cutoff, size)         │
                          └───────────────────┬──────────────────────────┘
                                              ▼
                          ┌──────────────────────────────────────────────┐
                          │ flushStep   = flushBeforeCompact   ?         │
                          │                 flushMemories(rc, flushInput) │
                          │               : Mono.empty()                 │
                          │ offloadStep = offloadBeforeCompact ?         │
                          │                 offloadMessages(...)          │
                          │                 → resolveOffloadPath(rc,…)    │
                          │               : Mono.just("")                │
                          │ ★ 严格串行：flushStep.then(offloadStep)       │
                          └───────────────────┬──────────────────────────┘
                                              ▼
                          ┌──────────────────────────────────────────────┐
                          │ summarizePrefix(summaryInput, config)         │
                          │  → 第 3 次独立 LLM 调用                       │
                          │  四段 prompt：SESSION INTENT / SUMMARY /      │
                          │               ARTIFACTS / NEXT STEPS          │
                          │  工具结果截断到 500 字符，ToolUse 入参全丢     │
                          └───────────────────┬──────────────────────────┘
                                              ▼
                          ┌──────────────────────────────────────────────┐
                          │ summaryMsg = buildSummaryMessage(             │
                          │                  summary, offloadPath)        │
                          │  id   = "__compaction_summary__:" +           │
                          │         UUID.nameUUIDFromBytes(content)  ← 稳定│
                          │  name = "__compaction_summary__"              │
                          │  role = USER  ★ 不是 SYSTEM                   │
                          │  compacted = [summaryMsg] + tail              │
                          └──────────────────────────────────────────────┘
```

**四个要点**：

1. **`truncateArgs` 默认是关的**（`truncateArgsConfig = null`）—— 文档里那段 `.truncateArgs(...)` 示例是**选择性开启**，不是默认行为。所以默认生效的是 **三步**：prune → 判阈值 → summarize。（另外还有 `ToolResultEvictionMiddleware` 在 `:2317`，是**链上另一个独立中间件**，80_000 字符落盘、留首尾约 2_000——它不属于 compactor，但客人感知到的「上下文瘦身」是它们一起做的。）
2. **`pruneToolResults` 默认是开的**，而且是**不花 LLM 钱**的那一层 —— 把超 2000 字符的工具结果换成 `头 + ...(N chars pruned)... + 尾`。它有一份排除名单（`read_file` / `memory_search` / `memory_get` / `session_search`），因为这四类的完整正文本身就是后面要用的信息。**判「要不要裁剪」用的是它能省下的量（≥ 20k token），不是总量**——省得不够多就不折腾。
3. **`triggerTokens == 0` 和 `keepTokens == -1` 都是「动态哨兵」不是「关」** —— 这是最容易答错的两个值。默认配置下两个触发阈值**同时生效**，`OR` 关系。
4. **`max(1, contextWindow / 2)` 那个夹紧分支是防抖** —— 当 `reserved(20_000)` 超过窗口本身（小窗口模型）时，`contextWindow - reserved` 会算成 ≤ 0。若不夹紧，阈值就是 0 或负数，**每轮推理都会触发压缩**，然后压缩完还是超阈值、再压缩——死循环式抖动。夹到窗口一半是个保守的兜底。

一句话版本：**「不是双阈值协商，是四条闸门先到先得：`truncateArgs`（默认关）、`pruneToolResults`（默认开，不花 LLM 钱）、按条数 50 触发、按 token 触发。后两个是 `OR`。而且 `triggerTokens = 0` 和 `keepTokens = -1` 都不是『关』，是『按上下文窗口动态算』——默认下两个阈值都是活的，token 阈值等于 `contextWindow - 预留 20000`，算不出窗口时兜底 160000。」**

#### 6.4.2 分割点与工具对保护：为什么要往回挪

`determineCutoffIndex` 算出一个**建议**分割点（粗略二分：以 `maxIter = Integer.SIZE - Integer.numberOfLeadingZeros(size) + 1` 步逼近，候选值夹在 `Math.min(candidate, size - 1)`；或者 `size - keepMessages`；或者动态 `keepTokens` 折算成条数），然后交给 `findSafeCutoffPoint` 做安全修正。**修正的核心只有一件事：不能让 `messages[cutoff]` 是一条 TOOL 消息。**

原因很硬：OpenAI 兼容接口**拒绝孤立的 tool 结果**——tool 消息必须能找到发起它的那条 ASSISTANT（带 `ToolUseBlock` 的那条）。如果 cutoff 正好切在「ASSISTANT 发起工具 → TOOL 返回结果」中间，被压掉的是 ASSISTANT、保留的是 TOOL，**下一次请求直接 400**。

修法：

```text
如果 messages[cutoff] 是 TOOL：
  ① 连续收集从 cutoff 开始的所有相邻 TOOL 消息的 id
  ② 往回搜，找到那个 ASSISTANT（它发起的 tool 调用 id 覆盖了上面这些）
  ③ 把 cutoff 挪到这条 ASSISTANT 之前
     → 即「ASSISTANT + 它引发的 TOOL 结果」整对都进被摘要的那一侧
  兜底：往回扫连续 TOOL 消息，跳过它们
```

**代价要说清楚**：这保证了协议合法，但**牺牲了那一轮的推理上下文**——发工具调用的那次推理被摘掉了，只剩摘要。同时它也让实际保留的尾部**比配置想要的更长**（`keepMessages` 是「至少」，不是「恰好」）。

还有一个必答的收尾：**`cutoff <= 0` 时整个压缩放弃，返回 `Optional.empty()`。** 宁可这一轮不压、上下文继续涨，也**不产出一条空摘要**把历史顶掉。这和 consolidation「输出为空就不推进水位」是同一种取向：**失败方向统统指向「不丢数据」。**

一句话版本：**「分割点先按 token/条数粗算，然后必须过一道安全检查：如果切点正好落在一条 TOOL 消息上，就要往回找到发起它的那条 ASSISTANT，把切点挪到它前面——让『工具调用 + 结果』整对进被摘要的一侧。因为 OpenAI 兼容接口不允许孤立的 tool 结果，切坏了下一轮请求直接 400。代价是那一轮的推理细节被摘要吞掉了。如果切点算出来 ≤ 0，整个压缩直接放弃——宁可这轮不压，也不产出空摘要。」**

#### 6.4.3 摘要消息的身份，和「摘要的摘要」怎么不出错

三个设计点，每一个都能单独展开：

**① 它是 `USER` 角色，不是 `SYSTEM`。**

```java
Msg.builder()
   .id(buildSummaryMessageId(content))
   .role(MsgRole.USER)
   .name(SUMMARY_MSG_NAME)          // "__compaction_summary__"
   .content(TextBlock.builder().text(...).build())
   .build();
```

摘要以**用户消息**的身份插在最前面（`compacted = [summaryMsg] + tail`）。因为多数 OpenAI 兼容模型不支持「历史中间插一条 system」，而一条 USER 消息放在最前面，在模型看来就是「用户一开始交代了这些背景」——**兼容性最好的位置**。

**② id 是内容寻址的稳定 id。**

```java
public static final String SUMMARY_MSG_NAME = "__compaction_summary__";
private static String buildSummaryMessageId(String content) {
    UUID stableId = UUID.nameUUIDFromBytes(content.getBytes(StandardCharsets.UTF_8));
    return SUMMARY_MSG_NAME + ":" + stableId;
}
```

`nameUUIDFromBytes` 是**确定性**的（MD5 派生的 v3 UUID）：**同样的摘要文本 → 同样的 id**。这不是为了好看——回想 flush 侧 offload 的去重就是按 `Msg.getId()` 做的。稳定 id 让「同一份摘要被重复 offload」变成幂等操作，而不是每次都追加一条重复的。

**③ 「进入下一次压缩」时，摘要要保留；「进入 flush」时，摘要要剔除。这个分叉是整节最精妙的一处。**

```java
List<Msg> summaryInput = new ArrayList<>(messages.subList(0, cutoff));  // ← 含旧摘要
List<Msg> flushInput   = filterSummaryMessages(summaryInput);           // ← 剔除旧摘要
```

同一个前缀，喂给**两个不同的下游**时被做了**相反**的处理：

| 下游 | 旧摘要怎么处理 | 为什么 |
|---|---|---|
| `summarizePrefix`（做新摘要） | **保留** | 不保留就「摘要的摘要」会丢掉更早的会话意图——每次压缩只看得到最近一段，越压越空 |
| `flushMemories`（抽长期记忆） | **剔除** | 旧摘要里的内容**在更早的那几轮已经 flush 过了**。再喂一遍 = 让日流水账无限长下去，而且直接违反 flush prompt 里「别重复已覆盖的」那条约定 |

**这是「读写权分离」的一个漂亮实例：同一份数据，给谁的视图不一样。** 说得出这一点，基本就证明是真读过而非背过。

**④ 摘要 prompt 的四个固定小节**：`## SESSION INTENT`（用户到底想干什么）/ `## SUMMARY`（做了什么）/ `## ARTIFACTS`（产出了什么文件）/ `## NEXT STEPS`（还差什么）。**这是让摘要「可继续工作」而不是「可读」**——`NEXT STEPS` 那节是给下一次推理的 agent 看的，不是给人看的。

**⑤ 摘要入参本身也被截断**：渲染消息时工具结果截到 **500** 字符，`ToolUseBlock` 的**入参直接丢掉**。所以摘要看到的是「调过哪些工具、结果大意」，而不是完整参数——这也是为什么 `ARTIFACTS` 那节能记住文件路径就够用了。

**⑥ 三条 fail-open 兜底**：摘要失败时不会中断对话，而是把 `"(Summary unavailable)"` / `"(Summarization failed: ...)"` 这样的文本**当作摘要写进上下文**（同时保留 `tail`）。**降级为「摘要质量差一点」，而不是「这轮对话挂掉」。**

#### 6.4.4 offload 与 flush 的先后关系：严格串行，而且 offload 的返回值是要用的

```java
Mono<Void>   flushStep   = config.isFlushBeforeCompact()   ? flushManager.flushMemories(rc, flushInput)... : Mono.empty();
Mono<String> offloadStep = config.isOffloadBeforeCompact()
        ? Mono.fromCallable(() -> { flushManager.offloadMessages(rc, messages, agentId, sessionId);
                                    return flushManager.resolveOffloadPath(rc, agentId, sessionId); })
              .onErrorResume(e -> { log.warn(...); return Mono.just(""); })
        : Mono.just("");

return flushStep.then(offloadStep)
        .flatMap(offloadPath -> summarizePrefix(summaryInput, config)
                .map(summary -> { Msg summaryMsg = buildSummaryMessage(summary, offloadPath.isBlank() ? null : offloadPath); ... }));
```

三个要点：

1. **严格串行，而且两步都在摘要 LLM 调用之前** —— `flush → offload → summarize`。**顺序不能换**：flush 要从上下文里抽事实，offload 要落全量原文，两个都必须**在原始消息被摘要替换之前**跑完。摘要一旦生成，原文就只存在于 offload 的文件里了。这正是 `flushBeforeCompact` / `offloadBeforeCompact` **默认都是 `true`** 的意义。
2. **offload 的返回值是被消费的** —— 它不是 fire-and-forget：`resolveOffloadPath` 拿到的路径会**嵌进摘要消息的正文**，告诉后续推理「被压掉的原文在这个文件里，需要细节去查」。所以摘要消息有两种措辞：拿到路径 → 指向文件；路径为空 → 退化成简单版，**但摘要照样做**。（这里和 6.3 的 offload 呼应：session jsonl 的去重靠 `Msg.getId()`，所以重复 offload 是幂等的。）
3. **三步失败的形状各不相同** —— 这是判断「你有没有真读过」的细节：

| 步骤 | 失败时 | 后果 |
|---|---|---|
| `flushMemories` | `Mono.empty()` | 这次不抽记忆，压缩照常进行 |
| `offloadMessages` | `Mono.just("")` | 不写 jsonl，但拿到空路径继续；摘要退化成简单措辞 |
| `summarizePrefix` | 兜底文本 | 上下文里写一条「摘要不可用」，对话继续 |

**没有一处失败会中断压缩，更不会中断对话。** 整条链是「尽可能多做，任何一步失败就退到次优解」——和 consolidation 的取向一致。

#### 图 7 · 时序图：一次推理前的压缩

```text
调用方  CompactionMiddleware  ConversationCompactor  MemoryFlushManager  模型  Workspace  AgentState
  │            │                      │                      │            │        │          │
  │─推理前────►│                      │                      │            │        │          │
  │            │─resolveEffectiveConfig(config, model)        │            │        │          │
  │            │  triggerTokens=0 → contextWindow-reserved    │            │        │          │
  │            │  keepTokens=-1   → 夹在 2000..8000           │            │        │          │
  │            │─compact(state.context)►│                    │            │        │          │
  │            │                      │─truncateArgs（默认关）│            │        │          │
  │            │                      │─pruneToolResults（默认开）        │        │          │
  │            │                      │  ★ 不花 LLM 钱的一层  │            │        │          │
  │            │                      │─shouldCompact?       │            │        │          │
  │            │                      │  条数 OR token，先到先得│           │        │          │
  │            │                      │─determineCutoffIndex ►│           │        │          │
  │            │                      │─findSafeCutoffPoint   │            │        │          │
  │            │                      │  切在 TOOL 上 → 往回找 ASSISTANT   │        │          │
  │            │                      │  → 挪到它之前（tool 对必须成对）   │        │          │
  │            │                      │  cutoff<=0 → 放弃，返回 empty      │        │          │
  │            │                      │─flushMemories(flushInput)────────►│        │          │
  │            │                      │  ← 旧摘要已被 filterSummaryMessages 剔除      │          │
  │            │                      │                      │─读 MEMORY.md─────►│          │
  │            │                      │                      │─读今日流水账─────►│          │
  │            │                      │                      │─2 条消息───►│     │          │
  │            │                      │                      │◄─抽取结果──│     │          │
  │            │                      │                      │─append 日流水账─►│          │
  │            │                      │◄─then────────────────│            │        │          │
  │            │                      │─offloadMessages(messages)─────────►│        │          │
  │            │                      │  load() + syncFromRemote()         │        │          │
  │            │                      │  按 Msg.getId() 去重 → append jsonl►│        │          │
  │            │                      │─resolveOffloadPath ───────────────►│        │          │
  │            │                      │◄─路径字符串─────────────────────────│        │          │
  │            │                      │─summarizePrefix(summaryInput)───►│          │          │
  │            │                      │  ★ 旧摘要保留：摘要是"累加"的     │          │          │
  │            │                      │  工具结果截 500 字符，入参丢弃    │          │          │
  │            │                      │◄─四段式摘要 / 兜底文本────────────│          │          │
  │            │                      │  buildSummaryMessage(summary, path)│         │          │
  │            │                      │   id = 内容寻址的稳定 UUID         │          │          │
  │            │                      │   role = USER（不是 SYSTEM）       │          │          │
  │            │◄─[summaryMsg] + tail─│                      │            │        │          │
  │            │─写回 state ──────────────────────────────────────────────┼────────►│          │
  │            │  ← 注意：这里会回写 AgentState，和 flush/consolidation 不一样        │          │
  │◄─继续推理──│                      │                      │            │        │          │
```

**读这张图的两个要点**：

1. **只有压缩会回写 `AgentState`** —— flush 和 consolidation 都只写 `Workspace`、从不碰会话状态（这是它们能安心异步的原因）。压缩不一样：它的产物**就是**新的上下文，必须回写。这也解释了为什么它挂在 `onReasoning`（热路径、同步）而不是 `onAgent`（收尾、异步）。
2. **LLM 调用被夹在「两个文件操作」中间** —— 前面是 flush + offload（必须先把原文和事实存好），后面是回写上下文。**这个顺序保证了一个不变量：任何时候摘要进了上下文，原文一定已经在 jsonl 里。** 这就是「压缩不丢信息」的真正根据——不是因为有备份策略，而是因为**顺序上不允许先丢后存**。

#### 6.4 收口（面试一句话）

> **「压缩是唯一挂在 `onReasoning` 上的——每次推理前检查一次，一个 ReAct 循环里可能跑 N 次，所以它必须是便宜的：默认先做不花 LLM 钱的 `pruneToolResults`（超 2000 字符的工具结果换成头尾预览，保护最近 40k token，`read_file` 这类排除），再判阈值。两个阈值是 `OR` 不是 `AND`，先到先得；而且 `triggerTokens = 0`、`keepTokens = -1` 都不是『关』，是『按上下文窗口动态算』。切点算完必须过安全检查——切在 TOOL 消息上就要往回挪到发起它的 ASSISTANT 之前，否则孤立的 tool 结果会让下一轮请求直接 400。摘要是 USER 角色、id 用内容寻址的 UUID（所以重复 offload 是幂等的）；进入下一次摘要时旧摘要保留，进入 flush 时旧摘要剔除——同一份数据给两个下游的视图相反。整条链严格串行：flush → offload → summarize，所以『摘要进了上下文』时『原文一定已经在 jsonl 里』，这是不丢信息的顺序保证，不是备份策略。」**

---

> ### 💡 这一层最反直觉的一个发现（讲出来是绝杀）
>
> **flush 根本没有「增量抽取」这回事。**
>
> 它每次都把 `state.getContext()` **全量**序列化之后发给模型，所谓「只提取新事实」完全靠 prompt 里那一句 *「skip anything already covered above」* 来约束——**去重是提示词级别的，不是代码级别的**。
>
> 两个直接推论：
> 1. **`THROTTLED` 的本质是在补一个架构缺口**，不全是为了省钱。不节流的话，每次 `call()` 都要重发一遍全量上下文。
> 2. **上下文越长浪费越严重**，且模型漏判「已覆盖」时就会产生重复条目——这也正是第一层必须「只追加、把去重推给后续合并」的原因之一。
>
> 对比本项目的做法：**用水位线 + 抽取下界做真增量**（`loadPending` 只取水位线之后的消息），重复在结构上就不可能发生，不需要赌模型自觉。**这是「为什么我自研」最硬的一条论据。**

---

---

## 附录 A · 关键数字速查

> **全部值均按 2.0.2 源码核对**（`MemoryConfig` / `CompactionConfig` / `PruneConfig` / `WorkspaceContextMiddleware` 的 Builder 默认值）。

### A-1 · Flush / Consolidation（第一层、第二层）

| 项 | 默认值 | 说明 |
|---|---|---|
| `flushTrigger` | `ALWAYS` | 每次 call 结束都 flush。**注意 `ALWAYS` 下 `shouldFlushNow` 直接 `return true`，唯一的 CAS 并发保护完全不生效** |
| `MemoryFlushMiddleware.SHARED_LAST_FLUSH_AT` | `static` `ConcurrentHashMap` | 节流表跨 `build()` 存活；键 = `isolationScope.name() + ":" + timerKeyFor(rc)` |
| `STALE_ENTRY_MAX_AGE` | 60 min | 冷 key 清理阈值，防 map 无限涨 |
| `MemoryMaintenanceMiddleware.DEFAULT_MIN_GAP` | 30 min | 后台 maintenance（含 consolidation）最小间隔 |
| `dailyFileRetentionDays` | 90 | 日流水账归档到 `memory/archive/` 的阈值 |
| `sessionRetentionDays` | 180 | `*.log.jsonl` 清理阈值 |
| `MemoryConsolidator.maxMemoryTokens` | 4_000 | **只写进 prompt 的软约束**；代码不校验、不截断 LLM 输出 |
| `maxChars` | `maxMemoryTokens × 4` = 16_000 | 硬编码 chars-per-token 估算，非真实 tokenizer |
| `STATE_FILE` | `.consolidation_state` | 水位文件（`memory/` 下），内容为 ISO-8601 `Instant` |
| 水位写入值 | `runStart` | **开始时刻**，不是完成时刻 → 中间窗口新追加的条目不会漏 |
| 空输出处理 | 不推进水位 | 自愈：下次 maintenance 重发同一批 → **at-least-once** |

### A-2 · MEMORY.md 注入（读侧）

| 项 | 默认值 | 说明 |
|---|---|---|
| `WorkspaceContextMiddleware.maxContextTokens` | 8_000 | **共享预算**：先减 session context / `AGENTS.md` / knowledge / additional，**剩下的才给 `MEMORY.md`** |
| `estimateTokens` | `text.length() / 4` | 全框架统一的估算启发式（consolidator 的 `×4` 是同一个） |
| `available <= 0` 时 | **不截断** | 那个 `if (available > 0 && …)` 直接跳过 → 记忆整份注入，「上限」在最坏情况失效 |
| `truncateToTokenBudget` 截断方向 | 保留头部 | `substring(0, maxChars) + 截断提示`，尾部条目先丢 |

### A-3 · Compaction（第三层）

| 项 | 默认值 | 说明 |
|---|---|---|
| `triggerMessages` | 50 | 按条数触发压缩（`> 0` 才参与判断） |
| `triggerTokens` | **0** | **0 = 动态，不是「关」**：`contextWindow − reserved`；算不出窗口 → `FALLBACK_TRIGGER_TOKENS = 160_000`；解出 `≤ 0` 时夹到 `max(1, contextWindow / 2)`（防抖） |
| `reserved` | 20_000 | 动态触发阈值和动态保留量的公共扣减项 |
| `keepMessages` | 20 | 压缩后**至少**保留的尾部条数（是下限，安全切点可能让实际更长） |
| `keepTokens` | **-1** | **-1 = 动态，不是「关」**：`min(8_000, max(2_000, usable × 0.25))` |
| `keepTokensMin` / `keepTokensMax` / `keepTokensRatio` | 2_000 / 8_000 / 0.25 | 上面那条夹紧式的下界、上界、比例 |
| `FALLBACK_TRIGGER_TOKENS` | 160_000 | 模型不报上下文窗口时的兜底触发值 |
| `flushBeforeCompact` | true | 压缩前先 flush（喂的是**剔除旧摘要**后的前缀） |
| `offloadBeforeCompact` | true | 压缩前先把全量原文写 jsonl，返回值路径会嵌进摘要正文 |
| `PruneConfig`（**默认 ON**） | protect 40_000 / minimum 20_000 / maxOutput 2_000 | 保护最近 40k token 的工具输出；超 2 000 字符的换成头尾预览 `...(N chars pruned)...`；**可裁剪量 ≥ 20k 才动手**；排除 `read_file` / `memory_search` / `memory_get` / `session_search` |
| `TruncateArgsConfig` | **`null` = OFF** | 开启后：25 条或 40k token 触发；超 2 000 字符的入参砍成「前 20 字符 + `...(argument truncated)`」；只处理保留窗口之前的 ASSISTANT `ToolUseBlock` |
| 摘要消息身份 | `role = USER`，`name = "__compaction_summary__"` | id = `name + ":" + UUID.nameUUIDFromBytes(内容)` → **内容寻址、稳定**，重复 offload 幂等 |
| 分割点安全修正 | 往回挪 | 切点落在 TOOL 消息上时，回找发起它的 ASSISTANT 并挪到其之前（防孤立 tool 结果）；`cutoff <= 0` → **放弃压缩** |
| 摘要入参截断 | 工具结果 500 字符 | `ToolUseBlock` 的入参**全部丢弃** |

### A-4 · 其他

| 项 | 默认值 | 说明 |
|---|---|---|
| `ToolResultEviction` 触发阈值 | 80_000 字符 | **链上独立中间件**（`:2317`），超了落盘、上下文留首尾各约 2 000 |
| `memory_search` 返回上限 | 30 条命中 | — |
| `LongTermMemory.timeout` | 60 s | Mem0 / ReMe 的 HTTP 超时（**可选外挂**，与 Harness 文件式记忆是两条独立的线） |

---

---

## 修订记录

> 记录本文档因**源码复核**而更新的地方。重读时先扫这一节，避免背到旧口径。
>
> **依据口径**：本文档第 6 层及本节的**唯一依据是 `2.0.2`**。源码来自 Maven Central 的 `agentscope-core-2.0.2-sources.jar` 与 `agentscope-harness-2.0.2-sources.jar`，本地解包后逐文件逐行读。**不再引用 `main` 分支的行号、sha 或 PR 号**——`main` 与 `2.0.2` 差异很大，引用它反而会误导。

### 一、`2.0.2` 「去漂移」总账（最重要的一次修订）

**背景**：第 6 层最初是照着 `main` 分支写的，而 Ragent 实际依赖 `2.0.2`。逐文件复核后发现**第 6 层大半内容在 `2.0.2` 上不存在**，已全部按 `2.0.2` 重写。

**五个符号在 `2.0.2` 上 grep 结果为 0 命中（即：根本没有）**：

| 符号 | 曾用在哪里 | `2.0.2` 的真相 |
|---|---|---|
| `MemoryBackgroundTasks` / `awaitQuiescence` | Q3.4、6.0 ③、图 3 | **不存在**。没有任何为记忆写入做的排空机制 |
| `TranscriptMiddleware` | Q3.4、6.0 ② | **不存在**。offload 就在 `MemoryFlushMiddleware#doFlush` 里 |
| `LocalPeriodicGate` | 附录 B、6.0 ④ | **不存在**。节流是静态表 + `AtomicReference.compareAndSet` |
| `FLUSH_QUEUES` / `FlushQueue`（running / pending 槽） | 6.2 锚点 5、6.0 ⑤ | **不存在**。没有队列、没有 pending 槽 |
| `drainFlushQueue` / `scheduleFlush` / `memory-flush:` 前缀 | 图 2、图 3、6.0 ④⑤ | **不存在**。键里没有 `memory-flush:` 前缀 |

**逐处修订清单**：

| # | 位置 | 原内容（错） | 改后（对照 `2.0.2`） | 依据（`2.0.2`） |
|---|---|---|---|---|
| 1 | 文首 | 「基于 `main` 分支源码」 | 改为「基于 `2.0.2` sources jar 逐文件逐行复核」，并加**版本漂移警告**（见 6.0） | `agentscope-core/harness-2.0.2-sources.jar` |
| 2 | Q1.3 触发点 1 | `doOnComplete(() -> scheduleFlush(agent, rc))` | **`concatWith(Mono.defer(() -> doFlush(...)).subscribeOn(boundedElastic()))`**；并注明**与该类 javadoc 第 44 行自相矛盾** | `MemoryFlushMiddleware.java:44` vs `:145-155` |
| 3 | Q3.3 | 把 `.truncateArgs(...)` 当成默认行为讲 | 补注 **`truncateArgsConfig` 默认 `null`（关）**；默认在跑的是 `pruneToolResults`；并补 `triggerTokens = 0` 是**动态**不是关 | `CompactionConfig` Builder 默认值 |
| 4 | Q3.4 全节重写 | ① flush 与 offload「同处异步」（归因错）② 「框架内建 5 秒 drain」 | ① flush 挂 `concatWith`（**会推迟 complete**），`subscribeOn` 只换线程 ② offload 是同一中间件里 `flushMono.then(offloadMono)` 的第二步、**不受 `FlushTrigger` 门控** ③ **`2.0.2` 没有任何记忆写入的排空机制**；core 的 `GracefulShutdownManager` 是**中断点检查**，不是等后台任务。反面结论：因为 flush 在请求流上，优雅停机反而顺带覆盖了它 | `MemoryFlushMiddleware.java:145-204`；`core/.../shutdown/GracefulShutdownMiddleware.java:63-91` |
| 5 | 6.0 全节重写 | 原为「三条未核实」列表 | 改为**五处口径修正表**（`doOnComplete`→`concatWith`、无 `TranscriptMiddleware`、无 `MemoryBackgroundTasks`、无 `LocalPeriodicGate`、无 `FlushQueue`），并标注「④⑤ 是讲了个不存在的机制，最危险」 | 见上表 |
| 6 | 图 1 静态地图 | 含 `TranscriptMiddleware`、`FlushQueue`、`LocalPeriodicGate` | 按 `2.0.2` 重画（中间件注册顺序 `:2278 → :2296 → :2311`、`onAgent: concatWith` 标注、**新增「唯一写者」列**） | `HarnessAgent.java:2240-2349` |
| 7 | 图 2 流程图 | 含 `scheduleFlush` / `begin()` / pending 槽 / `drainFlushQueue` / `doOnComplete` | 按 `2.0.2` 重画：`concatWith` → `doFlush` → `shouldFlushNow`（ALWAYS 直通 / THROTTLED 静态表+CAS）→ `flushMono.then(offloadMono)` → complete | `MemoryFlushMiddleware.java:138-204` |
| 8 | 图 3 时序图 | 含 `begin()` / `end()` / `drainFlushQueue` /「`doOnComplete` 换来的」 | 按 `2.0.2` 重画，并改结论为「**token 事件不受影响，但 complete 受影响**」 | 同上 |
| 9 | 6.2 锚点 1 | `doOnComplete` + PR #2777 | 改为「**注释写 `doOnComplete`、实现是 `concatWith`**」的注释-实现矛盾锚点（杀伤力更强） | `MemoryFlushMiddleware.java:44` / `:145-155` |
| 10 | 6.2 锚点 5 | `FLUSH_QUEUES` + running/pending 槽 | 改为「append 写盘 + **唯一并发原语是一个只在 `THROTTLED` 下生效的 CAS**」；补 `static` 表的理由与多副本倍数问题 | `MemoryFlushMiddleware.java:96-97, 217-264` |
| 11 | **新增 6.3** | — | **Consolidation** 全节：三重短路（图 4）、两个 `%d` 与 `×4`、唯一写者隔离、**「没有合并算法」**、时序图（图 5） | `MemoryConsolidator.java:133-254`；`MemoryMaintenanceMiddleware.java:135-274`；`WorkspaceContextMiddleware.java:194-225, 501-513` |
| 12 | **新增 6.4** | — | **Compaction summary** 全节：四条闸门（图 6）、切点与工具对保护、摘要身份与二次压缩、offload/flush 先后、时序图（图 7） | `ConversationCompactor.java:104-203`；`CompactionMiddleware.java:148-211`；`CompactionConfig.java` |
| 13 | 附录 A | `triggerTokens` 记成 `80_000（0 = 关）` 等错值 | 拆成四张表（Flush/Consolidation、注入侧、Compaction、其他），全部按 `2.0.2` Builder 默认值重列 | `MemoryConfig` / `CompactionConfig` / `PruneConfig` 默认值 |
| 14 | 附录 B | 「压缩有四套正交策略，默认全关」；`LocalPeriodicGate` + `"memory-flush:"` 键 | 逐条改正，并**新增 8 行**易错项（`triggerTokens=0` 是动态、双阈值是 `OR`、`concatWith` 不是 `doOnComplete`、无队列、4000 是软约束、consolidation 无算法、无文件锁、切点要回挪、旧摘要两个下游视图相反） | 见 6.3 / 6.4 |
| 15 | Q3.3 | 见 #3 | 同上 | 同上 |

> ⚠️ **#2 / #4 / #6 / #7 / #8 / #9 全部由同一个根因引起**：把 `doOnComplete` 当成了事实。它是本文档里**最容易被继承下去的一个错**——因为它听起来很合理（「异步、不拖慢、fire-and-forget」），所以一旦写进一层就会被后面几层反复引用。**记住结论：2.0.2 是 `concatWith`。**

### 二、修订后取得的两个「新证据」（比原来更强的说法）

| 位置 | 新说法 | 为什么更强 |
|---|---|---|
| 6.2 锚点 1 | **「该中间件的类注释写的是 `doOnComplete`，实现是 `concatWith`——注释和代码语义相反」** | 这是**只能逐行读源码**才能拿到的发现。javadoc 恰好是读代码的人最容易默认信任的东西，而它在这里是错的 |
| 6.0 / 6.3 / 6.4 | **「代码真正保证的只有输入怎么选，不保证怎么合并」** | 把「我读过实现」升级成「我知道哪些是代码保证的、哪些只是 prompt 里的期望」 |

### 三、原本「三条未核实」的最终结论

| 原存疑项 | 结论 | 依据 |
|---|---|---|
| `2.0.2` 的字段名是否与本文档（基于 `main`）一致 | **不一致，差异很大**，已完成「去漂移」全部改写（见上表） | 2.0.2 sources jar |
| `LocalPeriodicGate` 是否纯进程内、有无 SPI | **该符号在 `2.0.2` 不存在**。节流是 `static` 表 + CAS，**确实只在单 JVM 内**，且框架没留 SPI 扩展点——多副本要真限流必须自己换实现 | `MemoryFlushMiddleware.java:96-97, 237-264` |
| **core 的裸 `ReActAgent` 有没有可插入 flush 的等价扩展点** | **有，而且五个全有。** `io.agentscope.core.middleware.MiddlewareBase` 就在 **core** 里，声明 5 个 hook（`onAgent` / `onReasoning` / `onActing` / `onModelCall` / `onSystemPrompt`），`ReActAgent.Builder` 上有 `middleware(...)`。**但 core 的 `middleware/` 目录里唯一的实现是 `TaskReminderMiddleware`——记忆实现一个都没有，全在 harness 侧** | `core/io/agentscope/core/middleware/MiddlewareBase.java`；`core/io/agentscope/core/ReActAgent.java` |

→ **面试口径（这条最实用）**：不要说「core 没这能力」。要说「**挂载点 core 全给了，但框架在 core 里不带任何记忆实现；带记忆的是 `agentscope-harness`。我们只依赖 core，所以那三条链自己写。**」

**顺带解决洋葱顺序存疑点**：`order()` 默认 `1`，**数值越大越靠外层**，**同值按 Builder 注册顺序**；harness 里没有任何记忆中间件重写 `order()`。所以 `:2278 → :2296 → :2311` **就是真实洋葱顺序**，可以放心讲。

### 四、拆成两份文件 + 面试稿精简（本次）

**背景**：原单文件 1596 行，其中第 6 层占 901 行（56%），「背」和「查」两种用途挤在一起互相拖累。按用途拆开：

| 文件 | 定位 | 内容 |
|---|---|---|
| **`AgentScope记忆机制怎么做.md`**（面试稿） | **只背这一份** | 0 电梯版 → 第 1–5 层 → 第 6 层精简版（6.0 口径修正表 + 6.1 四个锚点）→ 附录 B 踩坑表 → 附录 C 反问。**579 行** |
| **本文（源码笔记）** | **被追问时翻** | 第 6 层完整版（901 行，含**图 1–7**）+ 附录 A 四张速查表 + 本修订记录。**1070 行** |

**面试稿精简按「A–D 无损」档执行**——删重复内容、示例配置代码块、考察点标签行、多余分隔线。**内容无丢失，但并非「话术一字未动」：前五层有 5 处引用块是「合并 / 改写」而非纯删（下表），背诵前这 5 处要按新措辞重读一遍。**

| 位置 | 动作 | 与原版的差异 |
|---|---|---|
| Q1.4 | 扩写 | 「两层读机制」块补进「读取先问 filesystem、没有再回落本地磁盘」，原版无此句 |
| Q2.2 | 合并 + 补写 | 并入原 Q2.4 全部内容，并加「这两个开关在 2.0.2 默认就是 `true`」 |
| Q3.2 | **净新增** | 补 Redis extension 警告（见下方 A 类说明），原版此处是代码块 |
| Q4.1 | **改写措辞** | 5 条代价压成 3 条，条目合并、句子重写 |
| Q5.3 | **改写措辞** | 原 1 句 + 3 bullet 压成 1 段，句子重写 |

**唯一「纯删除」的是原 Q2.4 整题**（3 行「为了让压缩可以做得比较激进…」），内容已并入 Q2.2，其余删项均为代码块 / 标签行 / 分隔线。

四个类别的具体动作：

| 类 | 动作 | 例 |
|---|---|---|
| **A｜重复的示例配置** | 删代码块，改为「默认值见 附录 A」的口径提醒 | `HarnessAgent.builder()` 24 行 mega-config、`.compaction()`、`.truncateArgs()` |
| **B｜标签行 / 分隔线** | 删掉每段话术前的 `**答题话术**：` + 段间 `---` | 共删 14 条段内分隔线 |
| **C｜前后重复的论证** | 后出现的压成一句 + 指回前面的题号 | 原 Q2.4 整题并入 Q2.2；Q5.3 三条问答压成一段 |
| **D｜交叉引用** | 修正指向第 6 层的失效编号 | L119 的 `6.2 锚点 1` → `6.1 锚点 A` |

**⚠️ 编号变动**：原 **Q2.4** 已被并入 Q2.2 删除，**原 Q2.5 顺位改为 Q2.4**（「为什么用 Markdown 文件做记忆，而不是向量库？」）。任何外部笔记若引用「Q2.5」，现在指错了。

**A 类顺带修掉的一处口径风险**：删 `RedisDistributedStore` / `RedisAgentStateStore` 代码块时，在 Q3.2 原位补了提醒——这两个类**既不在 `agentscope-core` 也不在 `agentscope-harness`**，`DistributedStore.java:61` 的 javadoc 自己写明属于 **`agentscope-extensions-redis`**。也就是「上分布式」要额外引一个 extension 包，不是换个 builder 参数的事；Ragent 没有这个依赖，别说成「我们配一下就行」。

---


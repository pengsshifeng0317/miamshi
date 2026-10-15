# Ragent 升级清单（RAG → Agentic RAG）

> **这份文档是干什么的**：记录 Ragent 项目从「RAG 问答系统」升级到「Agentic RAG 平台」过程中的所有升级项。以后要复习或追问，直接把本文件内容（或路径）给 AI，说一句「基于这份升级清单，帮我深入讲 XX」即可。
>
> **项目根目录（git 根）**：`D:\coding_location\java\zhishixingqiu\nageoffer_\ragent\ragent`
>
> **一句话定位**：Ragent 是一个基于 Spring Boot 4 + Java 17 的生产级 Agentic RAG 平台，用 AgentScope 2.0.2 把 RAG 能力工具化，交给 ReAct Agent 编排调用，叠加 MCP / 长期记忆 / 写操作确认 / Skills / 模型熔断降级 / 链路追踪。
>
> **本次更新**：补入 2026-09-05 → 09-20 的 35 个提交；重写「六、MCP 工具集成」；新增「七、链路追踪」「八、提示词槽位」「九、配套工程」三章；修正 4 处失效锚点。
>
> **档位前提**：本清单除注明外，均以 `ragent.engine.type=agent`（v2 ReAct 档）为前提。详见「二、架构层升级 · 3. 双档位」。

---

## 一、升级总览

**升级本质**：旧版是「工程师写死的检索管线」（检索 → 融合 → 生成），新版是「模型驱动的工具编排运行时」——Agent 引擎负责调度，中间件负责上下文治理，记忆/确认/技能/MCP 是挂在这个运行时上的四类能力，追踪贯穿全链。

### 关键提交时间线

| 提交 | 日期 | 动作 | 意义 |
|------|------|------|------|
| `17eaaa6f` | 08-04 | 新增智能体管理 | Agent 管理后台 |
| `020e5c3d` | 08-12 | 拆分模块并新增 Agent 执行架构 | 单体 → 7 模块，引入 Agent |
| `7f3a7f38` | 08-12 | 升级 Spring Boot 4 | Boot 3 → 4.1.0，Jackson 2→3 迁移 |
| `1cb684ed` | 08-30 | 实现 Agent 长期记忆 + 调试工具 | memory 子系统 + 回归验证台 |
| `24e6b4a4` | 09-02 | 写操作确认流程 | 工具执行前人工确认 |
| `45ec387e` | 09-02 | 新增 MCP 工具 | MCP 工具生态 |
| `c71f0750` | 09-03 | 集成 Skills | 技能手册 + 工具遮蔽 |
| `d0b744ea` | 09-05 | 重构 Agent 仪表盘 | 多引擎支持 |
| `05f045ff` | 09-06 | 中断提示独立 error 块 | 取消的流式表达 |
| `96c528d0` | 09-10 | LangFuse v4 自托管编排 | 追踪基建 |
| `177108bb` | 09-10 | **接入 LangFuse 链路追踪** | trace 子系统 |
| `27366d12` | 09-11 | **新增 DeepSeek 供应商** | 模型层扩展 |
| `745d8bbd` | 09-13 | 按编排模式控制 MCP 工具执行 | 双档位分流（WORKFLOW 才走 MCP） |
| `da32db76` | 09-14 | **MCP `_meta` 透传登录身份** | 工具侧身份 + 回环绑定 |
| `cdace75a` | 09-14 | Agentic v2 比特严选场景示例 | 完整示例域 |
| `5f3c35e3` | 09-15 | 优化 ReActAgent 缓存驱逐 | 状态缓存 |
| `e1ba294e` | 09-16 | 意图树按档位筛选叶子节点 | 双档位分流 |
| `67624931` | 09-17 | **完善任务取消支持** | TaskCancellation |
| `7c418b8d` | 09-17 | 工具目录提示词快照 | 目录重构 |
| `7e522563` | 09-18 | AgentScope 原生 Skill 对照测试 | 框架缺陷复现 |
| `fd58ec76` | 09-20 | **重构 MCP 集成，删除 v2 WorkFlow MCP** | rag 侧 MCP 与旧意图树工厂一并清零 |
| `d0146b5e` | 09-20 | 简化技能加载工具 | 移除展示名映射 |

### 升级维度总表

| 维度 | 旧版（RAG） | 新版（Agentic RAG） |
|------|------------|---------------------|
| 编排方式 | 固定检索管线 | 模型自主调工具（ReAct）；检索管线降级为一个 Tool |
| 档位 | 单一链路 | 部署级双档位 `workflow` / `agent` |
| 工具接入 | 应用内硬编码函数 + 应用内 MCP 执行链 | MCP 协议，连接/发现/Schema 全交给 AgentScope |
| 会话记忆 | 最近 N 轮拼接 | 三层记忆（短期裁剪 / 中期压缩 / 长期跨会话） |
| 安全 | 无 | 写操作人工确认 + 身份 `_meta` 透传 + 回环绑定 + 边界声明 |
| 扩展 | 流程写死 | Skills 渐进式披露 |
| 可观测 | 无 | OTel → LangFuse 全链追踪 + 契约版本 |
| 提示词 | 硬编码模板 | 槽位枚举 + 后台可配 + 快照缓存 |
| 取消语义 | 无 | 中断标记 + 异常链判定，贯穿模型/工具/流式/前端 |
| 框架 | Spring Boot 3 | Spring Boot 4 + AgentScope 2.0.2 |

---

## 二、架构层升级

### 1. 模块化拆分（提交 `020e5c3d`）

单体 `bootstrap` 拆成 7 个 Maven 模块：

| 模块 | 职责 |
|------|------|
| `framework` | 统一响应/异常、认证上下文、幂等、分布式 ID、MQ、Trace、SSE、**任务取消** |
| `infra-ai` | Chat/Embedding/Rerank/VLM 客户端、模型档位、路由、熔断、首包探测 |
| `system` | 用户认证、审计日志 |
| `rag` | RAG 问答、知识库、入库 Pipeline、意图树、检索、会话、**提示词槽位** |
| `agent` | Agent 执行架构（ReAct v2）、记忆、技能、确认、工具、**追踪**、**仪表盘** |
| `bootstrap` | 启动装配 |
| `mcp-server` | 独立 MCP 工具服务（含 enterprise / bit 两个示例域） |

**分层核心思想**：把「业务编排 / AI 供应商差异 / 通用基础设施」隔离开，切模型/向量库/对象存储时，问答主流程不重写。

### 2. Spring Boot 4 升级（提交 `7f3a7f38`）

- Boot 3 → 4.1.0，Java 17
- Jackson 2 → 3 迁移，用 `spring-boot-jackson2` 兼容模块过渡
- MyBatis-Plus 用 `mybatis-plus-spring-boot4-starter`

### 3. 双档位：`workflow` / `agent`（**新增**）

**核心文件**：`rag/.../config/OrchestrationMode.java`、`OrchestrationProperties.java`

由 `ragent.engine.type` 指定，大小写不敏感，无法识别时回落 `WORKFLOW`：

| 档位 | 含义 |
|------|------|
| `WORKFLOW` | v1 编排管线：意图分类 → 检索 → 合成，链路确定、延迟低 |
| `AGENT` | v2 ReAct 架构：主 Agent 决策，**RAG 管线降级为其中一个 Tool** |

**两个值得讲的设计决策**：

1. **是部署级决策，不开放后台切换**——枚举注释原文：*「属部署级决策（切换需重启，且 AGENT 依赖外部 ReAct 服务存活），因此不开放后台切换」*。
2. **配置类不挂在 `rag` 前缀下**——`OrchestrationProperties` 注释原文：*「不挂在 rag 下：AGENT 档里 RAG 管线只是主 Agent 的一个 Tool，档位在语义上包含 RAG 而非从属于它」*。

**档位如何渗透到各子系统**：

| 子系统 | 分流方式 |
|--------|----------|
| 意图识别 | `DefaultIntentClassifier`：Agent 档只识别 KB 叶子节点；WorkFlow 档额外保留 SYSTEM 应答节点（`e1ba294e`）；候选为空时跳过 LLM 调用直接返回空意图 |
| 提示词 | `AgentPromptSlot` 每个槽位声明自己的 `Set<OrchestrationMode>` 生效范围 |
| MCP 工具 | `745d8bbd` 先做档位分流，`fd58ec76` 直接把 WORKFLOW 那条链删掉——**WorkFlow 档现在根本不连 MCP**（见第六章） |
| 记忆/追踪 | `@ConditionalOnAgentEngine` 条件装配，WorkFlow 档下整套 bean 不加载 |
| 装配自证 | `AgentMcpClientsModeTest`：`engine.type=agent` → `hasSingleBean(AgentMcpClients.class)`；`workflow` → `doesNotHaveBean(...)` |

---

## 三、Agent 执行引擎（最核心升级）

### 引擎装配 —— `agent/.../config/ReActAgentProvider.java`（157 行）

- 用 **AgentScope 2.0.2** 的 `ReActAgent`（builder 装配）
- **单例复用 + 懒重建**：以「人设内容（`AGENT_MAIN` 槽位） + 工具目录指纹（`ResolvedCatalog.fingerprint()`）」比对决定是否重建，控制台改配置无需重启
- **旧实例不主动 close**：在途会话仍在其上流式输出，交由 GC 回收
- 会话状态存 PostgreSQL（`PgAgentStateStore`），按次加载，重建不丢历史
- `evictStateCache(userId, sessionId)` 只作用于已构建实例——*「清理动作不该顺手把 Agent 建起来」*；仅本节点有效，多节点各持一份缓存

### 七层 Middleware 链（洋葱模型，从外到内）—— **已从 4 层扩到 7 层**

```java
otelTracingMiddleware        // 最外层：span 要盖住记忆与压缩才量得到耗时
traceEnrichmentMiddleware    // 紧贴官方中间件内层，补业务属性
userMemoryMiddleware         // 记忆块插在人设与会话之间，压缩的 offset 比对自然吸收它
contextCompactionMiddleware  // 裁剪/压缩上下文
confirmDenialMiddleware      // 压缩之后：被压进摘要的拒绝结果已不在列表里，改写自然跳过
skillMaskingMiddleware       // 手册被压缩带走后遮蔽跟着复位，工具与手册同进同出
toolBatchMiddleware          // 最内层：批起止贴着真实执行，不随追踪开关变形
```

前两层用 `ObjectProvider.ifAvailable(...)` 条件装配，追踪关闭时不进链。

**顺序不是随手排的**——代码里每条都有理由（见上方注释）。这是很好的面试展开点：洋葱模型的层序由「谁来吸收谁的副作用」决定。

### 请求生命周期 —— `agent/.../service/impl/AgentChatServiceImpl.java`（385 行）

- 两条入口：`streamChat`（提问）、`confirmPendingTool`（确认卡片）
- `launchStream` 里 `agent.streamEvents(input, RuntimeContext)` 返回 `Flux<AgentEvent>`，经 `AgentStreamEventBridge` 转 SSE
- 整个 Agent 执行是**响应式**的（Project Reactor）
- 构建 `RuntimeContext` 时把身份塞进去：`.put(McpMeta.class, new McpMeta(AgentMcpMeta.ofUser(scope.userId())))`（`fd58ec76`）

### 生命周期句柄 —— `service/handler/AgentRunHandle.java`（295 行）

- `complete / cancel / fail` 三条出口 **CAS 互斥**，收尾体只跑一次
- **优雅中断**：`interruptUpstream()` 先 `agent.interrupt()` 等框架存盘（`GRACEFUL_INTERRUPT_WAIT_MS = 2000`），超时才 `dispose()` 断流，否则丢本轮状态
- 释放钩子队列保证每个钩子恰好执行一次

### 用户级并发闸门 —— `service/handler/AgentRunGate.java`（99 行）

- 一个用户同一时刻只跑一条流，Redis `RBucket.setIfAbsent` 抢占
- TTL = SSE 超时 × 2；`release` 用 `compareAndSet` 只放自己占的位（防误删别人的闸门）

### RAG 能力工具化 —— `agent/.../tool/KnowledgeSearchTool.java`（149 行）

- 整个 RAG 检索管线封装成 `search_knowledge` 工具（Agent 模式下 RAG 唯一入口）
- **工具描述由提示词槽位提供**：构造时注入 `description`（来自 `KNOWLEDGE_TOOL_DESCRIPTION`），改描述无需重建 Agent
- 走 `KnowledgeSearchFacade`，带改写兜底：取最近 2 轮做指代消解

### 任务取消与中断语义（`67624931`、`05f045ff`）—— **全新维度**

**框架层原语** —— `framework/.../cancellation/TaskCancellation.java`（70 行）

> 类注释原文：*判定一次失败是不是用户取消，只服务于不该把取消记成故障的出口。依据是中断标记与异常链：用户停止后运行句柄 dispose 上游，Reactor 随之中断执行线程。*

| 方法 | 用途 |
|------|------|
| `isCancelled()` | 下游**降级返回而非抛出**时，出口只剩中断标记可判 |
| `isCancellation(Throwable)` | 中断标记 + 沿 cause 链走查 `InterruptedException` / `CancellationException` |
| `asCancellation(Throwable)` | 还原成取消异常，**并补回被 `InterruptedException` 清掉的中断标记** |

最后一条是精华：`InterruptedException` 抛出时 JVM 会清中断标记，不补回来上游判据就失灵。

**与 `AgentRunHandle` 的关系是「补充」而非「替代」**：前者管「停止的编排」（顺序、存盘、释放），后者管「取消的识别」（中断标记 + 异常链）。`interruptUpstream()` 打标记，`TaskCancellation` 被三处复用。

**三处消费者**：

| 位置 | 处理 |
|------|------|
| `KnowledgeSearchTool`（+22） | ① 正常路径成功返回后**再判一次** `isCancelled()`——注释：*「RAG 逐层降级，取消到这里多半不是异常而是一份空结果，只在 catch 里判会漏掉」*；② catch 里 `isCancellation(e)` → `INTERRUPTED`；③ 反馈文案 `INTERRUPTED_MESSAGE = "用户已停止，本次知识检索未完成"`。核心口径：**取消不能报成 ERROR——模型见到失败会在下一轮重试检索，而用户已经喊停** |
| `McpToolProxy` | `isCancellation(e)` → `INTERRUPTED`，其余异常 → `CALL_FAILED_MESSAGE` |
| `ModelRoutingExecutor`（+9） | catch 里取消 → **不降级、不累计失败**，只 `healthStore.releaseHalfOpenPermit(permit)` 归还半开探测名额，再 `throw asCancellation(e)` |

**追踪侧对齐**：`AgentToolBodyTracer.settleFinished()` 在 `INTERRUPTED` 时用 `facts.markTerminated(toolCallId)` 对齐中断时刻，span 置 `StatusCode.ERROR` + `AgentErrorTypes.ofCancelledSpan(...)`——**span status 只有 OK/ERROR，分不出「被停」与「真报错」，所以结局另走属性**。

**流式侧：中断提示必须独立成块**（`05f045ff`，`AgentStreamEventBridge` +51）

- 新增 `DELTA_TYPE_ERROR = "error"` 与 `TextChannel.ERROR("error", AgentBlock.KIND_ERROR)`
- `onError()` 先判 `runHandle.isCancelled()`：取消导致的信号中断**不算错**，交 `finishCancelledStream`；否则单开 error 块 + 发增量 + `settleAndPersistMessage(content, INTERRUPTED)` + 发 `FINISH` 与 `DONE`（**改发 `finish` 而非 `fail` 关流**——*「finish 让前端把已流出的内容与工具块定型，口径与落库一致」*）
- 文案统一：`NOTICE_INTERRUPTED = "回复到这里中断了。如果上面有提交类操作，请先到对应业务系统核对是否生效，确认没生效再重新提问"`
- **为什么单独成块**（注释原文）：*「这是系统在说话，混进 answer 就跟模型的回答一个身份了；块和当场增量都要发，只塞进 content 的话历史回放有块就不读 content，刷新后这句就没了」*

**前端三文件配合**：

| 文件 | 改动与理由 |
|------|-----------|
| `useAgentStream.ts` | **删掉 `streamWithRetry`**（这就是「不是有重试吗」的正面回答）——*「只发一次，失败就交给上层：Agent 一轮里可能已经执行过写操作……重发就意味着整轮重跑，写操作再执行一遍，**与 HITL 的不重复提交直接冲突**」*；`readSseStream` 加 `terminated` 标志，没收到 `done` 就是连接被掐断，主动取消由上层收尾不算异常 |
| `agentChatStore.ts` | `appendText` kind 扩展 `"error"`；finish 里 `INTERRUPTED` 时兜底状态从 `awaiting` 改 `interrupted`——*「不能说未执行：那比已中断更强，且与后端落库相反，**会诱导用户再提交一次**」* |
| `AgentChatInput.tsx` | 停止中按钮 `Loader2` + `aria-busy` + `data-stopping` + `disabled`；免责行 `aria-live="polite"` 显示「正在停止并保存已生成内容…」；store 的 `requestStop(taskId)` 失败时只在流状态与 taskId 双双一致时复位，否则 `toast.error` |

**端到端链路**：点停止 → 前端**不中断 fetch**、置 `cancelRequested` + `requestStop` → 后端 `interruptUpstream()` 优雅打断 + 落库 → 流以 `cancel/finish + done` 收尾（异常则走 error 块 + `finish + done`）→ 前端渲染「已中断」中间态与独立提示块，**刷新后与落库口径一致**。

---

## 四、长期记忆系统（全新子系统）

### 注入 + 抽取双线

- **注入线（读）**：`AgentUserMemoryMiddleware` 每轮推理前，把跨会话事实块插在「人设之后、会话首条之前」
- **抽取线（写）**：轮次结束后异步跑 `AgentMemoryPipeline.extract()`

### 抽取 Pipeline 并发控制 —— `memory/AgentMemoryPipeline.java`

```
ensureBaseline(预建控制行=抽取下界) → loadPending(水位线后的新消息)
  → claim(抢占抽取权，靠唯一索引仲裁) → judge(LLM仲裁，事务外) → commit(短事务 + revision/水位双校验)
```

- baseline 必须在首条消息落库前建立，否则本轮消息永久漏抽
- 仲裁在**事务外**跑（慢 LLM 调用不占行锁），提交才进短事务

### LLM 一次调用完成「抽取 + 取舍」—— `memory/AgentMemoryJudge.java`

- LLM 输出**四态**：`NOOP`（不记，解析为 `null` 不入枚举）/ `ADD`（新增）/ `SUPERSEDE`（取代，靠 id 指认）/ `RETRACT`（撤销）；枚举 `AgentMemoryDecision.Action` 只含后三者
- **反 prompt 注入**：素材用 `<recent_turns nonce="随机串">` / `<existing_memories nonce="随机串">` 围栏包裹，用户消息里的围栏标签被 `neutralize` 中和

### 上下文裁剪与压缩（两级水位）—— `memory/AgentContextCompactionMiddleware.java`

- **50% 水位**：`AgentContextTrimmer` 把过老工具结果换成等长占位（保留原入参）
- **80% 水位**：`AgentContextCompactor` 触发摘要压缩
- 裁剪按「工具循环」切分，保护最近 N 轮 + 未闭合循环；**只换 tool_result 不动 tool_use**（不产生孤儿结果块）
- 裁剪白名单：`agent.memory.evictable-tools`，默认 `[search_knowledge]`

### 记忆受限合并 —— `memory/AgentMemoryConsolidator.java`（188 行）

容量顶到上限时的取舍策略；仅当记忆总量顶到上限时才调用一次，产物不进对话，按 JSON 数组整组替换。对应提示词槽位 `AGENT_MEMORY_CONSOLIDATION`。**这是原清单第八节列的「未展开深挖点 ③」，现已实现。**

### 记忆回归验证台（**新增**）—— `resources/regression/`

> 定位原文：*对一个**已经跑起来的**环境做行为验证：登录、按剧本串行提问、读库看状态、给判定。它不写任何业务数据，也不删任何东西，跑完的会话留在库里，随时可以再查。*

与 `resources/initializer` 是同一套 CLI（Java 17、不启动 Spring、复用 `common/` 的 HTTP 与 JDBC 客户端），分工是：**初始化器负责造出确定状态，回归台负责验证运行时行为**。

| 层 | 机制 | 剧本怎么验 | 现状 |
| --- | --- | --- | --- |
| 短期 | 工具结果清理（`AgentContextTrimmer`） | t01 植入知识库里查不到的锚点，t02~t07 把上下文撑大到触发清理，t08 要求原样复述锚点 | 已实现，硬断言 |
| 中期 | 安全前缀压缩 + 会话摘要（`AgentContextCompactor`） | t09 问只在 t02 说过一次的工位号；t14 问 t10 更正过的资产编号。两轮都**不许重新检索** | 已实现，硬断言 |
| 长期 | 跨会话用户事实（`AgentUserMemoryMiddleware` + 抽取管道） | t15 另开一条会话直接问工号 | 已实现，硬断言 |

**判定五态**：`PASS` / `FAIL`（非零退出）/ `PENDING`（该层未实现，答不出是正确行为）/ `UNEXPECTED`（未实现却命中，需人复核）/ `UNCOVERED`（本次没跑到该分支）。

**演进机制**：往后新增记忆层时，先在 `tier` 上写新值再实现，答不出只记 PENDING，实现之后加进 `MemoryTurnScript.IMPLEMENTED_TIERS` 就**自动转成硬断言**——这套剧本同时就是每一层的验收用例。

**两条中期断言的论证（很值得讲）**：共同前提是**被问的事实只说过一次、之后再没被复述**，所以一旦被压缩吃掉，答对就只可能来自摘要。

- **t09（保真）**：工位号 `B3-1207` 只在 t02 出现过，知识库里查不到、问题里也没带。答对说明用户原话被摘要 **verbatim** 带过来了。同轮 `forbid-tool`：又调一次检索就是 FAIL。
- **t14（取新）**：t01 说资产编号 `NB-2049`，t10 更正为 `NB-3160`。答旧值即 FAIL——说明第一代摘要里的旧值被逐代抄了下来。

> 反例（原清单之外的教训）：本剧本早先那版 t09 用的是 `归还|资产|交接` 三个关键词，**问题里就带着**，模型凭空编一句也能 PASS。关键词式断言只在上面的形态下才站得住。

**用实测数据推翻了一个假设（最有含金量的一段）**：

> 原文结论：「多数会话靠裁剪就把水位摁住」**实测不成立，上界只有压缩给得了**。裁剪只改写 `tool_result`，射程等于它在上下文里的占比，而那是**语料属性不是常量**。

08-25 关掉压缩跑满 15 轮，末轮 30575 字符的构成：

| 块类型 | 字符 | 占比 | 裁剪能动吗 |
| --- | --- | --- | --- |
| `text`（用户话 + 助手回答） | 13823 | 45.2% | ✗ |
| `thinking` | 8039 | 26.3% | ✗ 明令永不清理 |
| `tool_result` | 6851 | 22.4% | ✓ 但 11/26 块已是占位 |
| `tool_use` | 1862 | 6.1% | ✗ |

- 射程 **22.4%** 对下限 **20%**（占**当前总量**，分母还一直涨）——下限顶在射程天花板上，所以裁剪在 t04/t06 各成功一次之后**连续九轮空转**，上下文一路涨到预算的 **3.8 倍**。光 `thinking` 一项就比整个可裁剪面还大。
- **反证「压缩永远等不到」**：8000 预算、白名单原样留着跑满 15 轮，压缩照样发生 **4 次**，裁剪只成功 1 次。原因在 `KEEP_RECENT_CYCLES = 2`——每轮只有两三个循环，最近两个一保护，裁剪够不着「回收量 ≥ 当前总量 20%」那道下限。
- `summary-enabled: false` 是摘要模型不可用时的运维出口，但**不是有界的降级**：关掉之后没有任何东西摁得住水位，一直涨到模型真实窗口 overflow，而那一轮整个不落库。

**实测画像（本机 12 条会话、16 轮、25 次模型调用）**：单轮上下文增量峰值 8,810 字符；单次 ReAct 循环增量峰值 5,794 字符 / 3,667 token；单块 `tool_result` max 3,271 / p90 2,781 / 均值 725；固定开销（人设 + 工具 schema）2,855 token；约 1.6 字符/token；按峰值循环 × `max-iters: 10` 外推单轮上界约 **5.8 万字符**——**这条外推只对 `search_knowledge` 成立**，因为检索结果是再合成一次的答案，原始 chunk 进不了 Agent 上下文。

**摘要长度跨四代增长**：638 → 778 → 1106 → 1285 字符。

**机制回归**（另一个会写库的 CLI，`AgentMemoryMechanismMain --case all`）：验剧本读不出来的管道行为——M1/M2 并发下整批拒不拒、M3/M4 flush 与水位、M6 撤回后不注入、M7 取代目标失效只丢一条、M8 抽取下界两面、R1~R5 跨会话可见/撤回不复活/改口取新/水位不重不漏/容量上界。

**探针**：`AgentStateProbe`（纯只读读 `t_agent_state`，SQL 侧按 `AgentContextChars` 口径复算字符数、逐条 usage 是唯一权威 token 读数）+ `AgentMemoryProbe`（`t_agent_memory` 三表观测口 + 机制回归造现场用的写入口，伪造数据按 `99` 前缀物理删干净）。配套 `--session <会话ID>` 单会话排查 CLI，可查线上「为什么忘了某件事」。

---

## 五、写操作确认 + Skills（安全与扩展）

### 两段式写操作确认

1. **工具声明是否需确认**：`McpToolProxy.checkPermissions()` 返回 `ask`/`allow`
2. **确认后防篡改**：`AgentChatServiceImpl.resolveConfirmResults()` 从 Agent state 取待确认工具，前端只传 `approved` 布尔

**⚠️ `needsConfirm` 判定在 `fd58ec76` 里被收紧了**（收进 `AgentToolCatalog.McpToolBinding.of()`）：

```java
// 旧：null 视为「不需要确认」
needsConfirm = requireConfirm || Boolean.FALSE.equals(readOnlyHint);
// 新：null 视为「需要确认」，更保守
needsConfirm = confirmConfigured || !Boolean.TRUE.equals(readOnlyHint);
```

**默认值方向的改变是个好的安全叙事点**：服务端没声明 `readOnlyHint` 时，旧逻辑放行、新逻辑要确认——**把「未声明」当成不可信**。

### 拒绝口径修正 —— `confirm/AgentConfirmDenialMiddleware.java`

- 用户点「取消」后，框架默认塞 `"Permission denied by user"`，会被模型误判为权限不足
- 中间件改写成友好解释：「用户在确认卡片上点了取消，不是权限不足也不是故障」
- 只改用户取消（`DENIED + 该文案`），规则拒绝不动

### Skills 渐进式披露 —— `skill/AgentSkillMaskingMiddleware.java`

- 集合运算模型：`本轮遮蔽 = G − L`（G = 所有启用技能 tool_ids 并集，L = 已加载技能 tool_ids 并集）
- 手册加载前，易出错工具不出现在模型可见工具列表里，逼模型先 `load_skill`
- 手册被压缩带走时，工具一并收回（**工具与手册同生共死**）
- 遮蔽结论寄存在 `RuntimeContext`（`MASKED_TOOLS_ATTRIBUTE`）；**`McpToolProxy.callAsync()` 执行前复查同一份结论**（遮蔽是「列表隐藏 + 执行短路」两层，不只是不给看）；计算异常时**本轮按全量工具推理**（不阻塞主流程）

### 技能加载工具 —— `skill/SkillLoadTool.java`

- 技能清单挂在工具 `description` 上，正文等模型判断命中后再取
- 改手册无需重建 Agent
- `d0146b5e` 移除了展示名映射，简化为直接取正文

### 技能手册的双文件形态 —— `resources/initializer/bit-selection/skills/`

```
010-after-sale-apply.properties   # 元数据：skill-code / name / description / content-file / tool-ids / sort-order / enabled
after-sale-apply.md               # 正文：什么时候用 / 办事步骤 / 这几项怎么填
```

- 元数据里写清**什么时候该加载、什么时候不该**——例：`tool-ids=apply_after_sale`，注释说明*「query_order、query_after_sale 看自己的参数说明就能调对，用户单独问『我那张售后单到哪一步了』时不该先加载一份办事手册」*
- 正文是给模型的办事流程，含分流规则（如按签收天数分档、超过 15 天转保修）

### 与 AgentScope 原生 Skill 的对照（**新增，重要**）—— `agent/src/test/.../skill/`

生产路径仍是自研遮蔽中间件，但新增了两个**对照框架原生 Skill 机制**的测试（`7e522563`）：

- **`AgentScopeToolGroupConcurrencyTest`** — 复现 **AgentScope #2439**：*「单例 Agent 并发处理不同会话时，共享 Toolkit 的工具组状态会串话」*。类注释明确：**这是缺陷复现测试，断言的是当前框架的错误行为**；上游修复后工具结果应改为 SUCCESS，本测试也应随之删除或改成回归断言。
- **`AgentScopeSkillActivationLifecycleTest`** — 复现原生 Skill 的激活周期：*「工具组状态属于会话状态，**不随模型可见上下文一起压缩**」*（测试名 `shouldKeepSkillToolActiveAfterLoadResultIsCompactedOut`）。

**第二条正好解释了自研的理由**：本项目要的语义是「手册被压缩带走时工具一并收回」，而原生 Skill 的工具组状态不跟着压缩回收——**两者的生命周期根本不同**。第一条则是需要在面试里主动交待的风险面（见下）。

> ⚠️ **待你确认的口径**：`AgentRunGate` 是**按用户**闸的（一个用户同时只跑一条流），而 `#2439` 说的是**单例 Agent 被不同会话并发使用**时串话——**不同用户之间并不互斥**。这是已知风险、还是另有兜底？写进逐字稿前需要你定口径。

---

## 六、MCP 工具集成（**整章重写**）

> **为什么重写**：原清单的 `McpToolBridge.java` 已在 `fd58ec76` 删除（191 行），`rag/.../core/mcp/` **整个包（10 个文件）已消失**，连带 `IntentTreeFactory`（457 行）等旧意图链路文件。旧描述全部失效。

### 删除前的双路径

旧架构里 MCP 有**两条路径**：

| 路径 | 位置 | 谁在用 |
|------|------|--------|
| Agent 侧桥接 | `agent/.../tool/McpToolBridge.java`（手写 `extends ToolBase`，自带 `buildInputSchema()` / `resolveReadOnlyHint()`） | Agent 档 |
| RAG/WorkFlow 侧执行链 | `rag/.../core/mcp/`：`McpClientToolExecutor`(84) / `McpToolExecutor`(62) / `McpToolRegistry`(80) / `DefaultMcpToolRegistry`(123) / `McpClientAutoConfiguration`(108) / **`LLMMcpParameterExtractor`(415)** / `McpParameterExtractor`(47) / `McpExtractionResult`(54) / `McpCallMeta`(50) / `McpClientProperties` | WorkFlow 档 |

**两条路径的存在理由**：`AgentEngineConfiguration` 里曾有注释 *「MCP 不走 AgentScope 自带客户端（其 MCP SDK 0.17.0 被根 pom 的 1.1.2 压制），统一桥接 rag 既有连接」*——**SDK 版本冲突是旧桥的直接成因**。

**`fd58ec76` 把两条路径收敛成一条**：`pom.xml` 与 `mcp-server/pom.xml` 同步对齐 MCP SDK 版本，绕行路径不再需要，于是 rag 侧整包清零。`RetrievalEngine` 净删 **202 行**（`executeMcpAndMerge`、`mcpIntents` 及相关分支全移除，当前文件已无任何 `mcp` 引用）——**WorkFlow 档现在彻底不连 MCP**。

> **演进叙事（很值钱）**：`745d8bbd` 先做成「按档位分流：WORKFLOW 走 MCP、AGENT 跳过」，注释理由 *「AGENT 档下 MCP 工具的正当入口只有 Agent 自己的工具清单，那条路上有确认卡、技能遮蔽、readOnlyHint」*；`fd58ec76` 进一步发现 WorkFlow 那条提参同步调用链（LLM 提参三态 + 同步调用 + 拼检索上下文）在 Agentic 架构下**已无存在意义**，连它一起删了。**这是「先隔离、再删除」的典型收敛过程。**

### 现在的新架构（4 个类，全在 `agent/.../tool/`）

| 类 | 行数 | 职责 |
|----|------|------|
| `McpToolProxy` | 111 | *代理远程 MCP 工具调用，复用 AgentScope 的连接、Schema 与结果转换，并补充本项目的执行规则*。**`extends McpTool`（AgentScope 自带基类）**，只覆写 `checkPermissions()` 与 `callAsync()` |
| `AgentMcpClients` | 112 | *Agent 模式的 MCP 连接和工具发现由 AgentScope 持有*。唯一持有 AgentScope MCP 客户端的地方：`@PostConstruct init()` 遍历 `servers` → `McpClientBuilder.create(name).streamableHttpTransport(url).buildSync()` → `initialize()` → `listTools()`；存 `Map<String, RemoteTool>`（重名保留先连接者并 warn）；`@PreDestroy close()` |
| `AgentMcpMeta` | 43 | 身份元数据。由 rag 侧 `McpCallMeta` **重命名**而来（删掉 `userIdOf()` 与 `trim()`）。**`USER_ID_KEY = "com.nageoffer.ragent/userId"`** |
| `AgentMcpProperties` | 51 | 由 `McpClientProperties` **重命名**；配置键从 `rag.mcp.servers` 迁到 **`agent.mcp.servers`** |

**升级要点**：从「手写 `ToolBase` 适配 + 自己管 MCP 客户端 + 自己把 `JsonSchema` 转 Map」改为「**继承框架的 `McpTool` + 把连接、发现、Schema 转换全交给 AgentScope**」。`buildInputSchema()` / `resolveReadOnlyHint()` **两个方法已不存在**，其逻辑收进 `AgentToolCatalog.McpToolBinding.of()`：

```java
Boolean readOnlyHint = annotations == null ? null : annotations.readOnlyHint();
... McpTool.convertMcpSchemaToParameters(definition.inputSchema(), Set.of()),  // 替代 buildInputSchema
Boolean.TRUE.equals(readOnlyHint),                                              // 替代 resolveReadOnlyHint
confirmConfigured || !Boolean.TRUE.equals(readOnlyHint),                        // needsConfirm
```

**协作链**：`AgentMcpClients` 发现 `RemoteTool` → `AgentToolCatalog.resolveMcpTools()` 意图树与 MCP 注册表取交集 → `McpToolBinding.of()` 算 schema / readOnly / needsConfirm → `buildToolkit()` 注册 `McpToolProxy(binding)` → 代理持 `binding.remote().client()` 真正调用。

### 身份透传（`da32db76`）—— 全新能力

**只传一个字段**，键名带反向 DNS 前缀：`com.nageoffer.ragent/userId`，值为当前登录 userId。

**传递路径**：

```
AgentChatServiceImpl 构建 RuntimeContext (.put(McpMeta.class, ...))
  → McpToolProxy.callAsync() 取 ctx.get(McpMeta.class).entries()
  → client.callTool(getName(), input, meta)        [AgentScope McpClientWrapper]
  → 塞进 MCP CallToolRequest._meta（streamable HTTP）
  → mcp-server 各 executor：McpToolResults.userId(request)
  → null 则 McpToolResults.identityRequired(TOOL_ID)（isError=true）
```

**旧链路的坑（升级理由）**：旧 `McpToolBridge` 用 `runtimeContext.getUserId()`，注释警告*「切 boundedElastic 后 ThreadLocal 型 `UserContext` 取不到，会静默挂空用户」*。新链路把身份放进 AgentScope 的 `McpMeta` 对象随 `RuntimeContext` 传递，**规避线程切换丢身份**。

**安全边界（三层，都有原文可引）**：

1. **`AgentMcpMeta` 类注释**：*「身份只通过 `_meta` 传递，**不进入模型可控制的工具参数**。」*——身份若走工具参数，模型就能看见甚至篡改
2. **`McpToolResults.userId` 注释**：*「`_meta` 是 client-supplied 的扩展点，**规范明确服务端不得据此做认证决策**。本进程只绑本机、只服务同一个应用，用它圈定数据范围成立；**哪天要对外开放，得换成 OAuth 而不是加固这里**。」*
3. **配置侧落地**：`mcp-server/application.yml` 加 `server.address: 127.0.0.1`（只绑回环），commit message：*「mcp-server 绑定回环地址，_meta 为调用方自填不可作认证凭据」*

**服务端不校验真伪**，只做类型 + 空白校验，身份仅用于圈定数据范围。取不到身份**统一回绝**而非按 null 查；文案**不提「重新登录」**（有测试 `identityRequiredMustNotBlameTheUser` 守着）。相关测试：`McpToolResultsTest`（含 `userIdKeyMustStayInSyncWithCaller`、`failureMustNotLeakUnderlyingMessage` / `MustNotLeakExceptionType`）、`McpToolProxyTest.shouldPassThroughAgentScopeMeta`。

### 动态挂载 —— `tool/AgentToolCatalog.java`（332 行）

> 类注释：*主 Agent 工具目录：固定注册 search_knowledge，并按意图树配置挂载当前可用的 MCP 工具。*

- 意图树（`IntentNodeRegistry.listMcpToolNodes()`）与 MCP 注册表**取交集**自动挂载
- `RESERVED_TOOL_NAMES = { KnowledgeSearchTool.TOOL_NAME, MemoryFlushTool.TOOL_NAME, SkillLoadTool.TOOL_NAME }`；命中保留名或远端缺失 → 记入 `unavailableToolIds`，`buildToolkit()` 时逐个 warn
- `resolve(prompts)` → `ResolvedCatalog`（**快照**，指纹与 Toolkit 都从它派生）；`fingerprint()` **两级**：`McpToolBinding.fingerprint()` → `McpToolFingerprint`，聚合 → `ToolCatalogFingerprint`（注释：*「指纹只收生效值」「两处排序都为了指纹稳定」*）。技能只记 `hasSkills` 有无
- `displayNames` / `fieldLabels`：**从服务端 `JsonSchema` 的 `properties[].title` 按声明序提取，供确认卡展示**

> **一条完整的因果闭环（很好的面试点）**：`McpToolSchema` 坚持用 `LinkedHashMap` 保参顺序（而不是 `Map.copyOf`）→ 服务端 schema 参数序稳定 → `fieldLabels` 按声明序提取稳定 → **确认卡字段顺序每次启动都一样**。三个模块隔着网络，靠「保序」这一条串起来。

### MCP 工具开发骨架（`mcp-server`，**新增**）

| 类 | 职责 |
|----|------|
| `McpToolSchema` | 入参 schema 链式构造器，把各执行器手拼的 `Map.of(...)` 收成一处 |
| `McpToolException` | 工具异常（`cb4438c7`） |
| `McpToolResults` | 结果封装：`success/error/args/parseDate` 转 public 供各 executor 复用，`META_USER_ID`、`userId(request)`、`identityRequired(toolId)` |

`McpToolSchema` 里两条注释很值钱：

- `additionalProperties` 恒为 `false`：*「服务端不据此校验入参（SDK 的 `JsonSchemaValidator` 只管 `outputSchema`），这里纯粹是发给模型的约束，省掉它模型就可能自造参数名，而自造的键在 `handleCall` 里会被静默忽略」*——**说清了「这个约束到底约束谁」**
- 保序用 `LinkedHashMap` 而非 `Map.copyOf`：*「`Map.copyOf` 会打乱迭代顺序，参数顺序一乱，发给模型的 schema 与确认卡的字段顺序每次启动都不一样」*

---

## 七、链路追踪与可观测性（**全新子系统**）

> 提交 `177108bb`（09-10）+ 基建 `96c528d0`。包位置 `agent/.../trace/`，13 个主文件 + 12 个测试 + 一份可当逐字稿的 `README.md`。

### 技术栈与装配

- **OpenTelemetry** `opentelemetry-bom 1.62.0` → **OTLP HTTP** → **LangFuse v4**（自托管，`resources/docker/langfuse`）
- 装配类 `AgentTracingConfiguration`，两层条件注解：

```java
@ConditionalOnAgentEngine                                          // ragent.engine.type=agent
@ConditionalOnProperty(prefix = "agent.trace", name = "enabled", havingValue = "true")
```

**配置项**（`agent.trace.*`，默认 `enabled: false`）：

| 键 | 默认 | 说明 |
|----|------|------|
| `enabled` | `false` | 总开关 |
| `capture-content` | `true` | 内容采集（提示词/模型输出/工具出入参）；纯骨架改 `false` |
| `endpoint` | `http://localhost:3000/api/public/otel/v1/traces` | **LangFuse 只收 HTTP 的 OTLP，没有 gRPC 端点** |
| `public-key` / `secret-key` | 空 | 拼 `Authorization: Basic` |
| `max-field-chars` | `32768` | 单字段截断上限 |
| `max-attribute-chars` | `131072` | 单条属性总预算；超限退成带 preview 的合法 JSON |
| `service-version` / `environment` / `instance-id` | 空 | Resource 身份；`instance-id` 空则取 `HOSTNAME`，再退启动期随机 UUID |

其余细节：`BatchSpanProcessor` 批量导出；`Sampler.alwaysOn()` **显式声明不抽样**（注释说明待多租共享后端再按 trace 抽样，且要在根 span 上决定，避免把一棵树采成半棵）；`SpanLimits.setMaxAttributeValueLength(maxAttributeChars * 2)` 留高一档余量防 SDK 再切一刀；特殊头 `x-langfuse-ingestion-version: 4`（缺了直传数据最多延迟十分钟可见）。

### 与 AgentScope 官方中间件的关系：**继承 + 富化**

- `RagentOtelTracingMiddleware` **extends** `io.agentscope.core.tracing.OtelTracingMiddleware`，**只覆写 `onActing`**（有反射测试 `shouldOverrideActingHookOnly()` 守住这个边界）
- 装配时 bean 声明类型仍是 `OtelTracingMiddleware`，所以骨架、耗时、token 计数、Reactor 上下文透传仍走官方实现
- `AgentTraceEnrichmentMiddleware` 是独立的**内容装饰层**，紧贴官方中间件内层补业务属性

**为什么要自研一个**：官方给**整批**工具建一个 `execute_tool` 并写 `gen_ai.tool.*`，但它盖着的是一整批调用——单工具的耗时与成败一个都给不出；等工具体自己建 span 后，同一次执行会被两个工具语义节点各表达一遍。因此换成 `tool_batch`：只说「这是一批、几个人、整批跑没跑」，**工具语义留给真正的执行体**。

### span 结构

```
invoke_agent                      [官方 onAgent 建根 span]        type=agent
├── reasoning (每轮一个)           [自研 onReasoning 补建]         type=chain
│   └── chat                       [官方 onModelCall 建]           type=generation
└── tool_batch                     [自研 onActing 建，与 reasoning 兄弟] type=span
    └── execute_tool <toolName>    [AgentToolBodyTracer 建]        type=tool
```

**几个值得讲的细节**：

- `reasoning` 与 `tool_batch` 是**兄弟**而非父子——官方中间件在自研外层，`onActing` 的 `contextWrite` 只覆盖 reasoning 那条流；两者靠属性 `ragent.react.round_index` 归到同一轮
- span 名**低基数、不含动态参数**（调用 ID/序号进属性）
- `writeIdentity()` **在每一个 span 上**写 `langfuse.user.id` / `langfuse.session.id`——因为 **LangFuse 不回填父节点的这两项**，只写根 span 会导致 observation 级过滤失准（官方文档也这么要求）
- 业务反查键：`ragent.task.id` / `ragent.reply_to.message_id` / `ragent.confirm.message_id`（都是 PG 主键，**从消息记录可反查到 trace**）
- `ragent.contract.version = "2"`，只写根 span，供查询侧区分两代数据

**关键属性**（`RagentAttributes` / `GenAiAttributes` 三层不越界：`gen_ai.*` 标准区、`ragent.*` 自有区、`langfuse.*` 厂商区）：

| 属性 | 含义 |
|------|------|
| `ragent.run.outcome` | 整条 trace 结局：`completed`/`failed`/`interrupted`/`aborted`（按严重度递增 CAS 抢占） |
| `ragent.round.outcome` | 单轮结局，同取值——**span status 只有 OK/ERROR，分不出「被停」与「真报错」** |
| `ragent.react.round_index` | 本轮序号，四类节点同值，**跨兄弟节点归轮的钥匙** |
| `ragent.run.kind` | `ask`（首问）/ `confirm_resume`（确认续跑），链路形状不同 |
| `ragent.tool.batch.id` | 与 PG 块、SSE 帧的 batchId **同值**（`AgentToolExecutionFacts` 独家生成） |
| `ragent.tool.batch.outcome` | 批结局，优先级 `interrupted > awaiting > executed > denied` |
| `ragent.tool.status` | 工具七态，与 PG 块、SSE 帧 status 同一套取值 |
| `ragent.tool.span_scope` | 固定 `"body"`——只有工具执行体，**查表/校验/排队/重试都在 span 外** |
| `ragent.tool.error.source` | `tool_result_error` / `exception`——`error.type` 是闭集，细分放这里 |

`AgentErrorTypes` 枚举：`TOOL_ERROR("tool_error")`、`CANCELLED("cancelled")`、`TIMEOUT("timeout")`。

### `AgentToolBodyTracer.trace()` 的双重职责（**重点**）

它**不是纯观测**——里面的 `facts.markStarted / markEnded / markTerminated` 写的是**业务事实**，有两个业务读者：

1. **`AgentStreamEventBridge`** 读 `startedAt()/endedAt()/durationMs()`，投影到 `AgentBlock`（并置 `durationSource = DURATION_SOURCE_TOOL`），经 SSE 帧发前端、经落库写进 PG
2. **`AgentToolBatchMiddleware`** 读 `fact.startedAt() == null` 判断该调用**「没真进过工具体」**（框架前置失败），据此写 `SHORT_CIRCUIT_PRE_EXECUTION` 短路原因——SSE 与落库那头靠它把工具显示成「未执行」

> README 原文：*「方法体不能绕过，调用顺序不能改。」*

### `README.md` 里的三条硬结论（**逐字稿金矿**）

**① 属性取舍是量出来的，不是猜的**

删了 3 项属性（`langfuse.observation.model.name`、`gen_ai.conversation.id`、三个 span 事件），每项都有实测依据。怎么量的（值得原样保留，体现方法论）：

- Langfuse v4 的 `events_only` 模式下 `/api/public/traces`、`/api/public/observations` 全返回 "not available"，UI 也不好逐属性看 → **直接查 ClickHouse**：`events_full` 是强类型表，被认识的属性进 `input`/`output`/`provided_model_name`/`session_id` 列，不认识的落 `metadata_names`/`metadata_values` 数组，**两边都没有的就是被丢了**。原始 OTLP 载荷在 MinIO，按 `blob_storage_file_path` 列取。
- **判断一把键有没有用不能看它在现有数据里出不出现**（受同 span 上其它键遮蔽影响）。做法是**伪造一条只带这把键的 span 打进去**，看列填不填。
- span event 那条另有三个独立证据：伪造带 event 的 span 打进去 `metadata_names` 里一个没有；官方属性映射表里 event 没有目的地；真实数据里 awaiting 发生过 6 次、`observe()` 同一个 `if` 分支里属性落了 event 没落。

**② `AgentToolBodyTracer.trace()` 包在 `ToolExecutor.applyRetry` 内侧 → 给工具配重试前必须先处理 span 重订阅**

> 重试即重订阅，`Mono.deferContextual` 的体会重跑一遍，于是同一次工具调用建出**第二个 `execute_tool` span**。起止时刻更麻烦：`facts.markStarted/markEnded` 用的是 `compareAndSet(0L, ...)`，第二个 span 会套用第一次尝试的区间，**时间轴上两个节点重叠**。

现在不触发（全项目没有一处配 `ExecutionConfig.maxAttempts`）。**这是「追踪代码反过来约束业务配置」的典型案例。**

**③ 模型侧重试的语义错位（配置陷阱）**

`agent.max-retries` 名字叫 retries、值却直接喂给 `maxAttempts`，**语义是「含首次的总次数」**，且 `ModelConfig` 校验必须大于 0，所以 `1` 就是「不重试」的地板值。

真正生效的配置由三段拼出来：

- `OpenAIChatModel.Builder.build()` 装 `ExecutionConfig.MODEL_DEFAULTS`（5 分钟超时、3 次尝试）进模型 `defaultOptions`
- `ReActAgent.buildGenerateOptions()`（`ReActAgent.java:3631`）每次调用另建一份，只填 `maxAttempts(modelConfig.maxRetries())`
- `OpenAIChatModel.java:109` 是 `mergeOptions(options, configuredOptions)`：**调用级是 primary，模型默认是 fallback**

所以给 `defaultOptions` 配 `maxAttempts(1)` 是**无效的**（被调用级原样盖掉）。更要紧的是 `retryWhen` 加在整条 `doStream0` 之上、位置在 `onModelCall` 之下——**不管流已经吐过多少 chunk，出错就重订阅**，于是下游收到「前半截 + 完整第二遍」：累加的不只是本中间件的缓冲，还有框架的 `transformedText` 与 `ReasoningContext`、SSE 增量、落库正文。

> 现已把 `agent.max-retries` 降到 1 关掉，等实现「仅在首个 chunk 之前允许重试」再放开。README 原文：*「追踪如实反映了框架手里那份已被污染的状态，**改这个包等于把证据擦掉**。」*

### 测试覆盖（12 个测试类）

覆盖面很全，其中几个特别：

- `AgentTraceScenarioTest` — 端到端场景（单工具/并行/同名并行/挂起/全拒/await+resume/取消**两层都收口**）+ **全树不变量扫描**（span 名不含 ID、`gen_ai` 键白名单、`error.type` 封闭取值）
- `AgentToolBodyTracerCoverageTest` — **扫生产 `AgentToolCatalog`，断言每把注册工具恰好留下一个 `execute_tool <name>` span**（用测试守住「没有工具漏埋点」）
- `AgentTraceExporterLoggingTest` — 端点打不通时导出失败必须经 `SLF4JBridgeHandler` 落进应用日志
- `LangfuseReadbackProbe` — **一次性读回探针**，把完整树打到本机 LangFuse（`service.name=ragent-service-plan-d-probe`）后回读校验，**不属于回归集**
- 辅助：`CollectingSpanExporter`（收 span 供断言，因 span 结束后属性只能从这里读）、`MutableClock`（推进时间让 span 起止可被摆到已知位置）

### 纯重构如何自证没改行为（方法论，很值钱）

> 同一天还合并了几处复制粘贴。这类改动**单测全绿不算证完**——改的是收尾路径，得看真发出去的字节。

办法：**同一个探针跑两遍，中间夹着改动，然后按节点比取值数**。每类节点的取值数应当等于「单次运行里该类节点的不同内容数」，而不是它的两倍。

---

## 八、Agent 管理后台与提示词槽位（**新增**）

### 提示词槽位枚举 `AgentPromptSlot` —— 槽位元数据的唯一权威源

> 类注释原文：*槽位按功能命名而非按架构命名：**生效范围会随 v1/v2 演进变化，不编码进标识符**。*

每个槽位声明四件事：**生效档位集合**（`Set<OrchestrationMode>`）、**占位符变量集**、**分组**、**写作指导**。

| 槽位 | 生效档 | 特点 |
|------|--------|------|
| `SYSTEM_CHAT` 闲聊应答 | WORKFLOW | Agent 模式下由主 Agent 直接应答 |
| `AGENT_MAIN` Agent人设 | AGENT | 即 `ReActAgent` 的 `sysPrompt`，**为空直接抛异常** |
| `KNOWLEDGE_TOOL_DESCRIPTION` 知识库工具声明 | AGENT | **唯一一个不进对话消息的槽位**——它随工具定义下发，模型在调用前就要读懂 |
| `AGENT_CONTEXT_COMPACTION` 上下文压缩 | AGENT | 占位符 `{summary_max_chars}`；产物以历史消息身份回填 |
| `AGENT_MEMORY_EXTRACTION` 长期记忆抽取 | AGENT | 占位符 `{existing_memories}` `{recent_turns}` `{memory_max_chars}`；产物不入对话，按 JSON 解析后直接写库 |
| `AGENT_MEMORY_CONSOLIDATION` 长期记忆受限合并 | AGENT | 仅当记忆顶到上限时才调一次 |
| `AGENT_MEMORY_TOOL_DESCRIPTION` 记忆整理工具声明 | AGENT | 同 `KNOWLEDGE_TOOL_DESCRIPTION`，随工具定义下发 |
| `KB_ANSWER` 知识库应答 | 两档共用 | WorkFlow 下由主链路合成，Agent 下由 RAG Tool 内部合成 |

**每条的「写作指导」是这套设计最特别的地方**——例如 `KNOWLEDGE_TOOL_DESCRIPTION`：*「模型靠它判断要不要查知识库，此时还看不到检索结果；写清这个库覆盖哪类问题，**不必在这里规定回答风格**」*；`AGENT_CONTEXT_COMPACTION`：*「产物会以历史消息的身份回填进后续每一轮，而被它替代的原文届时已经删除」*——**把「这个槽位在什么时机、以什么身份、被谁读到」写进定义里**，而不是只给字段名。

### 解析与回落 —— `AgentPromptResolver`

- 读：**先铺内置智能体（`builtin`）作为基线，再让激活智能体（`active`）的非空槽位覆盖**
- 缓存：`AgentPromptCacheManager`；另有 `loadOwnPrompts(agentId)` 不做回落，供控制台编辑态展示
- 存储：`AgentProfileDO`（智能体档案，`builtin`/`active` 标志）+ `AgentPromptDO`（按 `agentId + slotKey`）
- 占位符填充与清理语义与 `PromptTemplateLoader#render` 一致

### Agent 仪表盘 —— `agent/.../admin/`（4 个类）+ `frontend/src/pages/admin/dashboard/`（6 个 tsx）

支持多引擎（`getEngine()`），Workflow 与 Agent 两套面板。**四个类的注释把设计取舍写得很干净，是很好的「性能与安全」展开点**：

| 类 | 注释原文要点 |
|----|-------------|
| `AgentDashboardService` | *独立于 Agent 执行链路的统计适配器，最多缓存 3 个窗口 × 2 个粒度，**无后台任务*** |
| `AgentDashboardReader` | *管理端专属读模型，聚合在 PostgreSQL 内完成，**永不加载消息/工具结果/记忆正文***；*不依赖聊天服务，也不访问框架运行状态；**只读快照事务限制查询时间**，避免首页拖慢业务连接池* |
| `AgentDashboardWindow` | *固定窗口和粒度白名单：**请求参数不得放大为无界扫描或成为 SQL 片段*** |
| `AgentDashboardPerformance` | *仅暴露聚合值，**不返回聊天正文、工具参数、记忆内容或用户标识*** |

暴露的指标：`Replies`（normal/interrupted）、`Tools`（done/failed/interrupted）、`Confirmations`（approved/denied/pending/expired）、`Memory`（compactions、contextReductionPct）等。

---

## 九、示例场景与配套工程（**新增**）

### 比特严选（bit-selection）示例域 —— `cdace75a`、`48b31c84`

`mcp-server` 新增 **49 个文件**，一个完整的电商客服域：

| 组成 | 内容 |
|------|------|
| MCP 工具 | `executor/bit/` 17 个类：订单（create/query/cancel/pay/address）、售后（apply/query）、购物车（query/update）、优惠券（query + `BitCouponRules`）、物流查询、商品搜索、工单提交 |
| 领域数据 | `dao/entity/` 10 个 DO（Order/OrderItem/Product/ProductSku/Cart/Coupon/UserCoupon/AfterSale/LogisticsTrace/Ticket）+ Mapper + Result |
| 技能手册 | 5 份（双文件形态，见第五节） |
| 知识库 | `resources/initializer/bit-selection/docs/knowledge/**` + 5 份 skills |
| 初始化器 | 一键造出确定状态 |

另有 `executor/enterprise/` 7 个企业域工具（资产、请假、会议室、销售、工单等）。另有 `BitOrderReleaser` / `BitPendingOrderTimeoutTask`（超时释放待支付订单）等**业务侧的真实复杂度**。

### 其它工程配套

- `resources/docker/langfuse` — LangFuse v4 自托管编排
- `resources/docker/elasticsearch-8.17.sh` — ES 8.17 启动脚本
- `AgentSkillAdminServiceImpl` / `AgentSkillToolOptionVO` — 技能后台管理（可引用工具来自意图树里已启用的 MCP 节点）
- MQ 事务消息回查类型转换异常修复（`b0f27491`）、模板数据 EOL 转换导致 Windows 检出 checksum 失配修复（`ca93991a`）

---

## 十、技术选型清单

| 分类 | 选型 |
|------|------|
| 语言/框架 | Java 17 + Spring Boot 4.1.0，前端 React 18 |
| Agent 框架 | AgentScope **2.0.2**（ReActAgent；`McpTool` / `OtelTracingMiddleware` / `AgentSkill` 均为框架原生） |
| 协议 | MCP SDK **1.1.2** |
| **可观测** | **OpenTelemetry 1.62.0 → OTLP HTTP → LangFuse v4（自托管）** |
| 模型供应商 | **DeepSeek（当前默认，`deepseek-flash`）**、阿里云百炼、硅基流动、Ollama、AIHubMix（均 OpenAI 兼容） |
| 向量库 | Milvus 2.6.6 |
| 关键词检索 | Elasticsearch 8.17 |
| 知识图谱 | LightRAG |
| 联网搜索 | You.com |
| 融合/精排 | RRF + Rerank |
| 数据库 | PostgreSQL + MyBatis-Plus 3.5.17 |
| 对象存储 | AWS S3 + 阿里云 OSS |
| 缓存/分布式 | Redisson 4.6.1 + Redis |
| 消息队列 | RocketMQ（事务消息） |
| 认证 | Sa-Token |
| 文档解析 | Apache Tika |
| 线程上下文 | Transmittable ThreadLocal（10 个专用线程池） |
| 其他 | Hutool、CommonMark、Batik、bizlog-sdk、Spotless |

> ⚠️ **已修正**：原清单写「阿里云百炼（主）」，实际 `agent.chat.provider: deepseek`。

---

## 十一、追问入口（预留深挖点）

### 原清单条目状态复核

| # | 原追问点 | 现状 |
|---|---------|------|
| ③ | 记忆合并淘汰 `AgentMemoryConsolidator` | ✅ 存活（188 行），**已实现**，有提示词槽位与回归用例 R5 |
| ② | 事件桥接 `AgentStreamEventBridge` | ✅ 存活，**541 → 695 行** |
| ② | 模型容错 `infra-ai` 档位/熔断/首包探测/降级 | ✅ 存活（`LlmFirstPacketProbe` / `RoutingLLMService` / `ModelRoutingExecutor`），**且已与取消语义打通** |
| — | RAG 侧检索：多通道并行召回、RRF 融合、召回预算漏斗 | ✅ 存活（`RetrievalEngine` + `retrieval/channel/`） |
| **⑤** | **MCP 参数提取 `LLMMcpParameterExtractor`** | ❌ **整条已删除**（415 行 + 整包被 `fd58ec76` 移除）——换成新的 MCP 集成架构，见第六章 |
| — | 入库 Pipeline：节点编排、条件执行、事务消息可靠性 | ✅ 存活 |

### 本期新增的追问入口

- **追踪包的三条硬结论**：属性取舍方法论（ClickHouse 实测 + 伪造 span 探针）、工具重试与 span 重订阅的冲突、模型重试语义错位（`max-retries` 名实不符）
- **`AgentScope #2439` 并发串话**：自研遮蔽 vs 原生 Skill 的生命周期差异；`AgentRunGate` 按用户闸门与跨会话并发的覆盖边界（**⚠️ 待确认口径**）
- **记忆回归台的实测画像**：裁剪射程 22.4% vs 下限 20%、上下文涨到预算 3.8 倍、单轮上界外推 5.8 万字符及其仅对 `search_knowledge` 成立的限定
- **双档位的语义边界与收敛过程**：为什么 `OrchestrationProperties` 不挂 `rag` 前缀；为什么先按档位隔离 MCP、再整体删除 WorkFlow 那条链
- **MCP 架构收敛**：为什么删掉 rag 侧整包（双路径 → 单路径 + SDK 版本对齐），`extends McpTool` 相比手写 `ToolBase` 省掉了什么
- **`_meta` 身份透传的三层安全边界**：为什么身份不走工具参数、为什么服务端不做认证决策、为什么绑回环地址
- **`needsConfirm` 默认值方向被反转**：`null` 从「放行」改成「要确认」
- **取消语义的端到端设计**：为什么前端删掉 `streamWithRetry`（与 HITL 不重复提交冲突）、为什么中断提示要独立成块
- **`McpToolSchema` 的 `additionalProperties: false` 与保序**：约束的是模型而不是服务端；保序如何一路影响到确认卡

---

## 附：关键代码锚点速查

| 升级项 | 核心文件 |
|--------|----------|
| **档位枚举** | `rag/.../rag/config/OrchestrationMode.java` |
| **档位属性** | `rag/.../rag/config/OrchestrationProperties.java` |
| **提示词槽位枚举** | `rag/.../rag/core/prompt/AgentPromptSlot.java` |
| **提示词解析** | `rag/.../rag/core/prompt/AgentPromptResolver.java` |
| Agent 引擎装配 | `agent/.../config/ReActAgentProvider.java` |
| 引擎配置/模型/状态 | `agent/.../config/AgentEngineConfiguration.java` |
| 请求编排 | `agent/.../service/impl/AgentChatServiceImpl.java` |
| 生命周期句柄 | `agent/.../service/handler/AgentRunHandle.java` |
| 并发闸门 | `agent/.../service/handler/AgentRunGate.java` |
| 事件桥接 | `agent/.../service/handler/AgentStreamEventBridge.java`（695 行） |
| **任务取消原语** | `framework/.../framework/cancellation/TaskCancellation.java` |
| 工具目录 | `agent/.../tool/AgentToolCatalog.java`（332 行） |
| 工具批中间件 | `agent/.../tool/AgentToolBatchMiddleware.java` |
| 工具执行事实源 | `agent/.../tool/AgentToolExecutionFacts.java` |
| RAG 工具化 | `agent/.../tool/KnowledgeSearchTool.java` |
| **MCP 代理（新）** | `agent/.../tool/McpToolProxy.java`（`extends McpTool`） |
| **MCP 连接（新）** | `agent/.../tool/AgentMcpClients.java` |
| **MCP 身份（新）** | `agent/.../tool/AgentMcpMeta.java` |
| **MCP Schema 构造器** | `mcp-server/.../mcp/executor/McpToolSchema.java` |
| **MCP 结果与身份回绝** | `mcp-server/.../mcp/executor/McpToolResults.java` |
| 记忆注入 | `agent/.../memory/AgentUserMemoryMiddleware.java` |
| 上下文压缩 | `agent/.../memory/AgentContextCompactionMiddleware.java` |
| 裁剪算法 | `agent/.../memory/AgentContextTrimmer.java` |
| 记忆 Pipeline | `agent/.../memory/AgentMemoryPipeline.java` |
| 记忆仲裁 | `agent/.../memory/AgentMemoryJudge.java` |
| 记忆受限合并 | `agent/.../memory/AgentMemoryConsolidator.java` |
| 拒绝口径 | `agent/.../confirm/AgentConfirmDenialMiddleware.java` |
| 技能遮蔽 | `agent/.../skill/AgentSkillMaskingMiddleware.java` |
| 技能加载 | `agent/.../skill/SkillLoadTool.java` |
| **追踪装配** | `agent/.../trace/AgentTracingConfiguration.java` |
| **追踪属性取舍记录** | `agent/.../trace/README.md`（**可当逐字稿用**） |
| **追踪富化中间件** | `agent/.../trace/AgentTraceEnrichmentMiddleware.java` |
| **工具体追踪 + 事实源** | `agent/.../trace/AgentToolBodyTracer.java` |
| **仪表盘读模型** | `agent/.../admin/AgentDashboardReader.java` |
| **回归验证台** | `resources/regression/README.md`（**可当逐字稿用**） |
| **记忆探针** | `resources/regression/agent-memory/AgentMemoryProbe.java` |
| 前端取消流 | `frontend/src/hooks/useAgentStream.ts`、`stores/agentChatStore.ts`、`components/AgentChatInput.tsx` |

> **已删除（原清单锚点，勿再引用）**：`McpToolBridge.java`、`rag/.../core/mcp/*`（10 个文件）、`rag/.../core/intent/IntentTreeFactory.java`（457 行）

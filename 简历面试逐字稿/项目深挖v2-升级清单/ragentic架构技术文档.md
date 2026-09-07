# 寿险智能问答平台 · ragentic（Agentic RAG）架构技术文档

> **本文档定位**：`架构技术文档.md` 的姊妹篇。那篇讲的是 v1「WORKFLOW 固定管线」（意图路由 → 多路检索 → 生成），这篇讲 v2「AGENT 执行架构」——把 RAG 能力工具化，交给 ReAct Agent 自主编排。
> **视角**：演进对比。贯穿全文回答一个面试题——「你们怎么从 RAG 演进到 Agent？改了哪、为什么、代价是什么、怎么兜底？」
> **配套清单**：`Ragent升级清单.md`（按提交时间线记录的升级项清单，本文件是对同一套代码的「架构视角」整理，两者互补）。
> **代码锚点**：项目根 `D:\codinglocation\ragent`，核心在 `agent/` 模块，复用 `rag/` 模块。

---

## 目录

1. [从 RAG 到 Agentic：升级的本质](#1-从-rag-到-agentic升级的本质)
2. [整体架构](#2-整体架构)
3. [端到端执行链路](#3-端到端执行链路)
4. [核心组件技术架构](#4-核心组件技术架构)
5. [技术选型（Agentic 增量）](#5-技术选型agentic-增量)
6. [数据模型（Agent 新增表）](#6-数据模型agent-新增表)
7. [关键设计原则与亮点](#7-关键设计原则与亮点)
8. [升级的代价与兜底](#8-升级的代价与兜底)
9. [已知局限与演进方向](#9-已知局限与演进方向)

---

## 1. 从 RAG 到 Agentic：升级的本质

### 1.1 旧架构（v1 WORKFLOW）的边界

旧版是一条**代码写死的固定管线**，`OrchestrationMode.WORKFLOW`：

```text
意图分类 → 问题改写 → 多路检索 → 融合/重排/闸门 → LLM 合成
```

这条管线有两个先天边界：

- **LLM 只在固定两个点出现**：问题改写、答案合成。中间的「去哪检索、调不调工具、调哪个」全是工程代码/规则，模型插不上手。
- **一次请求 = 一次检索 + 一次生成**：工具调用是「意图树节点静态绑定，命中才调一次」。用户问一个**多步骤组合任务**（「查一下我这张保单的现金价值，再对照条款算能不能退保」），流水线只能跑一轮检索一轮生成，没法「先查 A、再查 B、再综合」。

所以 v1 强在**链路确定、延迟低、可控**，弱在**表达不了组合任务、编排能力上限就是工程师预设的 if/else**。

### 1.2 升级本质：编排权从代码上移到模型

`OrchestrationMode` 枚举把这个转折写得很直白：

```text
WORKFLOW = v1 编排管线：意图分类 → 检索 → 合成，链路确定、延迟低
AGENT    = v2 ReAct 架构：主 Agent 决策，RAG 管线降级为其中一个 Tool
```

升级不是「把 RAG 拆了重写」，而是做了一件事——**把「检索」「工具」「记忆」「技能」全部抽象成 Tool，交给一个 ReAct 循环的 Agent 自主决策**。原来「工程师写死的顺序」变成「模型在循环里动态决定：先干嘛、要不要调工具、调几次、什么时候停」。

### 1.3 双模式共存（不是推倒重来）

关键设计：v1 和 v2 **并存**，不是替换。

- 由 `ragent.engine.type` 指定，是**部署级决策**（切换需重启，且 AGENT 依赖外部 ReAct 服务存活），不开放后台热切换。
- `agent` 模块所有 Bean 都标 `@ConditionalOnAgentEngine`（`@ConditionalOnProperty(ragent.engine.type = agent)`）——**workflow 侧零开销**，不启用 agent 时这套类根本不进容器。
- 提示词槽位 `AgentPromptSlot` 按 `Group` 分栏：`WORKFLOW 专属 / AGENT 专属 / COMMON 通用`。`KB_ANSWER`（知识库应答）是 COMMON——两种架构共用，只是「WorkFlow 下由主链路合成，Agent 下由 RAG Tool 内部合成」。

> **面试话术**：我们不是从 RAG 切到 Agent 的「大爆炸式重写」，而是加了一个 `agent` 引擎档位，`ragent.engine.type=agent` 一键切换，旧的 workflow 管线原样保留当退路。这比「重构」更稳，也更符合生产落地的节奏。

### 1.4 演进对比总表

| 维度 | 旧版 WORKFLOW（v1） | 新版 AGENT（v2 ReAct） |
|---|---|---|
| 编排方式 | 代码写死的固定管线 | 模型在 ReAct 循环里自主调工具 |
| LLM 出现位置 | 固定 2 点：改写、合成 | 每轮都在决策（思考 + 行动） |
| 检索 | 主链路必经的一步 | 降级成一个 Tool（`search_knowledge`） |
| 工具调用 | 意图树节点静态绑定，命中才调一次 | 动态编排，模型决定调哪个/几次/顺序 |
| 多步骤任务 | 不支持（一轮检索一轮生成） | 支持（多轮调多个工具再综合） |
| 上下文治理 | 滑动窗口 + 摘要（会话记忆层） | 50%/80% 两级水位 + 长期记忆注入 |
| 安全 | 无写操作确认 | 两段式确认 + 防篡改 |
| 扩展 | 流程写死 | Skills 渐进式披露 + MCP 动态挂载 |
| 状态 | 无 Agent 状态 | `PgAgentStateStore` 持久化、按次加载 |

---

## 2. 整体架构

### 2.1 分层架构图

```text
┌────────────────────────────────────────────────────────────────────────┐
│ 接入层   AgentChatController（SSE） / 确认入口 / meta 探活                │
├────────────────────────────────────────────────────────────────────────┤
│ 编排层   ReActAgent（AgentScope）—— 思考→行动→观察 循环，≤ maxIters=10    │
│           │  四层中间件（洋葱模型，外→内）                                │
│           │  userMemory → contextCompaction → confirmDenial → skillMask │
├────────────────────────────────────────────────────────────────────────┤
│ 工具层   search_knowledge（RAG 工具化）│ McpToolBridge │ flush_memory    │
│         │ load_skill（Skills 渐进披露）                                  │
├────────────────────────────────────────────────────────────────────────┤
│ 记忆层   注入线（跨会话事实块） + 抽取线（异步 Pipeline：judge→commit）      │
├────────────────────────────────────────────────────────────────────────┤
│ 治理层   AgentRunGate（用户级并发闸门） + AgentRunHandle（生命周期句柄）    │
├────────────────────────────────────────────────────────────────────────┤
│ 状态/存储 PostgreSQL（t_agent_state 等 8 张 Agent 表）· Redis · rag 模块  │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.2 agent 模块组件职责表

| 组件 | 类 | 职责 |
|---|---|---|
| 引擎装配 | `config/ReActAgentProvider` | 单例 ReActAgent + 懒重建 |
| 引擎配置 | `config/AgentEngineConfiguration` | 装配模型 + 状态存储 |
| 请求编排 | `service/impl/AgentChatServiceImpl` | 提问/确认两条入口，串起全链路 |
| 并发闸门 | `service/handler/AgentRunGate` | 用户级单流 |
| 生命周期 | `service/handler/AgentRunHandle` | complete/cancel/fail 三出口 |
| 事件桥接 | `service/handler/AgentStreamEventBridge` | AgentScope 事件 → SSE |
| 工具目录 | `tool/AgentToolCatalog` | 工具解析 + 快照指纹 |
| RAG 工具化 | `tool/KnowledgeSearchTool` | 检索管线收敛成一个工具 |
| MCP 桥接 | `tool/McpToolBridge` | 适配 MCP 执行器 + 权限 ask/allow |
| 记忆注入 | `memory/AgentUserMemoryMiddleware` | 跨会话记忆块注入 |
| 上下文压缩 | `memory/AgentContextCompactionMiddleware` | 50%/80% 两级水位 |
| 裁剪/压缩 | `memory/AgentContextTrimmer` / `AgentContextCompactor` | 具体算法 |
| 记忆 Pipeline | `memory/AgentMemoryPipeline` | 抽取主流程 |
| 拒绝口径 | `confirm/AgentConfirmDenialMiddleware` | 用户取消的文案修正 |
| 技能遮蔽 | `skill/AgentSkillMaskingMiddleware` | G−L 工具遮蔽 |
| 状态存储 | `state/PgAgentStateStore` | Agent 状态 PG 持久化 |

### 2.3 与 rag 模块的关系：降级复用

升级最聪明的一点：**整条 RAG 流水线被收敛成一个方法**。

```text
旧：rag 模块 = 主链路（意图路由 → 改写 → 检索 → 合成）
新：rag 模块 = 一个 Tool 的幕后实现

    search_knowledge（Tool）
         │
         ▼
    KnowledgeSearchFacade.search(query, recentHistory)
         │  内部仍是完整 RAG 窄口：
         │  改写 → 意图解析(KB-only 过滤) → 歧义引导 → 多通道检索 → KB_ANSWER 合成
         ▼
    返回可直接引用的答案文本
```

`KnowledgeSearchFacade` 的类注释把它定义为「Agent 模式下 rag 对外的唯一检索窄口」。三个复用点：

- **检索能力 100% 复用**：`search_knowledge` 工具背后就是原来的 `RetrievalEngine`（多路并行 + RRF + Rerank），一行没重写。
- **意图树拆成两用**：路由知识库（`filterKbOnly` 在 facade 内部用）+ 决定挂哪些 MCP 工具（`intentNodeRegistry.listMcpToolNodes()` 喂给 `AgentToolCatalog`）。
- **MCP 桥接复用**：`McpToolBridge` 把 rag 侧的 `McpToolExecutor` 适配成 AgentScope 的 `ToolBase`，不重接一遍 MCP 客户端。

> 一句话：**编排权上移给了模型，但检索/工具/意图这些「能力」原样复用**。这是「升级成 agentic」和「重写一个 agent」的本质区别。

---

## 3. 端到端执行链路

### 3.1 文字流程

```text
用户提问（SSE）
   │
   ▼
① AgentRunGate.acquire：Redis RBucket.setIfAbsent 抢用户级运行位（失败即拒）
   │
   ▼
② touchConversation + addUserMessage + ensureExtractionBaseline（记忆抽取下界）
   │
   ▼
③ agent.streamEvents(input, RuntimeContext{userId, sessionId}) → Flux<AgentEvent>
   │
   ▼
④ ReAct 循环（每轮：四层中间件改写 ReasoningInput → LLM 推理 → 工具调用 / 最终答案）
   │
   ▼
⑤ AgentStreamEventBridge：事件 → SSE（META / MESSAGE / TOOL / HINT / CONFIRM / FINISH / DONE）
   │
   ▼
⑥ 收尾：persistAssistantMessage → 释放闸门 → 驱逐状态缓存 → 异步记忆抽取
```

### 3.2 ReAct 执行循环

```text
                ┌────────────────────────────────────────────┐
                │        ReAct 循环（≤ maxIters = 10）         │
                │                                            │
                │   ① Reason（LLM 推理，产出思考 + 下一步）     │
                │        │                                   │
                │        ▼                                   │
                │   ② Act（决定调 Tool / 直接回答）             │
                │        │                                   │
                │        ├─ 调工具 → 执行 → ③ Observe（结果回填）│
                │        │              └──────── 回到 ① ──────┤
                │        │                                   │
                │        └─ 能回答了 → 输出最终答案 → STOP        │
                │                                            │
                │   熔断：跑满 10 轮未停 → 框架强制收尾          │
                └────────────────────────────────────────────┘
```

### 3.3 中间件洋葱时序

```text
用户输入
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ userMemoryMiddleware（最外层）                                 │
│   把跨会话记忆块插到「人设之后、会话首条之前」，只读不写          │
│   ┌───────────────────────────────────────────────────────┐ │
│   │ contextCompactionMiddleware                            │ │
│   │   50% 裁剪工具结果 / 80% 压缩摘要，同步上行消息列表       │ │
│   │   ┌─────────────────────────────────────────────────┐ │ │
│   │   │ confirmDenialMiddleware                          │ │ │
│   │   │   改写「用户取消」的拒绝文案（不是权限不足）          │ │ │
│   │   │   ┌───────────────────────────────────────────┐ │ │ │
│   │   │   │ skillMaskingMiddleware（order=0，最内层）    │ │ │ │
│   │   │   │   遮蔽未加载技能解锁的工具（G−L）             │ │ │ │
│   │   │   │        ↓ ReasoningInput                    │ │ │ │
│   │   │   │        LLM 推理                            │ │ │ │
│   │   │   └───────────────────────────────────────────┘ │ │ │
│   │   └─────────────────────────────────────────────────┘ │ │
│   └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 核心组件技术架构

### 4.1 ReAct 执行引擎（装配 + 配置）

**装配**（`ReActAgentProvider`）：

```text
ReActAgent.builder()
    .name("ragent")
    .sysPrompt(persona)                 // AGENT_MAIN 人设槽位
    .model(agentChatModel)              // OpenAIChatModel，单模型无 fallback
    .toolkit(toolCatalog.buildToolkit())// 从目录快照构建
    .maxIters(10) .maxRetries(2)
    .stateStore(PgAgentStateStore)
    .middleware(userMemory → contextCompaction → confirmDenial → skillMasking)
    .build()
```

**单例复用 + 懒重建**：以「人设内容 + 工具目录指纹」比对决定是否重建，控制台改配置后**下一次会话生效、无需重启**；旧实例不主动 close（在途会话还在其上流式输出，交 GC 回收）。

**配置**（`AgentProperties`，`agent:` 段）：

| 配置 | 默认 | 含义 |
|---|---|---|
| `agent.chat.provider/model` | — | 引用 `ai.providers` 解析 url/key/endpoint，单模型无 fallback |
| `agent.max-iters` | 10 | ReAct 循环上限，超出由框架熔断收尾 |
| `agent.max-retries` | 2 | 单次模型调用失败重试 |
| `agent.sse-timeout-ms` | 900000 | SSE 通道超时（15 分钟），到点回收上游 |

### 4.2 四层中间件链（顺序即正确性）

中间件顺序不是随意排的，每一条的相对位置都有**正确性依赖**（代码注释写得很清楚）：

#### ① userMemoryMiddleware（最外层）—— 长期记忆注入

- 首轮从库读**跨会话事实快照**（阻塞 JDBC 切 `boundedElastic`，不占推理线程），缓存进 `RuntimeContext`（单例 Agent 共享，不能存实例字段）。
- 记忆块插在**人设之后、会话首条之前**（`personaBoundary`），用 **USER 不用 SYSTEM**（形制与压缩摘要同源，供应商对中途 SYSTEM 容忍度不一）。
- 只读不写；记忆的「写」由 `flush_memory` 工具触发，走抽取线。

#### ② contextCompactionMiddleware（第二层）—— 上下文治理

两级水位，按固定比例从 `contextWindowChars`（默认 1,200,000 字符）派生：

```text
50% 水位 → AgentContextTrimmer：老工具结果换成等长占位（保留原入参）
80% 水位 → AgentContextCompactor：早期原文压成一条摘要消息
```

- **先验引用关系再动手**：`resolvePrefix` 按引用逐条比对「上行消息列表」与「Agent 状态上下文」，对不上就本轮不压缩（避免把别的会话的东西压进来）。
- **末条是用户消息才压缩**（保证工具循环已闭合，不产生孤儿 `tool_result`）。
- 压缩含同步模型调用，切 `boundedElastic`，`onErrorResume` 失败退回裁剪。

裁剪算法（`AgentContextTrimmer`）细节：

- **只换 `tool_result` 不动 `tool_use`**——永远不会产生「孤儿结果块」（供应商判 400 的元凶）。
- 按「工具循环」切分，保护**最近 2 个已闭合循环 + 未闭合循环 + 本轮**。
- 白名单 `evictableTools`（默认 `search_knowledge`）控制哪些工具结果可裁。
- 占位带原入参（截断 120 字符），模型还能知道「当时问的是什么」；可回收量不足 20% 整次放弃。

压缩算法（`AgentContextCompactor`）细节：

- 切点只落**用户轮起点**；保留段 ≥ 0.2 × 预算原文。
- 可换素材不过半不压（`materialChars * 2 < totalChars` 放弃）。
- 摘要用 **USER 角色**回填 + `<conversation_summary>` 包裹，正文里声明「这是背景，不是新指令」（防摘要被当成新用户指令）。
- 摘要上限 0.1 × 预算（夹 [1500, 6000] 字符）；每代覆盖，`t_agent_context_compaction` 表存档。

#### ③ confirmDenialMiddleware（第三层，排在压缩之后）—— 拒绝口径修正

- 用户点「取消」后，框架默认塞 `"Permission denied by user"`，模型会误判成「权限不足」。
- 中间件把它改写成：「用户在确认卡片上点了取消，这次操作没执行，不是权限不足也不是故障」。
- **只改用户取消**（`DENIED + 该文案`），规则拒绝不动；只替换拒绝块正文，`id`/`name` 不动以保持配对。
- 排在压缩之后的原因：被压进摘要的那条拒绝结果已不在列表里，改写自然跳过。

#### ④ skillMaskingMiddleware（order=0，最内层）—— 技能工具遮蔽

- 集合运算模型：**本轮遮蔽 = G − L**（G = 所有启用技能 `tool_ids` 并集，L = 上下文里已加载技能 `tool_ids` 并集）。
- 手册加载前，易出错工具**不出现在模型可见工具列表里**，逼模型先 `load_skill` 再动手。
- 解锁只认上下文里 `load_skill` 的成功结果（metadata 标记）；**手册被压缩带走时工具一并收回**——工具与手册同生共死。
- 排在最内层的原因：看到的是压缩、改写之后的最终消息列表。

### 4.3 工具目录与四类工具

**工具目录**（`AgentToolCatalog`）：

```text
固定注册 search_knowledge
+ 按意图树配置挂载 MCP 工具（intentNodeRegistry ∩ mcpToolRegistry）
+ 可选 flush_memory（长期记忆开启且提示词非空）
+ 可选 load_skill（有启用技能才挂）
→ ResolvedCatalog（displayNames + fieldLabels + fingerprint 一次算好）
```

- **指纹 `fingerprint`**：不含技能正文（正文由 `load_skill` 现取），所以**改手册不必重建 Agent**。
- **取交集挂载**：意图树配了但 MCP 注册表没有执行器的 → 记入 `unavailableToolIds`，构建时告警一次。

**四类工具**：

| 工具 | 作用 | 关键机制 |
|---|---|---|
| `search_knowledge` | RAG 管线唯一入口 | `readOnly`；带改写兜底（近 2 轮指代消解）；调 `KnowledgeSearchFacade` |
| `McpToolBridge` | MCP 执行器 → AgentScope Tool | 继承 `ToolBase` 接入权限检查 `checkPermissions` → `ask`/`allow` |
| `flush_memory` | 记忆整理 | **无参**（模型只有触发权没有内容写入权）；身份取 `RuntimeContext` 防篡改 |
| `load_skill` | 技能加载 | 技能清单挂 `description`；加载成功写 metadata 标记 |

**MCP 工具权限判定**（`McpToolBridge.checkPermissions`）：

```text
需要确认 = requireConfirm（意图树节点勾选）|| readOnlyHint == false（MCP annotations 显式声明）
需要确认 → PermissionDecision.ask  → 弹确认卡片
否则     → PermissionDecision.allow → 直接执行
```

### 4.4 长期记忆（定位：注入 + 抽取双线）

**注入线（读）**：`AgentUserMemoryMiddleware` 每轮推理前，把跨会话事实块插进上下文（见 4.2①）。

**抽取线（写）**：`AgentMemoryPipeline`，两个入口共用一条管道：

```text
ensureBaseline（预建控制行 = 抽取下界）
  → loadPending（水位线后的新消息）
  → claim（抢占抽取权，短事务双校验）
  → judge（LLM 仲裁，事务外跑，不占行锁）
  → commit（短事务提交，冲突/超容由提交侧兜住）
```

- **双触发**：后台 `BACKGROUND`（轮次结束异步，`minTurns=3` 门槛才值得叫一次模型）+ `FLUSH`（`flush_memory` 工具主动触发，不受门槛挡）。
- **LLM 一次调用完成「抽取 + 取舍」**：四种动作 `NOOP`（不记）/ `ADD`（新增）/ `SUPERSEDE`（取代，靠 id 指认）/ `RETRACT`（撤销）。
- **反 prompt 注入**：素材用 `<fence nonce="随机串">` 围栏包裹，用户消息里的围栏标签被 `neutralize` 中和。
- 容量顶到上限时先试一次受限合并（`AgentMemoryConsolidator`），压到 75% 即停。

> 记忆在「核心链路」里的定位一句话：**中间件负责「读」（注入），工具 + 管道负责「写」（抽取），两者解耦**。模型只能触发 `flush_memory`，记什么忘什么由服务端仲裁。

### 4.5 写操作确认（定位：两段式 + 防篡改）

```text
① 工具声明是否需确认：McpToolBridge.checkPermissions → ask/allow
② 用户确认/拒绝后继续执行：AgentChatServiceImpl.confirmPendingTool
     └─ 工具入参从 Agent 状态取（resolveConfirmResults），前端只传 approved 布尔 → 防篡改
```

- 前端只传「同意/拒绝」，**入参不从前端来**，而是从 Agent 状态里重取待确认的 `ToolUseBlock`，杜绝用户改参数。
- 有待确认工具时**不接受新问题**（否则框架报英文技术异常）；确认卡片落库失败则提示新建会话，不留 pending 卡死。

### 4.6 Skills 渐进式披露（定位）

- 技能 = 「针对具体办事场景写好的操作手册」+ `tool_ids`（加载后才解锁的工具）。
- `load_skill` 把技能清单挂在 `description` 上，模型判断命中后再取正文。
- `skillMaskingMiddleware` 保证「不看手册就办错的工具」在加载前不可见（见 4.2④）。

### 4.7 状态持久化（PgAgentStateStore）

- AgentScope 官方 2.0.2 的 `AgentStateStore` 只有 in-memory / JSON 文件 / Redis / MySQL 四种实现，**项目主存储是 PostgreSQL，故自实现**，挂 `t_agent_state`。
- `payload` 是 AgentScope 自有编解码的**不透明 JSON**，不与业务表建立结构约定（框架升级不破坏表结构）。
- 会话状态**按次加载**：流结束驱逐内存缓存，下一轮从 PG 重载；重建 Agent 不丢历史。

### 4.8 并发门控与生命周期

**并发闸门**（`AgentRunGate`）：一个用户同一时刻只跑一条流。

```text
acquire：RBucket.setIfAbsent("ragent:agent:running:{userId}", "taskId|conversationId")
          抢不到 → 直接拒绝「当前会话处理中」
TTL    ：sseTimeoutMs × 2（15min × 2 = 30min，长过任何活流，又不把用户挡到下个小时）
release：compareAndSet 只放自己占的位（防止 TTL 挤掉后误删下一轮的闸门）
```

- 与 `@IdempotentSubmit` 的区别：覆盖**整个流生命周期**，而非控制器返回 emitter 前的同步窗口。

**生命周期句柄**（`AgentRunHandle`）：

- `complete / cancel / fail` 三条出口 **CAS 互斥**，收尾体只跑一次。
- **优雅中断**：取消时先 `agent.interrupt()` 等框架存盘（最多 2 秒），超时才 `dispose()` 断流——顺序反了会丢本轮 Agent 状态。
- 释放钩子队列保证每个钩子恰好执行一次；`failed`/`forcedDisposal` 时补一次存盘（错误路径与强制断流框架来不及存）。

**事件桥接**（`AgentStreamEventBridge`）：AgentScope 事件流 → SSE 协议。

```text
TEXT_BLOCK_DELTA / THINKING_BLOCK_DELTA → MESSAGE（增量）
TOOL_CALL_START / TOOL_RESULT_*         → TOOL（工具进度块）
REQUIRE_USER_CONFIRM                    → CONFIRM（确认卡片）
HINT_BLOCK / EXCEED_MAX_ITERS           → HINT
AGENT_RESULT + 收尾                      → FINISH / DONE
```

- 前端块模型：`answer / reasoning / error / tool / confirm` 五种块，刷新前后看到的同一条。
- 工具结果截断 64000 字符；中断提示单独成 `error` 块（是系统在说话，混进 answer 就分不清身份了）。

---

## 5. 技术选型（Agentic 增量）

| 分类 | 选型 | 说明 |
|---|---|---|
| Agent 框架 | AgentScope 2.0.2（`ReActAgent`） | ReAct 循环 + 中间件 + 事件流，不自己造轮子 |
| 响应式 | Project Reactor（`Flux<AgentEvent>`） | Agent 执行全程响应式，取消/断流靠 `Disposable` |
| 状态存储 | PostgreSQL（自实现 `AgentStateStore`） | 官方无 PG 实现，自写挂 `t_agent_state` |
| MCP 版本 | 桥接 rag 既有连接，不走 AgentScope 自带 | 根 pom 用 MCP SDK 1.1.2 压制框架自带的 0.17.0 |
| 并发 | Redis `RBucket` + CAS | 用户级单流闸门 |
| 模型 | 单模型无 fallback（`OpenAIChatModel`） | `nativeStructuredOutputWithTools(false)` 兼容端点 |

---

## 6. 数据模型（Agent 新增表）

> Agent 引擎与 workflow 会话**两套表分立**（见 schema 注释「与 workflow 会话两套分立」）。

| 表 | 归属 | 核心字段（简） | 说明 |
|---|---|---|---|
| `t_agent_profile` | 人设 | id, name, description, builtin, active | 智能体人设配置 |
| `t_agent_prompt` | 提示词 | agent_id, slot_key, content | 提示词槽位（对应 `AgentPromptSlot`） |
| `t_agent_skill` | 技能 | skill_code, name, description, content, tool_ids(JSONB), enabled | 技能手册 + 解锁工具 |
| `t_agent_conversation` | 会话 | conversation_id, user_id, title, last_time | Agent 会话列表 |
| `t_agent_message` | 消息 | role, content, thinking_content, blocks(JSONB), message_status | 含块列表 + 思考文本 |
| `t_agent_state` | 状态 | user_id, session_id, state_key, payload(JSONB) | AgentScope 工作状态，不透明 JSON |
| `t_agent_context_compaction` | 压缩审计 | generation, summary, material_chars, context_chars_before/after | 追加型审计，应用侧无读路径 |
| `t_agent_memory` | 记忆 | user_id, content, source_type, invalid_at, superseded_by | 长期记忆事实表 |
| `t_agent_memory_extraction` | 抽取台账 | from/to_message_id, status, trigger_type, attempt_count | 部分唯一索引 = 分布式 claim |
| `t_agent_memory_control` | 记忆控制面 | user_id, revision | 抽取下界 + 并发 revision |

关键字段语义：

- `t_agent_message.blocks`（JSONB）：整条回复的块结构（answer/reasoning/tool/confirm），刷新后回放靠它。
- `t_agent_memory` 用 `invalid_at` / `superseded_by` 做**软失效**，部分索引只查 ACTIVE 行。
- `t_agent_memory_extraction` 的 `uk_agent_memory_extraction_processing`（`status='PROCESSING'` 部分唯一）即**分布式 claim**——同一会话同时只允许一次在飞抽取。

---

## 7. 关键设计原则与亮点

1. **编排权上移、能力复用**——ReAct 决定「怎么组合」，`search_knowledge`/`McpToolBridge` 复用「既有能力」，升级成本收敛在 `agent` 一个模块。
2. **双模式共存、条件装配**——`ragent.engine.type` 部署级切换，`@ConditionalOnAgentEngine` 让 workflow 侧零开销，随时可回退。
3. **中间件顺序即正确性**——记忆块插在「人设后、会话前」+ 排在压缩前，压缩中间件的引用比对才不永久失效；遮蔽排最内层，看到最终消息列表。
4. **两级水位治理上下文**——50% 先裁工具结果（低成本、不丢语义）、80% 再压摘要（高成本、只在顶不住时动），能省则省。
5. **只换 `tool_result` 不动 `tool_use`**——从根上杜绝「孤儿结果块」，避免供应商 400。
6. **写操作「声明确认 + 执行防篡改」**——`checkPermissions` 声明需确认，确认时入参从 Agent 状态重取、前端只传布尔。
7. **状态全部外置**——Agent 状态在 PG、并发位在 Redis、记忆在 PG，Agent 实例单例复用仍会话隔离，多实例可扩展。
8. **记忆读写解耦、服务端仲裁**——模型只有 `flush_memory` 触发权，记什么/忘什么由 judge/consolidator 仲裁，且反 prompt 注入。

---

## 8. 升级的代价与兜底

> 这是「从 RAG 到 Agent」最该讲清楚的一节：Agentic 不是免费的，每份「自主性」都换了一份「不确定性」，我们给每份代价配了兜底。

| 代价 / 风险 | 兜底机制 |
|---|---|
| 模型陷入死循环 / 跑偏 | `maxIters=10` 熔断 + `maxRetries=2` + `EXCEED_MAX_ITERS` 事件提示（不判失败，仍生成总结） |
| 工具结果撑爆上下文 | 50% 裁剪 + 80% 压缩，两级水位 + 白名单 |
| 写操作危险 | `requireConfirm` → `PermissionDecision.ask` → 确认卡片 → 前端只传 `approved` 防篡改 |
| 拒绝被误判为「权限不足」 | `confirmDenialMiddleware` 改写拒绝文案，如实告知「已取消」 |
| 工具爆炸 / 绕过前置步骤 | `skillMasking`（G−L 遮蔽）+ `McpToolBridge` 执行前复查同一份遮蔽结论 |
| 长循环中断丢状态 | `PgAgentStateStore` 按次加载 + 优雅中断先存盘再断流 + 失败收尾补存盘 |
| 并发失控 | `AgentRunGate` 用户级单流 + TTL + CAS 释放 |
| 单例共享 vs 会话隔离 | 状态外置 PG、流结束驱逐缓存、人设/目录变化才懒重建 |
| SSE 断开空跑到迭代上限 | `bindEmitterLifecycle` 监听 `onTimeout/onError/onCompletion`，断开即取消 |

**核心取舍**：v1 换确定性，v2 换灵活性。所以保留双模式——**能走确定管线的不必上 Agent**，多步骤组合任务才值得付 Agent 的延迟与成本。

---

## 9. 已知局限与演进方向

- **延迟**：Agent 一轮量级接近 RAG 单问全程，`maxIters=10` 最坏可放大数倍；演进方向是更细的迭代预算 + 早停策略。
- **单模型无 fallback**：`agent.chat` 只有一个 provider/model，Agent 场景下模型故障没有降级；演进方向是接入 `infra-ai` 的模型路由 + 三态熔断。
- **长期记忆抽取是异步批**：`minTurns=3` 门槛意味着短期内的对话不会立刻沉淀；演进方向是事件驱动 + 更细粒度的即时抽取。
- **压缩不可逆**：早期原文压成摘要后即删（只有审计表存档），无法回溯原文；演进方向是保留「摘要 → 原文」的指针可回查。
- **中间件是静态洋葱**：四层固定顺序，无法按会话动态插拔；演进方向是配置化的中间件链。
- **状态 payload 不透明**：`t_agent_state` 存框架自有 JSON，业务侧无法查询/审计状态细节；演进方向是框架升级后保留关键状态的结构化快照。

---

*本文档与 `架构技术文档.md`（v1 WORKFLOW）、`Ragent升级清单.md`（升级项清单）共同构成「RAG → Agentic RAG」的完整技术叙事；各组件细节以 `agent/` 模块源码为准。*

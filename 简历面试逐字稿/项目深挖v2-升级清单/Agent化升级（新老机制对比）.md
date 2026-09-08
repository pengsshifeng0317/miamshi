# Agent 化升级（新老机制对比 · RAG 流水线 → Agentic RAG）

> **这个问题什么时候被问**：讲完意图识别 / 多路检索 / 会话记忆后，面试官常会追一句「你们后来不是又加了一个 Agent 模式吗？这跟原来的链路什么关系？」——它考的是**你有没有架构层面的掌控感**，能不能说清"哪些是升级的、哪些是复用的、为什么要配套记忆层"。
>
> **一句话记住**：升级不是重写 RAG，而是把固定流水线**收进一个 `search_knowledge` 工具**，再让一个 ReAct Agent（AgentScope）**自己决定怎么编排它和 MCP 工具**——控制权从 Java 代码移交给了大模型。

---

## 🎤 开场总览（45 秒版，可直接背）

> 定位提示：先一句话亮明"决策权转移"这个本质，再点三件具体升级的事 + 复用程度 + 配套代价，末尾留钩子 → 接下文【时序图】【升级点】。

```text
一句话：把"下一步做什么"从 Java 代码交给大模型。
```

**我：**

我们原来是一条 **RAG 固定流水线**：用户进来，代码按顺序跑「会话记忆加载 → 问题重写 → 意图识别 → 多路检索 → 重排序 → 拼 Prompt → 流式生成」，每一步该不该做、先做谁，都是后端代码写死的，模型只在最后生成答案。

后来做了一个 **Agent 化升级**，核心是**把流程控制权交还给模型**：

第一，**老链路没删，被收编成一个工具**。我加了 `agent` 模块，用 AgentScope 搭了一个 ReAct Agent，把原来整套 RAG 管线封装成它的一个工具 `search_knowledge`——意图识别、多路检索、RRF、重排、证据闸门全在里面，只是从"主流程"降级成"工具的私有实现"。

第二，**意图树换了角色**。原来它是入口路由器，负责把所有问题分到 KB / MCP / 系统交互三路；现在入口不再做一次性的意图分类，改成模型自己在循环里判断要不要查库、查哪个库；MCP 工具也桥成了 Agent 的原生工具，模型可以**先查条款、再查实时销量/天气，最后综合回答**——这是老链路做不到的跨工具组合。

第三，**配套了一套两层记忆治理**。Agent 每走一步都要把整段"思考 + 工具往返"带进上下文，膨胀比对话快得多，所以我在它每次推理前挂了个中间件：上下文超过预算一半，就把老的 `search_knowledge` 结果换成占位文本；超过八成，再压成摘要落库——保证多步推理不爆窗口、不丢消息结构。

> 末尾留的钩子 → 面试官会顺着问：**具体流程图长什么样 / 到底升级了哪几点 / 底层表和记忆怎么分的**——正好接下面三节。老链路细节见「多路检索怎么做.md」「意图识别树怎么做.md」「会话记忆怎么做.md」。

---

## 一、老链路：固定流水线（代码编排 · 一次请求走一遍）

```text
用户问题
   │
   ▼
① 会话记忆加载（近 8 轮原文 + 增量摘要）          t_conversation / t_conversation_summary
   │
   ▼
② 问题重写 + 关键词归一化                        t_query_term_mapping（规则优先）
   │
   ▼
③ 意图识别树：一次 LLM 打分，≥0.35 取前 3，子问题 cap≤3    t_intent_node
   │
   ├─ 命中 MCP 节点 ──→ MCP 工具执行 ──→ 直接拼答案（短路）
   ├─ 命中 SYSTEM 节点 ─→ 系统交互直接答（短路）
   ├─ 歧义候选 ────────→ 澄清反问，本轮结束
   └─ 命中 KB 节点 ────→ 继续 ↓
   │
   ▼
④ 多路检索：作用域解析(≥0.6定向/<0.6全局)
     + 四通道并行(向量/关键词/图/联网)
     + RRF 融合 + Rerank + 证据闸门(≥0.2)          t_knowledge_chunk / t_knowledge_vector …
   │
   ▼
⑤ Prompt 组装（KB 上下文 + 记忆压缩 + 意图说明）
   │
   ▼
⑥ LLM 流式生成 ──→ SSE ──→ 用户
```

**特征**：①~⑤ 由 Java 服务按序调用、**写死**；全程**无回环**；模型只被打一次分、被喂一次最终 Prompt。

---

## 二、新链路：Agent ReAct 自主循环（模型编排 · 一次请求多步往返）

```text
用户问题
   │
   ▼
AgentChatService：会话落库(t_agent_conversation/message) → 并发闸门 → 取单例 ReActAgent
   │
   ▼
ReActAgent（AgentScope 引擎）                     ← ReActAgentProvider 装配
   sysPrompt = 人设（t_agent_prompt 的 AGENT_MAIN 槽位）   ← 替掉老"SYSTEM 短路"
   tools     = [search_knowledge + 按意图树挂的 MCP 工具]  ← AgentToolCatalog
   maxIters  = 10 / maxRetries = 2
   stateStore= t_agent_state（跨轮状态按次加载）
   middleware= 记忆中间件（轮内治理挂在这里）
   │
   ▼
┌──────────────── ReAct 循环（≤10 次）────────────────┐
│                                                     │
│  每次模型推理前 → 记忆中间件查水位：                   │
│    > 窗口50%  → Trimmer：老 search_knowledge 结果     │
│                换占位(不删 tool_use，永不产生孤儿块)   │
│    > 窗口80% 且末条是用户 → Compactor：压成摘要落库     │
│                                                     │
│  LLM 思考 → 二选一：                                 │
│   ├─ 够了 → 生成终答 → 出循环                         │
│   └─ 缺资料 → 调工具：                               │
│        · search_knowledge                            │
│           内部仍跑老管线：改写 → 只留KB意图 → 歧义引导  │
│           → 多通道检索 → 内层 LLM(temp=0) 合成答案文本 │
│        · MCP 工具（桥成原生工具，可多次/可组合）        │
│    工具结果回填 → 回到"LLM 思考"                      │
└─────────────────────────────────────────────────────┘
   │
   ▼
AgentStreamEventBridge：思考/工具进度/回答 → SSE → 用户
   │
   ▼（收尾）状态写回 t_agent_state → 内存驱逐 → 下轮从 PG 反序列化读回
```

---

## 三、新旧时序图对比

### 老链路时序（代码串一次，流式返回）

```text
用户   ChatService   记忆Service   Rewrite    Intent     RetrievalEngine    LLM      SSE
 │         │            │           │           │             │            │        │
 ├─问题────→│            │           │           │             │            │        │
 │         ├─加载历史────→│           │           │             │            │        │
 │         ├─改写+归一化─────────────→│           │             │            │        │
 │         ├─意图打分────────────────────→│(一次)    │             │            │        │
 │         ├─[MCP短路/澄清分支]           │           │             │            │        │
 │         ├─多路检索(并行4通道)──────────────────────────→│            │        │
 │         ├─RRF+Rerank+闸门(纯规则)───────────────────────┤            │        │
 │         ├─组装Prompt───────────────────────────────────────────────→│        │
 │         ├─────────────────────────────流式生成─────────────────────────→│        │
 │         │←──────────────────────────────────────────────SSE 逐字───────────│        │
```

### 新链路时序（Agent 编排，典型一轮=思考→查库→再思考→答）

```text
用户  AgentChatService    ReActAgent(引擎)   记忆中间件   search_knowledge   Facade(老管线内部)   SSE
 │         │                 │                 │             │                  │            │
 ├─问题────→│                 │                 │             │                  │            │
 │         ├─落库/闸门/取Agent→│                │             │                  │            │
 │         │                 ├─[≤50%水位放行]──→│             │                  │            │
 │         │                 ├─LLM思考#1:要查库   │             │                  │            │
 │         │                 ├─tool_use───────────────→│      │                  │            │
 │         │                 │                  ├─改写+KB意图+歧义引导              │            │
 │         │                 │                  ├─多通道检索───────────────────────→│            │
 │         │                 │                  ├─内层LLM temp=0 合成KB答案         │            │
 │         │                 │                 ├←─tool_result(答案文本)─────────────┤            │
 │         │                 ├─LLM思考#2:够了直接答 │       │                  │            │
 │         │                 ├─[工具进度事件]─────────────────────────────────────────────────→│
 │         │                 ├─终答文本────────────────────────────────────────────────────→│
 │         │←─状态落PG/内存驱逐─│                 │             │                  │            │
 │         │←────────────────────────────────────────────────────────SSE─────────────────│
```

> **升级后的新能力**：问「深圳明天能买重疾险吗，顺便看下上个月销量」→ Agent 思考#1 调 `search_knowledge`（读条款/健康告知）→ 思考#2 调 MCP 工具（天气 / 销量）→ 思考#3 **综合两路结果**给最终答复。老链路一次意图命中一个分支，做不了这种组合。

---

## 四、具体升级点（逐条能讲，附代码依据）

### 升级点 ①：流程控制权——代码写死 → 模型自主
老链路步骤在 Java 里按序写死；`ReActAgentProvider` 只给 Agent 三样东西：`sysPrompt`（人设）、`toolkit`（工具）、`maxIters=10`，剩下的"该做什么、做几步"由模型在 ReAct 里决定。
> 面试答法："以前流程是我用 Java 编排的，现在我把检索和 MCP 都做成工具，让模型自己排。"

### 升级点 ②：老流水线被"下沉"成一个工具，不是重写
`search_knowledge`（`KnowledgeSearchTool`）内部调 `KnowledgeSearchFacade.search`，那里**仍然按老管线走**：`rewriteWithSplit` 改写 → 意图解析（**只留 KB**，`filterKbOnly`）→ 歧义引导 → 多通道检索 → 内层 LLM **temp=0** 合成一段答案文本当 `tool_result`。
> 面试答法："改造只发生在编排层，底层检索、意图树、证据闸门全是复用的。"

### 升级点 ③：意图树从"入口路由器"降级成"两处被动配置"
- 老：入口一次意图分类，决定走 KB / MCP / SYSTEM / 澄清。
- 新：入口没有一次性分类。`filterKbOnly` 的注释点明了分工：**MCP 走原生工具，SYSTEM 由主 Agent 人设直接承担**，只剩 KB 意图进检索。
- 意图树剩下两处活：(a) 决定挂哪些 MCP 工具及怎么描述（`AgentToolCatalog` 拿 `t_intent_node.mcp_tool_id` 和 MCP 注册表求交集）；(b) 检索内部仍靠它定位 collection、歧义时命中就返回引导问句（跳过检索）。
- 配套存储：人设和工具描述不再写死，放 `t_agent_profile`（多个人设）+ `t_agent_prompt`（`slot_key` 槽位，如 `AGENT_MAIN`、`KNOWLEDGE_TOOL_DESCRIPTION`），改提示词不用改代码——这也是为什么 `ReActAgentProvider` 拿"人设文本 + 工具目录指纹"判断要不要**懒重建** Agent。

### 升级点 ④：指代消解/问题拆分的承担者变了
- 老：入口先强改写 + 归一化，再走意图。
- 新：主 Agent 上下文自带历史、能自己理解"它/这个"——工具参数要求模型传"**完整独立问题**"（`query` 的描述）；真没消解干净，工具内部才取近 2 轮做一次**兜底改写**。
- 子问题拆分：从"入口拆完、并行意图"变成"模型觉得复杂就**多调几次 `search_knowledge`**，每次一个独立 query"。

### 升级点 ⑤：跨工具组合（老链路做不到）
老链路 MCP 是"一次命中 → 短路执行 → 一问一答"；`McpToolBridge` 把 MCP 工具桥成 Agent 原生工具后，模型可以**先查知识库、再查实时数据、最后综合回答**。这是"Agentic RAG"相对"RAG + 工具"的核心卖点。

### 升级点 ⑥：记忆从"业务侧消息窗口"变"引擎级全量状态治理"
| 维度 | 老（会话记忆） | 新（Agent 记忆） |
|---|---|---|
| 记忆内容 | DB 对话消息（user/assistant），近 8 轮窗口 + 增量摘要 | AgentScope **完整状态**（含每轮 thinking + tool 往返），序列化到 `t_agent_state`，跨轮按次加载 |
| 治理时机 | 进请求时由记忆服务先加载好 | **每次模型推理前**由中间件查水位 |
| 治理手段 | 窗口截断 + 摘要文本 | 两层：>50% 裁工具结果（`Trimmer`，只换占位、不动 tool_use、白名单 `search_knowledge`）；>80% 压摘要（`Compactor`） |
| 为什么 | 对话历史也会长但可控 | Agent 每轮带整段工具往返，膨胀快得多，不治必爆 |
| 成本细节 | — | 刻意设 **20% 最小回收量门槛**：删太少就不动，保住模型 prompt 缓存；压缩记录以**追加审计**写 `t_agent_context_compaction`（generation / summary / 前后字符数），应用侧无读路径 |
| 兜底 | — | 记忆处理异常一律**降级放行**，永不阻断推理 |

### 升级点 ⑦：流式从"文本流"变"过程可观测流"
`AgentStreamEventBridge` 把 AgentScope 事件（思考、工具进度、完成）翻译成 SSE 事件，前端能看到"它在调什么工具、走到第几步"，`t_agent_message.blocks`（JSONB，reasoning/answer/tool 有序序列）用于**回放还原完整时间线**；老链路消息只有正文。

### 升级点 ⑧：外围新增"指挥与刹车"件
`AgentRunGate`（按用户并发闸门，先于一切副作用，被拒请求不留脏数据）；SSE 的 timeout/error/completion 三条终止信号都接了取消协议，防止"用户关页了，ReAct 还在后台跑满 10 次"。

---

## 五、一句话收尾（面试官问"改动多大"时直接背）

> "这次升级不是把 RAG 换掉，而是把原来的固定管线封装成一个知识检索工具，再用 AgentScope 搭一个会自己思考的 ReAct Agent 去编排它和 MCP 工具。**控制权从代码交给模型**，换来跨工具组合和更自然的交互；代价是 Agent 上下文膨胀快，所以我配套做了一套按水位先裁剪工具结果、再压缩成摘要的两层记忆治理，并用 20% 的最小回收量保住模型缓存——新老两条链路目前并存，老链路仍服务标准问答场景。"

---

## 🪝 面试官可能追问 + 应对

**追问 1：你到底是重写了 RAG 还是复用的？**
> 复用为主。检索/意图/重排/证据闸门都在 `KnowledgeSearchFacade` 里，我只是把它包成 `search_knowledge` 工具给 Agent 调。新增的只有 `agent` 模块这一层编排（AgentScope Agent + 工具桥 + 记忆治理）。

**追问 2：为什么保留老链路？两条并存不重复吗？**
> 老链路适合"标准知识问答"（确定性高、可回归、可评测）；Agent 模式适合需要**判断、组合多源、多轮确认**的问题。双轨并存：老链路继续吃 RAGAS 评测回归，Agent 走自己的会话与状态表，互不污染。schema 注释里也写了"v2 ReAct 与 workflow 会话两套分立"。

**追问 3：模型自己决定会不可控，怎么兜底？**
> 四层兜底：① `maxIters=10` + `maxRetries=2` 熔断；② 工具**白名单**（固定 `search_knowledge` + 意图树显式配置的 MCP），且检索工具 `isReadOnly=true`，没有危险写工具；③ 工具内部 `KB_ANSWER` 用 **temp=0** 生成、严格基于证据，`tool_result` 返回前还会抹掉内部 docId 锚点，防泄漏到模型可见文本；④ 歧义场景仍会命中意图引导，工具直接返回问句而不是瞎答。

**追问 4：Agent 上下文膨胀这么快，你具体怎么控？**
> 中间件在每次推理前查水位。预算设成 `context-window-chars=1200000`，两道门按比例派生：超过 50% 用裁剪器把**老的 `search_knowledge` 结果**换成 `[历史工具结果已省略，原长 N 字符，原入参 …]` 占位——**只换结果、不动 tool_use 请求**，所以永不产生孤儿块；超过 80% 才允许压缩成摘要（要调一次摘要模型）。只删白名单工具的**可重查结果**；本轮 + 最近 2 个已完成循环保护不删；回收量不足总字符 20% 就整次不动——因为删太碎会让每次请求的 prompt 前缀都变，模型缓存永远命中不了。

**追问 5：跨轮记忆靠什么？为什么下轮还要从 PG 读？**
> AgentScope 会把整段 ReAct 状态交给我的 `PgAgentStateStore` 序列化到 `t_agent_state`，下轮按 sessionId 反序列化读回，模型能看到上一轮完整的思考与工具结果（这是 agent 多轮一致的底气）。读回的是内存副本，`AgentChatServiceImpl` 在流结束就 `evictStateCache` 清内存缓存——避免常驻内存泄漏，代价只是一次反序列化。

**追问 6：Agent 怎么知道"该不该查库"？**
> 不靠规则，靠 `t_agent_prompt` 里 `KNOWLEDGE_TOOL_DESCRIPTION` 槽位教它"这个工具查什么、什么时候用、怎么把指代补全成独立问题"。描述写得不好，Agent 就会乱调或漏调——所以工具描述是可配置槽位，不是写死的字符串。

---

## 📦 涉及的表（分新旧两栏，均对照 schema 真实字段）

### 老链路核心表（Agent 化后大部分被复用 / 内部使用）

| 表 | 用途 | 关键字段 |
|----|------|---------|
| `t_intent_node` | 意图树：Agent 模式下改作 MCP 工具配置源 + 检索定位 | `kind`(0 KB/1 SYSTEM/2 MCP)、`mcp_tool_id`、`collection_names`、`description` |
| `t_query_term_mapping` | 术语归一化（改写兜底仍用） | `source_term`→`target_term`、`match_type`、`priority` |
| `t_knowledge_chunk` / `t_knowledge_vector` | 检索内容 / pgvector | `chunk.content`；`vector.collection_name`+`embedding(1536)` |
| `t_rag_trace_run` / `t_rag_trace_node` | 全链路 trace（Agent 模式同样打点） | `trace_id`、`node_type`、`duration_ms` |
| `t_conversation` / `t_message` / `t_conversation_summary` | 老链路标准问答的会话与记忆 | 窗口 8 轮 + `last_message_id` 增量摘要游标 |

### Agent 化新增表（`schema_pg.sql` 第 396 行起）

| 表 | 用途 | 关键字段 |
|----|------|---------|
| `t_agent_profile` | 智能体人设配置（可多套、`active` 生效） | `name`、`description`、`avatar`、`active` |
| `t_agent_prompt` | 人设 / 工具描述等提示词槽位，改提示词不改代码 | `agent_id`+`slot_key`(唯一)、`content`（存 `AGENT_MAIN`、`KNOWLEDGE_TOOL_DESCRIPTION` 等） |
| `t_agent_conversation` | Agent 会话列表 | `conversation_id`+`user_id` 部分唯一、`title` |
| `t_agent_message` | Agent 消息记录，`blocks` 保留运行轨迹可回放 | `role`、`content`、`thinking_content`、`blocks`(JSONB: reasoning/answer/tool 有序序列)、`reply_to_message_id`、`message_status`(NORMAL/INTERRUPTED) |
| `t_agent_state` | AgentScope 工作状态（跨轮记忆载体），payload 不透明 JSON | 主键 `user_id`+`session_id`+`state_key`、`payload` |
| `t_agent_context_compaction` | 上下文压缩事件，追加型审计日志 | `generation`、`summary`、`material_msg_count/chars`、`summary_chars`、`context_chars_before/after` |

---

## 📋 面试小抄（30 秒复盘）

- 本质：**决策权从代码移交给模型**；老 RAG 被收进 `search_knowledge` 工具。
- 三个关键词：**收编**（老管线→工具）、**重构意图树角色**（入口路由→工具挂载/检索定位）、**两层记忆治理**（50% 裁 / 80% 压）。
- 三个数字：`maxIters=10`、`context-window-chars=1200000`、白名单 `evictableTools=[search_knowledge]`。
- 一张表都不许说错：Agent 新增 `t_agent_profile/prompt/conversation/message/state/context_compaction`，老链路会话是 `t_conversation` 三张，**两套分立**。

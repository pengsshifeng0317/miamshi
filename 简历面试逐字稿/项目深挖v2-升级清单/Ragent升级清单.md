# Ragent 升级清单（RAG → Agentic RAG）

> **这份文档是干什么的**：记录 Ragent 项目从「RAG 问答系统」升级到「Agentic RAG 平台」过程中的所有升级项。以后要复习或追问，直接把本文件内容（或路径）给 AI，说一句「基于这份升级清单，帮我深入讲 XX」即可。
>
> **项目根目录**：`D:\coding_location\java\zhishixingqiu\nageoffer_\ragent`
>
> **一句话定位**：Ragent 是一个基于 Spring Boot 4 + Java 17 的生产级 Agentic RAG 平台，用 AgentScope 框架把 RAG 能力工具化，交给 ReAct Agent 编排调用，叠加 MCP / 长期记忆 / 写操作确认 / Skills / 模型熔断降级。

---

## 一、升级总览

**升级本质**：旧版是「工程师写死的检索管线」（检索 → 融合 → 生成），新版是「模型驱动的工具编排运行时」——Agent 引擎负责调度，中间件负责上下文治理，记忆/确认/技能/MCP 是挂在这个运行时上的四类能力。

### 关键提交时间线

| 提交 | 动作 | 意义 |
|------|------|------|
| `020e5c3d` | 拆分模块并新增 Agent 执行架构 | 单体 → 7 模块，引入 Agent |
| `7f3a7f38` | 升级 Spring Boot 4 | Boot 3 → 4.1.0，Jackson 2→3 迁移 |
| `17eaaa6f` | 新增智能体管理 | Agent 管理后台 |
| `1cb684ed` | 实现 Agent 长期记忆 | memory 子系统 |
| `24e6b4a4` | 写操作确认流程 | 工具执行前人工确认 |
| `c71f0750` | 集成 Skills | 技能手册 + 工具遮蔽 |
| `45ec387e` | 新增 MCP 工具 | MCP 工具生态 |
| `d0b744ea` | 仪表盘多引擎 | 重构 Agent 仪表盘 |

### 升级维度总表

| 维度 | 旧版（RAG） | 新版（Agentic RAG） |
|------|------------|---------------------|
| 编排方式 | 固定检索管线 | 模型自主调工具（ReAct） |
| 工具接入 | 应用内硬编码函数 | MCP 协议 + 动态挂载 |
| 会话记忆 | 最近 N 轮拼接 | 跨会话长期记忆 + 压缩摘要 |
| 安全 | 无 | 写操作人工确认 |
| 扩展 | 流程写死 | Skills 渐进式披露 |
| 框架 | Spring Boot 3 | Spring Boot 4 |

---

## 二、架构层升级

### 1. 模块化拆分（提交 `020e5c3d`）

单体 `bootstrap` 拆成 7 个 Maven 模块：

| 模块 | 职责 |
|------|------|
| `framework` | 统一响应/异常、认证上下文、幂等、分布式 ID、MQ、Trace、SSE |
| `infra-ai` | Chat/Embedding/Rerank/VLM 客户端、模型档位、路由、熔断、首包探测 |
| `system` | 用户认证、审计日志 |
| `rag` | RAG 问答、知识库、入库 Pipeline、意图树、检索、会话 |
| `agent` | Agent 执行架构（ReAct v2） |
| `bootstrap` | 启动装配 |
| `mcp-server` | 独立 MCP 工具服务 |

**分层核心思想**：把「业务编排 / AI 供应商差异 / 通用基础设施」隔离开，切模型/向量库/对象存储时，问答主流程不重写。

### 2. Spring Boot 4 升级（提交 `7f3a7f38`）

- Boot 3 → 4.1.0，Java 17
- Jackson 2 → 3 迁移，用 `spring-boot-jackson2` 兼容模块过渡
- MyBatis-Plus 用 `mybatis-plus-spring-boot4-starter`

---

## 三、Agent 执行引擎（最核心升级）

### 引擎装配 —— `agent/.../config/ReActAgentProvider.java`

- 用 **AgentScope 2.0.2** 的 `ReActAgent`（builder 装配）
- **单例复用 + 懒重建**：以「人设内容 + 工具目录指纹」比对决定是否重建，控制台改配置无需重启
- 会话状态存 PostgreSQL（`PgAgentStateStore`），按次加载，重建不丢历史

### 四层 Middleware 链（洋葱模型，从外到内）

```java
.middleware(userMemoryMiddleware)       // 注入跨会话记忆块
.middleware(contextCompactionMiddleware) // 裁剪/压缩上下文
.middleware(confirmDenialMiddleware)     // 改写用户取消的拒绝文案
.middleware(skillMaskingMiddleware)      // 技能工具遮蔽（最内层）
```

### 请求生命周期 —— `agent/.../service/impl/AgentChatServiceImpl.java`

- 两条入口：`streamChat`（提问）、`confirmPendingTool`（确认卡片）
- `launchStream` 里 `agent.streamEvents(input, RuntimeContext)` 返回 `Flux<AgentEvent>`，经 `AgentStreamEventBridge` 转 SSE
- 整个 Agent 执行是**响应式**的（Project Reactor）

### 生命周期句柄 —— `service/handler/AgentRunHandle.java`

- `complete / cancel / fail` 三条出口 **CAS 互斥**，收尾体只跑一次
- **优雅中断**：取消时先 `agent.interrupt()` 等框架存盘（最多 2 秒），超时才 `dispose()` 断流，否则丢本轮状态
- 释放钩子队列保证每个钩子恰好执行一次

### 用户级并发闸门 —— `service/handler/AgentRunGate.java`

- 一个用户同一时刻只跑一条流，Redis `RBucket.setIfAbsent` 抢占
- TTL = SSE 超时 × 2；`release` 用 `compareAndSet` 只放自己占的位（防误删别人的闸门）

### RAG 能力工具化 —— `agent/.../tool/KnowledgeSearchTool.java`

- 整个 RAG 检索管线封装成 `search_knowledge` 工具（Agent 模式下 RAG 唯一入口）
- 带改写兜底：取最近 2 轮做指代消解

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

---

## 五、写操作确认 + Skills（安全与扩展）

### 两段式写操作确认

1. **工具声明是否需确认**：`McpToolBridge.checkPermissions()` 返回 `ask`/`allow`，判断 = `requireConfirm || readOnlyHint == false`
2. **确认后防篡改**：`AgentChatServiceImpl.resolveConfirmResults()` 从 Agent state 取待确认工具，前端只传 `approved` 布尔

### 拒绝口径修正 —— `confirm/AgentConfirmDenialMiddleware.java`

- 用户点「取消」后，框架默认塞 `"Permission denied by user"`，会被模型误判为权限不足
- 中间件改写成友好解释：「用户在确认卡片上点了取消，不是权限不足也不是故障」
- 只改用户取消（`DENIED + 该文案`），规则拒绝不动

### Skills 渐进式披露 —— `skill/AgentSkillMaskingMiddleware.java`

- 集合运算模型：`本轮遮蔽 = G − L`（G = 所有启用技能 tool_ids 并集，L = 已加载技能 tool_ids 并集）
- 手册加载前，易出错工具不出现在模型可见工具列表里，逼模型先 `load_skill`
- 手册被压缩带走时，工具一并收回（**工具与手册同生共死**）

### 技能加载工具 —— `skill/SkillLoadTool.java`

- 技能清单挂在工具 `description` 上，正文等模型判断命中后再取
- 改手册无需重建 Agent

---

## 六、MCP 工具桥接

### 桥接适配 —— `agent/.../tool/McpToolBridge.java`

- 把 `McpToolExecutor`（rag 侧 MCP 执行器）适配成 AgentScope 的 `ToolBase`
- `buildInputSchema()`：MCP `JsonSchema` → AgentScope `inputSchema`
- `resolveReadOnlyHint()`：读 MCP `annotations.readOnlyHint`

### 动态挂载 —— `tool/AgentToolCatalog.java`

- 意图树（`IntentNodeRegistry`）与 MCP 注册表（`McpToolRegistry`）**取交集**自动挂载
- 有配置但无执行器 → 记入 `unavailableToolIds`，构建时告警

### 版本冲突规避

- AgentScope 自带 MCP SDK 0.17.0 被根 pom 的 1.1.2 压制
- MCP 客户端**不走 AgentScope 自带**，统一桥接 rag 既有连接（选「桥接复用」而非硬升级或双版本共存）

---

## 七、技术选型清单

| 分类 | 选型 |
|------|------|
| 语言/框架 | Java 17 + Spring Boot 4.1.0，前端 React 18 |
| Agent 框架 | AgentScope 2.0.2（ReActAgent） |
| 协议 | MCP SDK 1.1.2 |
| 模型供应商 | 阿里云百炼（主）、硅基流动、Ollama、AIHubMix（均 OpenAI 兼容） |
| 向量库 | Milvus 2.6.6 |
| 关键词检索 | Elasticsearch |
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

---

## 八、追问入口（预留深挖点）

这些是还没展开的细节，以后问 AI 时可以直接点：

- **③ 记忆合并淘汰**：`AgentMemoryConsolidator` 的合并算法（容量超限时的取舍策略）
- **② 事件桥接**：`AgentStreamEventBridge`（541 行，框架事件 → SSE 的完整映射）
- **② 模型容错**：`infra-ai` 的模型档位、三态熔断、首包探测、降级链路
- **RAG 侧检索**：多通道并行召回、RRF 融合、召回预算漏斗校验
- **⑤ MCP 参数提取**：`LLMMcpParameterExtractor`（LLM 辅助提参）
- **入库 Pipeline**：节点编排、条件执行、事务消息可靠性

---

## 附：关键代码锚点速查

| 升级项 | 核心文件 |
|--------|----------|
| Agent 引擎装配 | `agent/.../config/ReActAgentProvider.java` |
| 引擎配置/模型/状态 | `agent/.../config/AgentEngineConfiguration.java` |
| 请求编排 | `agent/.../service/impl/AgentChatServiceImpl.java` |
| 生命周期句柄 | `agent/.../service/handler/AgentRunHandle.java` |
| 并发闸门 | `agent/.../service/handler/AgentRunGate.java` |
| 事件桥接 | `agent/.../service/handler/AgentStreamEventBridge.java` |
| 工具目录 | `agent/.../tool/AgentToolCatalog.java` |
| RAG 工具化 | `agent/.../tool/KnowledgeSearchTool.java` |
| MCP 桥接 | `agent/.../tool/McpToolBridge.java` |
| 记忆注入 | `agent/.../memory/AgentUserMemoryMiddleware.java` |
| 上下文压缩 | `agent/.../memory/AgentContextCompactionMiddleware.java` |
| 裁剪算法 | `agent/.../memory/AgentContextTrimmer.java` |
| 记忆 Pipeline | `agent/.../memory/AgentMemoryPipeline.java` |
| 记忆仲裁 | `agent/.../memory/AgentMemoryJudge.java` |
| 拒绝口径 | `agent/.../confirm/AgentConfirmDenialMiddleware.java` |
| 技能遮蔽 | `agent/.../skill/AgentSkillMaskingMiddleware.java` |
| 技能加载 | `agent/.../skill/SkillLoadTool.java` |
| MCP 执行器接口 | `rag/.../core/mcp/McpToolExecutor.java` |

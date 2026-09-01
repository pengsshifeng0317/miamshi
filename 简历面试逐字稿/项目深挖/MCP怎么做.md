# 模拟面试逐字稿：MCP 怎么做

> 围绕简历产出——「基于 MCP 协议集成多业务工具，支持多工具并行调用，扩展智能问答与 Agent 能力」——的完整一问一答逐字稿。
> 包含：开场总览 → 追问 → 追问铺垫，最后附「追问钩子速览」和「面试建议」。
> 回答内容基于真实代码实现（`mcp-server/` 独立进程 + `rag/core/mcp/` 客户端 + `agent/tool/` 桥接 + `core/retrieval/` 调用侧）。

---

> 🕐 真实链路执行时序（开场先画这张图）：

```text
 LLM(Agent)           MCP Client(rag/core/mcp)      MCP Server(独立进程 :9099)        业务系统
    │                        │                             │                          │
    │ 需要外部工具能力          │                             │                          │
    │ ① 输出 tool_use 调用计划  │                             │                          │
    │────────────────────────►│ ② 查工具注册表/快照 fingerprint│                          │
    │                         │──JSON-RPC(工具名+参数)───────►│ ③ 执行工具                 │
    │                         │                             │──HTTP / RPC──────────────►│ 天气/销售/工单/请假...
    │                         │                             │◄─────────────────────────│ 业务结果
    │                         │◄───────────────────────────│ 结构化工具结果              │
    │◄────────────────────────│ ③ 结果注入上下文，LLM 继续     │                          │
    │ 多个工具 → 两层并行调用     │                             │                          │
    ▼                         ▼                             ▼                          ▼
```

## 🎤 开场总览（45 秒版，可直接背）

> 面试官问"MCP 在你们项目里是怎么用的？为什么选 MCP 而不是直接写 Java 方法调用？"时，先把下面这段一气呵成讲完，再落到 Q1 展开。
> 末尾留的钩子（协议化三层架构 / 工具自描述 Schema / LLM 提参三态 / 两层并行 + Agent 桥接）正好接 Q2 / Q5 / Q7 / Q9。

**我：** 我们做 MCP，是为了解决一个问题——知识问答平台光有"检索知识库"不够，用户会问"今天上海天气怎么样""查一下华东区销售数据"这类要**调用外部业务能力**的问题。这些能力不该写死在 RAG 链路里，所以用 **MCP（Model Context Protocol）** 把工具能力"协议化"。

核心是四件事：

**第一，三层架构，把工具能力"协议化"。** Server 是独立进程（`mcp-server`，跑在 9099，Spring Boot + Streamable HTTP），暴露天气、销售、工单、请假、资产、联网搜索等工具；Client 在 `rag/core/mcp`，启动时连接 Server、`listTools()` 动态发现工具、注册进工具注册表；消费侧是 RAG 检索链路 + Agent。为什么选 MCP 而不是硬编码 Java 方法调用：一是**解耦**——工具升级、新增不用改主服务，新增一个工具就是 Server 加个 executor + 意图树加个节点；二是**工具自描述**——Tool 自带 name/description/inputSchema，客户端不需要预先知道工具长什么样，启动时动态发现；三是**给 Agent 用**——模型要"看到工具描述 + 按 Schema 传参"才能自主调用。一句话：**MCP 是把业务能力标准化成"模型可发现、可调用、可理解"的工具协议**。

**第二，工具 = executor + 自描述 Schema。** Server 侧每个工具是一个 executor（比如 `WeatherMcpExecutor`），用 Tool 声明 name/description/inputSchema，Schema 里声明**必填、枚举、默认值**——这些不是摆设，后面 LLM 提参全靠它们。工具的复杂性全在 executor 内部，对外只是标准协议。

**第三，LLM 提参 + 三态分流（整个模块最有讲究的部分）。** 用户自然语言和工具 Schema 之间没有结构化映射，所以用 LLM 低温确定性提参（temperature 0.1 / topP 0.3），输出三态：SUCCESS 调工具；NEED_CLARIFICATION 必填缺参 → 注入澄清提示让模型追问（isError=false，作为正文进上下文）；FAILED 协议畸形或值非法 → 注入失败提示。原则一句话："**garbage 永不进工具入参**"——静默丢弃可选/有默认字段会让过滤条件无声扩大，比报错更危险。

**第四，两层并行 + 本地/远程统一 + Agent 桥接。** 子问题之间并行、子问题内多个 MCP 工具并行，单工具异常隔离不拖垮整批；本地/远程统一成 `McpToolExecutor` 接口，注册表按 toolId 存，消费方不知道工具是本地的还是远程的；Agent 侧用 `McpToolBridge` 把注册表桥接成 Agent 原生工具，工具目录走快照 + 指纹缓存，resolve 一次定格、buildToolkit 不回读注册表。

> 📦 涉及的表/存储（面试官问"MCP 工具存哪、表怎么设计的"时展开，平时不用全背）：
>
> 先说结论：**MCP 工具本身是代码注册的，不建独立数据库表**——Server 侧 6 个 executor（天气/工单/销售/联网搜索/资产/请假）在 `McpServerConfig` 里注册成 `SyncToolSpecification`；客户端 `DefaultMcpToolRegistry` 是内存 `Map<String, McpToolExecutor>`，启动时 listTools 发现后 put 进去。与 MCP 直接相关的表只有意图树一张：
>
> | 存储 | 用途 | 关键字段 |
> |----|------|---------|
> | 工具定义与注册（代码，非表） | mcp-server 6 个 executor + `McpServerConfig.tools(toolSpecs)`；客户端 `DefaultMcpToolRegistry` 内存注册表 | Tool(name/description/inputSchema 必填·枚举·默认)；`executorMap.put(toolId, executor)` |
> | `t_intent_node` | 意图树叶子挂 MCP 工具：kind=2 标记 MCP 节点，`mcp_tool_id` 挂工具 ID | `kind`(0 KB / 1 SYSTEM / 2 MCP)、`mcp_tool_id`、`param_prompt_template`(节点级自定义提参模板，Q8) |

讲到这里停一下，面试官大概率会顺着追问：Server 具体怎么暴露一个工具？参数怎么从自然语言提取的？多工具并行在哪？Agent 侧怎么调的？——正好接上 Q2 / Q5 / Q7 / Q9。

## Q1（开场总览）MCP 在你们项目里是怎么用的？为什么选 MCP 协议而不是直接写 Java 方法调用？

**我：** 先讲定位。我们的知识问答平台光有"检索知识库"不够——用户会问"今天上海天气怎么样？""查一下华东区销售数据"这类需要**调用外部业务能力**的问题。这些能力不该写死在 RAG 链路里，所以用 **MCP（Model Context Protocol）** 把工具能力"协议化"。

整体是**三层架构**：

| 层 | 模块 | 职责 |
|---|---|---|
| **Server**（独立进程） | `mcp-server`，跑在 9099 | 暴露 7 个工具：天气查询、销售数据、工单、请假、资产、联网搜索等 |
| **Client**（rag 模块） | `rag/core/mcp/` | 连接 Server、发现工具、注册进工具注册表、执行工具 |
| **消费侧** | RAG 检索链路 + Agent | 意图命中 MCP 节点 → 并行调用工具 → 结果注入上下文 |

> 三层架构和工具流先看这张图：

```text
┌─ 消费侧（调用工具）─────────────────────────────────────┐
│  意图命中 MCP 节点                                      │
│   ├─ 子问题之间并行（ragContextExecutor）                │
│   ├─ 子问题内多工具并行（mcpBatchExecutor）              │
│   └─ 提参：LLM 三态（SUCCESS / NEED_CLARIFICATION /     │
│      FAILED）                                          │
│      SUCCESS → 调工具；缺参 → 注入澄清提示让模型追问       │
│  Agent 侧：McpToolBridge → AgentToolCatalog 快照         │
└─────────────────────────┬───────────────────────────────┘
                          │ JSON-RPC（Streamable HTTP）
                          │ listTools() 发现 / callTool() 调用
┌─ Client（rag/core/mcp/）───────────────────────────────┐
│  McpClientAutoConfiguration（@PostConstruct 启动时连接） │
│  握手 initialize() → listTools() 动态发现               │
│  → McpClientToolExecutor → McpToolRegistry 注册         │
│  本地/远程统一实现 McpToolExecutor 接口                   │
│  连不上 → 跳过该 Server 注册，不阻塞主服务启动            │
└─────────────────────────┬───────────────────────────────┘
                          │ 独立进程 · 协议解耦
┌─ Server（mcp-server 独立进程 · :9099）──────────────────┐
│  Spring Boot + Streamable HTTP（Servlet 挂 /mcp）       │
│  暴露 7 个工具：天气 / 销售 / 工单 / 请假 / 资产 / 联网…   │
│  每个工具 = executor + Tool 声明                         │
│    （name / description / inputSchema：必填·枚举·默认）   │
└─────────────────────────────────────────────────────────┘
```

**为什么用 MCP 而不是硬编码 Java 方法调用**，三个理由：

1. **解耦工具提供方与消费方**：Server 是独立进程，暴露的是**标准协议**（JSON-RPC over HTTP），工具升级、新增不用改主服务。新增一个工具 = Server 加一个 executor + 意图树加一个节点，主服务零改动。
2. **工具自描述**：MCP 的 `Tool` 自带 name/description/inputSchema（JSON Schema）——**客户端不需要预先知道工具长什么样**，启动时 `listTools()` 动态发现。这是"硬编码"做不到的：硬编码是消费方写死提供方签名，MCP 是提供方声明、消费方发现。
3. **给 Agent 用**：Agent 需要"看到工具描述 + 按 Schema 传参"才能自主调用，MCP 的标准定义天然适配（后面 Q9 讲 Agent 桥接）。

一句话：**MCP 是把"业务能力"标准化成"模型可发现、可调用、可理解"的工具协议**。

**面试官追问**：Server 侧具体怎么暴露一个工具？协议层和工具定义长什么样？

## Q2（追问）MCP Server 怎么暴露工具？工具定义包含什么？

**我：** Server 是一个独立 Spring Boot 应用（`McpServerApplication`），核心在 `McpServerConfig`：

- **传输层**：`HttpServletStreamableServerTransportProvider`（MCP 官方 SDK 的 Streamable HTTP transport），注册成 Servlet 挂在 `/mcp` 路径。
- **服务端**：`McpServer.sync(transportProvider).serverInfo("ragent-mcp-server","0.0.1").tools(toolSpecs)` 构建同步服务端——所有工具以 `List<SyncToolSpecification>` 一次性暴露。

**每个工具是一个 executor**（如 `WeatherMcpExecutor`），用一个 `@Bean` 方法返回 `McpServerFeatures.SyncToolSpecification`，它由两部分组成：

> Q2 Server 暴露工具一图看懂：

```text
McpServerApplication（独立 Spring Boot）
   │
   ▼
传输层：HttpServletStreamableServerTransportProvider
   （MCP 官方 SDK Streamable HTTP，注册成 Servlet 挂 /mcp）
   ▼
服务端：McpServer.sync(transportProvider)
   .serverInfo("ragent-mcp-server", "0.0.1")
   .tools(toolSpecs)   ← 所有工具一次暴露
   ▼
每个工具 = 一个 executor（@Bean 返回 SyncToolSpecification）
   ① 工具声明 Tool.builder()
      .name(...) .description(...) .inputSchema(...)
      ├─ properties: city(String)
      │               queryType(enum current|forecast, default current)
      │               days(Integer, default 3)
      ├─ required: ["city"]
      └─ ★ 必填 / 枚举 / 默认值不是摆设
         → 客户端 LLM 提参全靠它们（Q5）
   ② 调用处理 (exchange, request) -> handleCall(request)
      从 request.arguments() 取参数 → 校验 → 执行业务
      → 返回 CallToolResult（成功/失败都带 isError 标记）
   ▼
工具内部是普通业务代码（Sales 多维筛选 / Weather 种子确定性）
   = "复杂性"全在 executor 内部，对外只是标准协议
```

1. **工具声明** `Tool.builder().name(...).description(...).inputSchema(...)`：
   ```java
   properties.put("city", Map.of("type","string","description","城市名称，如北京、上海"));
   properties.put("queryType", Map.of("type","string","enum",List.of("current","forecast"),"default","current"));
   properties.put("days", Map.of("type","integer","default",3));
   JsonSchema inputSchema = new JsonSchema("object", properties, List.of("city"), null, null, null);
   ```
   注意 Schema 里声明了**必填、枚举、默认值**——这些不是摆设，后面客户端 LLM 提参全靠它们（Q5 讲）。
2. **调用处理** `(exchange, request) -> handleCall(request)`：从 `request.arguments()` 取参数、校验、执行业务、返回 `CallToolResult`（成功/失败都带 `isError` 标记）。

工具实现内部是普通业务代码——比如 `SalesMcpExecutor` 按 region/period/product/salesPerson 多维筛选 + 汇总/排名/明细/趋势四种查询，`WeatherMcpExecutor` 按经纬度 + 日期种子生成确定性天气。**工具的"复杂性"全在 executor 内部，对外只是标准协议。**

**面试官追问**：客户端怎么知道有哪些工具？连接和发现是在启动时做的吗？

## Q3（追问）客户端怎么连接 Server、发现并注册工具？

**我：** `McpClientAutoConfiguration`，`@PostConstruct` 时执行：

1. 读配置 `rag.mcp.servers`（application.yaml：`name=default, url=http://localhost:9099`），**每个 server 连一遍**。
2. 建 `HttpClientStreamableHttpTransport` → `McpClient.sync(transport).clientInfo(...)` → **`client.initialize()` 握手**。
3. **`client.listTools()` 发现工具** → 每个工具包一个 `McpClientToolExecutor` → 注册进 `McpToolRegistry`。
4. 日志打"Server [x] 返回 N 个工具"；`@PreDestroy` 关闭所有客户端。

两个关键设计：

**失败降级**：连接 Server 异常只 `log.error` + **跳过该 Server 的工具注册，不阻塞主服务启动**。MCP 是"扩展能力"，工具连不上不该把主服务拖挂——这是"可选依赖"的定位，和会话记忆模块"记忆层永远不作为故障源"是一个思想。

> Q3 Client 连接发现一图看懂：

```text
McpClientAutoConfiguration（@PostConstruct）
   │
   ▼
① 读配置 rag.mcp.servers（name=default, url=http://localhost:9099）
   每个 server 连一遍
   ▼
② HttpServletStreamableHttpTransport → McpClient.sync(transport)
   clientInfo(...) → client.initialize() 握手
   ▼
③ client.listTools() 发现工具
   每个工具包一个 McpClientToolExecutor → 注册进 McpToolRegistry
   日志："Server [x] 返回 N 个工具"
   ▼
@PreDestroy 关闭所有客户端
   ▼
两个关键设计
   失败降级：连接异常 → log.error + 跳过该 Server 注册
     = 不阻塞主服务启动
     （MCP 是"扩展能力"= 可选依赖
       → 和记忆模块"记忆层永不作故障源"同一思想）
   URL 规范化：url.endsWith("/mcp") ? url : url + "/mcp"
     = 容忍配置漏写 /mcp 后缀，减少配错
   ▼
取舍：启动时一次性发现，不做运行时热发现
   工具集基本稳定，启动扫一次够用
   运行期反复 listTools 是浪费
   真有动态需求走"意图树配置驱动"（Q9）
```

**URL 规范化**：`serverUrl.endsWith("/mcp") ? url : url + "/mcp"`——容忍配置里漏写 `/mcp` 后缀，减少配错。

这里有个取舍：发现是**启动时一次性**的。为什么不做运行时热发现？因为工具集基本稳定、启动扫描一次够用，运行期反复 listTools 是浪费；真有动态需求走的是"意图树配置驱动"（Q9 讲）。

**面试官追问**：注册进去的工具执行器，统一接口是什么？本地工具和远程工具怎么共存？

## Q4（追问）工具注册表（McpToolRegistry）的设计

**我：** 核心是 `McpToolRegistry` 接口 + `DefaultMcpToolRegistry` 实现，把"工具执行器"统一抽象成 `McpToolExecutor`：

> Q4 工具注册表一图看懂：

```text
McpToolRegistry 接口 + DefaultMcpToolRegistry
   │
   ▼
统一抽象 McpToolExecutor
   ├─ Tool getToolDefinition()          工具自描述（给 LLM 看）
   ├─ CallToolResult execute(params)    执行
   └─ default getToolId() = name()
   ▼
本地/远程统一：McpClientToolExecutor（远程，调 McpSyncClient.callTool）
   也实现同一接口 → Map<String, McpToolExecutor> 按 toolId 存
   消费方根本不知道工具本地还是远程
   = 接新工具 = 注册一个实现
   ▼
两种注册来源
   ① 自动发现：@PostConstruct 遍历容器里所有 McpToolExecutor Bean
   ② 远程动态注册：McpClientAutoConfiguration 发现的远端工具 register()
   ▼
重复注册：executorMap.put 已存在 → log.warn("工具已存在，已覆盖")
   = 同名工具后者覆盖前者，防静默双写
   ▼
远程执行细节：远端调用异常不往外抛
   → 包成 isError=true 的 CallToolResult("远程调用失败: reason")
   = 失败以"工具结果"形态返回，不以异常打断链路
     上层拿 isError 标记自己决定，链路可控
```

```java
public interface McpToolExecutor {
    Tool getToolDefinition();                       // 工具自描述（给 LLM 看）
    CallToolResult execute(Map<String, Object> parameters);  // 执行
    default String getToolId() { return getToolDefinition().name(); }
}
```

**本地/远程统一**：`McpClientToolExecutor`（远程，通过 `McpSyncClient.callTool` 调远端）实现了同一接口。注册表里 `Map<String, McpToolExecutor>` 按 toolId 存，**消费方根本不知道工具是本地的还是远程的**——这层抽象让"接新工具 = 注册一个实现"。

**两种注册来源**（`DefaultMcpToolRegistry` 初始化）：
1. **自动发现**：`@PostConstruct` 遍历 Spring 容器里所有 `McpToolExecutor` Bean 注册（未来本地工具也走这）。
2. **远程动态注册**：`McpClientAutoConfiguration` 发现的远端工具 `register()` 进来。

**重复注册处理**：`executorMap.put` 已存在会 `log.warn("工具已存在，已覆盖")`——同名工具后者覆盖前者，防静默双写。

**远程执行器 `McpClientToolExecutor.execute` 还有个细节**：远端调用异常不往外抛，而是包成 `isError=true` 的 `CallToolResult`（"远程调用失败: reason"）。**调用失败以"工具结果"形态返回，而不是以异常打断链路**——上层拿到 isError 标记自己决定怎么处理，链路可控。

**面试官追问**：工具参数怎么来？用户说"查华东区销售"，`region=华东` 这种参数是硬匹配还是怎么提取的？

## Q5（追问）参数提取——为什么用 LLM 提参？三态结果怎么设计？

**我：** 这是整个 MCP 模块最复杂、也最有讲究的部分。用户自然语言（"帮我看看华东区这个月的销售排名"）和工具 Schema（region/period/product...）之间**没有结构化映射**，字段又多、有默认值，所以用 **LLM 从用户问题提取参数**（`LLMMcpParameterExtractor`）。

流程：
1. **无参工具短路**：`inputSchema.properties()` 为空 → 直接 `success(空 map)`，不用调 LLM。
2. **拼 prompt**：system = 提取规则模板，user = 工具定义（`buildToolDefinition` 把 Tool 转成可读的"工具ID/功能描述/参数列表（类型/必填/默认/枚举）"）+ 用户问题。
3. **调 LLM**：temperature 0.1 / topP 0.3 / thinking=false（**确定性任务，低温**），走标准档。
4. **结果分类**：`validateMcpParams` → `parseAndClassify` 逐参数按 Schema 分类，输出**三态**：

> Q5 提参流程一图看懂：

```text
LLMMcpParameterExtractor —— 自然语言 → 结构化入参
   ▼
① 无参工具短路：properties() 为空 → success(空 map)，不用调 LLM
   ▼
② 拼 prompt
     system = 提取规则模板
     user   = 工具定义（buildToolDefinition 转成可读描述）
              + 用户问题
   ▼
③ 调 LLM：temperature 0.1 / topP 0.3 / thinking=false
     确定性任务低温，走标准档
   ▼
④ validateMcpParams → parseAndClassify 逐参数按 Schema 分类 → 三态
   SUCCESS             参数提取正常         → 调用工具
   NEED_CLARIFICATION  必填且无默认的参数缺失/null → 不调用
                      → 注入澄清提示让 LLM 追问
   FAILED              协议畸形(空响应/非对象/解析失败)
                      或值非法(类型/枚举不对)   → 不调用，注入失败提示
   ▼
输出 McpExtractionResult(Status, params, missingRequired)
   三个静态工厂：success / needClarification / failed
   ▼
细节："模型省略 key 与显式输出 null 在实践中不可区分，
        同一业务情形不做分叉"
   （LLM 没输出某字段 vs 输出 null，本质都是"用户没说"
     → 都归 NEED_CLARIFICATION 的 userMissing，不为区分加复杂度）
```

| 结局 | 条件 | 消费端动作 |
|---|---|---|
| `SUCCESS` | 参数提取正常 | 调用工具 |
| `NEED_CLARIFICATION` | **必填且无默认**的参数缺失/null | **不调用**，注入澄清提示让 LLM 追问 |
| `FAILED` | 协议畸形（空响应/非对象/解析失败）或值非法（类型/枚举不对） | **不调用**，注入失败提示 |

设计要点：`McpExtractionResult(Status, params, missingRequired)` 是个 record，三个静态工厂方法 `success/needClarification/failed`。

**"模型省略 key 与显式输出 null 在实践中不可区分，同一业务情形不做分叉"**——这是注释里的原话，我转述下：LLM 没输出某个字段和输出 null，本质都是"用户没说"，归一类（NEED_CLARIFICATION 的 userMissing）处理，不为区分它们加复杂度。

**面试官追问**：你说"值非法判 FAILED"，具体怎么校验？比如 LLM 给了 `region=华中`（不在枚举里）会怎样？

## Q6（追问）提参校验的细节——为什么"垃圾值永不进工具入参"？

**我：** 校验在 `parseAndClassify` + `coerceAndValidate`，原则一句话（注释原话）：**"garbage 永不进工具入参"**。几个具体点：

> Q6 提参校验一图看懂：

```text
原则（注释原话）："garbage 永不进工具入参"
   ▼
1. 字段存在但值非法 → 一律 FAILED，无论必填与否
   例：region=华中（枚举里没有）、days="abc"（类型不对）
   为什么可选/有默认字段非法也判 FAILED 而不静默忽略？
   → 静默丢弃 = 过滤条件被无声移除
     例：用户问"华东区上月的销售"，LLM 输出 region=华中(非法)
       若被静默丢弃 → region 变"查全国"，过滤范围无声扩大
       比报错更危险
   = 宁可让这次调用失败，也不给错误的查询条件
   ▼
2. 类型保守转换 coerceType
   "数字串"→数字 / "true"→布尔 / 数组对象严格 instanceof
   转换失败返回 empty 判非法
   ★ parseDoubleOrNull 拒绝 NaN/Infinity
     Double.parseDouble 会接受 "NaN" 字面量
     但 NaN 不是合法 JSON 数值，必须拒掉
   ▼
3. 枚举容忍 enumContains
   先按值相等，再按字符串形态相等
   → 容忍 LLM 输出 3 而枚举声明是 3（Long vs Integer）字面差异
   ▼
4. 必填无默认缺失 → 归 userMissing（NEED_CLARIFICATION）而非 FAILED
   用户没给(可澄清) vs 模型不守协议(直接失败) = 两种失败语义
   ▼
5. 填默认值 fillDefaults（仅 SUCCESS）
   从 Schema default 补齐缺失可选参数（queryType→current, days→3）
```

**1. 字段存在但值非法 → 一律 FAILED，无论必填与否。** 比如 `region=华中`（枚举里没有）、`days="abc"`（类型不对）。为什么连**可选/有默认**字段非法也判 FAILED 而不静默忽略？注释写得很透：**"静默丢弃可选/有默认字段会让过滤条件被无声移除"**——比如用户问"华东区上月的销售"，LLM 输出 `region=华中`（非法）若被静默丢弃，`region` 就变"查全国"，**过滤范围无声扩大**，比报错更危险。宁可让这次调用失败，也不给错误的查询条件。

**2. 类型保守转换**：`coerceType` 按 Schema type 做转换——"数字串"→数字、"true"→布尔、数组/对象严格 instanceof；转换失败返回 empty 判非法。**注意 `parseDoubleOrNull` 拒绝 NaN/Infinity**：`Double.parseDouble` 会接受 `"NaN"` 字面量，但 NaN 不是合法 JSON 数值，必须拒掉。

**3. 枚举容忍**：`enumContains` 先按值相等，再按**字符串形态**相等——容忍 LLM 输出 `3` 而枚举声明是 `3`（Long vs Integer）这类字面差异。

**4. 必填无默认缺失 → 归 userMissing（NEED_CLARIFICATION）而非 FAILED**：这是"用户没给"和"模型不守协议"的区分——前者是用户真没提供、可澄清；后者是模型行为异常、直接失败。

**5. 填默认值**：仅 SUCCESS 才 `fillDefaults`，从 Schema 的 `default` 字段补齐缺失的可选参数——比如 `queryType` 默认 `current`、`days` 默认 `3`。

**面试官追问**：那工具调用本身呢？多个 MCP 工具是一次性全调，还是一个一个调？你说的"多工具并行"具体在哪？

## Q7（追问）多工具并行调用是怎么做的？

**我：** 在 `RetrievalEngine`。先讲整体链路：意图分类结果按 kind 分成 **KB 组**和 **MCP 组**（`NodeScoreFilters.kb/mcp`），每个子问题 `buildSubQuestionContext` 里 KB 走 `retrieveAndRerank`（多路检索），MCP 走 `executeMcpAndMerge`。

**"多工具并行"体现在两层**：

> Q7 多工具并行一图看懂：

```text
RetrievalEngine —— 意图按 kind 分组
   ├─ KB 组  → retrieveAndRerank（多路检索）
   └─ MCP 组 → executeMcpAndMerge
   ▼
"多工具并行"两层
   ① 子问题之间并行
      所有子问题 CompletableFuture.supplyAsync(..., ragContextExecutor)
   ② 子问题内部、多 MCP 工具并行
      executeMcpTools 对每个意图 supplyAsync(..., mcpBatchExecutor)
      "查天气"+"查销售" 并行调多个工具，互不等待
      = 串行会线性累加远端 HTTP 延迟，并行只取最慢的
        （和多路检索"多通道并行"、问题重写"多子问题并行"同一思想）
   ▼
并行 + 隔离
   单工具异常隔离：每个 task 内 try/catch
     → 异常包成 isError=true 的 CallToolResult，不拖垮整批
   结果按 toolId 分组：Collectors.groupingBy(ToolOutput::toolId)
     → 同工具多次调用归一起，供 formatMcpContext 格式化
   ▼
每个工具调用点 executeSingleMcpTool 三件事
   从注册表拿 executor
   → mcpParameterExtractor.extractParameters(question, tool, 节点自定义提参prompt)
   → 按三态分流（Q8）
```

1. **子问题之间并行**：所有子问题 `CompletableFuture.supplyAsync(..., ragContextExecutor)` 并行构建上下文。
2. **子问题内部、多个 MCP 工具并行**：`executeMcpTools` 对 `mcpIntentScores` 里每个意图 `supplyAsync(..., mcpBatchExecutor)`——**一个子问题命中多个 MCP 意图（比如"查天气"+"查销售"）就并行调多个工具**，互不等待。工具多 → 每个工具一次远端 HTTP 调用，串行会线性累加延迟，并行只取最慢的（和多路检索"多通道并行"、问题重写"多子问题并行"是同一个思想）。

并行+隔离的两个细节：
- **单工具异常隔离**：每个 task 内 `try/catch`，异常包成 `isError=true` 的 `CallToolResult`，**不会让一个工具挂了拖垮整批**。
- **结果按 toolId 分组**：`Collectors.groupingBy(ToolOutput::toolId)`，同工具的多次调用结果归一起，供 `formatMcpContext` 格式化。

**每个工具的调用点 `executeSingleMcpTool` 做三件事**：从注册表拿 executor → `mcpParameterExtractor.extractParameters(question, tool, 意图节点的自定义提参prompt)` → 按三态分流（Q8 讲）。

**面试官追问**：NEED_CLARIFICATION 时"注入澄清提示"是什么意思？为什么不直接报错让用户重说？

## Q8（追问）三态分流——NEED_CLARIFICATION 为什么不报错？

**我：** 这是"参数不齐"的正确处理姿势。用户说"查一下销售数据"但没说哪个地区哪个时段——如果直接报错"参数缺失"，体验是硬打断；如果编一个 `region=华东` 去查，就是**编造参数**，比不查更糟。

> Q8 三态分流一图看懂：

```text
executeSingleMcpTool 按三态分流（switch）
   SUCCESS            → executor.execute(params)       真正调远端
   NEED_CLARIFICATION → clarificationResult(toolId, missing) 注入澄清提示
   FAILED             → extractionFailedResult(toolId) 注入失败提示
   ▼
NEED_CLARIFICATION 的巧妙之处：isError=false
   生成提示："调用工具【sales_query】需要参数：region，
              但用户问题中未提供。请在回答中主动向用户询问这些信息，
              不要编造。"
   isError=false → 作为"正文"进上下文（而非"工具调用失败"段）
   → LLM 看到提示后，会在回答里主动向用户追问
     "请问您要查哪个地区？"
   = "让模型补全交互"，而不是"系统报错"
   ▼
对比 FAILED（isError=true）：协议畸形/值非法是系统性问题
   （不是用户少说一句）→ 进"工具调用失败"段
   LLM 如实说明工具不可用即可
   = 三态是三种不同的失败语义，各进上下文的不同位置
   ▼
细节：提参 customPromptTemplate 可在意图树节点级配置
   （IntentNode.paramPromptTemplate）
   → 不同业务工具挂不同提参提示词，让 LLM 更懂字段语义
```

所以 `executeSingleMcpTool` 按三态分流：

```java
return switch (extraction.status()) {
    case SUCCESS -> executor.execute(extraction.params());          // 真正调远端
    case NEED_CLARIFICATION -> clarificationResult(toolId, missing); // 注入澄清提示
    case FAILED -> extractionFailedResult(toolId);                  // 注入失败提示
};
```

**NEED_CLARIFICATION 的巧妙之处在 `isError=false`**（`clarificationResult`）：
- 生成一条提示文本："调用工具【sales_query】需要参数：region，但用户问题中未提供。请在回答中主动向用户询问这些信息，**不要编造**。"
- **`isError=false` 让它作为"正文"进上下文**（而不是"工具调用失败"段）——LLM 看到这条提示后，会在回答里**主动向用户追问**"请问您要查哪个地区？"。这是"让模型补全交互"而不是"系统报错"。

对比 `FAILED`（`isError=true`）——协议畸形/值非法是**系统性问题**（不是用户少说一句），进"工具调用失败"段，LLM 如实说明工具不可用即可。**三态是三种不同的失败语义，各自进上下文的不同位置。**

还有一个细节：提参的 customPromptTemplate 可以**在意图树节点级配置**（`IntentNode.paramPromptTemplate`）——不同业务工具可以挂不同的提参提示词，让 LLM 更懂这个工具的字段语义。

**面试官追问**：MCP 工具在 Agent 侧是怎么被调用的？和 RAG 检索链路里的调用有什么不同？

## Q9（追问）Agent 侧桥接——McpToolBridge 和工具目录快照

**我：** 除了 RAG 检索链路直接调 MCP，我们还把 MCP 工具**桥接给主 Agent**（ReAct 模式）自主调用。核心是 `McpToolBridge implements AgentTool`——把 `McpToolExecutor` 适配成 AgentScope 框架的原生工具：

> Q9 Agent 桥接一图看懂：

```text
McpToolBridge implements AgentTool
   ├─ 描述优先意图树：getDescription() 优先取意图树节点 descriptionOverride
   │    空则回落 MCP 服务端原始描述
   │    = 面向 Agent 的路由描述由业务配置，比工具作者写的更懂路由语义
   ├─ 异步执行：callAsync = Mono.fromCallable(...).subscribeOn(boundedElastic)
   │    不阻塞 Agent 主循环
   ├─ 结果标准化：buildResult 把 CallToolResult → ToolResultBlock
   │    （id/name/output/state），失败标记 ERROR 状态
   └─ 只读性声明：isReadOnly() 透传 readOnlyHint，缺省 false
        注释理由："猜错的两个方向不对等：
        把写工具当只读会放过重复副作用"
        → 没声明只读就按写工具处理 = 保守判断
   ▼
AgentToolCatalog 快照机制
   ├─ resolve() 一次性定格：意图树配置的 mcpToolId ∩ 注册表 求交集
   │    配了但当前没执行器 → unavailableToolIds（只打 warn）
   │    产出 ResolvedCatalog（bindings + displayNames + fingerprint 指纹）
   ├─ buildToolkit(catalog) 从快照建 Toolkit，过程中不回读注册表
   │    = "结果与快照指纹必然一致"
   │    指纹用于缓存：同指纹 Toolkit 复用
   │    重建前所有请求看到同一份工具集，避免每请求重新解析注册表
   └─ mcpToolCount() 探活：不走整份 resolve
        "探活不该被知识库工具声明缺失连坐"，只看 MCP 绑定数
   ▼
为什么 Agent 走桥接而不是 AgentScope 自带 MCP 客户端
   注释："SDK 版本被压制" → 客户端自己维护（rag/core/mcp/）
   桥接复用这份资产，工具注册表是唯一真源，避免两套客户端双轨漂移
```

- **描述优先意图树**：`getDescription()` 优先取意图树节点配置的 descriptionOverride，空则回落 MCP 服务端原始描述——**面向 Agent 的路由描述由业务配置，比工具作者写的更懂路由语义**。
- **异步执行**：`callAsync` 用 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`，不阻塞 Agent 主循环。
- **结果标准化**：`buildResult` 把 `CallToolResult` 转成 Agent 的 `ToolResultBlock`（id/name/output/state），失败标记 `ERROR` 状态。
- **只读性声明**：`isReadOnly()` 透传 MCP 的 `readOnlyHint`，**缺省 false**——注释理由：**"猜错的两个方向不对等：把写工具当只读会放过重复副作用"**。工具没声明只读就按写工具处理，这是保守判断。

**Agent 侧的工具目录**（`AgentToolCatalog`）值得单独讲，它有个**快照机制**：
- `resolve()` 一次性定格：意图树配置的 mcpToolId 与注册表**求交集**，配了但当前没执行器的工具收进 `unavailableToolIds`（只打 warn）；产出 `ResolvedCatalog`（含 bindings + displayNames + **fingerprint 指纹**）。
- `buildToolkit(catalog)` **从快照建 Toolkit，过程中不回读注册表**——"结果与快照指纹必然一致"。指纹用于缓存：同一指纹的 Toolkit 复用，重建前所有请求看到同一份工具集，避免每次请求重新解析注册表。
- `mcpToolCount()` 做探活：**不走整份 resolve**——"探活不该被知识库工具声明缺失连坐"，探活只看 MCP 绑定数。

**为什么 Agent 走桥接而不是直接用 AgentScope 自带 MCP 客户端**：注释写明"SDK 版本被压制"——我们的 MCP 客户端是自己维护的（`rag/core/mcp/`），桥接复用这份资产，工具注册表是唯一真源，避免两套客户端双轨漂移。

**面试官继续追问**：MCP 工具"调对没有"，你的端到端评测为什么没给这模块单独出分？

## Q10（追问）评测数据的边界：MCP 为什么不在端到端评测集里、工具调用正确性怎么验证

**我：** 这是 MCP 模块最诚实的一个边界，我主动讲清楚：

**第一，评估集的结构里根本没有"工具调用"的标注。** ragenteval 的每条样本标注的是问答目标（`query / intent_l2 / expected_doc_ids / ground_truth`），**没有 `tool_calls_gold`**——没有"这条问题应该调哪个工具、参数该是什么"的期望答案。没有 gold 就出不了准确率指标，这是评测集设计时的取舍：给每个工具、每个典型参数组合都标注期望调用，成本远高于问答标注，而收益有限——**因为工具调错的后果，其实会被问答指标接住**：该调天气工具却调了别的，答案必然错，`faithfulness` / `answer_relevancy` 自然掉。工具调用是"影子评测"的又一个例子，正确性通过端到端答案好坏间接暴露。

**第二，虽然没有工具 gold，但评测旁路有 `has_mcp` 字段，能验证"链路真的走了 MCP"**。`EvalRecord` 的旁路数据（`/rag/eval`）会记 `has_mcp` / `intent_pred` 这些观测值。所以我能回答两类问题：**"该走 MCP 的样本有没有真的走"**（对照 requires_mcp 场景，误走/漏走会被发现）和 **"不该走 MCP 的有没有误调"**（这其实和检索的行为指标一个思路——过召回的镜像）。它不给工具调用打分，但能保证"MCP 参与链路这件事本身没失效"。

**第三，提参正确性没有指标，我用三道工程防线兜底**：① **garbage 永不进工具入参**——解析不到合法参数宁可丢，防止静默扩过滤范围（副作用不可逆）；② **低温确定性提参**——温度拉低，减少提参的随机抖动，让"同一句话每次提参结果稳定"；③ **NEED_CLARIFICATION 注入 isError=false 提示**——提参信息不足时让模型主动追问用户，而不是带着残缺参数硬调。**这三条都不是指标，但每一条都是"工具调用出问题时的可控形态"**——评测测的是"好不好"，工程防线保的是"错了不崩、不静默扩大副作用"。

> 💡 面试官追问（真实被问）：没给工具调用出分，是不是说明 MCP 不重要？
>
> **我：** 反过来理解才对——工具调用比问答更需要防御。问答答错一次只影响这一次；工具调错一次可能产生真实副作用（写操作、范围扩大）。所以我宁可不出"工具选择准确率"这种好看的分数，也要守住三条防线：garbage 不进参、低温确定性、NEED_CLARIFICATION 让模型追问。**评测指标回答"效果如何"，工程防线回答"出错时是否可控"——对工具调用来说，后者优先级更高。**

> 💡 面试官追问：如果让你给 MCP 建一套评测，会怎么建？
>
> **我：** 三个指标，都对照真实工具行为：① **工具选择准确率**——标注 tool_calls_gold（该调 weather 的样本期望调 weather），跑完比对预测工具与 gold；② **提参正确率**——对照 gold 的期望参数 JSON，用结构比对（比 LLM 判分更硬）；③ **冗余调用率**——构造"不需要工具"的样本，看会不会误调，这对应检索的过召回。前两个衡量"调得准"，第三个衡量"别乱调"——三个合起来才是工具调用的完整画像。

> Q10 MCP 评测边界一图看懂：

```text
为什么 MCP 不在端到端评测集里
   评估集标注 = query / intent_l2 / expected_doc_ids / ground_truth
   没有 tool_calls_gold（无"该调哪个工具、参数是什么"的期望）
   → 没有 gold 就出不了准确率指标，是设计取舍
   ▼
但链路有效性可间接验证（影子评测）
   旁路 /rag/eval 记 has_mcp / intent_pred 观测值
   "该走 MCP 的有没有走"（漏走/误走会被发现）
   "不该走的有没有误调"（= 检索行为指标的镜像）
   → 不给工具打分，但保证链路没失效
   ▼
提参正确性没指标 → 三道工程防线兜底
   ① garbage 永不进工具入参：解析不到宁可丢，防静默扩范围
   ② 低温确定性提参：减少随机抖动，同句话结果稳定
   ③ NEED_CLARIFICATION：信息不足让模型追问，不硬调
   = 评测回答"好不好"，防线保"错了不崩、不静默扩大副作用"
```

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里出现的钩子 | 面试官大概率会追 | 你准备好了没 |
|---|---|---|
| "三层架构：Server 独立进程 + Client + 消费侧" | 为什么 Server 独立部署 | ✅ Q1 |
| "MCP 工具自描述，客户端动态发现" | 和硬编码调用的本质区别 | ✅ Q1 |
| "Streamable HTTP + /mcp Servlet" | 传输层怎么选 | ✅ Q2 |
| "Schema 里声明必填/枚举/默认值" | Schema 是提参的依据 | ✅ Q5 |
| "启动时 listTools 一次性发现" | 为什么不做运行时热发现 | ✅ Q3 |
| "连不上只跳过注册，不阻塞启动" | 可选依赖的降级 | ✅ Q3 |
| "本地/远程统一 McpToolExecutor 接口" | 抽象的价值 | ✅ Q4 |
| "远程调用异常包成 isError 不抛出" | 失败形态可控 | ✅ Q4 |
| "低温确定性提参" | LLM 提参的工程约束 | ✅ Q5 |
| "garbage 永不进工具入参" | 静默丢过滤条件的危险 | ✅ Q6（过滤范围无声扩大） |
| "拒绝 NaN/Infinity" | 对宽松解析的防御 | ✅ Q6 |
| "枚举按字符串形态容忍" | Long vs Integer 字面差异 | ✅ Q6 |
| "必填缺失=用户没给，值非法=模型不守协议" | 两种失败语义区分 | ✅ Q5/Q6 |
| "多工具并行 + 单工具异常隔离" | 并行不拖累、挂了不连坐 | ✅ Q7 |
| "NEED_CLARIFICATION 注入 isError=false 提示" | 让模型追问而非报错 | ✅ Q8 |
| "意图树节点级提参 prompt" | 提参模板可配置 | ✅ Q8 |
| "McpToolBridge 描述优先意图树" | 路由描述由业务定 | ✅ Q9 |
| "readOnlyHint 缺省 false" | 写工具当只读的副作用 | ✅ Q9 |
| "工具目录快照 + 指纹缓存" | 每次请求不回读注册表 | ✅ Q9 |
| "探活不走整份 resolve" | 探活不被连坐 | ✅ Q9 |
| "MCP 为什么没进评测集" | 工具调用没有 gold | ✅ Q10（无 tool_calls_gold，被问答指标间接接住） |
| "工具调对没有怎么证明" | 提参正确性怎么验证 | ✅ Q10（has_mcp 旁路 + 三道工程防线） |

---

## 📋 面试建议（对真实面试的 3 条建议）

1. **把"协议化"讲成架构判断，而不是名词堆砌**。面试官问"MCP 是什么"如果只答"MCP 是模型上下文协议"就输了。高分答法是 Q1 的三层架构 + 三个理由（解耦/自描述/给 Agent 用），并点出"工具 Schema 是协议的核心资产"——因为它既给客户端提参当依据、又给 Agent 当路由描述。**把 MCP 当"协议层"讲，而不是"API 调用"。**
2. **"LLM 提参的三态"是你的专属高光**。大多数人聊 MCP 只会说"客户端调 server 的工具"，很少人讲透"自然语言参数怎么变成结构化入参"。`SUCCESS / NEED_CLARIFICATION / FAILED` 三态 + "garbage 永不进工具入参"（静默丢过滤条件会无声扩大范围）+ "必填缺失是用户没给、值非法是模型不守协议"——这是真实实现里踩过坑才有的认知，一定要主动展开。
3. **两个 ⚠️ 钩子提前备好**：①「Agent 侧为什么要自己桥接 MCP 而不是用框架自带客户端」——答"SDK 版本被压制 + 注册表是唯一真源，避免双轨漂移"（McpToolBridge 注释原话）；②「工具目录快照/指纹缓存怎么保证一致性」——答"resolve 一次定格，buildToolkit 从快照建、不回读注册表，指纹必然一致"。这两个点是你在 Agent 侧区分度的来源。

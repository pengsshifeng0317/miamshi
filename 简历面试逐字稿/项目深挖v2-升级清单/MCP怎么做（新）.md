# 模拟面试逐字稿：MCP 怎么做（新）

> 简历产出：基于 MCP 协议集成多业务工具，支持多工具并行调用，扩展智能问答与 Agent 能力
> 覆盖模块：MCP（mcp-server 独立进程 + rag/core/mcp 客户端 + agent/tool 桥接 + 意图树）
> 熟练度：🔴
> 最后复习：2026-09-08

> 围绕简历产出——「基于 MCP 协议集成多业务工具，支持多工具并行调用，扩展智能问答与 Agent 能力」——的完整一问一答逐字稿。
> 回答内容基于**当前最新代码**（HEAD ~2026-09-06），是对旧《MCP怎么做.md》的全面升级：工具从 6 个扩到 **11 个并分读/写**，新增 **写操作执行前确认** 与 **技能手册遮蔽** 两条治理链路。
> 包含：开场总览 → 追问 → 追问铺垫，最后附「追问钩子速览」和「面试建议」。

---

## 📦 先对一下版本（老稿 → 新稿，心里要有数）

| 维度 | 旧版（MCP怎么做.md） | 新版（本文） |
|---|---|---|
| Server 工具 | 6 个查询工具（天气/销售/工单/请假/资产/联网） | **10 个 executor 类 / 11 个工具**，8 读 + 3 写 |
| 读写标注 | 无（只有 readOnlyHint 透传，缺省 false） | **强制自报**：启动时 `requireReadOnlyHint` 校验，不声明不让起 |
| 写工具 | 没有（全是查询） | `leave_submit` / `asset_renewal_submit` / `meeting_room_book` |
| 写操作确认 | 没有 | **意图树 require_confirm → Agent 确认卡 → AWAITING_CONFIRM → confirm 端点续跑** |
| 技能 | 没有 | **load_skill + 技能遮蔽**：手册不加载，写工具模型都看不见 |
| McpToolBridge | `implements AgentTool` | **`extends ToolBase`**（接入框架权限检查，readOnly 进构造） |
| Agent 工具目录 | resolve 一次定格 + fingerprint 指纹 | 保留，另加 memory/skill 工具 + ReActAgentProvider **单例复用** |

> 一句话主线（开场先立住）：老版解决「怎么把外部工具能力协议化接进来」，新版解决「**接进来之后怎么管住它**」——查询自由并行、**写操作层层设闸**。

---

## 🎤 开场总览（45 秒版 · 端到端主线，可直接背）

> 面试官问"MCP 在你们项目里是怎么用的？为什么选 MCP？"时，把下面这段**端到端主线**一气呵成讲完——它就是你的开场总览：怎么把工具接进来（装配）→ 只读怎么直调 → 写操作怎么设闸。讲的时候不画图，锚点图只在你被要求"把链路从头捋一遍 / 画一下"时才掏出来。
> 末尾留的钩子（强制自报读写 / LLM 提参三态 / 两层并行 / 写操作确认卡 / 技能遮蔽）正好接 Q2 / Q5 / Q7 / Q10 / Q11。

**我：** 我们做 MCP，是回答一个问题——知识问答平台不能只会"检索知识库"，用户会问"上海今天天气怎么样""查一下华东区销售数据""帮我请个假"，这些要调**外部业务工具**。我们没有把它们写死在 RAG 链路里，而是用 **MCP 协议**接进来，三层角色：**`mcp-server` 独立进程（跑 9099）负责"自报能力 + 执行"；Client 在启动时 `listTools()` 发现注册；消费侧是 Agent + RAG 检索链路，决定怎么用**。这里有个边界要先讲清：Client 和消费侧其实在 bootstrap **同一个 JVM**，真正跨进程的只有到 mcp-server 那一段 `/mcp` HTTP——所以 Client 是 1:1 的协议翻译、没有业务判断，业务判断全在消费侧，这两层职责因此能分开。

为什么这么分层？关键在**工具自描述**：每个工具带 name / description / inputSchema，模型才"看得懂、能按 Schema 传参"自主调用；读写也从这个根上分开——`McpServerConfig.requireReadOnlyHint` 在**启动时**强制每个工具用 annotations 自报只读还是写操作，**不声明整个服务不让起**。现在 Server 是 10 个 executor 类、11 个工具：8 个只读随便调，3 个写操作层层设闸。

调用就分两条路。**只读**：模型读自描述 → 决定调 weather_query → 权限门 `allow` 直通 → 桥把阻塞调用切到 boundedElastic → Client 序列化成 **JSON-RPC 跨 `/mcp`** 打到 mcp-server → 执行完结果回填 → 模型二次推理组织人话 → SSE 推给前端。

**写操作**：模型想调 leave_submit（mcpId） → `McpToolBridge`（`extends ToolBase`）返回 `ask` → **引擎不执行，先停下来**：弹确认卡，消息以 `AWAITING_CONFIRM` 落库挂起。用户点"同意"走 confirm 端点，**从 Agent 状态里取回工具调用原件续跑**——前端只传同意/拒绝，参数篡改不了；点"取消"，`AgentConfirmDenialMiddleware` 把框架那句冷冰冰的 "Permission denied by user" 改写成模型能如实转告用户的话。再叠一道**技能遮蔽**：请假、订会议室这类不看手册就办错的操作，模型在 `load_skill` 之前**根本看不见**这个工具。一句话收束：老版解决"怎么把工具接进来"，新版解决"接进来后怎么管住它"——**查询放开、写操作层层设闸**。

> 🧭 背完上面，心里只留**一张锚点图**——被要求画链路时画这一张就够（主链 + 只读直调 + 写确认两分支）：

```text
 用户提问（前端 → bootstrap，9090/SSE）
      │
      ▼
 bootstrap（消费侧 Agent/检索 + MCP Client，同一 JVM）
      │  模型读工具自描述 → 决定调 MCP 工具
      ▼
 McpToolBridge 权限门（extends ToolBase）
   ├─ 只读(allow) → JSON-RPC /mcp ──► mcp-server(9099) 执行 ──► 结果回填 → 二次推理 → SSE
   │
   └─ 写/需确认(ask) → 引擎停 · 弹确认卡 · AWAITING_CONFIRM 落库
        ├─ 同意：confirm 端点 → 从 Agent 状态取回调用原件 → /mcp ──► mcp-server 执行
        └─ 取消：中间件把 Permission denied 改写 → 如实告知
```

> 这块刻意只留一张图。真被追问，各分支一句话带开：装配怎么发现注册 → Q3/Q4；LLM 提参为什么三态 → Q5/Q6；多工具怎么并行 → Q7；桥为什么从 implements 改成 extends ToolBase → Q9；确认续跑的防篡改细节 → Q10；技能遮蔽同生共死 → Q11；Server 启动强校验怎么做的 → Q2。

> 📦 涉及的表/存储（面试官问"MCP 工具存哪、确认状态存哪"时展开）：
>
> 先说结论：**MCP 工具本身是代码注册的**——Server 侧 11 个工具在 executor 里注册成 `SyncToolSpecification`；客户端 `DefaultMcpToolRegistry` 是内存 `Map`。和 MCP 直接相关的表是意图树一张，确认状态挂在 Agent 消息上：
>
> | 存储 | 用途 | 关键字段 |
> |----|------|---------|
> | 工具定义与注册（代码，非表） | mcp-server 10 个 executor 注册 11 个工具；客户端内存注册表 | `Tool(name/description/inputSchema + annotations.readOnlyHint)`；`executorMap.put(toolId, executor)` |
> | `t_intent_node` | 意图树叶子挂 MCP 工具：kind=2、`mcp_tool_id` 挂工具 ID；写工具节点勾 `require_confirm` | `kind`(0 KB/1 SYSTEM/2 MCP)、`mcp_tool_id`、`require_confirm`(默认 0)、`param_prompt_template` |
> | `t_agent_message` | Agent 确认卡落库：`message_status=AWAITING_CONFIRM` 挂起，续跑后改回 NORMAL；`blocks` 里存 confirm 块（pending/approved/denied/expired） | `message_status`(扩到 VARCHAR(32))、`blocks` JSON |

> DB 升级脚本：`upgrades/v2.0.0/260830_agent_tool_confirm.sql`（`t_intent_node` 加 `require_confirm SMALLINT NOT NULL DEFAULT 0`；`t_agent_message.message_status` 扩宽）+ `260903_agent_skill.sql`。

---

## Q1（开场总览）MCP 在你们项目里是怎么用的？为什么选 MCP 协议而不是直接写 Java 方法调用？

**我：** 先讲定位。我们的知识问答平台光有"检索知识库"不够——用户会问"今天上海天气怎么样？""查一下华东区销售数据""帮我提个年假申请"这类需要**调用外部业务能力**的问题。这些能力不该写死在 RAG 链路里，所以用 **MCP（Model Context Protocol）** 把工具能力"协议化"。

整体是**三层架构**：

| 层 | 模块 | 职责 |
|---|---|---|
| **Server**（独立进程） | `mcp-server`，跑在 9099 | 暴露 **11 个工具**：天气 / 销售 / 工单 / 请假 / 资产 / 会议室 / 当前日期 / 联网搜索，8 读 + 3 写 |
| **Client**（rag 模块） | `rag/core/mcp/` | 连接 Server、发现工具、注册进工具注册表、执行工具 |
| **消费侧** | RAG 检索链路 + Agent | 意图命中 MCP 节点 → 并行调用 → 结果注入上下文；写操作走 Agent 确认卡 |

**Server 现在的完整工具清单（心里要有数，面试官会点名抽查）：**

| executor 类 | 工具 ID | 读写 | 业务 |
|---|---|---|---|
| WeatherMcpExecutor | weather_query | 读 | 城市天气（含 20 城经纬度 + 季节性随机） |
| SalesMcpExecutor | sales_query | 读 | 销售多维统计/排名/趋势 |
| TicketMcpExecutor | ticket_query | 读 | 客户工单查询 |
| LeaveMcpExecutor | leave_query | 读 | 假期余额/请假记录查询 |
| LeaveApplyMcpExecutor | leave_submit | **写** | 提交请假申请 |
| AssetMcpExecutor | asset_query | 读 | 名下 IT 资产查询 |
| AssetRenewalMcpExecutor | asset_renewal_submit | **写** | 提交资产换新工单 |
| MeetingRoomMcpExecutor | meeting_room_query + meeting_room_book | 读 + **写** | 一个 executor 出两个工具：查空闲 + 订会议室 |
| CurrentDateMcpExecutor | current_date | 读 | 当前日期/星期/时区（相对日期换算） |
| YouComSearchMcpExecutor | youcom_search | 读 | You.com 联网搜索 |

> 细节加分项：**MeetingRoomMcpExecutor 一个类出两个 @Bean 工具**（query + book）。两个工具共享同一份 `occupiedSlots`——占用时段按「会议室 + 日期」定种子确定性生成，**同一天反复查结果稳定，预订才能据此判冲突**。设计上让"查完再订不撞上随机结果"。

**为什么用 MCP 而不是硬编码 Java 方法调用**，三个理由：

1. **解耦工具提供方与消费方**：Server 是独立进程，暴露的是**标准协议**（JSON-RPC over HTTP），工具升级、新增不用改主服务。新增一个工具 = Server 加个 executor + 意图树加个节点。
2. **工具自描述**：MCP 的 `Tool` 自带 name/description/inputSchema——**客户端不需要预先知道工具长什么样**，启动时 `listTools()` 动态发现。
3. **给 Agent 用**：Agent 需要"看到工具描述 + 按 Schema 传参"才能自主调用，MCP 的标准定义天然适配。

一句话：**MCP 是把"业务能力"标准化成"模型可发现、可调用、可理解"的工具协议；而第二版加上了"可管住"——读写分层、写操作确认、技能遮蔽。**

**面试官追问**：你说工具要自报读写、启动强制校验？Server 侧具体怎么暴露一个工具？

## Q2（追问）MCP Server 怎么暴露工具？"强制自报读写"具体指什么？

**我：** Server 是一个独立 Spring Boot 应用，核心在 `McpServerConfig`：

- **传输层**：`HttpServletStreamableServerTransportProvider`（MCP 官方 SDK 的 Streamable HTTP transport），注册成 Servlet 挂在 `/mcp`。
- **服务端**：`McpServer.sync(transportProvider).serverInfo("ragent-mcp-server","0.0.1").tools(toolSpecs)`，所有工具以 `List<SyncToolSpecification>` 一次性暴露。

**每个工具是一个 executor**，用一个 `@Bean` 方法返回 `McpServerFeatures.SyncToolSpecification`，由工具声明 + 调用处理两部分组成。工具声明里除了 name/description/inputSchema，**多了一样东西——annotations（读写标注）**：

```java
// McpToolAnnotations 预定义两种
public static final ToolAnnotations READ_ONLY = new ToolAnnotations(null, true, false, true, false, null);
public static final ToolAnnotations WRITE      = new ToolAnnotations(null, false, false, false, false, null);

// WeatherMcpExecutor（读）：查询工具
return Tool.builder()
        .name("weather_query")
        .description("查询城市天气信息，支持查看当前实时天气和未来多天天气预报…")
        .inputSchema(inputSchema)
        .annotations(McpToolAnnotations.READ_ONLY)
        .build();

// LeaveApplyMcpExecutor（写）：提交请假申请
return Tool.builder()
        .name("leave_submit")
        .description("为当前登录员工提交请假申请，提交后进入直属经理审批流程。"
                + "本工具会产生真实业务副作用，日期与事由必须来自用户明确说明…")
        .inputSchema(inputSchema)
        .annotations(McpToolAnnotations.WRITE)   // ← 写工具，readOnlyHint=false
        .build();
```

**强制校验**在 `McpServerConfig` 里：构建 `McpSyncServer` 前先跑 `requireReadOnlyHint(toolSpecs)`——

```java
static void requireReadOnlyHint(List<McpServerFeatures.SyncToolSpecification> toolSpecs) {
    List<String> undeclared = toolSpecs.stream()
            .filter(spec -> spec.tool().annotations() == null
                    || spec.tool().annotations().readOnlyHint() == null)
            .map(spec -> spec.tool().name())
            .toList();
    if (!undeclared.isEmpty()) {
        throw new IllegalStateException(
                "以下 MCP 工具未声明 readOnlyHint，请在 annotations 里显式写明只读或写操作: " + undeclared);
    }
}
```

**为什么强制？** 因为 Agent 侧要判断"这个工具该不该拦下来让用户确认"，而它只能通过工具的 `readOnlyHint` 做判断。如果工具作者忘了声明，Agent 不知道这是个写操作，就会**当成普通查询直接放行**——写操作没确认就执行了。所以把"必须声明"提前到**启动时 fail-fast**，谁漏了谁别想启动，而不是运行期才暴露。

> Q2 Server 暴露工具 + 读写标注一图看懂：

```text
McpServerApplication（独立 Spring Boot）
   │
   ▼
传输层：HttpServletStreamableServerTransportProvider
   （Streamable HTTP，注册成 Servlet 挂 /mcp）
   ▼
服务端：McpServer.sync(transportProvider).serverInfo("ragent-mcp-server","0.0.1")
   └─ 构建前先 requireReadOnlyHint(toolSpecs)   ← 启动强校验
       每个工具必须声明 annotations.readOnlyHint，漏一个就抛异常
   ▼
.tools(toolSpecs)  ← 注入容器里所有 SyncToolSpecification Bean
   ▼
每个工具 = 一个 @Bean SyncToolSpecification = 工具声明 + 调用处理
   ① Tool.builder()
      .name(工具ID) .description(路由语义)
      .inputSchema(properties/required/枚举/默认值/title中文名)
      .annotations(READ_ONLY | WRITE)          ← ★ 读写自报
   ② (exchange, request) -> handleCall(request)
      取参数 → 校验 → 执行业务 → CallToolResult(带 isError)
   ▼
写工具额外动作：校验失败返回中文错误
   "单次请假跨度已超过 30 天上限" / "结束时间不晚于开始时间"
   → 让模型回头问用户，而不是自己补全（Q5 同一个思想）
```

**写工具内部和读工具不一样的地方**：除了执行，它有**业务校验**。比如 `LeaveApplyMcpExecutor.validate()`——假期类型必须是枚举里的、事由必填、日期格式对、结束不早于开始、**单次跨度 ≤ 30 天**（注释：拦住模型把"休一阵子"放大成整年）；`MeetingRoomMcpExecutor` 预订会校验时段落在 09:00-21:00、单次 ≤ 4 小时、**和确定性占有时段做冲突检测，冲突时把已占用时段一起返回**让模型换时段而不是硬试。校验不通过返回**中文错误说明**——目的是让模型回头问用户，而不是自己编参数重试。

**面试官追问**：客户端怎么知道有哪些工具？连接和发现是在启动时做的吗？

## Q3（追问）客户端怎么连接 Server、发现并注册工具？

**我：** `McpClientAutoConfiguration`，`@PostConstruct` 时执行：

1. 读配置 `rag.mcp.servers`（application.yaml：`name=default, url=http://localhost:9099`），**每个 server 连一遍**。
2. 建 `HttpClientStreamableHttpTransport` → `McpClient.sync(transport).clientInfo(...)` → **`client.initialize()` 握手**。
3. **`client.listTools()` 发现工具** → 每个工具包一个 `McpClientToolExecutor` → 注册进 `McpToolRegistry`。
4. 日志打"Server [x] 返回 N 个工具"；`@PreDestroy` 关闭所有客户端。

两个关键设计：

**失败降级**：连接 Server 异常只 `log.error` + **跳过该 Server 的工具注册，不阻塞主服务启动**。MCP 是"扩展能力"，工具连不上不该把主服务拖挂。

**URL 规范化**：`serverUrl.endsWith("/mcp") ? url : url + "/mcp"`——容忍配置漏写 `/mcp` 后缀。

这里有个取舍：发现是**启动时一次性**的。为什么不做运行时热发现？工具集基本稳定、启动扫一次够用，运行期反复 listTools 是浪费；真有动态需求走的是"意图树配置驱动"——**改哪个工具可用，改的是意图树节点，不是重新 listTools**。

**面试官追问**：注册进去的工具执行器，统一接口是什么？本地工具和远程工具怎么共存？

## Q4（追问）工具注册表（McpToolRegistry）的设计

**我：** 核心是 `McpToolRegistry` 接口 + `DefaultMcpToolRegistry`，把"工具执行器"统一抽象成 `McpToolExecutor`：

```java
public interface McpToolExecutor {
    Tool getToolDefinition();                       // 工具自描述（含 annotations，给 LLM 和权限判断看）
    CallToolResult execute(Map<String, Object> parameters);  // 执行
    default String getToolId() { return getToolDefinition().name(); }
}
```

**本地/远程统一**：`McpClientToolExecutor`（远程，通过 `McpSyncClient.callTool` 调远端）实现了同一接口。注册表里 `Map<String, McpToolExecutor>` 按 toolId 存，**消费方根本不知道工具是本地的还是远程的**。

**两种注册来源**：① `@PostConstruct` 自动发现容器里所有 `McpToolExecutor` Bean；② `McpClientAutoConfiguration` 发现的远端工具 `register()` 进来。重复注册 `log.warn("工具已存在，已覆盖")`——同名后者覆盖前者。

**远程执行器 `McpClientToolExecutor.execute` 一个细节**：远端调用异常不往外抛，而是包成 `isError=true` 的 `CallToolResult`（"远程调用失败: reason"）。**失败以"工具结果"形态返回，不以异常打断链路**。

**面试官追问**：工具参数怎么来？用户说"查华东区销售"，`region=华东` 是硬匹配还是怎么提取的？

## Q5（追问）参数提取——为什么用 LLM 提参？三态结果怎么设计？

**我：** 这是整个模块最复杂、也最有讲究的部分。用户自然语言（"帮我看看华东区这个月的销售排名"）和工具 Schema（region/period/product...）之间**没有结构化映射**，所以用 **LLM 从用户问题提取参数**（`LLMMcpParameterExtractor`）。

流程：
1. **无参工具短路**：`inputSchema.properties()` 为空 → 直接 `success(空 map)`，不调 LLM。
2. **拼 prompt**：system = 提取规则模板，user = 工具定义（把 Tool 转成可读的"工具ID/功能描述/参数列表（类型/必填/默认/枚举）"）+ 用户问题。
3. **调 LLM**：temperature 0.1 / topP 0.3 / thinking=false（**确定性任务，低温**）。
4. **结果分类**：`parseAndClassify` 逐参数按 Schema 分类，输出**三态**：

| 结局 | 条件 | 消费端动作 |
|---|---|---|
| `SUCCESS` | 参数提取正常 | 调用工具 |
| `NEED_CLARIFICATION` | **必填且无默认**的参数缺失/null | **不调用**，注入澄清提示让 LLM 追问 |
| `FAILED` | 协议畸形（空响应/非对象/解析失败）或值非法（类型/枚举不对） | **不调用**，注入失败提示 |

设计要点：`McpExtractionResult(Status, params, missingRequired)` 是个 record，三个静态工厂方法 `success/needClarification/failed`。

**"模型省略 key 与显式输出 null 在实践中不可区分，同一业务情形不做分叉"**——注释原话：LLM 没输出某字段和输出 null，本质都是"用户没说"，归一类（NEED_CLARIFICATION）处理。

**面试官追问**：你说"值非法判 FAILED"，具体怎么校验？比如 LLM 给了 `region=华中`（不在枚举里）会怎样？

## Q6（追问）提参校验的细节——为什么"垃圾值永不进工具入参"？

**我：** 校验在 `parseAndClassify` + `coerceAndValidate`，原则一句话（注释原话）：**"garbage 永不进工具入参"**。几个具体点：

1. **字段存在但值非法 → 一律 FAILED，无论必填与否**。比如 `region=华中`（枚举里没有）、`days="abc"`（类型不对）。为什么**可选/有默认**字段非法也判 FAILED 而不静默忽略？注释写得很透：**"静默丢弃可选/有默认字段会让过滤条件被无声移除"**——用户问"华东区上月的销售"，LLM 输出 `region=华中`（非法）若被静默丢弃，`region` 就变"查全国"，**过滤范围无声扩大**，比报错更危险。宁可让这次调用失败，也不给错误的查询条件。
2. **类型保守转换** `coerceType`：按 Schema type 转换——"数字串"→数字、"true"→布尔、数组/对象严格 instanceof；失败返回 empty 判非法。**注意 `parseDoubleOrNull` 拒绝 NaN/Infinity**：`Double.parseDouble` 会接受 `"NaN"` 字面量，但 NaN 不是合法 JSON 数值，必须拒掉。
3. **枚举容忍** `enumContains`：先按值相等，再按字符串形态相等——容忍 LLM 输出 `3` 而枚举声明是 `3`（Long vs Integer）字面差异。
4. **必填无默认缺失 → 归 userMissing（NEED_CLARIFICATION）而非 FAILED**："用户没给（可澄清）"和"模型不守协议（直接失败）"是两种失败语义。
5. **填默认值 `fillDefaults`**：仅 SUCCESS 才从 Schema default 补齐缺失可选参数（queryType→current、days→3）。

> 面试官若追问"**提参 FAILED 之后有没有重试/降级？**"——
> 答：**没有自动重试**。FAILED 是"模型不守协议 / 值非法"的系统性问题，重试只是再花一次 LLM 调用、大概率还错。我们的处理是**直接不调工具**，注入一条 `isError=true` 的失败提示进上下文，让模型如实告诉用户"这个工具暂时用不了"。工具结果不能带病生成，宁缺勿滥。真正的"补全交互"只发生在 NEED_CLARIFICATION 那条路上（Q8）。

**面试官追问**：那工具调用本身呢？多个 MCP 工具是一次性全调还是一个一个调？"多工具并行"具体在哪？

## Q7（追问）多工具并行调用是怎么做的？

**我：** 在 `RetrievalEngine`。意图分类结果按 kind 分成 **KB 组**和 **MCP 组**（`NodeScoreFilters.kb/mcp`，MCP 组只看 kind=2 且 mcpToolId 非空），每个子问题 KB 走 `retrieveAndRerank`，MCP 走 `executeMcpAndMerge`。

**"多工具并行"体现在两层**：

1. **子问题之间并行**：所有子问题 `CompletableFuture.supplyAsync(..., ragContextExecutor)` 并行构建上下文。
2. **子问题内部、多 MCP 工具并行**：`executeMcpTools` 对每个意图 `supplyAsync(..., mcpBatchExecutor)`——一个子问题命中多个 MCP 意图就并行调，**串行会线性累加远端 HTTP 延迟，并行只取最慢的**。

**线程池参数**（`ThreadPoolExecutorConfig`，README 待补点，直接背）：
- `mcpBatchExecutor`：核心线程数 = CPU 核数，最大线程数 = 核数 << 1，keepAlive 60s，**SynchronousQueue**（不排队、直接交给线程），**CallerRunsPolicy**（线程池打满时由提交线程兜底执行，不丢任务），线程名带 TTL。这层只跑"等远端 HTTP"的 MCP 调用，CPU 密集度低，所以配大线程数。
- `ragContextExecutor`：核数 << 2，给子问题并行构建上下文用。

并行 + 隔离的两个细节：
- **单工具异常隔离**：每个 task 内 try/catch，异常包成 `isError=true` 的 `CallToolResult`，**一个工具挂了不拖垮整批**。
- **结果按 toolId 分组**：`Collectors.groupingBy(ToolOutput::toolId)`，同工具多次调用归一起，供 `formatMcpContext` 格式化。

**每个工具的调用点 `executeSingleMcpTool` 做三件事**：从注册表拿 executor（拿不到 log.warn + 返回 null）→ `mcpParameterExtractor.extractParameters(question, tool, 节点自定义提参prompt)` → 按三态分流（Q8）。

**面试官追问**：NEED_CLARIFICATION 时"注入澄清提示"是什么意思？为什么不直接报错让用户重说？

## Q8（追问）三态分流——NEED_CLARIFICATION 为什么不报错？

**我：** 这是"参数不齐"的正确处理姿势。用户说"查一下销售数据"但没说地区时段——直接报错"参数缺失"是硬打断；编一个 `region=华东` 去查就是**编造参数**，比不查更糟。

`executeSingleMcpTool` 按三态分流：

```java
return switch (extraction.status()) {
    case SUCCESS -> executor.execute(extraction.params());          // 真正调远端
    case NEED_CLARIFICATION -> clarificationResult(toolId, missing); // 注入澄清提示
    case FAILED -> extractionFailedResult(toolId);                  // 注入失败提示
};
```

**NEED_CLARIFICATION 的巧妙之处在 `isError=false`**：生成提示"调用工具【sales_query】需要参数：region，但用户问题中未提供。请在回答中主动向用户询问这些信息，**不要编造**。"——`isError=false` 让它作为**正文**进上下文（而不是"工具调用失败"段），LLM 看到后会**主动向用户追问**"请问您要查哪个地区？"。这是"让模型补全交互"而不是"系统报错"。

对比 `FAILED`（`isError=true`）——协议畸形/值非法是**系统性问题**，进"工具调用失败"段，LLM 如实说明工具不可用即可。**三态是三种不同的失败语义，各自进上下文的不同位置。**

还有个细节：提参 customPromptTemplate 可以在**意图树节点级配置**（`param_prompt_template`）——不同业务工具挂不同提参提示词。

**面试官追问**：你说写操作在 Agent 侧要被拦下来确认？模型要调一个写工具，具体会发生什么？

## Q9（追问）Agent 侧桥接——McpToolBridge 为什么从 implements AgentTool 改成 extends ToolBase？

**我：** 新版最大的结构变化在这里。老版 `McpToolBridge implements AgentTool`，只做"把 `McpToolExecutor` 适配成框架能调的工具"；新版改成 **`extends ToolBase`**——因为只有继承框架的 `ToolBase`，才能**接入框架的权限检查体系**（checkPermissions），写操作的确认才有地方挂。

```java
public class McpToolBridge extends ToolBase {

    private final McpToolExecutor executor;   // 底层 MCP 执行器（本地或远程）
    private final boolean requireConfirm;      // 意图树配置的执行前确认开关
    private final Boolean readOnlyHint;        // Server 声明的 readOnlyHint（null=未声明）

    public McpToolBridge(McpToolBinding binding) {
        super(ToolBase.builder()
                .name(binding.toolId())
                .description(resolveDescription(binding))          // 优先 binding，否则 MCP 原始描述
                .inputSchema(buildInputSchema(binding.executor())) // 把 JSON Schema 转成 Agent 输入
                .readOnly(Boolean.TRUE.equals(resolveReadOnlyHint(binding.executor()))));
        ...
    }
```

关键三件事：

**① 权限检查（确认卡的总开关）：**
```java
@Override
public Mono<PermissionDecision> checkPermissions(Map<String, Object> toolInput, PermissionContextState context) {
    if (!needsConfirm()) {
        return Mono.just(PermissionDecision.allow("该工具未配置执行前确认"));
    }
    return Mono.just(PermissionDecision.ask("该操作会产生实际业务影响，执行前需要你确认"));
}

private boolean needsConfirm() {
    return requireConfirm || Boolean.FALSE.equals(readOnlyHint);  // 意图树勾了 OR Server 声明是写
}
```

**needsConfirm 有两个来源，任意一个命中就确认**：意图树节点勾了 `require_confirm`，**或者** Server 的 annotations 声明它是写操作（readOnlyHint=false）。第二个是**漏勾时的兜底**——就算接入方忘了在意图树勾确认，只要工具自报是写操作，Agent 一样会拦。这就是 Q2"强制自报读写"的价值：**双保险，两边都要过关**。

**② 技能遮蔽兜底**：`callAsync` 开头先检查 `maskedBySkill(param)`——如果这个工具属于某个**没加载的技能**（runtimeContext 里的 `MASKED_TOOLS_ATTRIBUTE`），直接拒绝并提示"先调用 load_skill 取手册"（Q11 展开）。

**③ 执行与结果标准化**：`callAsync` 用 `Mono.fromCallable(execute).subscribeOn(Schedulers.boundedElastic())` 异步执行不阻塞 Agent 主循环；`execute` 里 `executor.execute(new HashMap<>(param.getInput()))`；`buildResult` 把 `CallToolResult` 转成 Agent 的 `ToolResultBlock`（id/name/output/state），失败标记 `ERROR`，成功 `SUCCESS`。

**resolveDescription 的一个细节**：`getDescription()` 优先取 binding 的 description（来自意图树节点的描述，业务配置的、更懂路由语义），空才回落 MCP 服务端原始描述。

**面试官继续追问**：这个 bridge 是谁建出来的？Agent 每次请求的工具集怎么来的、会不会每次重新解析？

## Q10（追问）写操作确认卡的完整链路——从意图树 require_confirm 到用户点确认后继续跑？

**我：** 这是新版整个 MCP 模块最完整、最有讲究的一条链路，我分三段讲清楚：**拦下来 → 落库发卡 → 续跑**。

> Q10 写操作确认全链路一图看懂：

```text
【① 拦下来：为什么只有这个工具会被拦】
意图树节点(写工具, require_confirm=true)
   └→ AgentToolCatalog.resolveMcpToolBindings
        意图树 kind=2+mcpToolId ∩ 注册表 = bindings
        requireConfirm = 任一节点 isRequireConfirm()  → McpToolBinding.requireConfirm=true
   └→ buildToolkit 注册 McpToolBridge(requireConfirm=true, readOnlyHint=false)
   └→ 模型要调它 → McpToolBridge.checkPermissions → needsConfirm()=true → ask
   └→ AgentScope 框架收到 ask → 工具块状态 ASKING → 不再执行，等用户裁决

【② 落库发卡：挂起，等用户】
AgentStreamEventBridge.onRequireUserConfirm(event)
   └→ 过滤内部工具 → toConfirmCall(工具ID + displayName + fields 字段展示 + arguments 原始入参)
   └→ confirm 块 status=pending（整卡一次决策，不逐条勾选）
   └→ openToolBlocks 里该工具 running → awaiting（"等待用户裁决"）
   └→ settleAwaitingConfirm：
        persistAssistantMessage(已生成正文, AWAITING_CONFIRM)   ← 消息落库挂起态
        send CONFIRM 事件(AgentConfirmPayload(messageId, title, pending))  ← 前端渲染确认卡
        send DONE "[DONE]"
   （落库失败 → settleUnpersistedConfirm：HINT"系统繁忙，这一步没有执行，请新建会话"）

【③ 续跑：用户点头或拒绝后，从 Agent 状态取原件继续】
POST /agent/v1/chat/confirm  ConfirmRequest(conversationId, messageId, approved)
   └→ AgentChatServiceImpl.confirmPendingTool → guardedStart(runGate 并发锁)
   └→ startConfirmRun：
        resolveConfirmResults：从 agent.getAgentState(context) 取
           最后一条 assistant 消息里 state==ASKING 的 ToolUseBlock（原件！）
           → ConfirmResult(approved, toolCall)
        conversationService.settlePendingConfirm：confirm 块置 approved/denied，消息改回 NORMAL
        launchStream(resumeMsg)：空正文消息仅带 METADATA_CONFIRM_RESULTS 续跑
           用户同意 → 工具真正执行（原参数）
           用户拒绝 → 模型带着"已取消"继续作答（拒绝文案被中间件改写，见下）
```

**几个最容易考的设计点，逐个说：**

**1. 为什么参数从 Agent 状态取原件、前端只传同意/拒绝？** 这是防篡改。用户点击确认时传的只是 `approved=true/false`，待执行的**工具与入参从 `agent.getAgentState(userId, conversationId).getContext()` 里取**——就是当初模型提出、卡在 ASKING 的那个 `ToolUseBlock`。前端永远碰不到真实参数，改不了"请 5 天"成"请 50 天"。接口注释原话：**"工具入参从 Agent 状态取，前端只传同意/拒绝，防止篡改。"**

**2. 确认状态挂起用什么表达？** 消息以 `AWAITING_CONFIRM` 落库——这是 `AgentMessageStatus` 里**唯一的非终态**，续跑成功后会改回 NORMAL。为什么不能用一个普通状态？因为 AWAITING_CONFIRM 意味着"这条会话还没完"，`streamChat` 顶部会查 `hasPendingConfirm`——**上一步操作还在等你确认时不接受新问题**，抛"请先确认或取消"。这防止用户把确认卡晾着、又去问别的问题，导致框架状态错乱。

**3. confirm 块怎么结算？** `settleConfirmBlock` 找到消息里 status=pending 的 confirm 块，用户同意置 `approved`、拒绝置 `denied`、超时/状态丢了置 `expired`（`expirePendingConfirm` 兜底，防止会话一直卡死），然后消息改回 NORMAL 解除对新提问的阻塞。卡片有了终态，前端的"待确认"角标才会消失。

**4. 用户拒绝之后，模型怎么措辞？** 这里有个隐藏坑——AgentScope 框架被拒时返回的拒绝结果是**英文 "Permission denied by user"**，模型照抄这句话回复用户会非常生硬。我们用 `AgentConfirmDenialMiddleware`（一个中间件）在消息进模型前把它改写掉：

```java
// 只改写"用户取消"的拒绝（ToolResultState.DENIED + 文案精确匹配），规则拒绝不动
private static final String DENIAL_EXPLANATION = """
        用户在确认卡片上点了取消，这次操作没有执行。这不是权限不足，也不是系统故障。
        请如实告诉用户操作已取消；等用户确认后再重新发起。""";
```

这样模型收到的是一条中文语义说明，会自然地回复"好的，请假申请没有提交，需要的话您可以再告诉我"。**注意这个中间件排在被压缩中间件之后**——被压进摘要的那条拒绝结果已经不在列表里，改写自然跳过。

**面试官追问**：这个确认只发生在 Agent 会话里。有没有更"重"的写工具，光确认还不够、还要按固定流程办？

## Q11（追问）技能遮蔽——为什么有的写工具模型连"看"都看不到？load_skill 是干嘛的？

**我：** 确认卡解决的是"**同不同意**"，技能解决的是"**会不会办**"。有些写操作有**固定前置步骤、不看手册就容易办错**——比如请假申请，正规流程是**先查假期余额、确认额度够，再提交**；订会议室是**先查空闲、拿到会议室 ID 和时段，再预订**。如果模型没看过手册就直接调 `leave_submit`，它很可能跳过查余额直接提交，或者拿一个编造的会议室 ID 去订。

所以我们加了**技能（Skill）**：把这类写工具挂进技能的 `tool-ids`，**加载手册之前，从模型视野里把工具整个遮掉**。

**配置长这样**（`skills/010-leave-apply.properties` 注释写得很清楚）：

```properties
# tool-ids 是「加载手册后才解锁的工具」，不是「手册会用到的工具」：只放不看手册就会办错的写操作，
# 模型必须先取手册再动手。leave_query 只看自己的参数说明就能调对，用户单独问「还剩几天年假」时
# 不该先加载一份业务手册，所以不挂进 tool-ids，正文按名字引用即可
skill-code=leave_apply
name=请假办理
description=员工要提交请假申请时用这个技能；只查假期余额、只问制度的不用加载
content-file=skills/leave-apply.md
tool-ids=leave_submit        # ← 只放写工具
enabled=true
```

同理 `meeting_room_booking` 的 `tool-ids=meeting_room_book`。**关键设计：tool-ids 只放"不看手册就会办错的写操作"，只读查询工具（leave_query/meeting_room_query/current_date）不挂**——它们自描述够用，单独问"还剩几天年假"不该被逼着先加载一份业务手册。

**遮蔽怎么实现？** `AgentSkillMaskingMiddleware`（排在中间件最内层 order=0，看到的是压缩与改写后的最终消息列表）：

```text
记 G = 所有启用技能 tool-ids 的并集
    L = 上下文里已加载技能 tool-ids 的并集
本轮遮蔽 = G − L
   → 不在任何 tool_ids 里的工具始终可见
   → 一个工具可挂多份手册，任一加载即放行
```

每次推理前，中间件 `maskedTools(input.messages())`：
- 从上下文消息里挑出 `load_skill` 的成功结果（ToolResultState.SUCCESS + 未被压缩驱逐 + metadata 带 `ragent_loaded_skill` 标记）→ 得到"已加载"的技能码集合 L；
- 对每个"启用了但没加载"的技能，把它的 tool-ids 记进遮蔽映射 `toolId → skillCode`；
- 已加载技能的 tool-ids 从遮蔽里**移除**（放行）；
- 把遮蔽映射放进 `runtimeContext`（`MASKED_TOOLS_ATTRIBUTE`），**并把遮蔽的工具从本轮传给模型的工具列表里过滤掉**——模型这轮根本"看不到"能调 leave_submit。

**两个"同生共死"的设计，是重点加分项：**

1. **遮蔽只认上下文里那条 load_skill 结果，不认别的。** 上下文压缩把手册正文带走了，工具就跟着收回——中间件注释原话："手册被压缩带走时工具一并收回，两者同生共死"。这保证模型不会在手册已经不在上下文里时，还凭记忆里的旧调用去硬调工具。
2. **双保险兜底。** 就算遮蔽漏了（比如模型照着历史里的旧工具调用硬闯），`McpToolBridge.callAsync` 开头的 `maskedBySkill` 检查会再拦一次，返回"这个工具属于技能 X，手册还没加载，本次调用没有执行，请先调用 load_skill"。

**load_skill 工具本身**（`SkillLoadTool`）是个**普通只读工具，永远可见**——否则模型没法解锁。它的 description 会动态拼出"可用技能清单"（skill-code/name/说明），参数 `skill_code` 是枚举（只收启用的技能码）；调用成功返回**手册正文 + 本技能解锁的工具清单**，并在 metadata 上打 `ragent_loaded_skill` 标记（遮蔽中间件就认这个）。这里还有个细节：**加载成功文本在正文调用时才从注册表取**，所以改手册内容**不需要重建 Agent**。

> 一个真实场景串起来（面试可复述）：
> 用户说"帮我请 5 天年假"→ 模型看到的工具列表里**没有** leave_submit（被遮了）→ 模型调 load_skill(skill_code=leave_apply) → 拿到手册："先查假期余额，确认够 5 天再提交……" → 手册解锁 leave_submit → 模型先调 leave_query 确认余额 → 再调 leave_submit → 因为是写操作，弹确认卡 → 用户点确认 → 才真正提交。

**面试官追问**：那 Agent 的工具目录本身怎么组织？每次请求都重新解析一遍注册表吗？

## Q12（追问）Agent 工具目录的快照与缓存——为什么重建前所有请求看到同一份工具集？

**我：** 工具目录 `AgentToolCatalog` 有个**快照机制**：

- `resolve()` 一次性定格：**意图树配置的 mcpToolId 与注册表求交集**。配了但当前没执行器的工具收进 `unavailableToolIds`（只打 warn，不打断——注释："配了但当前没执行器→unavailableToolIds"）；产出 `ResolvedCatalog`（含 knowledge 工具描述、可选 memory 工具描述、bindings、unavailableToolIds、skills、displayNames、fieldLabels、**fingerprint 指纹**）。
- `buildToolkit(catalog)` **从快照建 Toolkit，过程中不回读注册表**——"结果与快照指纹必然一致"。指纹用于缓存：同一指纹的 Toolkit 复用。

**注意新版 catalog 的组成变了**：buildToolkit 除了注册 MCP 桥接，还会按需注册 `SkillLoadTool`（hasSkills 时）、知识库工具、可选记忆工具。MCP 工具解析 `resolveMcpToolBindings` 只取**意图树里 kind=2 且 mcpToolId 非空的节点**去和注册表求交集。

**Provider 是单例的**（`ReActAgentProvider`，懒重建）：`getAgent()` 每次先解析人设 persona + `catalog.resolve()`，**只有 persona 变了或 catalog.fingerprint 变了才重建** Agent（双重检查锁），否则直接复用缓存实例。匹配条件 `persona.equals + fingerprint.equals` 双条件。重建前所有请求看到同一份工具集——避免每个请求都重新解析注册表、也避免请求之间工具集不一致。快照 fingerprint 和 Toolkit 同源，随 ActiveAgent 一起交给调用方。

> 探活细节：dashboard 的 MCP 探活走 `mcpToolCount()`，**不走整份 resolve**——"探活不该被知识库工具声明缺失连坐"，只看 MCP 绑定数。

**面试官继续追问**：Agent 会话状态呢？重建 Agent 会丢历史吗？

> 答：会话状态在 `PgAgentStateStore` 里**按次加载**，Agent 是单例、状态不入 Agent 字段，重建不丢历史。旧实例不主动 close——在途会话还在上面流式输出，交给 GC 回收。

**面试官继续追问**：写工具确认了、技能也加载了，工具调用"调对没有"，你的评测有覆盖吗？

## Q13（追问）评测数据的边界：读写治理这些链路怎么验证？端到端评测为什么还是没给工具调用出分？

**我：** 先说实话：**端到端评测集里依然没有"工具调用"的标注**。ragenteval 每条样本标的是 `query / intent_l2 / expected_doc_ids / ground_truth`，**没有 `tool_calls_gold`**——没有 gold 就出不了工具选择准确率。这是设计取舍：给每个工具每个参数组合都标注期望调用，成本远高于问答标注；而且**工具调错的后果会被问答指标接住**——该调天气却调了别的，答案必错，`faithfulness`/`answer_relevancy` 自然掉。

**新版读写治理引入后，验证方式变了两块：**

**第一，链路有效性靠旁路观测 + 行为日志。** 评测旁路记 `has_mcp`/`intent_pred`，能验证"该走 MCP 的样本有没有真走、不该走的有没有误调"。写工具这边，我们看的是**行为日志**：确认卡被拒的比例、被拒后模型是否如实收敛不再重试硬闯、`load_skill` 调用是否发生在对应写工具之前。这些不是自动指标，但能暴露"治理链路有没有在起作用"。

**第二，治理本身是"人机闭环"，比自动评测更硬。** 写操作确认卡和技能遮蔽，本质上是把"安全"交给**人在回路**：模型再自信，动手前也要用户点头；再看不懂流程，也要先取手册。这一层不依赖评测分数兜底——它把"模型自主操作"降级成"模型提方案、人拍板"。

**三道工程防线依然在（老版就有的，继续背）：** ① garbage 永不进工具入参——防静默扩过滤范围；② 低温确定性提参——同句话每次提参结果稳定；③ NEED_CLARIFICATION 注入 isError=false 提示——信息不足让模型追问不硬调。这三条加新版的确认卡 + 技能遮蔽，就是完整的"**错了不崩、不静默扩大副作用、动手前有人把关**"。

> 💡 面试官追问（真实可能被问）：写工具确认卡 + 技能遮蔽会不会拖慢 Agent？模型每次办事都要多两轮交互。
>
> **我：** 会多一点延迟，但这是**主动选择的成本**。查询类工具（天气/销售/工单/余额）完全不受影响——它们不 require_confirm、也不挂技能 tool-ids，走 Q7 的并行直调，Agent 和 RAG 检索链路都能秒级拿到结果。被拦的只有三类写操作：提交请假、资产换新、订会议室。对它们，多一次确认/多一次取手册，换的是"模型不会替你做出有真实业务副作用的事"——对 2B 系统这是**必须买的保险**。真要优化，未来可以对同一用户的重复操作做"白名单记忆"（比如今天已确认过同款请假格式），但那是后话。

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里出现的钩子 | 面试官大概率会追 | 你准备好了没 |
|---|---|---|
| "三层架构：Server 独立进程 + Client + 消费侧" | 为什么 Server 独立部署 | ✅ Q1 |
| "10 个 executor / 11 个工具，8 读 3 写" | 写工具都有哪些、为什么它们是写 | ✅ Q1（工具清单表） |
| "MeetingRoomMcpExecutor 一个类出两个工具" | 为什么查询和预订放一起 | ✅ Q1（共享确定性占用时段，查完再订不撞随机） |
| "强制自报读写，requireReadOnlyHint 启动校验" | 不声明会怎样 / 为什么启动时拦 | ✅ Q2（fail-fast，防运行期才发现写工具没确认） |
| "写工具返回中文错误让模型回头问用户" | 和服务端校验的关系 | ✅ Q2 |
| "Streamable HTTP + /mcp Servlet" | 传输层怎么选 | ✅ Q2 |
| "启动时 listTools 一次性发现" | 为什么不做运行时热发现 | ✅ Q3（意图树配置驱动） |
| "连不上只跳过注册，不阻塞启动" | 可选依赖的降级 | ✅ Q3 |
| "本地/远程统一 McpToolExecutor 接口" | 抽象的价值 | ✅ Q4 |
| "远程调用异常包成 isError 不抛出" | 失败形态可控 | ✅ Q4 |
| "低温确定性提参" | LLM 提参的工程约束 | ✅ Q5 |
| "garbage 永不进工具入参" | 静默丢过滤条件的危险 | ✅ Q6（过滤范围无声扩大） |
| "提参 FAILED 后不重试" | 为什么不自动重试/降级 | ✅ Q6（系统性错误，宁缺勿滥） |
| "两层并行：子问题并行 + 工具并行" | 线程池参数 / 单工具异常隔离 | ✅ Q7（mcpBatchExecutor: CPU~CPU×2, SynchronousQueue, CallerRunsPolicy） |
| "NEED_CLARIFICATION 注入 isError=false 提示" | 让模型追问而非报错 | ✅ Q8 |
| "McpToolBridge extends ToolBase 而非 implements" | 为什么必须继承才能接入确认 | ✅ Q9（接入框架权限体系） |
| "needsConfirm = require_confirm 勾选 OR readOnlyHint=false" | 双保险、漏勾兜底 | ✅ Q9 |
| "AWAITING_CONFIRM 是唯一非终态" | 会话挂起、新提问被阻塞 | ✅ Q10 |
| "确认参数从 Agent 状态取原件，前端只传同意/拒绝" | 防篡改怎么做 | ✅ Q10（防"请 5 天变 50 天"） |
| "拒绝文案从 'Permission denied by user' 改写" | 框架英文拒绝对用户生硬 | ✅ Q10（AgentConfirmDenialMiddleware） |
| "tool-ids 只放写工具、只读不挂" | 为什么查余额不用先加载手册 | ✅ Q11 |
| "遮蔽 = G − L，压缩带走手册工具跟着收回" | 解锁与手册同生共死 | ✅ Q11 |
| "load_skill 正文调用才取注册表，改手册不用重建 Agent" | 热更新 | ✅ Q11 |
| "工具目录快照 + 指纹缓存，resolve 一次定格" | 每次请求不回读注册表 | ✅ Q12 |
| "ReActAgentProvider 单例，persona+fingerprint 双条件重建" | 请求间工具集一致性 / 会话状态不丢 | ✅ Q12 |
| "确认卡+技能遮蔽 = 人在回路，比自动评测更硬" | MCP 为什么还没进评测集 | ✅ Q13 |
| "写工具确认/取手册会不会拖慢 Agent" | 交互成本的取舍 | ✅ Q13（💡 查询工具不受影响，写操作是必须买的保险） |

---

## 📋 面试建议（对真实面试的 3 条建议）

1. **把"管住写操作"讲成架构升级，而不是罗列新功能。** 面试官如果看过你简历的 MCP 描述，大概率想知道"除了接查询工具，你有没有考虑副作用"。高分答法是 Q1 的主线：**"查询自由并行、写操作层层设闸"**——强制自报读写（Q2）、确认卡（Q10）、技能遮蔽（Q11）三层，每层解决一个不同的问题（不知道是不是写 → 知道写但要不要执行 → 会不会按正确流程办）。这条主线让 MCP 从"工具接入"升维到"Agent 安全操作"。

2. **"确认参数从 Agent 状态取原件、前端只传同意/拒绝"是你的防篡改高光。** 大多数候选人聊工具确认只会说"加个弹窗"，很少人讲透"为什么参数不能从请求里带回来"。答这个点时主动抛出"前端改不了请 5 天成 50 天"，再补上拒绝文案被中间件改写、AWAITING_CONFIRM 挂起阻塞新提问——这是真实实现里踩过交互坑才有的认知。

3. **两个 ⚠️ 钩子提前备好：** ①「为什么 McpToolBridge 从 implements 改成 extends ToolBase」——答"只有继承框架的 ToolBase 才能接入 checkPermissions 权限检查，写操作的 ask 才有地方挂"；②「技能遮蔽为什么只认上下文里的 load_skill 结果」——答"遮蔽和解锁同生共死，手册被上下文压缩带走时工具必须跟着收回，否则模型会凭旧记忆硬闯——而 McpToolBridge.callAsync 的 maskedBySkill 是第二道兜底"。这两点是你比"只会接 MCP"的候选人多出的治理深度。

---

> ⚠️ 版本记录：本文对应真实代码 HEAD ~2026-09-06（Agent 确认链路 + 技能 + 读写标注均在码）。旧《MCP怎么做.md》仍对应 2026-08-25 前的代码（仅 6 个查询工具、无读写分层/确认卡/技能）。复习时以本文 + 实际代码为准。

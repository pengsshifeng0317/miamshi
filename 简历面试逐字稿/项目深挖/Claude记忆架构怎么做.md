# 模拟面试逐字稿：Claude 的记忆架构设计

> 业界方案深挖——面试官问"你了解 Claude 的记忆系统是怎么设计的吗？"或"如果让你参考 Claude 设计一个 Agent 记忆系统，你会怎么设计？"的完整一问一答逐字稿。
> 回答内容基于 Anthropic 官方文档 + Claude Code 真实实现，与本项目自研的「会话记忆」模块（滑动窗口 + 增量摘要）对照讲，体现"既懂业界方案、又做过自研取舍"。
> 事实以官方文档为准（2026 年状态）。

---

> 🕐 真实链路执行时序（开场先画这张图）：

```text
 会话开始(新会话)    Claude Code 运行时        磁盘记忆(MEMORY.md+主题文件)      selectRelevantMemories
    │                      │                          │                              │
    │ 启动会话               │                          │                              │
    │─────────────────────►│ ① 静态层：加载 CLAUDE.md   │                              │
    │                      │─────────────────────────►│ (四个作用域按优先级合并)       │
    │                      │ ② 动态层：加载 MEMORY.md   │                              │
    │                      │─────────────────────────►│ (只当索引，200行/25KB)        │
    │                      │ ③ 轻量召回(最多 5 条)      │                              │
    │                      │──────────────────────────────────────────────────────►│ 扫描各文件 frontmatter
    │                      │◄──────────────────────────────────────────────────────│ 相关性排序取 top-5
    │                      │ ④ 注入上下文(静态指令+索引+召回要点)                      │
    │ 用户提问              │                          │                              │
    │─────────────────────►│ 带着记忆回答               │                              │
    │◄─────────────────────│                          │                              │
    │                      │ ⑤ extractMemories：每轮扫描 │                              │
    │                      │─────────────────────────►│ 值得记的事实→写主题文件+更新索引│
    ▼                      ▼                          ▼                              ▼
```

## 🎤 开场总览（45 秒版，可直接背）

> 面试官问"Claude 的记忆架构是怎么做的？"或"如果让你设计一个 Agent 记忆系统，你会怎么设计？"时，先把下面这段一气呵成讲完，再落到 Q1 展开。
> 末尾留的钩子（三套系统 / 三层成本模型 / selectRelevantMemories / 自研增量摘要取舍）正好接 Q1 / Q2 / Q8 / Q10。

**我：** 先纠正一个常见误区——**Claude 的记忆不是一个系统，而是三套独立系统**：网页聊天的 claude.ai 消费级 Memory、给 API/SDK 用的 Memory Tool、开发者工作流里的 Claude Code 记忆。它们存储介质各不相同，但遵循同一个第一性原则——**记忆是压缩的要点，不是归档的流水账**（notebook 而非 archive），都不存完整对话。我们自研的会话记忆模块，就是照着这个思想做的取舍。

核心是四件事：

**第一，先讲清三套系统，用三层记忆理论做骨架。** 工作记忆=上下文窗口（贵，有硬上限）、情景记忆=会话历史（中，按需访问）、语义记忆=长期要点（便宜）。记忆架构的本质是把贵的东西压成便宜的：**会话 → 摘要 → 要点**。Claude 在 Claude Code 里就是这么干的——动态层只存 MEMORY.md 索引，主题文件按需懒加载。

**第二，索引与内容分离、按需召回，控制注入量。** MEMORY.md 只当索引（200 行/25KB），主题文件懒加载，`selectRelevantMemories` 每请求只召回 ≤5 条。落到我们自研模块就是：**滑动窗口 + 增量摘要**——原文只留最近 8 轮当工作记忆，更早的压成一条摘要当情景记忆，加载时摘要置前、历史在后。

**第三，增量摘要的关键是"只摘新增、不重复摘"。** 我们按 assistant 消息异步触发，触达 9 轮才开始；摘要覆盖约一半原文窗口，**只有这段重叠滑出窗口才再次生成**（避免每次全量重摘）；用 Redisson 分布式锁防并发重复摘要；用 FAST 档模型生成，合并历史摘要去重、**冲突以本轮为准**——这和 Claude 防记忆过时的思路一致。

**第四，feedback 记忆防行为漂移 + 安全层防记忆投毒。** 用户纠正类 feedback 是最高价值记忆；写记忆的接口做白名单校验，防止外部内容通过记忆持久化污染后续所有会话。

> 📦 涉及的表（面试官问"表怎么设计的"时展开，平时不用全背）：
>
> | 表 | 用途 | 关键字段 |
> |----|------|---------|
> | `t_conversation` | 会话列表，一个会话一条，按 `user_id` 隔离 | `conversation_id`+`user_id` 唯一；`title`、`last_time`（按用户+时间排序） |
> | `t_conversation_summary` | 会话摘要（与消息表分离存储），增量摘要的落库 | `last_message_id` 标记摘到哪条；`content` 存摘要正文 |
> | `t_message` | 会话消息原文，滑动窗口的载体 | `role`/`content`；`thinking_content`/`sources`/`retrieved_chunks`/`recommended_questions`(JSONB) |
> | `t_message_feedback` | 消息反馈（feedback 记忆，防行为漂移） | `vote`(1/-1)、`reason`、`comment` |
> | `t_sample_question` | 示例问题，WelcomeScreen 引导提问（非记忆主链路） | `title`/`description`/`question` |

讲到这里停一下，面试官大概率会顺着追问：你说的"三套系统"具体差别在哪？三层成本模型怎么算？`selectRelevantMemories` 是怎么从一堆记忆里挑出相关的？你们自研的增量摘要踩过什么坑？——正好接上 Q1 / Q2 / Q8 / Q10。

## Q1（开场总览）Claude 的记忆系统整体是什么架构？

**我：** 先纠正一个常见误解——**Claude 的记忆不是一个系统，而是三套独立系统**，分别服务不同场景：

| 系统 | 服务对象 | 存储介质 | 谁写入 |
|---|---|---|---|
| **claude.ai 消费级 Memory** | 网页/桌面/移动聊天 | Anthropic 服务器上的条目 | Claude 自动 + 用户显式 |
| **Memory Tool**（`memory_20250818`） | Agent SDK / API 应用 | 你的文件系统（`/memories`） | 模型通过工具调用 |
| **Claude Code 记忆** | 开发者工作流 | 磁盘上的 Markdown 文件 | 静态 CLAUDE.md 人写 + Auto Memory 机器写 |

> 三套系统对照（第一性原则：都不存完整对话，只存"提炼后的信息"）：

```text
┌─ ① claude.ai 消费级 Memory ────────────────────────┐
│  服务对象：网页/桌面/移动聊天用户                       │
│  存储：Anthropic 服务器上的条目（按类别组织）           │
│  写入：Claude 自动 + 用户显式"记住…"                  │
│  机制：条目非转录 · 与 Chat Search(RAG) 互补          │
│  控制：Settings 可看/编辑/删除 · pause · reset ·      │
│        incognito 不写不读不搜                        │
│  局限：记忆多 → fading memory（细节被噪声淹没）         │
└─────────────────────────────────────────────────────┘

┌─ ② Memory Tool（memory_20250818）──────────────────┐
│  服务对象：Agent SDK / API 应用                       │
│  存储：客户端文件系统 /memories（纯文件，无DB无向量库）  │
│  写入：模型输出 tool_use → client-executed handler 执行 │
│  命令：view / create / str_replace / insert / delete /│
│        rename                                       │
│  安全层：路径前缀强制 · 目录穿越防护 · 扩展名白名单      │
│  Prompt 强制：ALWAYS VIEW YOUR MEMORY DIRECTORY      │
└─────────────────────────────────────────────────────┘

┌─ ③ Claude Code 记忆 ───────────────────────────────┐
│  服务对象：开发者工作流                                │
│  存储：磁盘 Markdown（~/.claude/projects/.../memory/）│
│  写入：静态 CLAUDE.md（人写）+ Auto Memory（机器写）    │
│  CLAUDE.md 四作用域：Managed/User/Project/Local       │
│  MEMORY.md 只当索引（200行/25KB）+ 主题文件按需懒加载   │
│  召回：selectRelevantMemories 选 ≤5 条注入              │
└─────────────────────────────────────────────────────┘
```

三套的共同点是**都不存完整对话**，而是存"提炼后的信息"。这是整个记忆设计的第一性原则：**记忆是压缩的要点，不是归档的流水账**——官方自己把它定位成"personal notebook（随身笔记本）而不是 archive（档案库）"。

**面试官追问**：这三套对应"三层记忆"理论的哪一层？

## Q2（追问）先讲理论框架——工作记忆 / 情景记忆 / 语义记忆怎么落？

**我：** 记忆系统领域有个经典的三层模型，Claude 的架构正好是它的工业级映射：

1. **工作记忆（working memory）= 上下文窗口**：当前会话内模型能直接看到的全部内容。Claude 用 context window 承载，**有硬上限**——这正是所有记忆系统要压缩的原因。
2. **情景记忆（episodic memory）= 会话历史**：过去发生了什么。Claude 的做法不是全量保留，而是**按需访问**——claude.ai 里可以"读过去的会话"（列出历史会话 + 读转录），Claude Code 里是增量摘要压缩。
3. **语义记忆（semantic memory）= 长期事实与偏好**：跨会话沉淀下来的"用户是谁、喜欢什么、项目背景"——这是**记忆系统的核心价值**，也是三套系统都在做的东西。

> 三层记忆理论 ↔ Claude 落地：

```text
┌─ 工作记忆 = 上下文窗口 ──────────────────────────────┐
│  当前会话模型直接看到的全部内容，有硬上限              │
│  贵：每次请求全量进 context → 所以要压缩              │
├─ 情景记忆 = 会话历史 ────────────────────────────────┤
│  过去发生了什么，按需访问                             │
│  中：claude.ai 可读过去会话 / Claude Code 增量摘要    │
├─ 语义记忆 = 长期事实与偏好 ──────────────────────────┤
│  用户是谁、喜欢什么、项目背景                        │
│  便宜：一小撮要点 ← 记忆系统的核心价值               │
└────────────────────────────────────────────────────┘
  核心思想：把贵的东西压成便宜的（会话→摘要→要点）
```

**关键洞察**：三层的**成本模型完全不同**。工作记忆贵（每次请求全量进 context）、情景记忆中（可按需捞）、语义记忆便宜（一小撮要点）。所以记忆架构的本质是——**把贵的东西尽量压成便宜的东西**：会话 → 摘要（情景→情景压缩）、摘要 → 要点（情景→语义）。我们自研的会话记忆模块就是在这个思想下做的：滑动窗口只留最近 8 轮当工作记忆、增量摘要压缩历史当情景记忆。

**面试官追问**：那 claude.ai 那个记忆，具体怎么工作？

## Q3（追问）claude.ai 消费级 Memory 的机制

**我：** 这是 2025 年 9 月上线面向普通用户的记忆，机制上几个关键点：

**1. 存的是"条目"不是"转录"**：记忆是 AI 生成的、按类别组织的条目（你的角色/项目/沟通偏好/技术偏好/正在做的工作），**不是完整对话记录**。这是"notebook 而非 archive"定位的直接落地。

> Q3 claude.ai 消费级 Memory 一图看懂：

```text
claude.ai 消费级 Memory —— 五个机制要点
   ① 存"条目"不是"转录"
      AI 生成、按类别组织的条目
      角色/项目/沟通偏好/技术偏好/正在做的工作
      = notebook 而非 archive 的直接落地
   ▼
   ② 三种写入方式
      你显式说"记住我是 Java 后端"
      Claude 对话中自己发现并保存（实时，非定时任务）
      Claude 按需读过去会话（列历史 + 读转录，不能全文检索）
   ▼
   ③ 与 Chat Search 互补（付费计划）
      记忆管"长期要点"
      搜索管"捞任意历史"（RAG，带引用）
      = 分工：要点 vs 全文
   ▼
   ④ 用户完全可控
      Settings 可看/编辑/删除 · pause（保留但停用）· reset（不可逆清空）
      incognito 模式：不写入、不读取、不搜索记忆
   ▼
   ⑤ 诚实局限：fading memory
      记忆越多 → 相关细节越被噪声淹没
      所有"把记忆塞进上下文"方案的共性代价
```

**2. 三种写入方式**：
- **你显式告诉它记住**（"记住我是 Java 后端"）；
- **Claude 在对话中自己发现并保存**（工作相关的上下文，实时读写不是定时任务）；
- **Claude 按需读过去的会话**（列出历史会话 + 读转录，但不能全文检索）。

**3. 与 Chat Search 互补**：付费计划里能对历史对话做 **RAG 检索**（表现为一个工具调用，带引用）。记忆管"长期要点"，搜索管"捞任意历史"，两者分工。

**4. 用户完全可控**：Settings > Memory 里能看全部条目、逐条编辑/删除；可以 pause（保留记忆但停用）或 reset（不可逆清空）。隐私兜底是 **incognito 模式**——该模式下不写入、不读取、不搜索记忆。

**5. 一个诚实的局限**：因为记忆文件要被加载进 context window，**记忆越多，相关细节越容易被噪声淹没**——官方明确提到大型记忆库会导致 "fading memory" 问题。这是所有"把记忆塞进上下文"方案的共性代价。

**面试官追问**：Agent SDK 那个 Memory Tool 和这个有什么不同？它是怎么设计的？

## Q4（追问）Memory Tool（memory_20250818）的设计——客户端执行的工具

**我：** 这是给**Agent 应用**用的记忆原语，设计上有一个很妙的点：**它是 client-executed tool（客户端执行工具）**——不是模型自己去读写文件，而是：

```
模型在回复里输出 tool_use 请求（比如 view /memories）
  → 应用侧 MemoryToolHandler 执行文件操作
  → 把 tool_result 返回给模型
```

**为什么这么设计**？因为让模型直接操作文件系统太危险，而把这个权限**收敛到客户端一个 handler 里**，所有安全策略都在一处实施：

> Q4 Memory Tool 设计一图看懂：

```text
Memory Tool（memory_20250818）设计解剖
   ▼
★ 核心：client-executed tool（客户端执行工具）
   模型输出 tool_use → 应用侧 MemoryToolHandler 执行 → 返回 tool_result
   为什么：让模型直接读写文件太危险
     权限收敛到客户端一个 handler，安全策略集中一处实施
   ▼
存储 = 纯文件系统
   /memories 目录下普通文件（.md 等）
   无数据库、无向量库 → 简单、可迁移、用户看得见摸得着
   ▼
六命令
   view          读文件/列目录
   create        建/覆盖文件
   str_replace   精确替换（要求 old_str 唯一，防歧义编辑）
   insert        按行号插入
   delete        删除
   rename        重命名
   ▼
安全层四道防线
   ① 路径前缀强制   所有路径必须以 /memories 开头
   ② 目录穿越防护   Path.resolve() + relative_to() 挡 ../
   ③ 扩展名白名单   create 只允许 .txt/.md/.json/.py/.yaml
   ④ 根保护         不能删除 /memories 本身
   ▼
记忆行为强制约定
   "IMPORTANT: ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING ELSE"
   = 把"记忆优先检查"写进 prompt，防 Agent 开局忘了自己有记忆
```

**1. 存储是纯文件系统**：所有记忆就是 `/memories` 目录下的普通文件（`.md` 等），**没有数据库、没有向量库**——简单、可迁移、用户看得见摸得着。

**2. 六个命令**：`view`（读文件/列目录）、`create`（建/覆盖文件）、`str_replace`（精确替换，**要求 old_str 唯一**，防歧义编辑）、`insert`（按行号插入）、`delete`、`rename`。

**3. 安全层是重头戏**：
- **路径前缀强制**：所有路径必须以 `/memories` 开头，拒绝 `/etc/passwd` 这类；
- **目录穿越防护**：`Path.resolve()` + `relative_to()` 挡 `../`；
- **扩展名白名单**：create 只允许文本类（.txt/.md/.json/.py/.yaml）；
- **根保护**：不能删除 `/memories` 本身。

**4. 记忆行为的强制约定**：官方注入一条系统提示——**"IMPORTANT: ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING ELSE"**（做事前先看记忆目录）。这是把"记忆优先检查"写进 prompt，防止 Agent 开局就忘了自己有记忆。

**面试官追问**：Claude Code 的记忆呢？CLAUDE.md 是干嘛的？

## Q5（追问）Claude Code 静态记忆层——CLAUDE.md 的四个作用域

**我：** Claude Code 的记忆分**静态层（人写）和动态层（机器写）**两块。先说静态层 `CLAUDE.md`——它是**用户/团队写的常驻指令**，会话开始就加载，等于是"给每个新会话注入的长期记忆"。

> Q5 CLAUDE.md 四作用域一图看懂：

```text
CLAUDE.md —— 静态记忆层（人写、常驻指令）
   会话开始就加载 = 给每个新会话注入的长期记忆
   ▼
优先级从低到高（越近当前目录越优先）
   Managed   /etc/claude-code/CLAUDE.md   组织级管理员下发全局规则
      → User   ~/.claude/CLAUDE.md        个人对每个项目的通用偏好
         → Project  ./CLAUDE.md、.claude/rules/*.md   进仓库、团队共享
            → Local    CLAUDE.local.md   私有、git 忽略
   ▼
加载机制：从工作目录向上遍历目录树，每层都加载
   工作目录 ──► ./CLAUDE.md ──► ../CLAUDE.md ──► … ──► ~/.claude ──► /etc/claude-code
   ▼
条件加载：.claude/rules/*.md
   按主题拆分指令（coding-style.md / testing.md）
   frontmatter 的 paths glob → 只在相关目录生效
   = 静态记忆的本质：把约定提前固化、跟着代码库版本走
```

**四个作用域，优先级从低到高（越近当前目录越优先）**：

| 类型 | 位置 | 谁用 |
|---|---|---|
| **Managed** | `/etc/claude-code/CLAUDE.md` | 组织级管理员下发的全局规则 |
| **User** | `~/.claude/CLAUDE.md` | 个人对每个项目的通用偏好 |
| **Project** | `./CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md` | 进仓库、团队共享 |
| **Local** | `CLAUDE.local.md` | 私有、git 忽略 |

**加载机制**：从工作目录向上遍历目录树，每层都加载。`.claude/rules/*.md` 支持**按主题拆分指令**（coding-style.md / testing.md），还能用 frontmatter 的 `paths` glob 做**条件加载**（只在相关目录生效）。

这个设计给我的启发：**静态记忆的本质是"把约定提前固化"**——与其每次让 Agent 自己摸索，不如把团队规范写进代码库跟着版本走。我们项目里维护的这套规则集（`~/.claude/CLAUDE.md` + `.claude/rules/`）就是这么用的。

**面试官追问**：那"Auto Memory / MEMORY.md"呢？这是 Claude 自己写的记忆？

## Q6（追问）Claude Code 动态记忆层——Auto Memory 与 MEMORY.md

**我：** 这是 Claude Code 的**自动记忆**（默认开启）：Claude 在会话中发现值得记的东西，自己写进磁盘。存储结构是：

```
~/.claude/projects/<sanitized-git-root>/memory/
├── MEMORY.md          # 索引（每个会话自动加载）
├── user_role.md       # 单主题文件（按需加载）
├── feedback_testing.md
└── project_context.md
```

**核心设计是两个文件的两级结构**：

1. **MEMORY.md 只当"索引"，不当"内容库"**——它被强约束在 **200 行 + 25KB 以内**，每行是一个 ~150 字符的主题文件链接。为什么这样设计？因为 MEMORY.md 是**每个会话都要自动加载**的，塞内容会吃掉上下文；所以只放目录、主题文件按需读。

> Q6 Auto Memory 两级结构一图看懂：

```text
Auto Memory —— 动态记忆层（机器写，默认开启）
   Claude 在会话中发现值得记的东西，自己写进磁盘
   ▼
两级结构 = 索引与内容分离
   ① MEMORY.md = 只当"索引"，不当"内容库"
      强约束 200 行 + 25KB
      每行 = 一条 ~150 字符的主题文件链接
      为什么：MEMORY.md 每个会话都自动加载
        → 塞内容会吃掉上下文，只放目录
   ② 主题文件 = 按需懒加载
      user_role.md / feedback_testing.md / project_context.md …
      只有 Claude 判断需要时才用文件工具打开
      = 控制每次会话的上下文开销
   ▼
细节
   所有 worktree 共享同一记忆目录（按 canonical git root 归一）
   存储位置可被环境变量/设置覆盖
   默认路径故意排除 project 设置
   （自动记忆目录不该被仓库内文件控制 = 安全考量）
```
2. **主题文件按需加载**：只有 Claude 判断需要时，才用文件读取工具打开某个主题文件——**懒加载**，控制每次会话的上下文开销。

细节：所有 worktree 共享同一个记忆目录（按 canonical git root 归一）；存储位置支持环境变量/设置覆盖（且默认路径故意**排除 project 设置**——因为自动记忆目录不该被仓库内文件控制，这是安全考量）。

**面试官追问**：存什么不存什么？记忆的 schema 长什么样？

## Q7（追问）四类记忆 + frontmatter + 自动 vs 显式

**我：** 记忆**不是什么都存**，它有个严格的取舍原则：**只存"从代码库和 git 历史推不出来的信息"**——架构、模式、已修 bug 都在 git 里，存了只会过时（stale）。所以存的是：非代码可推导的上下文（项目目标、正在做的事、决策）、行为反馈、外部依赖指针。

**每条记忆用 frontmatter 元数据标注四类**（封闭分类）：

> Q7 四类记忆 + 写入策略一图看懂：

```text
记忆取舍原则（第一道闸）
   只存"从代码库和 git 历史推不出来的信息"
   架构/模式/已修 bug 都在 git 里 → 存了只会过时(stale)
   → 存：非代码可推导的上下文 / 行为反馈 / 外部依赖指针
   ▼
四类记忆（frontmatter 元数据，封闭分类）
   user       你的角色/经验/时区/偏好           长期身份
   feedback   对 AI 行为的纠正与确认          ★价值最高
              从失败+成功两端都记 → 防行为漂移
   project    非代码可推导的项目背景           目标/进行中的工作/决策
   reference  外部系统指针/未文档化依赖        环境配置
   ▼
两种写入路径
   自动检测   用户纠正/重复模式/重要决策/报错修复/特定 flag/未文档化依赖
   显式保存   用户说"记住……"，保证逐字精确
   最佳实践：规则/约束用显式存（保精确）、上下文知识用自动存（保覆盖）
   ▼
对照自研：我们的"合并式摘要、冲突以本轮为准"
   = 用"本轮事实优先"纠偏 feedback 类记忆的行为漂移
```

| 类型 | 内容 | 价值 |
|---|---|---|
| **user** | 你的角色/经验/时区/偏好 | 长期身份 |
| **feedback** | 对 AI 行为的纠正与确认 | **价值最高**——从失败和成功两端都记，防行为漂移 |
| **project** | 非代码可推导的项目背景 | 目标/进行中的工作/决策 |
| **reference** | 外部系统指针、未文档化的依赖 | 环境配置 |

**两种写入路径**：
- **自动检测**：用户纠正、重复出现的模式、重要决策、报错与修复、特定命令 flag、未文档化依赖。
- **显式保存**：用户说"记住……"，**保证逐字精确**。最佳实践是**规则/约束用显式存、上下文知识用自动存**——显式保精确、自动保覆盖。

这个设计让我想到我们自研会话记忆里的一个对应点：我们的"合并式摘要"（冲突以本轮为准）解决的就是 feedback 类记忆的**行为漂移**问题——Claude 的做法是从失败+成功两端都记录来防漂移，我们是靠"本轮事实优先"来纠偏。

**面试官追问**：记忆文件是每个会话全量读吗？召回怎么做的？

## Q8（追问）Smart Recall——怎么从一堆记忆里挑出相关的？

**我：** 这是记忆系统**最容易做砸**的一环——记忆文件全塞进上下文会淹没重点（fading memory），所以 Claude Code 用了**轻量侧查询**做召回：

**`selectRelevantMemories()`**：一个轻量的 Sonnet 侧查询，扫描各文件的 frontmatter，**每次选最多 5 条相关记忆**注入上下文。有三个工程细节：

> Q8 Smart Recall 一图看懂：

```text
Smart Recall —— 从一堆记忆里挑出相关的
   痛点：记忆全塞进上下文 → fading memory 淹没重点
   方案：轻量侧查询 selectRelevantMemories()
   ▼
流程
   扫描各文件 frontmatter（description）
     → 相关性排序
        → 每次最多注入 5 条相关记忆进上下文
   ▼
三个工程细节
   ① description 字段当搜索关键词
      写记忆时 description 要像搜索词一样写
   ② recentTools 去噪
      正在用的工具：工具说明不注入（读了白读）
      但该工具的历史 warning/gotcha 仍然选
      = 区别对待"工具说明"和"工具教训"
   ③ alreadySurfaced 去重
      已给过模型的记忆不再重复给
   ▼
配套后台：extractMemories（每轮对话后扫描）
   自动抽取值得记的事实
   跳过主 Agent 已写过的段 → 避免重复写入
   ▼
本质 = 轻量 RAG
   记忆当文档库 · description 当检索字段 · 相关性排序取 top-5
   对照自研：我们的"重叠滑出才重摘" = 同样控制注入量、防上下文撑爆
```

1. **description 字段当搜索关键词**：每条记忆的 frontmatter 里那个一句话 description，就是给召回做相关性判断用的——**所以写记忆时 description 要像搜索词一样写**。
2. **`recentTools` 去噪**：正在用的工具，它的文档就不注入（省得读了也白读），但该工具的历史 warning/gotcha 仍然选——**区别对待"工具说明"和"工具教训"**。
3. **`alreadySurfaced` 去重**：已经给过模型的记忆不再重复给。

配套还有一个**后台服务 `extractMemories`**：每轮对话结束后扫描，自动抽取值得记的事实（跳过主 Agent 已经写过的那几段，避免重复写入）。

**这套"索引 + 按需召回 + 注入 ≤5 条"的设计，本质是一个轻量 RAG**——把记忆当成文档库、description 当检索字段、相关性排序取 top-5。我们自研会话记忆里的"重叠滑出才重摘摘要"和它思路一致：**都是控制注入量、避免上下文被记忆撑爆**。

**面试官追问**：Claude 怎么保证记忆可信、可控？还有没有别的记忆形态？

## Q9（追问）用户控制 / 隐私 / Agent Memory

**我：** 分三块：

> Q9 用户控制 / 隐私 / Agent Memory 一图看懂：

```text
记忆可信、可控 —— 三块
   ① 用户控制
      /memory 命令：全屏编辑器，列出所有记忆文件按类型分组
        可查看/编辑/增删，改动立即刷新内存缓存
      说"记住 X"立即保存 / "忘掉 Y"查找并删除
      incognito 会话不写不读
      claude.ai 侧 pause/reset；企业账号默认关闭需组织开启
   ▼
   ② 安全性设计
      记忆目录路径默认排除 project 设置控制（防仓库内文件劫持）
      Memory Tool 安全层（路径前缀/穿越防护/扩展名白名单）
        = 把"模型读写文件"关进笼子
      ★ 底线：记忆是注入 prompt 的内容
        被污染 = 持久化投毒，比单会话注入危害大得多
   ▼
   ③ Agent Memory（v2.1.33+）
      每个子 Agent 一个独立记忆空间
        ~/.claude/agent-memory/<agent-name>/MEMORY.md + 主题文件
      作用域跟随 settings 层级
      意义：不同职责的 subagent 记忆隔离
        "检索 agent 的记忆污染不了代码审查 agent"
      对照自研：我们"每类 subagent 独立上下文" = 同一隔离思想
```

**1. 用户控制**：`/memory` 命令打开全屏编辑器，列出所有发现的记忆文件按类型分组，可查看/编辑/增删，**改动立即刷新内存缓存**；对 Claude 说"记住 X"立即保存、"忘掉 Y"查找并删除。隐私上 incognito 会话不写不读；claude.ai 侧能 pause/reset，企业账号默认关闭需组织开启。

**2. 安全性设计**：记忆目录路径默认排除 project 设置控制（防仓库内文件劫持）；Memory Tool 的安全层（Q4 的路径前缀/穿越防护/扩展名白名单）把"模型读写文件"关进笼子。**这是 Agent 记忆的底线**——记忆是注入 prompt 的内容，一旦被污染就是持久化投毒，比单会话注入危害大得多。

**3. Agent Memory（v2.1.33+）**：给**子 Agent 每人一个独立记忆空间** `~/.claude/agent-memory/<agent-name>/MEMORY.md` + 主题文件，作用域跟随 settings 层级。**意义**：不同职责的 subagent 记忆隔离，避免"检索 agent 的记忆污染了代码审查 agent"——和我们项目里"每类 subagent 独立上下文"是同一个隔离思想。

**面试官追问**：它有什么局限？如果你照 Claude 抄一版，会怎么取舍？

## Q10（追问）Claude 方案的局限 + 如果我们设计会怎么抄

**我：** 先讲 Claude 方案的**已知局限**，这比吹它更能体现深度：
- **per-project 无全局记忆**：MEMORY.md 只在项目内生效，没有跨项目的全局等效物（`~/.claude/CLAUDE.md` 只有静态指令，没有自动记忆）——社区靠第三方工具补。
- **无垃圾回收**：删掉的项目在 `~/.claude/projects/` 留下孤儿记忆。
- **上下文窗口是硬约束**：记忆文件要进 context，记忆多了就 fading memory。
- **自动记忆有"过度写入"风险**：所以 Claude Code 用 `alreadySurfaced` 去重 + 跳过主 Agent 已写的段。

**如果让我为我们的寿险问答 Agent 照这套架构设计一版记忆**，我的取舍是：

> Q10 局限 + 我们的取舍一图看懂：

```text
Claude 方案四个已知局限
   ① per-project 无全局记忆：MEMORY.md 只在项目内生效
      无跨项目自动记忆（~/.claude/CLAUDE.md 只有静态指令）
   ② 无垃圾回收：删掉的项目留孤儿记忆
   ③ 上下文硬约束：记忆要进 context，多了就 fading memory
   ④ 自动记忆过度写入风险 → alreadySurfaced 去重 + 跳过已写段
   ▼
照抄给寿险问答 Agent：五条取舍
   ① 三层结构照抄，存储换我们已有的 Redis
      静态约定层 = system prompt 工程（代码内维护）
      动态要点层 = 记忆条目库（四类照搬，Redis Hash）
      会话层 = 滑动窗口 + 增量摘要
   ② 索引 + 按需召回照抄
      Redis sorted set 索引，每请求只注入 ≤N 条相关记忆
   ③ feedback 类记忆最高优先级
      （合并式摘要、冲突以本轮为准）
   ④ 记忆投毒防护照抄：写记忆接口白名单校验（字段/长度/类型）
   ⑤ 不抄 Memory Tool 纯文件存储
      Redis 更可控：TTL/隔离/审计
      但"client-executed tool 收敛权限"的思路照搬
   ▼
最值得抄的不是存储选型，是三个设计原则
   记忆是压缩的要点不是完整流水账
   索引与内容分离、按需召回控制注入量
   feedback 防行为漂移、安全层防记忆投毒
```

1. **三层结构照抄，但存储换成我们已有的 Redis**：CLAUDE.md 的"静态约定层"对应我们的 system prompt 工程（业务人设/规则，代码内维护）；Auto Memory 的"动态要点层"对应我们的记忆条目库（user/feedback/project/reference 四类照搬，Redis Hash 存）；会话层保留我们的滑动窗口 + 增量摘要。
2. **索引 + 按需召回照抄**：MEMORY.md 索引 → 我们的记忆目录索引（Redis sorted set，按最近访问/相关性），每请求**只注入 ≤N 条相关记忆**（借鉴 selectRelevantMemories 的 ≤5），用 description 当检索关键词。
3. **feedback 类记忆优先**：结合我们"合并式摘要、冲突以本轮为准"，把用户纠正类记忆提到最高优先级——这是防行为漂移最值钱的部分。
4. **记忆投毒防护照抄安全层**：写记忆的接口做白名单校验（只允许特定字段/长度/类型），防止外部内容通过记忆持久化污染后续所有会话。
5. **不抄的部分**：Memory Tool 的纯文件存储——生产 Agent 我们用 Redis 反而更可控（TTL、隔离、审计）；但"client-executed tool 收敛权限"的思路照搬，工具调用权限收敛到一层。

一句话总结我的认知：**Claude 记忆架构最值得抄的不是存储选型，而是三个设计原则——"记忆是压缩的要点不是完整流水账"、"索引与内容分离、按需召回控制注入量"、"feedback 记忆防行为漂移、安全层防记忆投毒"。**

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里出现的钩子 | 面试官大概率会追 | 你准备好了没 |
|---|---|---|
| "三套独立系统" | claude.ai / Memory Tool / Claude Code 区别 | ✅ Q1 |
| "notebook 而非 archive" | 记忆压缩的第一性原则 | ✅ Q1/Q2 |
| "工作/情景/语义三层映射" | 记忆理论框架 | ✅ Q2（三层成本模型不同） |
| "三种写入方式 + RAG 检索互补" | 消费级记忆机制 | ✅ Q3 |
| "fading memory 问题" | 记忆进上下文的共性代价 | ✅ Q3 |
| "client-executed tool" | 为什么不让模型直接读写文件 | ✅ Q4（权限收敛到 handler） |
| "路径前缀 + 穿越防护 + 扩展名白名单" | Memory Tool 安全层 | ✅ Q4 |
| "ALWAYS VIEW YOUR MEMORY DIRECTORY" | 记忆优先检查的 prompt 约定 | ✅ Q4 |
| "CLAUDE.md 四个作用域" | 静态记忆分层 | ✅ Q5（越近越优先） |
| "MEMORY.md 200行/25KB 只当索引" | 索引与内容分离 | ✅ Q6 |
| "主题文件按需懒加载" | 控制上下文开销 | ✅ Q6 |
| "四类记忆 + 自动vs显式" | 记忆 schema 与写入策略 | ✅ Q7 |
| "只存 git 推不出来的信息" | 防 stale 记忆 | ✅ Q7 |
| "selectRelevantMemories ≤5 条" | 轻量 RAG 召回 | ✅ Q8（description 当关键词、去噪、去重） |
| "feedback 记忆防行为漂移" | 最高价值记忆类型 | ✅ Q7/Q10 |
| "记忆投毒防护" | Agent 记忆的安全底线 | ✅ Q9/Q10 |
| "per-project 无全局记忆、无 GC" | Claude 方案的局限 | ✅ Q10 |
| "抄三个原则不抄存储选型" | 借鉴的取舍 | ✅ Q10 |

---

## 📋 面试建议（对真实面试的 3 条建议）

1. **开场先纠误区："Claude 记忆不是一套，是三套"**。面试官问"Claude 记忆架构"时，多数候选人只会答 claude.ai 那个消费级记忆。你第一句话把它拆成 消费级 Memory / Memory Tool / Claude Code 三套，就已经赢了一半——这表明你真的看过官方文档，而不是道听途说。
2. **用"三层记忆理论"做骨架、用"三个设计原则"做落点**。回答的加分点是框架感：工作/情景/语义三层映射 + "记忆是压缩的要点不是流水账"、"索引与内容分离按需召回"、"feedback 防漂移 + 安全防投毒"。面试官要的不是你背 MEMORY.md 上限 200 行这种细节，而是**你能不能抽象出可迁移的设计原则**。
3. **主动对照自己的自研会话记忆**（⚠️ 你项目的实际筹码）。每讲一个 Claude 的设计，就接一句"我们自研会话记忆里对应的做法是……"：三层成本模型 ↔ 滑动窗口+增量摘要；selectRelevantMemories ↔ 我们的重叠滑出才重摘；feedback 防漂移 ↔ 我们的冲突以本轮为准。**"懂业界方案 + 自己实现过取舍"是 1 年经验候选人能给出的最强信号**，比单方面复述 Claude 高级得多。

> 附：事实核对来源——[Anthropic Memory Tool 官方文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)、[Anthropic Cookbook: Memory Tool & Cross-Session Learning](https://deepwiki.com/anthropics/claude-cookbooks/6.2-memory-tool-and-cross-session-learning)、[Claude 记忆与聊天搜索官方说明](https://support.claude.com/en/articles/11817273-use-claude-s-chat-search-and-memory-to-build-on-previous-context)、[Claude Code Memory 文档](https://mintlify.wiki/killlowkey/claude-code/concepts/memory)。

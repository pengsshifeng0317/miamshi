# 模拟面试逐字稿：RAGAS 评测怎么做

> 围绕简历产出——「基于 RAGAS 的端到端评测，覆盖检索/延迟/回答质量 20+ 指标，支持 A/B Diff 与人工校正；驱动 faithfulness +10.3%、答案相关性 +8.0%」——的完整一问一答逐字稿。
> 包含：开场总览 → 追问 → 追问铺垫，最后附「追问钩子速览」和「面试建议」。
> 回答内容基于真实代码实现（`ragenteval/` Python 评测模块：`eval/rag/` 下 runner / metrics / pipeline / report 全套）。

---

> 🕐 真实链路执行时序（开场先画这张图）：

```text
 评测脚本(runner)      生产聊天接口(SSE)      评测旁路(/rag/eval)      自建指标 + RAGAS judge      报告
    │                       │                        │                        │                │
    │ ① 登录拿 sa-token      │                        │                        │                │
    │──────────────────────►│                        │                        │                │
    │ ② 调真实链路(逐条样本)  │                        │                        │                │
    │──SSE /rag/v3/chat────►│ 流式 response/thinking │                        │                │
    │ ③ 手写 SSE 解析器       │◄──────────────────────│                        │                │
    │──JSON /rag/eval──────►│                        │ 旁路取检索证据           │                │
    │◄──────────────────────│◄──────────────────────│ (docIds/contexts)       │                │
    │ ④ 合成 EvalRecord      │                        │                        │                │
    │────────────────────────────────────────────────►│ 意图/检索/行为/延迟(自建) │                │
    │───────────────────────────────────────────────►│ RAGAS 五指标(judge)      │                │
    │                                                 │ n_runs均值/降级/重试     │                │
    │                                                 │◄───────────────────────│ report 四产物
    │                                                 │                        │ 失败样本=优化清单
    ▼                       ▼                        ▼                        ▼                ▼
```

## 🎤 开场总览（45 秒版，可直接背）

> 面试官问"你们的 RAGAS 评测是怎么做的？"时，先把下面这段一气呵成讲完，再落到 Q1 展开。
> 末尾留的钩子（评估集怎么标 / Runner 怎么拿全链路 / 自建指标怎么算 / judge 怎么配）正好接 Q2 / Q3 / Q4 / Q5。

**我：** 我们做这套 RAGAS 评测，是为了解决一个问题——RAG 是一条"意图识别 + 重写拆分 + 多路检索 + 重排 + 证据闸门 + 生成"的多段流水线，任何一段改个阈值、权重、topK，都可能让整体变好或变坏。没有评测就没有尺子，改完不知道自己改对没有。所以我们搭了端到端评测，用数据驱动优化。

核心是四件事：

**第一，评估集人工标注，问题带答案、答案带出处。** 每条样本都标了标准答案、召回出处 reference_doc_ids、意图标签 intent_l2，还有"该不该查"的行为标记（MCP / 兜底 / 拒答）。评测不跑空炮，每条都有据可查。

**第二，评测跑真实链路，不搭影子系统。** runner 用 sa-token 登录生产接口，逐条样本走真实的 SSE 聊天链路；同时打一个评测旁路 `/rag/eval`，把检索证据（docIds / contexts / 子问题 / 意图叶子）单独取出来，合成一条 EvalRecord——一次问答，一份完整可回放的评测记录。

**第三，指标分两套，各管一段。** 自建指标盯链路：意图 Top-1 准确率、检索 Hit@K / Recall@K / MRR@10、行为红线（误拒率 / 兜底率 / 过召回率）、延迟（TTFT / 总耗时）；RAGAS LLM-as-judge 盯答案：faithfulness、answer_relevancy、answer_correctness、context_precision、context_recall。两套合起来才是"覆盖检索 / 延迟 / 回答质量"。

**第四，报告四产物，失败样本就是优化清单。** 每次评测出 report.md / per_sample.csv / failures.jsonl / slides.html，A/B diff 直接对比两版参数看指标涨跌；失败样本逐个人工校正再回流评估集——评测不是终点，是优化闭环的起点。

> 📦 涉及的表（面试官问"这些数据都存在哪"时展开，平时不用全背）：
>
> | 表 | 用途 | 关键字段 |
> |----|------|---------|
> | `t_rag_trace_run` | Trace 运行记录，一次问答一条链路，评测可回放核对 | `trace_id`(全局唯一)、`status`(RUNNING/SUCCESS/ERROR)、`duration_ms`、`entry_method`、`extra_data`(JSON) |
> | `t_rag_trace_node` | Trace 节点记录，链路每个环节 | `trace_id`+`node_id` 唯一、`parent_node_id`/`depth` 拼树、`node_type`(REWRITE/RETRIEVE/LLM)、`status`、`duration_ms` |
> | `t_knowledge_chunk` | 评测旁路第一跳：chunkId → 文档雪花 id | `id`、`doc_id`、`content` 正文、`kb_id` |
> | `t_knowledge_document` | 评测旁路第二跳：雪花 id → 业务码 | `id`、`doc_name`(剥后缀对齐 reference_doc_ids)、`kb_id` |
> | `t_intent_node` | 意图叶子节点，算 Top-1 准确率的比对基准 | `intent_code`、`kind`(0 KB/1 SYSTEM/2 MCP)、`collection_names`(JSONB)、`mcp_tool_id` |

讲到这里停一下，面试官大概率会顺着追问：评估集每条样本到底标了什么？Runner 怎么拿一条完整的 EvalRecord？自建指标和 RAGAS 分别怎么算？——正好接上 Q2 / Q3 / Q4 / Q5。

## Q1（开场总览）你们的 RAGAS 评测体系是怎么搭的？为什么 RAG 系统必须有一套评测？

**我：** 先说为什么做评测——RAG 链路是"意图识别 + 重写拆分 + 多路检索 + 重排 + 证据闸门 + 生成"的**多段流水线**，任何一段改个参数（阈值、权重、topK）都可能让整体变好或变坏。没有评测就相当于**没有尺子**：改一处不知道自己改对没有，上线全靠运气。所以我们自建了一套端到端评测，用**数据驱动**来推动优化。

整体是一个 **Python 评测模块（ragenteval）**，数据流是一条清晰的流水线：

```
eval_set_v1.jsonl           评估集（人工标注）
   │  load_samples()
   ▼
List[EvalSample]            一条样本（输入）
   │  runner.run()          调真实生产链路跑一遍
   ▼
List[EvalRecord]            跑完的完整记录（runs/*.jsonl）
   │  score.score()         算所有指标
   ▼
List[MetricResult]          每个指标一个结果
   │  report / diff
   ▼
report.md / per_sample.csv / failures.jsonl / slides.html
```

**CLI 五个命令**：`run`（跑链路）→ `score`（算指标）→ `report`（出报告）→ `diff`（A/B 对比）→ `all`（一条龙）。

**20+ 指标分两组**：
- **自建指标**：意图 Top-1 准确率、检索 Hit@K / Recall@K / MRR@10、行为红线（误拒率 / 兜底率 / 过召回率）、延迟（TTFT/总耗时）——覆盖"链路各段"，RAGAS 不管这些。
- **RAGAS LLM-as-judge**：faithfulness、answer_relevancy、answer_correctness、context_precision、context_recall——覆盖"回答质量"。

为什么两套都要：**自建指标盯链路（检索得对不对、该不该查）、RAGAS 盯答案（答得忠不忠实、切不切题）**。简历里"覆盖检索/延迟/回答质量"就是指这两套合起来。

**面试官追问**：评估集是怎么建的？每条样本都标了什么？

## Q2（追问）评估集 eval_set_v1.jsonl 怎么设计？

**我：** 评估集是一行一条样本的 JSONL，每条 `EvalSample` 字段：

| 字段 | 作用 |
|---|---|
| `query_id / query` | 样本 ID + 用户问题 |
| `intent_l1 / intent_l2` | 标注的意图（用于意图 Top-1 准确率） |
| `difficulty` | easy / medium 等 |
| `requires_rag` | **该不该走知识检索**（核心字段，行为指标靠它） |
| `expected_doc_ids`(+nice) | **检索的 ground truth**——正确证据的文档 ID 列表 |
| `ground_truth` | 标准答案（RAGAS 的 reference） |
| `expected_answer_type` | 期望的答案类型（如 fault_troubleshooting） |
| `trap_type` | 陷阱类型（如 single_fault） |
| `eval_metrics` | 这条样本要评哪些指标 |

实际样本长这样：`F1-01` / "我的扫地机充不进电了" / FEEDBACK / F1_故障报告 / easy / requires_rag=true / expected_doc_ids=[FAQ_VAC_001, CODE_VAC_001, MANUAL_VAC_001] / trap_type=single_fault / 很长的 ground_truth 排查步骤。

三个设计点：

> Q2 评估集一图看懂：

```text
eval_set_v1.jsonl —— 一行一条样本，EvalSample 字段
   ├─ query_id / query              样本 ID + 用户问题
   ├─ intent_l1 / intent_l2         标注意图（意图 Top-1 准确率）
   ├─ difficulty                    easy / medium
   ├─ requires_rag ★核心            该不该走知识检索（行为红线靠它）
   ├─ expected_doc_ids (+nice)      检索 ground truth：正确证据文档 ID 列表
   ├─ ground_truth                  标准答案（RAGAS reference）
   ├─ expected_answer_type / trap_type  期望答案类型 / 陷阱类型
   └─ eval_metrics                  这条样本要评哪些指标
   ▼
三个设计点
   ① requires_rag = 行为红线锚点
      区分"该查没查(误拒)" vs "不该查查了(过召回)"
      = 业务红线指标，RAGAS 完全不覆盖
   ② expected_doc_ids 标文档级、不标 chunk 级
      → 评测要的是"证据进没进 top-K"
        标到文档粒度人工标注成本可控
   ③ ground_truth 每条人工写（RAGAS reference）
      → 标注质量直接决定评测可信度
      = 评估集要小而精，不是越大越好
```
1. **`requires_rag` 是行为红线的锚点**——它能区分"该查的没查（误拒）"和"不该查的查了（过召回）"。这是业务红线指标，RAGAS 本身完全不覆盖。
2. **`expected_doc_ids` 标的是文档级、不是 chunk 级**——因为评测要的是"证据进没进 top-K"，标到文档粒度人工标注成本可控。
3. **`ground_truth` 每条都人工写**——这是 RAGAS 的 reference，answer_correctness 和 context_recall 都拿它当标准。**标注质量直接决定评测可信度**，所以评估集要小而精，不是越大越好。

**面试官追问**：你说的 runner 是调 mock 还是调真实链路？检索证据从哪拿的？

## Q3（追问）Runner——怎么拿一条完整 EvalRecord？

**我：** 核心原则：**端到端调真实生产链路，不 mock**。`runner.run()` 干四件事：

> Q3 Runner 一图看懂：

```text
runner.run() —— 端到端调真实生产链路，不 mock
   ▼
① 登录拿 sa-token
   ▼
② 每条 query 调两个接口
   ├─ SSE  GET /rag/v3/chat   真实生产链路
   │     聚 response / thinking / first_token_ms / latency_ms / final_status
   └─ JSON GET /rag/eval      评测旁路（app.eval.enabled=true）
        取 retrievedDocIds / retrievedChunkIds / retrievedContexts / intentLeafIds
   → 为什么需要旁路？
     SSE 流里只有最终回答，检索证据（recall 指标要的 docIds/contexts）不体现
     → 必须从旁路拿；两条路合起来 = 一条完整 EvalRecord
   ▼
③ 手写 SSE 解析器 parse_sse_stream（踩过的坑）
   requests.iter_lines() 会吞掉跨 HTTP chunk 边界的事件分隔符(空行)
   → 中间事件静默丢失
   自己从字节流逐事件切分（\r\n\r\n 和 \n\n 两种分隔、event:/data: 解析）
   --debug 保留每条 query 原始 SSE 字节流，出问题可回溯
   ▼
④ 文档 ID 反向映射：doc_id_map.json
   ragent 内部 ragent_doc_id → 业务文档 ID
   → expected_doc_ids 和 retrieved_doc_ids 才能比对
   --workers 多线程并行加速 / --sleep 控制节奏防把生产压垮
```

**1. 登录**拿 sa-token。

**2. 每条 query 调两个接口**：
- **SSE `GET /rag/v3/chat`**——真实生产链路，聚 `response / thinking / first_token_ms / latency_ms / final_status`。
- **JSON `GET /rag/eval`**——**评测旁路**（需 `app.eval.enabled=true`），取回检索证据 `retrievedDocIds / retrievedChunkIds / retrievedContexts / intentLeafIds`。

为什么需要旁路？因为 SSE 流里只有最终回答，**检索证据（recall 指标需要的 docIds/contexts）在回答里不体现**，必须从旁路拿。两条路合起来才构成一条完整 `EvalRecord`。

**3. 手写 SSE 解析器**（`parse_sse_stream`）——这是踩过的一个坑：用 `requests.iter_lines()` 会把**跨 HTTP chunk 边界的事件分隔符（空行）吞掉**，导致中间事件丢失。所以自己从字节流里逐事件切分（处理 `\r\n\r\n` 和 `\n\n` 两种分隔、`event:`/`data:` 字段解析）。测试时还能 `--debug` 保留每条 query 的原始 SSE 字节流，出问题可回溯。

**4. 文档 ID 反向映射**：`doc_id_map.json` 把 ragent 内部 `ragent_doc_id` 映回业务文档 ID，这样 `expected_doc_ids` 和 `retrieved_doc_ids` 才能比对。`--workers` 多线程并行加速（默认 1 保持顺序），`--sleep` 控制节奏防把生产压垮。

**面试官追问**：自建指标除了检索，还评了哪些？为什么这些要自己写而不是靠 RAGAS？

## Q4（追问）自建指标四组——意图 / 检索 / 行为 / 延迟

**我：** 因为 RAGAS 只评"回答质量"（faithfulness 那五个），**链路各段的正确性它完全不管**。自建指标补齐：

> Q4 自建指标四组一图看懂：

```text
为什么自建：RAGAS 只评"回答质量"(faithfulness 五个)
   → 链路各段的正确性它完全不管
   ▼
① 意图 intent.py
   intent_top1 = 预测 intent_pred == 标注 intent_l2 的比例
   为什么必须测：意图错 → 后续召回、答案大概率全错
   = 最上游的闸门
   ▼
② 检索 retrieval.py
   Hit@K（K=1/3/5/10 top-K 命中率）
   Recall@K（must + inclusive 两种口径）
   MRR@10（首个命中的名次倒数）
   样本过滤：只统计 requires_rag=true 且 reference_doc_ids 非空
   → SYSTEM 兜底类样本不污染检索指标
   ▼
③ 行为红线 behavior.py（三个业务红线 = 产品红线，不是学术指标）
   误拒率    requires_rag=true 但召回为空（该查没查）
   兜底率    requires_rag=true 但回答出现"未检索到相关文档"
             （检索失败被兜底话术接管）
   过召回率  requires_rag=false 却走了 RAG 召回（不该查的查了）
   → 直接对应用户真实体感的"三类翻车"
   ▼
④ 延迟 latency.py：TTFT P50/均值 + 总耗时均值
   设计取舍：体感卡点是正式回答首字(type=response 首个 delta)
     不是完整流（完整流耗时随 token 线性涨，不反映"卡顿"）
   小样本下 P95/P99 退化为极值 → 改用均值
```

**1. 意图（intent.py）**：`intent_top1` = 预测 intent_pred == 标注 intent_l2 的比例。为什么必须测？**意图错 → 后续召回、答案大概率全错**，它是最上游的闸门。

**2. 检索（retrieval.py）**：`Hit@K`（top-K 命中率，K=1/3/5/10）、`Recall@K`（召回比例，must + inclusive 两种口径）、`MRR@10`（首个命中的名次倒数）。**样本过滤关键**：只统计 `requires_rag=true` 且 `reference_doc_ids` 非空——"SYSTEM 兜底类样本不污染检索指标"。

**3. 行为红线（behavior.py）**，三个业务红线：
- **误拒率**：requires_rag=true 但召回为空（该查没查）。
- **答案兜底率**：requires_rag=true 但回答里出现"未检索到与问题相关的文档内容"（检索失败被兜底话术接管）。
- **过召回率**：requires_rag=false 却走了 RAG 召回（不该查的查了，应走 SYSTEM 话术）。

这三条直接对应用户真实体感的"三类翻车"，是**产品红线**，不是学术指标。

**4. 延迟（latency.py）**：`TTFT P50/均值` + 总耗时均值。设计取舍：**对话产品的体感卡点是正式回答首字到达（type=response 首个 delta），不是完整流**——完整流耗时随 token 数线性增长，不反映"卡顿"；且小样本下 P95/P99 退化为极值，改用均值。

**面试官追问**：RAGAS 那五个指标具体是什么？judge 模型怎么配的？

## Q5（追问）RAGAS LLM-as-judge——五个指标和 judge 配置

**我：** `ragas_judge.py` 封装五个指标：

> Q5 RAGAS 五指标一图看懂：

```text
ragas_judge.py 封装五个指标
   faithfulness       幻觉检测：response 是否忠实于 retrieved_contexts
                     LLM 拆 claim 逐条核对上下文
   answer_relevancy   切题：回答是否相关用户问题
                     反向生成问题，与 user_input 算 embedding 余弦
   answer_correctness 与参考答案语义+事实一致
                     claim F1 + embedding 相似度
   context_precision  retrieved_contexts 里有用信息比例
                     LLM 判断哪些 chunk 对回答有用
   context_recall     retrieved_contexts 是否覆盖 reference 所需
                     LLM 比对 reference 逐条
   ▼
judge 配置（环境变量）
   JUDGE_MODEL 默认 gpt-5.4-mini
   EMBEDDING_MODEL 默认 text-embedding-3-large
   走 aihubmix（API 中转）
   API key 缺失 → RuntimeError 直接报错
   = 不再走兜底硬编码 key（安全红线）
   ▼
样本过滤 filter_evaluable
   只评 response / retrieved_contexts / reference 三项齐全
   且 final_status=success 的样本
   其余记 skip_reason 到 meta、不参与均值
   为什么：RAGAS 三输入缺一不可
     response 空 → 评不了 faithfulness
     contexts 空 → 评不了 context_recall
     拒答/报错的样本本身不该评质量
   ▼
细节：judge 用 JSON mode（response_format: json_object）
   强制合法 JSON，避免中文引号导致 OutputParserException
   = LLM 评测必踩的坑
```

| 指标 | 评什么 | 机制 |
|---|---|---|
| **faithfulness** | 幻觉检测——response 是否忠实于 retrieved_contexts | LLM 拆 claim 逐条核对上下文 |
| **answer_relevancy** | 切题——回答是否相关用户问题 | 反向生成问题，与 user_input 算 embedding 余弦 |
| **answer_correctness** | 与参考答案的语义+事实一致 | claim F1 + embedding 相似度 |
| **context_precision** | retrieved_contexts 里有用信息的比例 | LLM 判断哪些 chunk 对回答有用 |
| **context_recall** | retrieved_contexts 是否覆盖 reference 所需信息 | LLM 比对 reference 逐条 |

**judge 配置**（环境变量）：`JUDGE_MODEL` 默认 `gpt-5.4-mini`、`EMBEDDING_MODEL` 默认 `text-embedding-3-large`，走 **aihubmix**（API 中转）。API key 缺失直接 `RuntimeError` 报错——**不再走兜底硬编码 key**（这是条安全红线）。

**样本过滤**（`filter_evaluable`）：只评 **response / retrieved_contexts / reference 三项齐全且 final_status=success** 的样本，其余记 skip_reason 到 meta、不参与均值。为什么要过滤？**RAGAS 三输入缺一不可**——response 空评不了 faithfulness，contexts 空评不了 context_recall，拒答/报错的样本本身不该评质量。

**细节**：judge 用 **JSON mode**（`response_format: json_object`）强制 LLM 输出合法 JSON，**避免中文引号等导致 OutputParserException**——这是 LLM 评测必踩的坑。

**面试官追问**：LLM-as-judge 方差大是出了名的，你们怎么保证分数可信？

## Q6（追问）LLM-as-judge 的工程化——方差、降级、重试

**我：** 这是评测可信度的核心，四个手段：

> Q6 可信度工程化一图看懂：

```text
四个手段
   ① n_runs 并发跑 N 次取均值（--ragas-n）
      同批数据并发跑 N 次独立评测，每条样本取各次均值
      LLM judge 单次打分随机性大，多跑取平均是直接手段
      默认 n_runs=1（省 API），正式发版对比开 3 次
   ▼
   ② 三级降级（_run）
      ① batch + JSON mode
      ② batch 无 JSON mode
      ③ 逐条 eval 隔离问题样本
      前两层失败打印原因自动回退
      第三层逐条跑：单条失败返回 NaN 不影响其它样本
      = 不因一条坏样本丢整批结果
   ▼
   ③ 失败重试
      所有 run 都拿不到分的 (query_id, metric) 逐条重试
      超时从 900 放宽到 1200（长文本 judge 需要更久）
      重试仍失败才保留 None 并打印
   ▼
   ④ 两个 model 细节
      - reasoning 模型不发 temperature
        _is_reasoning_model 识别 gpt-5/o1/o3/o4 前缀
        这些模型不接受采样参数，硬发会报错
        answer_relevancy 设 strictness=1
      - metric 对象必须 copy.copy()
        RAGAS evaluate() 会临时把 llm/embeddings 写进 metric
        不能跨线程共享，多 run 并发必须每份独立 copy
```

**1. n_runs 并发跑 N 次取均值压制单次方差**（`--ragas-n`）：同批数据并发跑 N 次独立评测，每条样本取各次均值。**LLM judge 单次打分随机性大，多跑取平均是直接手段**。默认 n_runs=1（省 API），正式发版对比开 3 次。

**2. 三级降级**（`_run`）：RAGAS batch eval 可能整批失败——顺序是 **①batch + JSON mode → ②batch 无 JSON mode → ③逐条 eval 隔离问题样本**。前两层失败后打印原因自动回退；第三层逐条跑，**单条失败返回 NaN 不影响其它样本**，不因一条坏样本丢了整批结果。

**3. 失败重试**：所有 run 都拿不到分的 `(query_id, metric)`，**逐条重试、超时从 900 放宽到 1200**（长文本 judge 需要更久），重试仍失败才保留 None 并打印。

**4. 两个 model 细节**：
- **reasoning 模型不发 temperature**：`_is_reasoning_model` 识别 `gpt-5/o1/o3/o4` 前缀，这些模型不接受采样参数，硬发会报错；且 answer_relevancy 设 `strictness=1`。
- **metric 对象必须 `copy.copy()`**：RAGAS `evaluate()` 会临时把 llm/embeddings 写进 metric，**不能跨线程共享**，多 run 并发时必须每份独立 copy。

**面试官追问**：score 之后怎么出报告？失败样本怎么定位？"人工校正"是怎么做的？

## Q7（追问）score 聚合 + 报告产物 + 人工校正

**我：** `score()` 按顺序算：intent → retrieval → behavior → latency → ragas（`--skip-ragas` 可跳过省 API），结果落 `reports/<run>/_scores.json`——**report 阶段直接读它复用，不用重算**（RAGAS 很贵）。

> Q7 score + 报告一图看懂：

```text
score() 按顺序算
   intent → retrieval → behavior → latency → ragas
   （--skip-ragas 可跳过省 API）
   结果落 reports/<run>/_scores.json
   → report 阶段直接读它复用，不用重算（RAGAS 很贵）
   ▼
数据质量自检 sanity_check_doc_id_alignment
   校验 ragent 给 chunk 维度 docId 与 contexts 文本 frontmatter 还原的 docId 是否一致
   不一致 → Hit@1/MRR 会失真，要尽早发现
   = 而不是等指标出来才发现被骗
   ▼
report 四类产物
   report.md       自建+RAGAS 整体表 + 按 intent_l2 分层
                   （哪个意图挂一目了然）
   per_sample.csv  每条样本一行、所有指标横向铺开
   failures.jsonl  扩口径失败样例（hit@5 miss / answer_correctness<0.5
                   / faithfulness<0.5 / 误拒 / 过召回）
                   = "评测驱动优化"的关键：失败样本就是优化清单
   slides.html     汇报用（swiss/magazine 主题）
   ▼
人工校正（markdown.py manual override）
   RAGAS 是 LLM 自评、有方差 → 关键样本允许人工打分优先
   per_sample.csv 给 RAGAS 指标各留一列 xxx_manual
   人工填了用人工值、没填回退 RAGAS
   = "manual value first, fallback to ragas"
```

score 里还有个**数据质量自检** `sanity_check_doc_id_alignment`：校验 ragent 给的 chunk 维度 docId 与 contexts 文本里 frontmatter 还原的 docId 是否一致——**不一致时 Hit@1/MRR 会失真，要尽早发现**，而不是等指标出来才发现被骗。

`report` 出四类产物：
- **report.md**：自建 + RAGAS 整体表 + **按 intent_l2 分层**（哪个意图挂一目了然）。
- **per_sample.csv**：每条样本一行、所有指标横向铺开，方便分析单条。
- **failures.jsonl**：扩口径失败样例——**Hit@5 miss、answer_correctness<0.5、faithfulness<0.5、误拒、过召回**。这是"评测驱动优化"的关键：失败样本就是优化清单。
- **slides.html**：汇报用（swiss/magazine 主题）。

**人工校正**（`markdown.py` 的 manual override）：RAGAS 是 LLM 自评、有方差，**关键样本允许人工打分优先**。机制：per_sample.csv 里给 RAGAS 指标各留一列 `xxx_manual`，人工填了就用人工值、没填回退 RAGAS——**"manual value first, fallback to ragas"**。这回答了简历里"支持人工校正"。

**面试官追问**：简历里"faithfulness +10.3%、答案相关性 +8.0%"这个数，怎么算出来的？

## Q8（追问）A/B Diff 与指标口径——+10.3% 和 +8.0%

**我：** 用 `diff` 命令做 **A/B 对比**（`report/diff.py`），对比的是**同一评测集、两次 run**（优化前后各跑一遍 `run → score`）：

> Q8 A/B Diff 一图看懂：

```text
diff 命令做 A/B 对比（report/diff.py）
   同一评测集、两次 run（优化前后各跑 run → score）
   python -m eval rag diff v1_20260806_225842 v1_20260807_181028
   ▼
机制：读两份 _scores.json
   每个指标 overall 逐项 Δ = B − A
   输出终端表格 + markdown
   按意图分层对比
   （core_metrics：hit@5 / recall@5 / mrr@10 /
      faithfulness / answer_correctness）
   ▼
★ 退化检测是亮点——不看所有 Δ，只标显著退化
   LOWER_IS_BETTER 集合（误拒率/过召回率/延迟类）
     → Δ 超过阈值 = 退化
   REGRESSION_THRESHOLD 每指标独立阈值
     faithfulness 0.05 / answer_relevancy 0.05
     hit@5 0.03 / TTFT 500ms
     → 超过阈值打 ⚠，避免"0.003 的抖动也当问题"
   ▼
+10.3% / +8.0% 口径（诚实讲）
   faithfulness +10.3% = 某次优化后整体均值相对优化前 +10.3 个百分点
   答案相关性 +8.0% = 同口径 answer_relevancy 提升
   指标都是比例值，Δ 是百分点差（0.82→0.923 = +10.3pp）
   = 不是相对增幅
```

```
python -m eval rag diff v1_20260806_225842 v1_20260807_181028
```

**机制**：读两份 `_scores.json`，把每个指标的 overall 逐项算 `Δ = B − A`，输出终端表格 + markdown，并做**按意图分层对比**（core_metrics：hit@5 / recall@5 / mrr@10 / faithfulness / answer_correctness）。

**退化检测**是亮点——不是看所有 Δ，而是**只标显著退化**：
- `LOWER_IS_BETTER` 集合（误拒率/过召回率/延迟类）：Δ 超过阈值 = 退化。
- `REGRESSION_THRESHOLD` 每指标独立阈值（如 faithfulness 0.05、answer_relevancy 0.05、hit@5 0.03、TTFT 500ms）——**超过阈值打 ⚠**，避免"0.003 的抖动也当问题"。

**+10.3% / +8.0% 口径**（诚实讲）：
- **faithfulness +10.3%** = 某次优化（比如证据闸门收紧/检索质量提升）后，faithfulness 整体均值相对优化前提升 10.3 个百分点。
- **答案相关性 +8.0%** = 同口径的 answer_relevancy 提升。

对比时注意：**指标都是比例值，Δ 是百分点差（如 0.82→0.923 是 +10.3pp），不是相对增幅**。这个口径我面试时会主动讲清。

**面试官追问**：评测怎么真正推动优化？不是测完就完了吗？

## Q9（追问）评测驱动优化的闭环

**我：** 这是评测体系存在的意义，是一条**闭环**：

```text
评测驱动优化 · 闭环（面试画这个胜过报数字）：

      ┌────────────── 优化闭环 ────────────────┐
      │                                       │
      ▼                                       │
① 跑基线：run → score → report（发版前存档）    │
      │                                       │
      ▼                                       │
② 改参数 / 改链路（阈值、权重、topK、闸门…）     │
      │                                       │
      ▼                                       │
③ 重跑 + diff（A/B 对比两次 run）              │
   ├─ 显著退化（超 REGRESSION_THRESHOLD）→ 回滚 │
   └─ 达标 → 收                               │
      │                                       │
      ▼                                       │
④ failures.jsonl 失败样本 = 优化清单           │
   hit@5_miss            → 查检索（召回深度/重排）│
   faithfulness_low      → 查生成（幻觉/上下文） │
   intent_top1 低        → 查意图树             │
   refused_when_required → 查证据闸门误杀       │
   over_retrieved        → 查作用域放太宽       │
      │                                       │
      └────── 回到 ②（改 → 重跑 → diff 验证）───┘
```

1. **跑分 + diff**：发版/改参前先跑基线，改完再跑，diff 看变化。
2. **看 failures.jsonl 定失败**：每条失败样本都有 query_id、意图、检索了哪些 doc、回复预览、失败原因——**失败样本就是优化清单**。
3. **按原因分流定位**：
   - `hit@5_miss` → 查检索段（召回深度/重排）；
   - `faithfulness_low` → 查生成段（幻觉，是不是上下文不够/答案超纲）；
   - `intent_top1` 低 → 查意图树；
   - `refused_when_required` → 查证据闸门是不是误杀；
   - `over_retrieved` → 查意图/作用域是不是放太宽。
4. **改 → 重跑 → diff 验证**：确认没有把别的指标带崩（**退化检测就是防止"拆东墙补西墙"**）。

诚实讲，**评测是一面镜子，不是答案本身**——它告诉你"哪里坏了"，但"怎么修"要靠对链路的理解；而且 LLM-as-judge 的分数只能反映"自评视角的质量"，不代表真实用户满意度，所以关键样本要人工复核（Q7 的人工校正）。

**面试官追问（收尾）**：这套评测你踩过最大的坑是什么？

## Q9b（追问）评测系统踩过的坑

**我：** 挑四个有代表性的：

> Q9b 四个坑一图看懂：

```text
四个有代表性的坑
   ① SSE 跨 chunk 吞事件（runner）
      requests.iter_lines() 在事件分隔符跨 HTTP chunk 边界时
      把空行吞掉 → 中间事件静默丢失，回答少了中段
      解决：手写逐字节事件切分
   ▼
   ② LLM judge 中文引号解析炸
      judge 输出里的中文标点让 RAGAS OutputParser 抛异常
      解决：JSON mode 强制合法 JSON
   ▼
   ③ answer_correctness 静默 NaN
      内部 embedding 相似度在备用域名上偶发网络挂起
      默认 180s 超时 → 指标被静默记为 NaN，不报错但分数缺失
      解决：超时调大到 600s + 内置重试（quickstart 注释原话）
   ▼
   ④ metric 对象跨线程共享
      多 run 并发时 evaluate() 把 llm 临时写进共享 metric → 结果串了
      解决：每份 copy.copy()
   ▼
共同点：评测链路本身也是一条要上生产的系统
   它自己也会坏、也会静默出错
   → 评测代码到处是"显式报错、不静默吞、降级可见"
     评测的可靠性当第一优先
```

1. **SSE 跨 chunk 吞事件**（runner）——`requests.iter_lines()` 在事件分隔符跨 HTTP chunk 边界时把空行吞掉，中间事件静默丢失，最后发现回答少了中段。解决：手写逐字节事件切分。
2. **LLM judge 中文引号解析炸**——judge 输出里的中文标点让 RAGAS 的 OutputParser 抛异常。解决：JSON mode 强制合法 JSON。
3. **answer_correctness 静默 NaN**——它的内部 embedding 相似度在备用域名上偶发网络挂起，默认 180s 超时导致指标被**静默记为 NaN**，不报错但分数缺失。解决：超时调大到 600s + 内置重试（quickstart 注释里原话）。
4. **metric 对象跨线程共享**——多 run 并发时 evaluate() 把 llm 临时写进共享 metric，结果串了。解决：每份 `copy.copy()`。

这几个坑的共同点是：**评测链路本身也是一条要上生产的系统，它自己也会坏、也会静默出错**——所以评测代码里到处是"显式报错、不静默吞、降级可见"，把评测的可靠性当第一优先。

**面试官继续追问**：评估集是人工标的，那评测数据本身——一条 EvalRecord 里哪些是标的、哪些是系统跑的？20+ 指标是不是每条都算全？有没有样本被排除在分母外？

## Q10（追问）评测数据的真相：一条 EvalRecord 从哪来、分母怎么过滤、20+ 指标怎么摊

**我：** 评测数据不是"评估集跑一下就有"，它是**三段拼起来的一条完整记录**（`EvalRecord`，`runs/*.jsonl` 一行一条）：

> Q10 评测数据三段来源一图看懂：

```text
EvalRecord = 三段数据拼成（一次问答 → 一份全息记录）
   │
   ① 评估集复制   eval_set_v1.jsonl 直接带过来
      query / intent_l2 / requires_rag / reference_doc_ids
      = 判分的"标准答案"，永远不动
   │
   ② 真实链路     SSE /rag/v3/chat 流式拿
      response / thinking / latency_ms / first_token_ms / final_status
      = 回答质量 & 延迟指标的数据源
   │
   ③ 评测旁路     JSON /rag/eval（app.eval.enabled=true）
      retrieved_doc_ids / intent_pred / has_kb / has_mcp / trace_id
      = 检索证据 & 意图判定的数据源
   ▼
一次问答，三段合成一条
   "问题 + 答案 + 证据 + 意图" 全息可回放
   = 20+ 指标的原材料，全在这一条里
```

1. **EvalRecord 分三段来源**：评估集复制（标准答案）、真实链路 SSE 拿的（回答/延迟）、评测旁路 `/rag/eval` 拿的（证据/意图）。**为什么必须拆三段**：答案和证据不在同一份数据里——SSE 流里只有最终回答，检索证据（docIds/contexts）不体现，必须从旁路拿，两路拼起来才是完整记录。这也是 `schemas.py` 注释里"禁止裸 dict、数据沿三个类型搬一遍"的原因——字段一处定义，读写两端共享，新增字段不会悄悄漏。

2. **20+ 指标不是每条都算全，每个指标有自己"有资格被评"的样本集**：

| 指标组 | 分母过滤（eligible） |
|---|---|
| 检索（Hit@K / Recall@K / MRR） | 只统计 `requires_rag=true` **且** `reference_doc_ids` 非空——SYSTEM 兜底类样本不污染检索指标 |
| 行为（误拒 / 兜底 / 过召回） | 按 `requires_rag` 分流：true 组看误拒/兜底，false 组看过召回 |
| 意图（Top-1） | `intent_l2` 为空的样本不算（触发澄清的没有标准答案） |
| RAGAS 五指标 | `filter_evaluable`：response / contexts / reference 三项齐全 **且** `final_status=success`，其余记 skip_reason 不参与均值 |

3. **所以一条 EvalRecord 能摊出 20+ 指标，但每个指标的分母是它的"合格样本子集"**：`intent_top1 + hit@1/3/5/10 + recall@K(must+inclusive 两种) + mrr@10 + 行为3 + 延迟3 + ragas5` 差不多 25 个，全部从同一条记录算出来，只是各自过滤条件不同——**分母是脚本里写死的 eligibility，不是事后挑的**。

> 💡 面试官追问（真实被问）：既然有分母过滤，+10.3% 这些数字是不是"挑出来的"？
>
> **我：** 不是。分母规则是**在评测脚本里写死的**（`metrics/*.py` 各自的 eligible 过滤、`filter_evaluable`），不是事后挑样本。而且 +10.3% 是 **diff 同一评测集、两次 run 的整体均值之差**——两次 run 用同一批样本、同一条过滤逻辑，两边口径完全一致，差得出来才是真变化。真要挑数字，得把每轮的过滤条件一起改，那不是评测，是表演。

> 💡 面试官追问：你前面说"标注质量决定评测可信度"，怎么防评估集本身骗人？
>
> **我：** 三道闸：① **硬字段校验**——requires_rag / reference_doc_ids 是判分基准，缺了样本直接不参与相关指标，标注不全自己露馅；② **docId 对齐自检**（`sanity_check_doc_id_alignment`）——评测数据里 chunk 维度 docId 和 contexts 文本还原的 docId 不一致时 Hit@1/MRR 会失真，score 阶段先报警，而不是等指标出来才发现被骗；③ **人工校正回流**（manual override 优先，人工值覆盖 RAGAS）——关键样本的标注质量有人兜底。评估集和被测系统一样，是持续维护、防恶化的资产。

---

## 🪝 追问钩子速览（每段答案埋了什么）

| 答案里出现的钩子 | 面试官大概率会追 | 你准备好了没 |
|---|---|---|
| "没有评测就没有尺子" | 评测对 RAG 系统的价值 | ✅ Q1 |
| "自建指标 + RAGAS 两套" | 为什么 RAGAS 不够 | ✅ Q1/Q4（RAGAS 不管链路） |
| "requires_rag 是行为红线锚点" | 评估集设计 | ✅ Q2 |
| "expected_doc_ids 标文档级" | 标注成本取舍 | ✅ Q2 |
| "端到端调真实链路不 mock" | 评测可信度 | ✅ Q3 |
| "为什么要有 /rag/eval 旁路" | SSE 流里没有检索证据 | ✅ Q3 |
| "手写 SSE 解析器" | 跨 chunk 吞事件的坑 | ✅ Q3/Q9b |
| "行为红线：误拒/兜底/过召回" | 产品红线指标 | ✅ Q4 |
| "TTFT 卡点是首字不是完整流" | 延迟口径 | ✅ Q4（小样本 P95 退化） |
| "RAGAS 五指标语义" | faithfulness 怎么算、answer_relevancy 机制 | ✅ Q5（claim 拆分 / 反向生成） |
| "API key 缺失直接报错不兜底硬编码" | 安全红线 | ✅ Q5 |
| "JSON mode 防中文引号" | LLM 评测必踩坑 | ✅ Q5 |
| "n_runs 跑 N 次取均值" | LLM judge 方差压制 | ✅ Q6 |
| "三级降级 + 逐条隔离" | 评测自身可靠性 | ✅ Q6 |
| "reasoning 模型不发 temperature" | 模型差异 | ✅ Q6 |
| "docId 对齐自检" | 数据质量失真预警 | ✅ Q7 |
| "manual override 人工优先" | 人工校正机制 | ✅ Q7 |
| "LOWER_IS_BETTER + 退化阈值" | A/B diff 怎么判退化 | ✅ Q8（0.05 阈值防抖动） |
| "+10.3% 是百分点不是增幅" | 指标口径诚实性 | ✅ Q8 |
| "失败样本 = 优化清单" | 评测驱动优化闭环 | ✅ Q9 |
| "评测也是一条要上生产的系统" | 评测自身的可靠性 | ✅ Q9b |
| "EvalRecord 三段数据源" | 评测数据从哪来、为什么答案和证据分开拿 | ✅ Q10（评估集/真实链路/旁路三段拼） |
| "分母过滤：requires_rag=true、三项齐全才评" | 数字可信度、是否挑样本 | ✅ Q10（分母写死 eligibility，diff 同口径） |

---

## 📋 面试建议（对真实面试的 3 条建议）

1. **把评测讲成"优化闭环"而不是"跑分工具"**。大多数人聊评测只会说"用 RAGAS 算了几个分数"，区分度在于：评测 → failures.jsonl 定位 → 按原因分流（hit@5_miss 查检索、faithfulness_low 查生成）→ 改 → diff 验证不拆东墙补西墙。主动讲这个闭环 + "+10.3% 就是这么推出来的"，就立住了"数据驱动"的工程方法。
2. **"评测自身的可靠性"是你的专属高光**。很少有人想到评测链路自己也会静默出错：SSE 跨 chunk 吞事件、中文引号解析炸、embedding 挂起静默 NaN、metric 跨线程串。你能讲出这四个坑 + "显式报错不静默吞"的原则，面试官会认为你是真跑过大规模评测的人。
3. **两个 ⚠️ 钩子提前备好**：①「+10.3%/+8.0% 的确切口径」——答"同一评测集、优化前后两次 run 的整体均值之差，是百分点差不是相对增幅，n_runs=3 取均值压制 judge 方差，关键样本人工复核"；②「为什么自建指标而不全靠 RAGAS」——答"RAGAS 只管 answer quality，链路各段（意图/检索路由/行为红线/延迟）它完全不覆盖"，顺带展开行为红线三件套（误拒/兜底/过召回）——这两个是简历指标背后的真实工程认知。

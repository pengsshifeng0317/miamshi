# TTL 异步链路透传 · 面试指南

> 主题：`TransmittableThreadLocal` 包装线程池，解决 `CompletableFuture` 异步任务丢失 traceId、链路 `t_rag_trace_node` 断链的问题。
> 使用方式：按叙事线背诵 → 应对追问 → 进阶深挖。全文约 3–4 分钟口述。

---

## 一、面试讲述稿（结构化）

### 第1段 · 背景

> 我在做一个 RAG 检索服务，检索阶段需要对向量召回、关键词召回、知识库检索做**多路并行**，我用 `CompletableFuture` 提交到线程池。项目里有完整的链路追踪，traceId 存在日志的 MDC 里。

讲清两个关键点：① 为什么用多线程（并行提速）；② traceId 的载体（MDC）。

### 第2段 · 现象

> 上线后发现一个链路问题：主链路的 traceId 是 A，但检索这一段（`t_rag_trace_node`）的日志 traceId 要么是空的，要么是另一个值，**链路图上这段是断的**，排查问题时没法把检索耗时、调用链对上。

要点：现象要具体——"哪一段断了""对上不了"。

### 第3段 · 根因（最加分的一段）

> 我定位到根因分两层：

> **第一层，丢值。** traceId 存在 `ThreadLocal`（MDC 底层就是它），ThreadLocal 是**线程私有**的。主线程把 traceId 放在自己的 ThreadLocalMap 里，而 `CompletableFuture.supplyAsync` 的任务是**在线程池的工作线程上执行**——工作线程的 ThreadLocal 里没有主线程的值，所以 `MDC.get("traceId")` 读到 null，日志就丢了。

> **第二层，串值。** 线程池线程是**复用**的。工作线程干完上一个任务，如果 ThreadLocal 没清，会带着上一个任务的 traceId 去执行下一个任务，日志张冠李戴——比丢值更难排查。

加分点：主动说出"丢值 + 串值"两个问题，体现对线程池特性的理解。

### 第4段 · 为什么不直接用 JDK 的 InheritableThreadLocal

> 我第一反应是用 JDK 自带的 `InheritableThreadLocal`，但它只在 **new Thread() 创建那一刻**从父线程拷贝一次值，而线程池是**复用线程**的，复用时不触发拷贝，所以它解决不了线程池场景。

要点：对比讲，体现你考虑过替代方案、知道为什么不行。

### 第5段 · 解决方案

> 我用阿里的 `TransmittableThreadLocal`（TTL）解决。核心就两步：
> 1. 把 `ThreadLocal` 换成 `TransmittableThreadLocal`，并替换 `MDCAdapter`（`TtlMDCAdapter`），让 traceId 能进 TTL 的管理范围；
> 2. 用 `TtlExecutors.getTtlExecutorService()` 包装线程池，把 `CompletableFuture.supplyAsync` 的线程池参数换成包装后的池。

### 第6段 · 原理（核心考点）

> TTL 的核心机制是**在任务提交时捕获、执行时注入、执行后还原**，三个阶段：

> **① capture（捕获）**：发生在**父线程、提交那一刻**。TTL 通过一个全局注册表（holder）遍历所有 TTL 实例，把父线程此刻的值快照下来，藏在任务里。

> **② replay（回放）**：发生在**工作线程、任务执行前**。把快照里的 traceId 写入工作线程的 TTL，同时备份工作线程原本的值。

> **③ restore（还原）**：发生在**任务执行完**。把工作线程的 TTL 恢复成执行前的样子，清掉这次的 traceId，防止串到下一个任务。

> 关键点在于**传递的时机是'任务提交'而不是'线程创建'**——这正好绕开了线程池复用的问题，每次提交都拿到父线程的最新值。

### 第7段 · 落地代码

> 代码上：入口用 `Filter` 在 `doFilter` 里 `MDC.put("traceId", UUID)`，请求结束 `MDC.remove`。业务层把线程池 `TtlExecutors.getTtlExecutorService()` 包一层，`supplyAsync(fn, ttlPool)`。执行任务的线程池用包装后的池子，traceId 就能透传到所有异步分支。

### 第8段 · 踩坑/深度（区分度）

> 我踩过两个坑：
> 一是 `CompletableFuture.supplyAsync(fn)` **不带线程池参数**时默认用 `ForkJoinPool.commonPool()`，那个全局池我没包装，所以一开始 traceId 还是丢的——必须**显式传包装后的池子**。
> 二是 traceId 是在日志 MDC 里的，光有 TTL 不够，还得把 `MDCAdapter` 换成 TTL 的实现，traceId 才能真正被透传。
> 另外 TTL 传递的是**引用**，如果里面装的是可变对象要自己注意线程安全，traceId 这种 String 没问题。

---

## 二、面试官追问 Q&A

| 追问 | 你答什么 |
|---|---|
| 为什么 ThreadLocal 跨线程读不到？ | ThreadLocal 是线程私有 Map，值存在当前线程的 ThreadLocalMap，其他线程访问不到 |
| capture 是在哪条线程上做的？ | 父线程，提交任务那一刻——正因为还在父线程才能读到值 |
| restore 为什么必要？ | 线程池复用，不还原会把这次的 traceId 串给下一个任务 |
| MDC 和 ThreadLocal 什么关系？ | MDC 是日志框架封装的线程私有结构（底层 ThreadLocalMap），所以才会丢 |
| 除了 TTL，还有什么方案？ | ① 手动在任务里 set/remove；② InheritableThreadLocal（线程创建才有效）；③ 显式传参把 traceId 当方法参数传（侵入代码）；④ 上微服务链路组件如 SkyWalking/Sleuth，agent 层面透传 |
| commonPool 的问题怎么彻底解决？ | 用 TTL 的 `TtlForkJoinPool` 替换 commonPool，或所有 supplyAsync 都显式传 ttlPool |

---

## 三、进阶深挖：ForkJoinPool 透传底层

面试官如果往下追 commonPool / ForkJoinPool，答这套。

### 3.1 ForkJoinPool 和 ThreadPoolExecutor 的本质区别

```
ThreadPoolExecutor（普通线程池）
  提交线程 ──放入──▶ 共享任务队列 ◀──拉取── 工作线程
  每个任务由哪个线程执行：基本确定（谁从队列拿走就谁干）

ForkJoinPool（工作窃取）
  每个工作线程有自己的双端队列
  干完自己队列后，会去偷别人的队列任务（work-stealing）
  每个任务由哪个线程执行：不确定（可能被任意线程偷走）
```

**关键推论**：在 ForkJoinPool 下，"提交线程"和"执行线程"**完全脱钩**，而且执行线程随时可能变。所以 TTL 绝不能依赖"提交线程在提交时把值放进执行线程"——那做不到。

### 3.2 TTL 的处理思路（与普通池一致，但更强调"执行时"）

```
  提交线程（父）                   执行线程（可能是被偷的线程）
        │                                │
  ①capture 快照，藏进任务包装            │
        │──────────────────────────────▶│ 任务被某个线程执行
        │                            ②replay：快照写入执行线程 TTL
        │                            ③任务本体执行（日志带 traceId ✅）
        │                            ④restore：执行完还原
```

核心：**capture 在提交线程做，replay/restore 一定在真正执行任务的线程上做**（因为 work-stealing 下执行线程不确定）。这要求包装任务的 run 方法里自己做 replay/restore，而不是提交线程替它做。

### 3.3 实现层面怎么做

- `TtlForkJoinPool`：TTL 提供的 ForkJoinPool 子类，重写 `submit/invoke/execute` 等方法，把任务包成 `TtlForkJoinTask`（内部带 capture 快照，执行时 replay/restore）。
- `ForkJoinTask` 的 fork/join 也要透传：`TtlForkJoinTask` 对子任务同样包装，保证嵌套 fork 也不断。
- `commonPool` 是**静态 final 单例**，不能直接换实例。TTL 通过 `TtlForkJoinWorkerThreadFactory`（注册到系统属性 `java.util.concurrent.ForkJoinPool.common.threadFactory`）给 commonPool 换线程工厂，让工作线程挂上 TTL 能力。

### 3.4 和普通线程池对比如下，方便口述

| 维度 | ThreadPoolExecutor | ForkJoinPool |
|---|---|---|
| 任务分配 | 共享队列，谁取谁执行 | 各线程私有队列 + work-stealing |
| 执行线程确定性 | 相对确定 | **不确定**（会被偷） |
| TTL 处理 | 提交时 capture，执行线程 replay/restore | 同样三段式，但 replay/restore 必须在执行线程内做 |
| 包装方式 | `TtlRunnable`/`TtlCallable` | `TtlForkJoinTask` + `TtlForkJoinWorkerThreadFactory` |
| commonPool 替换 | 不需要 | 静态 final，只能换线程工厂 |

### 3.5 一句话收尾

> 普通池靠"提交时捕获 + 执行时注入"，ForkJoinPool 因为 work-stealing 执行线程不确定，原理一样但 replay/restore 必须放在任务自身内部执行；commonPool 是静态单例，TTL 通过替换线程工厂接入。

---

## 四、面试节奏建议

1. 先讲现象和根因（第2–3段），这是判断你是否真做过的分水岭——不要一上来背 TTL 原理。
2. 原理三段式（capture/replay/restore）讲得慢、讲清楚。
3. 踩坑两点（commonPool、MDCAdapter）主动抛出，证明不是背的。
4. 全程 3–4 分钟，面试官有追问再展开。

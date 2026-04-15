# 从 Agent 到 Flink 到 Compiler——我发现的一些意外关联

*从图结构到抽象层次，一次跨越三个时代的技术观察*

Last Updated: 2026-04-15

---

2026 年 4 月一个下午，我在和 Claude 讨论我正在做的 Agent 系统和 Pydantic AI、LangGraph 的架构差别。

聊着聊着，话题滑到了 LangGraph 的 channel 设计、Flink 的 typed dataflow、Spark 的 DAG 调度、Scala 的 Actor，然后滑到了 1974 年的 Kahn Process Networks 和 1973 年的 Hewitt Actor Model。聊到一半的时候 AI 说了一句话：

> "你做过数据中台、用过 Spark 和 Flink，又在做 Agent，你正好站在两个时代的缝隙上——既见过旧世界的武器，又能给新用户（LLM）打造接口。"

我停下来盯着屏幕看了一会儿。

---

我 2023 年毕业，大学里有两门课学得特别认真——**操作系统**和**编译原理**。编译原理作业里我写过一个小编译器 demo，觉得"把一段源码变成可执行的东西"这件事非常美。

第一份工作做**数据中台**——主要是搭基础设施、服务别人用。少部分任务自己要写 Spark 批处理和 Flink 实时处理，半路出家，跑起来就行。checkpoint 机制、反压、一致性语义，用着但并不真懂。2025 年开始转做 Agent，现在主要做的就是一个 Agent 系统。

那天下午和 Claude 聊到一半我突然发现：我现在做的 Agent 里，我正在实现的那套东西——**节点的持久化状态、基于 channel 的依赖、带类型的执行图、失败恢复**——**一部分是我大学学过但当时不知道有用的东西，另一部分是我工作里短暂用过的东西**。

- 操作系统教的 state 管理、进程调度、IPC，现在是 Agent 里的 memory、scheduler、multi-agent hand-off。
- 编译原理教的 AST、IR、代码生成，现在是各种形态的 Agent 执行图——LLM Planner 生成的 Plan 图、n8n / Coze / Dify 里拖拽出来的流程图、LangGraph 里工程师手写节点但由引擎执行的图。写图的作者不同，但本质都是"一张 IR 交给图引擎执行"。
- Flink 里我没搞懂过的 checkpoint、反压、watermark，现在是我要给 Agent 自己写的东西。

我通过和 Claude 聊天，一点点被它带着把那些老东西串起来。聊到一半我意识到：**我大学有些了解的、工作时边干边学的那些东西，其实都是给我现在正在做的事情准备的**。

我是一个资历不深的工程师，写这篇不是要给任何人讲历史。敢写是因为这个观察不需要 30 年经验——你只要在几个技术栈之间浅浅地穿行过（编译原理 + OS + 数据中台 + Agent 碰巧是我的路径），在和 AI 讨论自己正在做的东西时，它会把你过去学过的、用过的、还没搞懂过的东西重新串起来。这件事在 ChatGPT 之前不会发生。

写 LangGraph、Pydantic AI 的那些作者大概早就意识到自己在重演什么。我只是作为一个**半路出家的后来者**，第一次意识到的时候被震撼了，想把这个震撼分享出来。

---

## 一、那张对照表

开场里我随手列了三行对应关系。真实情况是——**远不止三行**。

整个 Agent 圈在 2024-2026 年里冒出来的"新概念"，几乎每一个都在计算机系统工程 20-50 年前的遗产里有一个精确对应物。不是"思想上相似"，是**解决的是同一个问题、用的是同一套思路**。

我把手上 20 多个 Agent 框架源码研究笔记里出现过的对应关系整理成一张表，按系统工程的五大经典子系统归类。这张表不完整，但已经够震撼：

### A. 状态与持久化

| Agent 概念 | 历史对应 | 年代 |
|---|---|---|
| LangGraph checkpointer | Chandy-Lamport snapshot / Flink Checkpoint | 1985 / 2014 |
| Agent long-term memory | 虚拟内存 / 文件系统 | 1960s / 1970s |
| Session state (Google ADK) | PHP / Rails session | 1990s / 2000s |
| Prompt cache | CPU L1/L2/L3 cache | 1960s |
| Context compaction | GC / LRU page eviction | 1960s / 1970s |

### B. 并发与通信

| Agent 概念 | 历史对应 | 年代 |
|---|---|---|
| Multi-agent hand-off | Actor Model (Hewitt) / Unix pipe | 1973 |
| Tool use | syscall / RPC | 1970s / 1980s |
| Sub-agent spawn | Unix fork() / Erlang spawn | 1970s / 1986 |
| MCP (Model Context Protocol) | CORBA IDL / gRPC proto / LSP | 1991 / 2015 / 2016 |

### C. 调度与执行

| Agent 概念 | 历史对应 | 年代 |
|---|---|---|
| ReAct loop | REPL / JS event loop | 1960s / 1990s |
| Planner | Volcano / Cascades query optimizer | 1994 / 1995 |
| Workflow DAG (Mastra / Dify) | Airflow / Luigi / Dagster | 2014+ |
| Dataflow firing | Kahn Process Networks / Synchronous Dataflow | 1974 / 1987 |
| pydantic_graph typed edge | Lustre / Esterel | 1984 / 1987 |

### D. 资源管理

| Agent 概念 | 历史对应 | 年代 |
|---|---|---|
| Token budget | Memory allocation / quota | 1960s |
| Rate limit / backpressure | Flink credit-based backpressure | 2014+ |

### E. 可观测性

| Agent 概念 | 历史对应 | 年代 |
|---|---|---|
| Langfuse / Helicone trace | Dapper / Jaeger / OpenTelemetry | 2010 / 2017 |
| Typed output boundary | Contract / schema validation | 1970s+ |

### 不是列表，是子系统

这张表的真正含义不是"某个 Agent 功能对应某个历史工具"。更深的含义是——**把右边所有东西拼起来，你得到的是一个完整的操作系统**。

Agent 圈里散落着 checkpoint、scheduler、IPC、protocol、cache、GC、APM——这些东西在一个成熟的操作系统（Unix、Windows、JVM）里全都是**必须同时存在的子系统**。你不能只做 scheduler 不做 memory，不能只做 IPC 不做权限。

**Agent 圈正在做的事情不是"加 AI 功能"，而是在搭一个新的操作系统**——只是这次底下不是 CPU，而是 LLM。

挑两个第三章不会再展开的对应讲一下——

**Planner = Query Optimizer**。AgentScope 等一些研究性 Agent 框架里有"先出 Plan、再执行"的设计，Plan 是一棵带依赖的任务树。这就是数据库的**查询优化器**——SQL 先被解析成逻辑计划、再被优化成物理执行计划。Goetz Graefe 1993-95 年的 Volcano/Cascades 框架已经把这套讲透。**Agent 圈目前 "Plan generation" 的水平大概相当于 1985 年的查询优化器**——能跑、但没有 cost model、没有多方案比较。

**Prompt cache = CPU cache**。Anthropic 和 OpenAI 2024 年先后推出的 prompt caching，本质是**在 LLM 这个贵的计算单元外加一层带 TTL 的缓存**。这和 1960 年代 IBM 给 CPU 加 L1 cache 解决的是同一个问题：**计算单元太贵，用一层廉价高速存储摊销重复访问**。

表里剩下的那些（checkpoint、multi-agent、dataflow 等）下一章会逐个展开。

**这不是巧合。下一章讲为什么这种重演必然发生**。

---

## 二、为什么重演必然发生

先说结论：**计算机系统工程有四件永恒的事——状态、并发、容错、抽象。每次换一个新的"执行基底"（从大型机到 PC，从 PC 到分布式，从分布式到 LLM），这四件事都要在新基底上重做一遍**。

### 2.1 四件永恒的事

- **State**：信息要存在哪里、怎么改、崩了怎么恢复
- **Concurrency**：多件事同时做，怎么不冲突、不死锁、不竞争
- **Fault tolerance**：任何一个零件都会坏，系统要有弹性
- **Abstraction**：复杂度必须被分层管理，否则写不下去

这四件事 70 年前就存在，70 年后也会存在。**变的是执行基底，不变的是这四件事**。

> **Hardware changes, software principles don't.**

不管底下是打孔卡、晶体管、GPU 还是 LLM——只要它是一种"按规则把输入变成输出"的东西，你在它上面搭应用时就逃不开这四件事。

### 2.2 LLM 是最新的"新媒介"

现在有了一个新的执行基底——LLM。它有几个前所未有的特性：

- **概率**：同样输入每次输出略不同
- **慢**：单次调用 100ms-30s，比任何其他计算单元都慢三个数量级
- **贵**：每次调用要真金白银花 token
- **有语言理解能力**：能接受自然语言指令，能产出结构化输出

但 LLM 作为系统的一部分运行时，**还是要处理四件永恒的事**——要有状态（记得之前说过什么）、要有并发（多个 Agent 或多个 tool 并行）、要有容错（超时、失败、幻觉）、要有抽象（不能把所有逻辑堆在一个 prompt 里）。

所以新媒介 + 老问题 = **必然重演**。

Agent 圈正在做的事，就是把"状态、并发、容错、抽象"这四件事在 LLM 这个新基底上**重新实现一遍**。checkpoint 重新做、scheduler 重新做、message passing 重新做、protocol 重新做、observability 重新做。

**每个"新"的东西都有一个老答案**，因为问题本身是老的。

### 2.3 重演的经济学——为什么不直接复用老轮子

既然老答案都在，为什么不直接用？两个原因：

**1. 老轮子针对旧媒介优化，不适配新场景**。Flink checkpoint 是为"每秒百万条、单条处理毫秒级"设计的；Akka Actor mailbox 是为微秒级消息设计的。Agent 里一次 LLM 调用要 5 秒、消息动辄几百 KB——这些老实现的很多优化在 Agent 场景里要么失效、要么是浪费。**复用 ≠ 照搬**，老答案提供的是**思路**，不是**实现**。另一边，老轮子的 API 本身也对新手不友好：Akka、Flink、Erlang OTP 都要啃几个月才上手——而 Agent 圈的从业者大多是 Python 应用层上来的，直接用老工具的学习成本可能比重新造一个还高。

**2. Agent 圈的从业者大多不知道老轮子存在**。一个 2024 年入行做 Agent 的工程师，极大概率没学过分布式系统论文、没读过 PL 经典、没写过流处理——遇到 "Agent 怎么从失败中恢复" 这个问题时，第一反应是自己想一个方案，而不是去搜 1985 年的论文。**这件事一部分是遗憾，一部分是必然**。每一代工程师都会重新发明前人的东西，这是计算机文明的常态。但如果能意识到自己在重演，至少可以把重演做得更快、更好、不踩同样的坑。

到这里为止讲的都是高层对应。但"高层相似 ≠ 本质同构"——下一章挑三个具体对应关系，讲透它们在理论、工业化、Agent 应用三代之间的精确传承。每一个都是我自己碰过的——一个我写过一点、一个我学过、一个我现在正在做。

---

## 三、三个深挖——从理论到 Agent 的精确传承

### 3.1 Case 1: Checkpoint 恢复——从 Chandy-Lamport 到我的 Agent

**起点：1985 年的一个分布式系统难题**

想象一群互相发消息的计算机在协作。某一时刻，你想"拍一张照片"——把所有机器在同一瞬间的状态保存下来。

听起来简单？**根本做不到**。

因为这些机器没有共享时钟。你让 A 机器"现在存一下状态"时，A 存完状态的那一刻，B 机器可能已经给 A 发了一条消息但 A 还没收到。那 A 存的状态和 B 存的状态就对不上——B 记得自己发了消息，A 不知道。**这不是一致的快照**。

1985 年，K. Mani Chandy 和 Leslie Lamport 发表了一篇叫 *Distributed Snapshots: Determining Global States of Distributed Systems* 的论文，用一个非常漂亮的算法解决了这个问题：

1. 发起方先保存自己的状态，然后往所有 channel 发一个特殊 marker
2. 每个节点收到 marker 时：保存自己的状态，然后往下游发 marker
3. 在收到 marker 之前从某个 channel 来的消息，都算是"快照一部分"，要记录下来
4. 最后把所有节点状态 + 所有 channel 上"在途消息" 合起来——就是一个一致快照

这个算法的核心洞察是：**用一个"标记消息"把"快照之前"和"快照之后"划分开**。你不需要全局时钟，你需要的是**一个可以在 channel 里传播的边界**。

**Kahn 的 KPN（1974）+ Chandy-Lamport 的 snapshot（1985）**，这两个工作合起来构成了后续所有"容错分布式数据流"系统的理论基础。

**2010s：Flink 把它工业化**

Flink 把 Chandy-Lamport 工业化。它把"marker"叫做 **checkpoint barrier**——一个特殊的消息，和普通数据消息一起在流里流动。barrier 经过一个算子时，算子把自己的状态持久化；所有算子都对齐到同一个 barrier 时，一个 checkpoint 就完成了。

这是 Flink "exactly-once semantics" 的基础。你在 Flink 程序里写的那一行：

```java
env.enableCheckpointing(5000)
```

背后就是这篇 1985 年的论文。**每次打开 Flink 任务的日志，里面那些 "Checkpoint N triggered" 的记录，本质是在按 Chandy-Lamport 的算法拍照**。

我在数据中台时调过一次 Flink 任务，遇到过 checkpoint 失败——那时候我的处理方式是"加大超时、重启任务、能跑就行"。背后的算法我当时根本不知道，日志里那些"checkpoint barrier timeout"的字样我也没深究。这是大多数 Flink 使用者的真实状态——**用着工具，不真懂工具**。

**2024+：LangGraph 和 Agent 的持久化**

LangGraph 是 LangChain 的图执行引擎。它的一个核心特性是**可恢复**——一个长流程 Agent 跑到一半挂了，可以从上一个节点的状态恢复继续执行。

它的 checkpointer 是怎么做的？看源码你会发现——它保存每个节点执行完之后的**完整图状态快照**，包括 channel 里的 pending 消息。存储后端可以是 SQLite、Postgres、Redis。

**这就是 Chandy-Lamport 换了一身 Python 皮**——channel、marker、状态快照、channel 里的在途消息，概念一一对应。

**同一个算法，40 年服务三代工程师**：
- 第一代：分布式系统研究者（Chandy-Lamport 时代）
- 第二代：流处理工程师（Flink 时代）
- 第三代：Agent 工程师（现在）

每一代的使用者都觉得自己在解决"新问题"。每一代都少数人意识到自己在重演。

---

### 3.2 Case 2: 编译器 IR——从大学 demo 到 LLM 作为编译器

这一段可能不少读者会陌生——写过编译器的人不多。

**传统编译器的分层**

大学编译原理课讲的那条 pipeline：源码经过词法分析、语法分析、语义检查，生成 AST、再生成 IR（LLVM IR / 字节码），经过优化 pass、最后输出目标机器码。这是编译器 70 年的主要成就——**自动化了"已经被人结构化的源码"到"机器指令"的翻译**。

我大学时写过一个简单的编译器 demo——词法分析、语法分析、生成一个简化的 IR、最后生成目标代码。写到 IR 那一步时，我印象很深的是：**IR 是一种"机器能看懂但不是机器指令"的中间表示**。它比源码更接近机器、但比机器码更易操作。IR 是整个编译器的骨架。

当时我觉得编译器是一种"把人的意图精确落到机器能做"的魔法。

**但我当时没意识到的是**：这个魔法只覆盖了一整条"意图→机器码"链的**下半段**。

完整的链其实是这样：

```
模糊的人类意图  →  结构化的源码  →  IR  →  机器码
                ↑                 ↑
             人写              编译器写
```

**左边那个箭头一直是人做的**。编译器再强大，也只能处理"已经被人写成源码"的东西。"意图→源码"这一步 70 年来没有任何工具能替代人类。

**LLM 补全了编译栈**

LLM 做的事情，换个角度看，就是**自动化了"意图→结构化输出"这一层**。

你给 LLM 一段自然语言任务描述，它能输出：
- Python 代码（一种源码 IR）
- JSON（一种结构化数据 IR）
- SQL（一种查询 IR）
- Plan 图（Agent 场景下的 plan IR）

**Agent 的 Planner 本质就是一个编译器**——输入自然语言意图，输出一张带依赖的图（Plan）。

具体的映射：

| 传统编译器 | Agent 系统 | 对应关系 |
|---|---|---|
| 源码 | 自然语言任务 | 输入 |
| Lexer + Parser | LLM 的理解阶段 | 解析 |
| AST | Plan 的中间表示 | 结构化内部表示 |
| 类型检查 | 字段对齐 / schema 校验 | 静态检查 |
| IR | Plan 图 | 可执行中间表示 |
| 优化 Pass | （目前基本没人做）| 优化 |
| 代码生成 | Plan 实例化 | 最终产物 |
| VM / CPU | Agent runtime | 执行环境 |

**LLM 是 70 年来第一个能自动化编译栈最上层的东西**。

这件事的含义比它表面看起来大得多。过去 70 年软件工程的所有方法论——OOP、设计模式、DDD、敏捷、TDD——**本质都是在帮人类做好"意图→源码"这一步**。因为这一步只能人做，所以大家发明了一堆方法教人做得更好。

现在这一步可以被机器做一部分了。

**但 LLM 有一个传统编译器没有的新问题**

不过，等一下——**LLM 作为编译器有一个根本缺陷**。

传统编译器是**确定的、可复现的**。`gcc foo.c` 两次，得到同样的二进制。编译器有 bug 是大新闻——GCC 每个 bug 都有编号，会被严肃修复。

LLM 作为编译器是**概率的、不可复现的**。同样 prompt 两次，编出来的 Plan 可能完全不同。这在编译器理论里是**反常**——传统编译器根本不会设计成这样。

所以 Agent 作为一个"使用概率编译器的系统"，必须处理一个传统软件里没有的问题：**编译失败不是 bug，是运气**。

传统编译器:
```
编译失败 → 报错 → 人看错误 → 修代码 → 重编
```

Agent 编译:
```
LLM 生成失败/错误 → 重试？重新写 prompt？验证？回滚？
```

这导致 Agent 的 runtime 必须内置 **retry、verify、fallback、replan** 作为一等公民——这些在传统 VM 里根本不存在。这块第五章再展开。

**LLM 是一种新型编译器，Agent 是它的配套 VM**。Agent 工程师现在做的事，本质是**在给一种全新的"概率编译器"设计运行时**——这件事 70 年前没发生过。

---

### 3.3 Case 3: Dataflow——从 KPN 到 LangGraph 的 Channel

**1974 年：一个并发计算模型**

Gilles Kahn 1974 年发表了一篇 12 页的论文 *The Semantics of a Simple Language for Parallel Programming*，提出一个极简的并发模型：

1. **Process**：独立的顺序计算单元
2. **Channel**：process 之间只能通过无界 FIFO 通信
3. **读阻塞**：从 channel 读数据，没数据就等
4. **写非阻塞**：往 channel 写永远能写

没有锁、没有共享内存、没有全局时钟。Kahn 证明了一件反直觉的事：**只要每个 process 是确定性函数，整个网络的输出就是确定的——不管调度顺序如何**。这叫 **Kahn's Principle**。

它为"用 channel + 节点组织并发计算"这条路提供了数学基础。后续 50 年所有 dataflow 系统都从这里衍生。

**50 年后，这些系统还在跑**

KPN 的工业后代不是"被遗忘的研究原型"——它们是现在每天都在处理几十亿条数据的生产系统：

- **Lustre / Esterel**（INRIA，1980s）——用于空客飞控软件。你每次坐 A320、A380，飞机上的飞控逻辑就是 Lustre 编译出来的代码
- **Synchronous Dataflow (SDF)**（Edward Lee, UC Berkeley, 1987）——给每个端口声明"消费速率"，使 dataflow 图可以在编译期静态调度
- **Apache Flink**（2014）——阿里双十一的实时数据大屏、Uber 的动态定价、Netflix 的流媒体质量监控，底层都跑在 Flink 上
- **Apache Beam**（2016）——Google 内部大量流处理作业的抽象层

Flink 的 DataStream API 是 KPN 的一个扩展变体——加上了 watermark（时间语义）、keyed state（分区）、backpressure（有界 buffer）。每次你在 Flink 里写：

```java
DataStream<Event> events = source.keyBy(...).map(...).filter(...)
```

你就是在写一张 KPN 图。

**这不是考古学，是当前还在生产运行的基础设施**。

**LangGraph 里的 Channel 和 Reducer**

快进到 2024 年的 LangGraph。它的状态模型里有两个核心概念：

- **Channel** —— 每个 state 字段背后是一个 channel，节点之间通过 channel 传递消息
- **Reducer** —— 多个节点向同一个 channel 写入时，用 reducer 合并。声明方式是 `Annotated[list, add_messages]`——告诉引擎"这个字段是一个 channel，新值用 add_messages 合并到旧值"

这两个概念和 KPN / Flink 那套 channel + reducer 是**同一套抽象的再次应用**。LangGraph 的 `Annotated[list, add_messages]` 本质就是 Flink 的 keyed state + reduce function 在 Python/LLM 场景下的翻译。pydantic_graph 的"从 return type annotation 推导边"则是 typed dataflow 的一种变体。

**同一个抽象，两种完全不同的工作负载**：

| | Flink | LangGraph |
|---|---|---|
| 消息频率 | 每秒百万条 | 每任务几十条 |
| 消息结构 | 同构（`Event` 或类似类型） | 异构（每条可能完全不同） |
| 图生命周期 | 月/年级（长 running job） | 秒/分钟级（每任务一张） |
| 典型负载 | CPU / 网络 bound | LLM 调用 bound |

同样的 channel + reducer 既能撑起每秒百万条的实时流，又能撑起 LLM agent 的消息传递。**抽象的普适性在这里体现得非常明显**——好的抽象一旦成型，会在不同时代、不同场景反复找到新的应用。

**我自己做的 Agent 和这件事的关系**

我做的 Agent 里有一套"字段级依赖"设计——每个节点声明需要哪些字段、产出哪些字段，Planner 在生成 Plan 时按字段对齐自动连接节点。

设计这套的时候，我没看过 Kahn 的论文、没深读过 Ptolemy II，也没仔细比对过 LangGraph 的 channel 模型。我当时的想法很朴素："Planner 生成 Plan 时老是字段名对不上，我得给它加点约束"。

后来和 Claude 聊才知道——**我独立重新发明了 dataflow programming 的一个变体**。

这件事让我既沮丧又兴奋。沮丧：原来我以为的"新设计"50 年前就有人做过、20 年前工业化过、去年在 LangGraph 里已经以 channel + reducer 的名字存在了。兴奋：Edward Lee 他们那些 paper 里解过的问题（调度、反压、fault tolerance、类型系统），我可以直接去读答案，不用再绕弯。

**独立重新发明前人答案**，是 Agent 圈 90% 工程师的常态。能意识到自己在重演的人，只占剩下的 10%。

---

## 四、但不要忘了 Bitter Lesson

到这里为止，前三章给读者建立的印象是——"Agent 圈的很多设计其实是老遗产的复活"。

但这个印象**只对一半**。如果我只写到这里就收尾，会给读者一个危险的误导：**以为 Agent 设计的未来就是"把 KPN/Actor/Chandy-Lamport 搬过来"**。

这个方向大概率是**错的**。

### 4.1 现实：ReAct 在主导

我第一篇 blog 里调研过当前主流 Agent 框架的执行模型，结论是——**几乎所有主流框架的底层都是 ReAct 循环**。Mastra、Google ADK、Pydantic AI、LangChain classic、Claude Code 的 agent loop——全是 ReAct。

ReAct 是什么？它**几乎没有结构**。LLM 每一步看当前上下文、决定下一步做什么、执行、看结果、再决定。一个 while 循环 + 一个 prompt。**没有 Plan、没有 dataflow 图、没有字段对齐**。

这个"粗糙"的模型在主流框架里**压倒性地赢**过精致的 Plan-and-Execute、dataflow、结构化 Planner。

为什么？

### 4.2 Sutton 的 Bitter Lesson

2019 年 Richard Sutton 写过一篇叫 *The Bitter Lesson* 的短文，讲的是 AI 研究 70 年最反复印证的一个教训：

> 所有"用人类知识/结构/工程约束来帮 AI"的方法，**长期都被"scale + 通用学习"打败**。

例子：国际象棋手工评估 → Deep Blue 暴搜 → AlphaZero 自对弈；计算机视觉手工特征 SIFT/HOG → CNN 扫光；NLP pipeline（parse → NER → coref）→ LLM 扫光。**每次都一样**：人类工程师精心设计的结构，短期有效，长期被更大规模的纯学习扫光。

### 4.3 P&E / dataflow 是同类尝试

Plan-and-Execute、字段对齐、程序化校验、typed dataflow——这些 Agent 架构设计本质都是"**工程手段补救 LLM 能力不够**"：字段对齐防止 LLM 乱起字段名，程序化校验防止输出垃圾数据，P&E 防止 LLM 走到半路迷路，typed graph 防止连错节点。前提假设是 LLM 会犯错，所以要加一层护栏。

那么反过来问：**如果 LLM 足够强，这些护栏还需要吗？**

历史先例摆在那——**1970 年 GCC 生成的汇编比人手写慢 30%**，所以"性能敏感代码必须手写汇编"是当时行业共识。到 2000 年，GCC 追平人类；到 2010 年，GCC 超过人类。**手写汇编从"最佳实践"变成"技术债"**。

那套"手写汇编" 的经验智慧不是错的——它在当时就是对的。只是它的前提（编译器不够强）消失了。

**Plan-and-Execute、字段对齐这些东西在今天 LLM 水平下是对的**。但它们可能是"手写汇编"那个位置——**随着 LLM 变强，它们会从必要变成多余**。

2023 年 GPT-4 连续正确执行 5-10 步 agent loop 就算不错。2025 年的前沿模型已经能长时间无人监督完成复杂任务，Anthropic 内部有 demo 让 Claude 连续工作 7 小时完成一个完整任务。

**智能这一轴两三年之间有了数量级的跃升**。如果这个速度持续——很多"为了保护 LLM 不放飞"设计的东西，会变成多余。

### 4.4 Qualifier：重演不是接管

所以前三章讲的"复活"，不能理解为"老范式要接管 Agent 主流"。

更准确的表述是：**老范式会在关键场景守住**——

- 审计场景（金融、医疗、政府）——必须有可追溯的 plan
- 预算敏感场景——必须能预估 token 成本
- 高确定性场景——不能有幻觉
- 长流程场景——必须有 checkpoint

**这些场景不是主流，但它们存在**。

主流会继续 ReAct。关键场景会继续用"老范式改造"的结构。两者长期共存。

**优雅的结构反复输给暴力的 scale，这是计算机史最反复印证的教训**。但反复输不等于消失——每次被主流抛弃后，老范式都在某个具体场景里活下来：**Erlang 活在电信、Haskell 活在金融、Lustre 活在航空**。Agent 圈的"概率约束层"可能就是 dataflow 下一个栖身之所。

### 4.5 我个人的更新

写这篇文章的过程中，我和 Claude 还讨论过"P&E（Plan-and-Execute）会不会成为主流"这个问题。一开始我偏向"会"——毕竟我自己做的 Agent 就是 dataflow 风格的 P&E 架构。

但在被 Bitter Lesson 反驳之后，我更新了立场——**大概率不会**。我做的东西更合适的定位不是"Agent 的主流架构"，而是"Agent 在关键场景下的安全底盘"。ReAct 是引擎，我做的是底盘和安全带。

这不是撤退，是更准确的定位。**我押的不是"dataflow 必胜"，是"关键场景永远需要约束"——后者几乎是永真命题**。

---

## 五、真正新的东西

承认 Bitter Lesson 之后，还有一件事要说清楚——**并不是什么都在重演**。

Agent 圈有三件事是**真正没有先例**的。如果只讲重演不讲这些，会给读者另一个错误印象——"哦原来 Agent 什么新东西都没有"。这也不对。

### 5.1 LLM 作为概率编译器

第三章 Case 2 里提过：LLM 作为编译器有一个传统编译器没有的特性——**概率**。

传统编译器：
- 相同输入 = 相同输出
- 编译失败 = 编译器 bug（严重问题，必须修）
- 优化 pass 是确定性变换

LLM 编译器：
- 相同输入 ≠ 相同输出（有 temperature / sampling）
- 编译失败 = 运气（重试可能就好了）
- 优化 pass 这个概念还不存在

**这是编译器理论中从未处理过的情况**。确定性是编译器定义的一部分——"compiler is a function from source to target"。LLM 作为编译器破坏了函数性。

所以围绕 LLM 编译器，必须发展一整套新的工程学：

- **重试策略**（retry with backoff）——传统编译器不需要
- **验证器**（verifier / checker）——执行前验证 LLM 产出
- **回滚机制**——执行到一半发现产出有问题
- **多候选 + 选择**（类似 Beam Search 但对输出）
- **Cost model**——哪次重试值得、哪次该放弃

这些东西在传统编译器里要么不存在，要么是边角料。**在 Agent 系统里是一等公民**。

**最接近的历史先例**：JIT + 自适应优化、speculative execution。但那些仍然是确定性的，只是决策延后。LLM 是真的概率。**这一块是 Agent 系统对计算机科学的真正贡献之一**——我们不得不发明"概率编译器"的工程学。

### 5.2 运行时动态图

Flink 图是编译期固定的，提交 job 之前就定型。然后跑几个月、几年都不变。图一对多——**一张图处理亿万条同构数据**。

Agent 图是运行时生成的，每次任务一张新图。跑几秒几分钟就扔掉。图一对一——**一张图处理一次具体任务**。

这个差别不是量变，是质变。

```
Flink  图生命周期：月/年级
Agent  图生命周期：秒/分钟级
```

一个推论：**图的经济学完全不同**。

Flink 花两周让工程师手调一个静态拓扑值得——因为要跑一年。Agent 一次任务 30 秒，没人会为这 30 秒手画 DAG——所以必须让 LLM 运行时生成图。

**当"画图的成本" < "一次任务的价值"时，运行时画图才成立**。LLM 的便宜让这个不等式第一次在计算机史上成立。

这件事挑战了 70 年软件工程的核心教义——"**抽象是为了复用**"。过去所有的设计模式、OOP、函数式、微服务……都在追求"写一次，用 N 次"。

Agent 里出现了一种新的计算经济学——**"不值得抽象，现场具现更便宜"**。对每个任务当场画一张一次性的图，比维护一张通用的复用图更划算。

**这是一种从未有过的"一次性软件"思维**——Flink 抽象是为了复用，Agent 图是为了不复用。

### 5.3 自然语言作为 IR

过去所有的编译器 IR 都是**形式语言**——LLVM IR、JVM 字节码、WebAssembly、SSA form、GCC GIMPLE。

形式语言的特征：
- 语法严格
- 可以静态分析
- 编译器之间可以互转
- 机器能完全理解

Agent 里出现了一种新型 IR——**半结构化自然语言**。

最典型的例子是 plan："先调用搜索工具，然后用搜索结果生成摘要，如果摘要长度超过 500 字就分段输出"——这是一个 plan，它不是 JSON、不是 DSL、是自然语言描述了一个执行流程。

为什么用自然语言？因为——

- LLM 之间对齐容易（不用学新的 DSL）
- 表达力强（形式语言塞不下的隐式规则可以用自然语言写）
- 生成成本低（LLM 天然生成自然语言）

代价是：
- 无法静态分析（传统编译器工具链全失效）
- 不可精确复现
- 解析依赖另一个 LLM

这里有一个有趣的历史对照——**Cobol**。1959 年 Cobol 设计时就想"让编程语言看起来像英语"：

```cobol
ADD 1 TO COUNTER.
MOVE "HELLO" TO MESSAGE.
```

结果呢？**Cobol 是最严格的形式语言之一**——每个词的含义固定、不能有歧义。它只是**形式语言穿了英语的皮**。

Cobol 式 "伪自然语言" 失败了 70 年。真·自然语言作为 IR 一直是工程师的梦——直到 LLM 让它第一次可行。

**但这里有一个深刻的不对称**——自然语言 IR 可行，不是因为我们终于让计算机"理解"了自然语言，而是因为我们在两边都用 LLM：写 plan 的是 LLM，解释 plan 的也是 LLM。**自然语言 IR 只在 LLM 之间的对话里可行**。人类工程师看 plan 仍然要一条条解析。

---

## 六、对从业者的含义：读什么

写到这里，不能以"希望大家拥抱 Agent 时代"这种空话收尾。我想给一个**具体的可执行建议**——

**Agent 圈未来 5 年真正的技术突破，很大一部分不在 Agent 圈内部，在老遗产里**。

理由是：
- Agent 圈的论文产出速度 < 遗产消化速度
- 每个"Agent 新问题"几乎都能在老领域找到 30 年研究
- 看 Agent 圈 blog 的信息密度远低于看经典论文

所以如果你做 Agent、遇到一个具体工程问题、花一个下午搜索——**大概率能在老文献里找到一个比 Agent 圈的帖子质量高 10 倍的答案**。

下面这个清单按我自己的使用频率排序，每条标个粗略难度（★ 一星 = 入门、★★★★ 四星 = 硬核论文）：

### 阅读清单

| 想深入 | 读什么 | 难度 |
|---|---|---|
| Agent checkpoint / 恢复 | Chandy-Lamport 1985 《Distributed Snapshots》（12 页） | ★★ |
| Agent 并行调度 / 反压 | Flink 官方 paper + credit-based backpressure 设计文档 | ★★★ |
| Agent 类型系统 / 依赖图 | Edward Lee《Introduction to Embedded Systems》第 3 章 | ★★★ |
| Agent 增量执行 / replan | React Fiber 架构文档 + Umut Acar 的 incremental computation 教程 | ★★★ |
| Multi-agent 消息传递 | Joe Armstrong《Programming Erlang》前 6 章（强推，只看这 6 章也值） | ★★ |
| Agent Planner 优化 | Goetz Graefe 1993《Volcano Query Optimizer》| ★★★★ |
| 图调度 / Dataflow 基础 | Edward Lee 的 Ptolemy II 项目网页 overview | ★ |
| LLM-as-OS 视角 | Maurice Bach《The Design of the Unix Operating System》前三章 | ★★ |

### 一个心法

不是要每个工程师都变成 Leslie Lamport。我自己也读不完上面所有东西。

但**每隔几年做一个新东西时，花一个下午搜一下"这个问题学术界有没有解过"**——这个习惯值 10 个具体技术栈。

2026 年大多数工程师的第一反应是**"问 AI 怎么做"**，而不是"搜论文看看"。AI 做得很好的一件事，是**帮你把散落的历史脉络快速串起来**——就像这篇文章的起点那样。但如果要深入某个具体问题（设计 Agent 的 checkpoint、选 reducer 语义、排查一个死锁），经典文献提供的是 AI 生成答案背后那套问题分析空间和权衡。**两者不是替代关系，是不同深度**。

做工程师久了会发现：只有扎进过原始文献，才能从"能跑"升级到"知道为什么能跑"。

---

## 结尾：武器库和时代窗口

### 我的武器库

那天和 Claude 聊完后，我花了一个晚上把讨论记录整理成一份文档。写到一半我意识到——**我人生中学过的那些"当时觉得没用的东西"，一件都没浪费**。

让我把自己的武器库列一遍，作为一个鼓励同样半路出家的工程师的具体例子：

- **大学编译原理**——当年写那个 demo 时觉得是课程作业，现在支撑我理解 Planner 的本质、理解为什么 Plan 是一种 IR、理解编译器 70 年的工具链为什么对 LLM 编译也适用
- **大学操作系统**——当年学进程调度、IPC、虚拟内存时觉得是理论课，现在支撑我理解 multi-agent hand-off、理解 memory compaction、理解为什么 Agent 需要一个 scheduler
- **数据中台时期的 Flink 碎片经验**——当年看 checkpoint 日志看不懂，现在至少知道哪里该去查 Chandy-Lamport
- **短暂写过的 Scala 和 Actor**——当年觉得别扭就放弃了，现在在 LangGraph 的 multi-agent 里又看到了同一套抽象

**这些碎片在当时是分散的、看起来没用的**。现在它们以一种我完全没预料到的方式全都串起来了。

如果你是一个 2020 年代入行的工程师，你的武器库可能比你以为的更大。大学里那些不喜欢的课、工作里没搞懂就跳过的技术、自己研究一半就放下的项目——**这些碎片都在等一个机会被激活**。

### 时代窗口

有一件事我越想越觉得**紧迫**：

**70 年 PL 遗产被 LLM 激活这件事，在一个职业生涯里只发生一次**。

- 2022 年前不会发生——没有 LLM
- 2030 年代可能已经过去——后来者会把"Actor 模型适合 multi-agent"这种观察当成常识，就像我们今天看"链表适合插入密集场景"一样
- **现在（2026）恰好是窗口期**——很多人在用 LangGraph / Pydantic AI，但还没人系统讲出来这些东西背后的 50 年脉络

在一个明显的技术浪潮里，要站到"能看见重演"的位置上，需要的不是 30 年经验——需要的是：

1. 在 2-3 个技术栈之间穿行过（不深也没关系）
2. 对历史有一点好奇
3. 遇到一个契机（和 AI 讨论、读一篇论文、碰到一个 bug）让所有碎片串起来

这种时刻一辈子可能只出现一两次，**值得留意**。

---

*原文：<https://github.com/Lin-Guanguo/llm-agent-research/blob/main/blog.4.chinese.md>*

*本文研究素材公开在 <https://github.com/Lin-Guanguo/llm-agent-research>，包括 20+ 个 Agent 框架的源码笔记、完整的对照表、以及这篇 blog 的讨论过程记录。*

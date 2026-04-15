# Agent / Workflow 架构范式调研

Last Updated: 2026-04-08
Topic Summary: 系统梳理市面上 Agent / Workflow 的主要实现范式（Workflow vs Agent 二分、ReAct 系、Plan-and-Execute 系、Reflection 系、Search 系、多 Agent、Dataflow），追溯 typed dataflow 的 50 年理论谱系，对照定位自己 Agent 2.0 的设计位置，记录实现过程中碰到的四个 fan-in/fan-out 工程痛点及其在其他领域的成熟对应，补充介绍 Operational Transformation / Edit-Script 模式作为另一条同时覆盖多个问题的工业路线。最后一节是 2026-04-08 基于 4 个并行调研 subagent 的实证发现：**开源生态里"活着"的 Plan-and-Execute 实现极少，主流框架都已收敛到"ReAct + TodoWrite scratchpad"**。贯穿全文的主题是"独立重新发明前人答案"。

## 背景

实现 Agent 2.0 时自然写出了 Planner + 程序化校验 + 字段对齐机制，事后才发现这其实命中了学术界 ReWOO / LLMCompiler / Plan-then-Execute 这套体系。借此机会把市面上 Agent 架构整体过一遍，建立清晰的分类坐标，方便后续做架构决策时不再"重新发明轮子"。

起点是两篇资料：
- LangChain 博客 *Planning Agents*
- arxiv 2509.08646 *Plan-then-Execute*

---

## 一、最顶层的二分：Workflow vs Agent

Anthropic 在 *Building Effective Agents* 里给的定义目前最被广泛采用：

| 类别 | 定义 | 控制权 |
|------|------|--------|
| **Workflow** | LLM 和工具通过**预定义的代码路径**编排 | 代码（人类）控制流程 |
| **Agent** | LLM **动态决定**自己的流程和工具调用 | LLM 控制流程 |

"固定流程"就是 Workflow，"ReAct" 是典型 Agent。**Plan-and-Execute 是中间态**——计划阶段是 LLM 决策，执行阶段按计划走的固定流程。

另一条正交的轴是**控制流的拓扑形状**：链 (Chain) / 环 (Loop) / 树 (Tree) / 图 (DAG)。几乎所有 Agent 架构都可以用 (谁控制 × 拓扑形状) 这个二维表格来定位。

---

## 二、Workflow 模式（Anthropic 五种）

代码主导，LLM 是组件：

| 模式 | 一句话 | 拓扑 |
|------|--------|------|
| **Augmented LLM** | LLM + 工具 + 检索 + 记忆，最小积木 | 单点 |
| **Prompt Chaining** | A 的输出喂给 B，中间可加程序化校验 | 链 |
| **Routing** | 分类后路由到不同子 prompt / 模型 | 分叉 |
| **Parallelization** | Sectioning（独立子任务并行）或 Voting（同一任务多次投票） | 并行 |
| **Orchestrator-Workers** | 中央 LLM 动态拆分子任务、分发、合成 | 动态树 |
| **Evaluator-Optimizer** | 一个生成、一个评审、循环改进 | 环 |

---

## 三、Agent / Planning 架构（按出现顺序）

LLM 主导决策，论文体系内的主要变种：

| 架构 | 核心思想 | 拓扑 | 出处 |
|------|---------|------|------|
| **ReAct** (2022) | Thought→Action→Observation 的循环，每步现想现做 | 环 | Yao et al. |
| **Self-Refine / Reflexion** (2023) | 生成后自我批判 + verbal feedback 写入记忆，再重做 | 环 + 记忆 | Madaan / Shinn et al. |
| **Tree of Thoughts (ToT)** (2023) | 同时展开多条思路，按价值剪枝、回溯 | 树 | Yao et al. |
| **Plan-and-Execute** (2023) | 先一次性规划完整步骤，再顺序执行 | 链 (先一步思考) | LangChain / BabyAGI 系 |
| **ReWOO** (2023) | Planner 输出带变量占位符 `#E1` 的计划，Worker 顺序填充 | 链 + 变量绑定 | Xu et al. |
| **LLMCompiler** (2024) | Planner 流式输出**任务 DAG**，调度器并行执行 | DAG | Kim et al. |
| **LATS** (2024) | ReAct + Reflexion + MCTS 树搜索的统一体 | 树 + 回溯 | Zhou et al. |
| **Plan-then-Execute (P-t-E)** (2025) | P&E 的工程化范式，重点是**安全 / 控制流完整性 / 防 prompt injection** | 链 + 重规划环 | arxiv 2509.08646 |

---

## 四、按"性质"分类的六大类形态

除了固定流程 / ReAct / Plan-and-Execute 之外，市面上至少还有这些性质不同的形态：

### 1. 反思类 (Reflection-based)
Self-Refine、Reflexion、Evaluator-Optimizer。本质是"在 ReAct 的循环外再套一层评审循环"。如果"程序化校验"会触发重做，就属于这一类的简化版（评审者从 LLM 换成了规则代码）。

### 2. 搜索类 (Search-based)
ToT、LATS、RAP。把 Agent 状态空间当作搜索树，用 BFS/DFS/MCTS 探索。代价高、能力强，主要用于推理 benchmark。

### 3. 并行编译类 (Compiler-style)
LLMCompiler 是代表，ReWOO 是它的简化版。Planner 不再输出"步骤列表"而是输出"依赖图"，调度器负责并行调度。LLMCompiler 论文中实测比 ReWOO 快 ~3.6x。

### 4. 多 Agent / 分工类 (Multi-Agent)
- **Orchestrator-Worker**（Anthropic）：单一 leader 分发
- **辩论 / Society of Mind**：多个 agent 互相辩论
- **CrewAI / AutoGen 的 role-based**：每个 agent 有固定角色和工具集
- **Hierarchical Agents**：树状的 manager → worker

### 5. 环境驱动 / Computer-Use 类
Claude Computer Use、SWE-Agent、OpenInterpreter 这一类。结构上仍是 ReAct，但"动作空间"是 GUI/shell，反馈是截图/输出，工程难点在 ACI (Agent-Computer Interface) 而非控制流。

### 6. 状态机 / Graph 编排类（工程派）
LangGraph、Pydantic Graph、Burr。这些不是新算法，而是把上述任何一种架构用**显式状态机**写出来，便于调试和持久化。**这是当前生产环境最主流的形态**——不是某个固定模式，而是把 plan/react/reflect 拼成业务需要的有限状态机。

---

## 五、Planner 阶段如何"串联"步骤的 4 种机制

Plan-and-Execute 系架构里，最核心的工程问题是：**Planner 在生成计划时如何表达步骤之间的数据依赖**。这件事大约有 4 种实现风格，从最松散到最严格：

### 机制 1：字符串占位符（ReWOO 风格）

最早、最简单。Planner 输出一段纯文本计划，用约定标记表示"这一步的产物"：

```
Plan: Find the population of France
#E1 = Search[population of France]

Plan: Find the population of Germany
#E2 = Search[population of Germany]

Plan: Compute the sum
#E3 = Calculator[#E1 + #E2]
```

- Planner 写的时候 `#E1` 就是个普通字符串
- Worker 阶段顺序执行，每跑完一步把 `#En` 的值存到 dict
- 跑下一步前做**字符串替换**

**优点**：极简单。**缺点**：没类型、没 schema、只能顺序执行、runtime 才发现 `#E5` 是空的。

### 机制 2：函数调用 + 变量引用（LLMCompiler 风格）

把 Planner 的输出当成伪代码：

```
1. search("population of France") -> $1
2. search("population of Germany") -> $2
3. add($1, $2) -> $3
4. join()
```

- **Task Fetching Unit** 解析 Planner 输出，建 AST
- 通过 `$1`/`$2` 引用识别出 task 之间的**依赖图（DAG）**
- 调度器按拓扑顺序派发，**任何依赖已就绪的任务立刻并行启动**

**优点**：真正的并行（LLMCompiler 比 ReWOO 快 3x 的核心原因）、能检测循环引用。**缺点**：仍是位置参数无命名字段。

### 机制 3：结构化 JSON + 显式 depends_on（生产派主流）

LangGraph、CrewAI、Anthropic Tool Use 这类 SDK 用得最多：

```json
{
  "tasks": [
    {"id": "t1", "tool": "search", "args": {"q": "France"}, "deps": []},
    {"id": "t2", "tool": "search", "args": {"q": "Germany"}, "deps": []},
    {"id": "t3", "tool": "add", "args": {"a": "{{t1.result}}", "b": "{{t2.result}}"}, "deps": ["t1", "t2"]}
  ]
}
```

- 依赖既可以**显式声明**（`deps`），也可以从 `{{ti.result}}` 推导
- JSON Schema 给 Planner 强约束（structured output / tool use 强制格式）

**优点**：Planner 出格概率小、能验证、能可视化。**缺点**：仍是按 ID 引用整个对象，没有字段级类型。

### 机制 4：Typed Dataflow / 字段对齐

每个节点声明**输入 schema 和输出 schema**，引擎自动按字段名对齐：

```
NodeA:  in = {user_query}                  out = {search_results, refined_query}
NodeB:  in = {search_results}              out = {summary}
NodeC:  in = {refined_query, summary}      out = {final_answer}
```

- Planner 只挑节点和给出**字段映射**
- 系统维护 **blackboard / context dict**
- 节点触发规则：**`required_inputs ⊆ blackboard.keys()` → fire**
- 跑完写回输出字段，触发下游

这种语义在 CS 里有正式名字：**dataflow programming** 或 **petri net firing semantics**。对应工程系统：Airflow、Prefect、Tekton、Bazel、Make。Agent 圈里最接近的实现是 **LangGraph 的 channel/state 模型** 和 **DSPy 的 Signature**。

**独特优势**：
1. **静态验证** — plan 生成完之后、执行之前就能扫一遍图，required 字段是否都有上游产出
2. **天然并行** — 任何"输入字段齐了"的节点都可立刻 fire
3. **类型约束 → 减少幻觉** — Planner 只能用已声明的字段名，等于把"接口契约"硬编码进 prompt
4. **可重入 / 可恢复** — blackboard 是显式状态，崩溃后从持久化恢复

### 四种机制对比

| 维度 | ReWOO | LLMCompiler | JSON+deps | Typed Dataflow |
|------|-------|-------------|-----------|----------------|
| 依赖表达 | 字符串 `#E1` | 位置 `$1` | ID 引用 | **字段名对齐** |
| 引用粒度 | 整段文本 | 整个返回值 | 整个返回值 | **单个字段** |
| 触发语义 | 顺序 | 拓扑并行 | 拓扑并行 | **数据就绪触发** |
| 类型检查 | 无 | 语法层 | schema 层 | **字段级 schema** |
| 静态校验 | 无 | 部分 | 部分 | **完整 DAG + 类型** |
| Planner 负担 | 写文本 | 写伪代码 | 写 JSON | 只挑节点 / 字段 |

---

## 六、补充背景：Typed Dataflow 的理论谱系

机制 4 那种"声明输入输出 schema、字段对齐、ready 即 fire"的语义，**不是 LLM Agent 时代的发明**。它是一个有 50 年历史的独立研究领域——**dataflow programming**——在 LLM 上的重新发现。

### 编年史

| 时代 | 代表 | 贡献 |
|------|------|------|
| **1974** | **Kahn Process Networks (KPN)** — Gilles Kahn 的论文 | 数据流编程的数学基础：进程通过 FIFO channel 通信，节点在输入就绪时才 fire。**所有"字段对齐 → 触发"语义的祖先** |
| **1980s** | **Lustre / Esterel / Signal** — 法国 INRIA 的同步数据流语言 | 用于航空航天的硬实时系统（空客飞控就是 Lustre 写的）。引入了类型化的同步数据流 |
| **1980s-90s** | **Lucid / Sisal / Id** — 函数式数据流语言 | 学术上的探索，证明数据流模型适合自动并行化 |
| **2000s** | **Apache Flink / Storm / Spark Streaming** | 把数据流模型工业化为大数据流处理引擎 |
| **2010s** | **TensorFlow 1.x / Theano / PyTorch (autograd)** | 深度学习的"计算图"本质就是 typed dataflow——tensor 是 typed payload，op 是节点 |
| **2010s** | **Functional Reactive Programming (FRP)** — Elm / Rx / Cycle.js | 把 dataflow 引入前端 UI |
| **2020s** | **MLIR / XLA / TVM** — 编译器界的 dataflow IR | 把神经网络计算图当 dataflow IR 编译优化 |
| **2023+** | **LangGraph / DSPy / Burr** | LLM 时代把 dataflow 重新发明在 Agent 编排上 |

### 这个领域的核心研究问题

dataflow 圈几十年反复在解决的一批问题，**和现在做 Agent 时碰到的问题一一对应**：

| 数据流领域的经典问题 | 在 Agent 里的对应 |
|---------------------|------------------|
| **Type system for dataflow** — 节点的输入输出类型如何静态检查 | 字段名 / schema 对齐 |
| **Scheduling** — 节点 ready 后谁先跑、能否并行 | Plan 执行调度 |
| **Backpressure** — 慢消费者如何不被快生产者压垮 | Agent 调用 LLM 限速 |
| **Fault tolerance / replay** — 节点崩溃后如何恢复 | Agent 中途失败重启 |
| **Determinism** — 同样输入是否产生同样输出 | Agent 可重现性 |
| **Cycle detection** — 数据流图能否有环、什么时候允许 | 重规划循环何时合法 |
| **Higher-order dataflow** — 节点能否动态生成新节点 | Planner 动态拆任务 |

### 关键学术名词

如果想顺这个方向深挖：

- **Kahn Process Networks (KPN)** — 理论根基
- **Synchronous Dataflow (SDF)** — Edward Lee (UC Berkeley) 主导的模型，可静态调度
- **Dataflow Process Networks** — Lee & Parks 的统一框架
- **Stream Processing Systems** — 现代工业界的工程化版本
- **Computational Graphs** — 深度学习语境下的同样东西

Edward Lee 在 UC Berkeley 的 **Ptolemy II** 项目几乎是 typed dataflow 的"百科全书式"实现，跑了 25 年。他的教科书 *Embedded System Design with Models of Computation* 把所有变体都整理过。

### 实用启示

LangGraph、DSPy 这些 Agent 框架的 channel/state 机制，几乎全是从 Lee 那派的 dataflow 理论里直接搬过来的——只是大多数 Agent 框架的作者没明说这个谱系。

**当 Agent 编排往更复杂的方向走（动态拆任务、并行 DAG、容错恢复、流式输出）时，直接去看流处理 / Flink / 数据流编程的论文，比看 Agent 圈的文章信息密度高得多**。Agent 圈很多"新设计"在 dataflow 圈是 30 年前就解决了的老问题。

---

## 七、Agent 2.0 的定位

对照上面这套坐标系，自己实现的 Agent 2.0 大约站在这个位置：

```
基础形态:    Plan-and-Execute (Planner 一次出完整计划)
   +
增强 1:      字段对齐 → 接近 Typed Dataflow / DSPy Signature 风格
                       （比 ReWOO 的字符串占位符更严格）
   +
增强 2:      程序化校验 → Evaluator-Optimizer 的轻量化
                       （评审者是规则而非 LLM）
```

也就是说，"自然实现"出来的东西恰好命中了 **P&E + Typed Dataflow + Evaluator** 三种模式的交集。比教科书 ReWOO 多走了两步：
1. 从"位置参数"升级到"命名字段"（schema）
2. 从"顺序执行"升级到"数据就绪触发"（dataflow semantics）

这俩升级正好把 ReWOO 的两个主要缺点都补掉了，工程上与 LangGraph / DSPy 同代。

可继续探索的方向：
- **重规划循环 (Replanning)** — 执行失败 / 校验不过时，把失败信息回灌给 Planner 重生成，而不是从头再来。P-t-E 论文重点强调
- **DAG 化** — 如果步骤有可并行的，从串行 dataflow 升级到完整 DAG 调度
- **HITL 检查点** — Plan 是天然的 review point，比 ReAct 容易插入人工审核

工程落地要注意的坑：**字段名漂移**。Planner 是 LLM 时偶尔会按语义而不是按字段名拼接（`refined_query` 写成 `query_refined`）。两种解法：(a) 强 schema 把所有字段名作为枚举塞进 structured output schema；(b) 用轻量 LLM/embedding 做语义对齐 fallback。推荐 (a)。

---

## 八、关键工程问题：Plan-and-Execute 的四个 fan-in/fan-out 痛点

### 8.0 前提：为什么 ReAct 没有这些问题，Plan-and-Execute 有

这一节要讲的四个问题，**是 Plan-and-Execute 模式特有的**。ReAct 模式不会碰到它们——原因是两种模式对"顺序决策权"的分配完全不同：

| 模式 | 顺序决策权 | 代价 |
|------|-----------|------|
| **ReAct** | 每一步都由 LLM 当场判断下一步做什么 | 大量 LLM 调用，高延迟、高成本，结果不稳定 |
| **Plan-and-Execute** | 必须在 Plan 阶段就想清楚所有步骤和依赖 | LLM 调用少、可预测，但**必须用"数据流"机制显式表达所有依赖** |

**关键 insight**：ReAct 把"步骤之间的数据依赖"这件事**隐式交给 LLM 的上下文来维持**——前一步的输出直接进下一步的 prompt，LLM 自己决定用不用、怎么用。而 Plan-and-Execute 把这些依赖**外化为 plan**，所以需要一整套形式化机制来表达 "A 的输出如何流到 B"、"多个输入如何汇到一个节点"、"list 的每个元素如何被批量处理"、"部分变化如何只传播给受影响的下游"。

换句话说，**选择 Plan-and-Execute 的代价就是进入 dataflow 的问题域**。这不是实现得不好，而是架构选择的必然结果——一旦"想"和"做"解耦，就必须用数据流的语言重新表达原本藏在 LLM 上下文里的那些隐式依赖。

下面四个问题，就是走进这个问题域后几乎一定会碰到的核心痛点。每一个都对应着其他领域（dataflow / 流处理 / 增量计算 / 前端 reconciliation）几十年积累的成熟理论。

---

### 8.1 问题一：多输入汇集到一个节点（fan-in）

**痛点表现**：tuple 语法作为隐式合并机制，能表达"全部到齐就打包"，但表达不了"任一到就触发用最新"、"k of n 投票"、"race"、"zip vs combineLatest" 等更灵活的语义。

**理论对应**：**Fan-in / Join combinators**

主要理论谱系：
- **Kahn Process Networks (1974)** — 禁止非确定性 merge，是"默认情况下 fan-in 必须是确定的"这条原则的来源
- **Synchronous Dataflow / SDF** (Edward Lee, 1980s) — 每个端口声明消费速率，支持"端口 a 攒够 1 个 + 端口 b 攒够 3 个"这种细粒度规则
- **Tagged-Token Dataflow** (MIT, 1980s) — 用 (iteration_id, instance_id) tag 解决"哪个 a 配哪个 b" 的配对问题，特别适合"循环 + 重规划"场景
- **Petri Nets** — 最通用的 fan-in 形式化，提供 deadlock/livelock 分析工具
- **Functional Reactive Programming (FRP)** — 把 fan-in 浓缩为一组 combinator 小词典：

| Combinator | 语义 |
|-----------|------|
| `zip(a, b)` | 1:1 配对，等两边各出一个（对应当前 tuple 语法） |
| `combineLatest(a, b)` | 任一方有新值就 fire，用对方最新值 |
| `merge(a, b)` | 任一方有就直接 forward，不配对 |
| `withLatestFrom(trig, sample)` | trigger 触发时，sample 当前值 |
| `forkJoin(a, b)` | 等所有完成，取每个最后一个值 |
| `race(a, b)` | 谁先到用谁，丢弃其他 |
| `concat(a, b)` | 先 forward a 全部，再 forward b |

**推荐方案**：把 fan-in 从"隐式 tuple"提升为**一等节点**，提供 zip / combineLatest / forkJoin 等内置 join 节点。Planner 显式选择语义。Tuple 可以保留为 ZipJoin 的语法糖。

**Reducer vs Join 的正交性**：Reducer 解决"怎么合并"（replace / concat / sum / merge_dict / union_set），Join 解决"什么时候触发"（zip / combineLatest / race / quorum）。这两件事正交，不是替代关系。LangGraph 的 `Annotated[T, reducer]` 模型是 reducer 派的代表；FRP combinators 是 join 派的代表。

---

### 8.2 问题二：对 list 每个元素做同样操作（fan-out map）

**痛点表现**：需要对一个 list 的每个元素做同样的处理（LLM 调用或节点执行），再聚回一个 list。目前没有专门的原语，只能手动展开。

**理论对应**：**Dynamic Task Mapping / ParDo / Scatter-Gather**

这件事在不同社区至少有 7-8 个名字：

| 社区 | 名字 |
|------|------|
| 函数式编程 | `map` / `fmap` |
| MapReduce | "map 阶段" / scatter-gather |
| Apache Beam | `ParDo` |
| Flink / Spark | `map` / `flatMap` |
| **Airflow 2.3+** | **dynamic task mapping** (`.expand()`) |
| Argo Workflows | `withItems` |
| Tekton Pipelines | `matrix` |
| **LangGraph** | **`Send` API** |
| LangChain | `batch` / `RunnableEach` |

**关键难点**：
1. **动态 fan-out** — list 长度运行时才知道，节点数量不能在 plan 生成时写死。Airflow 2.3 专门为此加了 `.expand()`，是历史上改动最大的功能之一
2. **Gather 语义** — 顺序保留？失败处理？部分结果？流式 vs 全等？并行度控制？

**推荐方案**：把 **MapNode** 作为新的一等节点原语。声明 `input_list_field + apply_node + output_list_field`，调度器在运行时展开为 N 个并行实例，按 index 聚回 list。v1 先支持单节点 body（不支持子图嵌套和 nested map），失败策略默认 `fail_fast`。

> **类比**：Map 节点在 dataflow 里的地位，相当于 `for` 循环在命令式语言里的地位。所有成熟的 dataflow / workflow 系统最后都加了这个原语（Airflow / Argo / Tekton / Beam / Flink / LangGraph 无一例外）。

---

### 8.3 问题三：LLM 批量处理 + 只对变化项做下游（差分传播）

**痛点表现**：LLM 需要一次看到整个 list（为了跨项一致性、风格统一），然后返回整个 list + 标注哪些有更新。希望下游节点**只对变化的项**继续处理，没变的项复用上次的结果。

**理论对应**：**Differential Dataflow / Incremental Computation**

完整的研究谱系：

| 时间 | 系统 | 贡献 |
|------|------|------|
| 2006 | **Self-Adjusting Computation** (Umut Acar, CMU) | 理论奠基 |
| 2013 | **Naiad** (Microsoft Research) | 首个工业级 differential dataflow |
| 2013 | **Differential Dataflow 论文** (Frank McSherry) | (data, time, delta) 三元组模型 |
| 2015+ | **Timely / Differential Dataflow** (Rust) | 开源实现 |
| 2019+ | **Materialize** | 流式 SQL 数据库，核心就是这一套 |
| 工程派 | **Salsa** (Rust) | rust-analyzer 的增量编译框架 |
| 工程派 | **Jane Street Incremental** (OCaml) | 金融系统用 |
| 前端派 | **Solid.js / Svelte signals** | FRP 细粒度 reactive |

**核心抽象**：值不再是"当前状态"，而是 (element, time, delta) 三元组序列。下游操作符只处理 delta，不处理全量。"只对变化的部分做下游"是**语言层面自动的**，不需要手写"检测变化"的代码。

**推荐方案**（针对 LLM 场景的裁剪）：

1. 加 `KeyedList[T] = Map[ID, T]` 作为一等类型
2. LLM 节点标记 `mode: bulk`，表示必须整个 list 一次处理
3. 调度器在 bulk 节点执行后自动算 diff（按 id 匹配 + 内容 hash）
4. 下游节点声明处理模式：
   - `per_item` — 只对 diff 中的 item fire
   - `whole_list` — 看完整新 list
   - `aggregate` — diff 非空则 fire 一次
5. Hash cache 做 memoization：没进 diff 的 key，下游结果直接复用

**关键约束**：用 prompt 强制 LLM 保留每个 item 的稳定 id 字段，不允许编造。id 是整个机制的地基。

---

### 8.4 问题四：有序集合的增量更新（插入/删除中间项）

**痛点表现**：list 既需要身份追踪（按 id），又需要保持顺序。在序列中间 insert 或 delete 时，其他项的**内容**没变、只是**位置**变了。下游节点对这种"纯位置变化"应该有不同的反应，但 naive diff 会把"插入一项"变成"后面全部 dirty"。

**理论对应**：**有序集合 + 增量计算** 的交叉问题

**核心 insight**：**"位置"和"内容"是两个独立的维度**。naive diff 把它们混在一起，导致过度失效。

**三个经典解法**（推荐做法是三者组合）：

1. **React Reconciliation + `key` prop** — 工程界最成熟的方案。按 key 匹配而非按位置匹配，key 相同的组件复用
2. **Fractional Indexing / CRDT** (Figma / Linear / Notion / Yjs / Automerge) — 让 position 本身就是稳定的。在 0.1 和 0.2 之间插入用 0.15，其他项的 position 编号完全不变
3. **Self-Adjusting Computation** (Salsa / Jane Street Incremental) — 每个计算声明自己依赖什么，按依赖做选择性 recompute

**推荐方案**（三者组合）：

```
type OrderedKeyedList[T] = List of {
  id:      stable_id,      # LLM 必须保留
  rank:    float,          # 调度器维护，LLM 不碰
  content: T,
}
```

**Diff 算法按 (id, content, rank) 三维计算**：
- `content_changed` — id 存在但 content 变了
- `rank_changed` — id 存在但 rank 变了
- `added` — 新 id
- `removed` — 消失的 id

**下游节点声明 sensitivity（敏感度）**：

| sensitivity | 触发条件 |
|-------------|---------|
| `content_only` | 只对 `content_changed ∪ added` fire |
| `content_and_rank` | 对 `content_changed ∪ rank_changed ∪ added` fire |
| `neighbors` | 对上述项 + 它们的邻居 fire |
| `whole_list` | 任何变化都 fire，输入整个新 list |

**关键细节**：
- Rank 由调度器维护，**不交给 LLM**。LLM 只负责返回"语义顺序"，调度器事后按顺序为新元素分配介于邻居之间的 rank
- "移动"被识别为 rank 变化而非 remove+add（因为 id 还在）
- 这本质上是把 **"React key 匹配" + "fractional indexing 稳定位置" + "Salsa 依赖追踪"** 拼起来

---

### 8.5 反思：这些都不是新问题

上面四个问题有一个共同特征：**每一个都对应着其他领域几十年积累的成熟理论，而自己在碰到的时候完全不知道**。

| 问题 | 理论 | 研究年代 |
|------|------|---------|
| Fan-in 合并 | FRP combinators / KPN / SDF | 1974 起 |
| List 批量处理 | ParDo / Dynamic Task Mapping | 2004 起（MapReduce） |
| 差分传播 | Differential Dataflow / Self-Adjusting Computation | 2006 起 |
| 有序集合增量 | React Reconciliation / CRDTs / Fractional Indexing | 2013 起 |

**它们不是为 LLM Agent 而发明的**——多数甚至在 LLM 火起来之前十几年就已经有成熟实现。但一旦选择 Plan-and-Execute 架构，这些问题就会**同时**涌现出来，所以独立实现时就会"自然重新发明"其中一部分。

这件事带来两个可直接执行的教训：

**教训 1：碰到"感觉这件事不该这么难"的工程问题时，先去搜相关理论领域。** 优先搜这几个关键词：

- **Dataflow programming** — fan-in/fan-out/调度问题
- **Stream processing (Flink / Beam)** — 批量/流式/窗口/join
- **Incremental computation (Salsa / Jane Street Incremental)** — 部分重算问题
- **CRDTs** — 有序 + 身份 + 并发的组合问题
- **React reconciliation** — 有身份的集合的更新问题
- **Build systems (Bazel / Nix)** — 依赖 + 缓存 + 部分重建

**教训 2：LLM Agent 圈的"新设计"有很大一部分是旧问题换新包装。** 当你看到"XX 框架提出了 YY 模式"，可以下意识问一句："这在 dataflow / 流处理 / 增量计算圈对应什么？" 多数时候能直接找到更成熟的答案。

**更深的观察**：不是自己实现得不好，而是进入了一个**交叉区**——"LLM + Plan-and-Execute dataflow" 这块交叉区之前没人系统做过，所以碰到的每个问题都在两个领域之间"落缝"。但缝两边的领域各自都很成熟，前人的答案可以直接搬。对 Agent 架构选型的 meta-启示是：**如果你选 ReAct，你在付"LLM 当场决策"的运行时代价；如果你选 Plan-and-Execute，你在付"把隐式依赖外化"的工程代价，而这部分代价恰好是 dataflow 领域用 50 年才解决完的问题**。两种代价都真实存在，只是形式不同。

---

## 九、补充：Operational Transformation 与 Edit-Script 模式

§八 列出的四个问题可以分别独立解决，但业界还有另一条路线——用**一个统一的数据结构**把 §8.2 (map) + §8.3 (差分传播) + §8.4 (有序增量) 同时包住。这条路线叫 **Operational Transformation (OT)**，起源于 1989 年的协作编辑研究，到今天是一个有 40 年历史的成熟领域。

### 9.1 核心数据结构

这套模式的核心只有两件东西：

```
ops:    <结构变更的操作序列>       # Add / Edit / Delete 等
state:  <应用 ops 后的最终状态>    # materialized view / snapshot
```

这叫 **"Operation Log + Materialized View"** 或 **"Edit Script + Snapshot"** 模式。**同时保留 log 和 view** 是核心——log 提供"如何从旧状态到新状态"的增量信息，view 提供"当前快照"用于直接读取和 fallback（当 log 无法重放时）。

这个组合在不同领域有不同叫法：

| 领域 | 名称 |
|------|------|
| 数据库 | Write-Ahead Log (WAL) + Materialized View |
| 协作编辑 | **Operational Transformation (OT)** |
| 事件溯源 | Event Log + Aggregate State |
| 富文本编辑 | **Quill Delta format** |
| 版本控制 | Patch + Working Tree |
| 编辑器协议 | **LSP `TextDocumentContentChangeEvent`** |
| Android UI | **DiffUtil Edit Script** |
| 函数式编程 | Free Monad 的"指令 + 解释器" |

### 9.2 两个主要流派

OT 有两个主要流派，选择不同会显著影响实现复杂度：

**流派 A：Cursor-based (Quill Delta / ShareJS)**

```
{ retain: 10 }       # 保留前 10 个
{ insert: 'hello' }  # 插入 'hello'
{ delete: 3 }        # 删除 3 个
```

每个 op 相对于**光标位置**。顺序敏感、紧凑、易读。Quill.js 和 ShareJS 用这种。

**流派 B：Absolute-indexed (Jupiter / Google Wave)**

```
insert at index 10, insert at index 15, delete at index 20
```

每个 op 有**绝对索引**。随机访问方便，但需要明确定义"索引指的是哪个坐标空间"（原始 list？当前 snapshot？上一个 op 之后？）。Google Wave、Jupiter 系统、以及大部分自研的 list-edit 协议走这条路。

### 9.3 工业级实现参考

真正生产环境在用的实现，按 ROI 排序：

1. **Quill Delta** ([spec](https://github.com/quilljs/delta)) — 最干净、最小、最易懂的 OT 实现。支持 compose / transform / invert 三个核心操作。**推荐作为入门读物**
2. **LSP `TextDocumentContentChangeEvent`** — Language Server Protocol 用了 10 年的文本同步协议。每个 change 带 `range`（旧文档坐标系）和 `text`（新内容），version 号标识文档版本
3. **Android `DiffUtil`** — RecyclerView 的列表差分工具，把两个 list 变成 INSERT/REMOVE/MOVE/CHANGE 脚本，只对变化项重建 ViewHolder
4. **Automerge** ([automerge.org](https://automerge.org/)) — OT 向 CRDT 演化后的产物，支持 move 语义和离线合并
5. **ShareJS / ShareDB** — 经典的 OT 服务器端实现，Google Wave 的开源后继
6. **Yjs** — 另一个主流 CRDT 实现，生态广（Liveblocks、Tiptap 都用）

### 9.4 LLM 场景下的独特变种："LLM-as-op-producer"

经典 OT 的 ops 来自**用户按键**——编辑器捕获每次击键转成 op。LLM 时代出现了一个新变种：**让 LLM 直接产出 ops**，而不是让系统事后 diff。这个模式在 2023–2025 年间从"隐性工程实践"固化成"显性设计模式"，现在几乎是所有主流 AI coding 工具的事实标准：

| 系统 | 如何让 LLM 产出 ops |
|------|---------------------|
| **Cursor — Apply AI Edit** | LLM 返回"这段换成那段"，而非整个新文件 |
| **GitHub Copilot — Inline Edit** | 同上 |
| **Claude Code Edit 工具** | Tool schema 就是 `{old_string, new_string}`，强制 LLM 以编辑意图表达 |
| **Aider — editblock 格式** | 让 LLM 输出 `SEARCH / REPLACE` 块 |
| **OpenAI structured outputs** | 约束 LLM 输出结构化的编辑指令 |

**这个变种为什么重要**：

1. **保留意图信息** — LLM 明确标注"这是 EDIT 不是重写"，系统可以区分"小改"和"大改"。事后 diff 永远丢失这种区分（两者可能产生相同的最终状态）
2. **省去 diff 计算成本** — 不需要跑 Myers 算法
3. **更易审计** — ops 是显式的，可以打日志、做权限检查、做回滚
4. **更 LLM-friendly** — LLM 本来就擅长以"指令序列"形式输出，这比"输出整个新状态让系统猜变化"更贴合 LLM 的生成范式
5. **更省 token** — 不需要 LLM 把没变的部分完整复述一遍

经典 OT 文献里没有覆盖这个模式，因为 1989–2015 年间根本不存在"能以指令形式输出结构化变更"的生成式模型。但从 2023 年开始，它就是 LLM 应用处理"修改现有状态"类任务的默认选择。

### 9.5 OT 会碰到的四个经典问题

任何走 OT 路线的系统，早晚会碰到这四个问题。它们在 OT 领域研究了 30 年，有成熟答案但也有成熟的坑：

**问题 1：Operation Composition（op 组合）**

把两批 ops 合并成一批：`ops1 ∘ ops2 = ops12`。听起来简单，实际上需要证明 `apply(ops12, s) ≡ apply(ops2, apply(ops1, s))` 对所有状态 `s` 成立。

**关键坑**：index 的坐标空间必须仔细定义。如果 ops2 的 index 是相对 `apply(ops1, s)` 的，就要把它投影回相对 `s` 的坐标。ShareJS 的 `type.compose()` 就是专门做这件事的，**不要自己从头写**——这是 OT 领域有 30 年 bug 史的地方。OT 领域有专门的理论工具叫 **TP1/TP2 properties**（Transformation Property 1/2）来证明 transform 函数的正确性。

**问题 2：Move 语义的表达**

"把第 3 项移动到第 1 位"在 OT 里有两种表达：
- **Move 作为一等 op**：保留身份，下游可以复用对该项的处理结果
- **分解为 Delete + Insert**：简单但丢失身份，下游以为是"删一个 + 新增一个"

Quill Delta **没有 move**，Automerge / Yjs 有。**这是 OT vs CRDT 的经典分歧点**。对 LLM 场景的启示：如果 LLM 经常重排序列表而内容基本不变，必须支持 move，否则下游计算（比如为每项生成图片 / TTS）会被浪费。

**问题 3：Convergence / State Reconciliation（状态收敛）**

当多个节点对"基准状态"的认知不一致时——比如下游重启过、历史状态丢失、上游产生的 ops 无法重放——需要一个"重新对齐"的机制。

分布式系统里这叫 **state-based reconciliation**，最常用的策略是 **Last-Writer-Wins (LWW)**——直接用最新的 snapshot 覆盖旧状态。更复杂的策略包括 vector clock、version vector 等。**多数单用户 LLM 场景用 LWW 就够了**，但要清楚这是一个简化策略，在多源头并发写入场景会丢数据。

**问题 4：Ops 的并发语义 vs 顺序语义**

同一个位置被 EDIT 两次，应该是错误还是累积？

- **并发语义**（LLM 批量产出场景）：同位置冲突是错误，早暴露 bug
- **顺序语义**（用户逐步编辑场景）：后一次覆盖前一次，累积生效

**两者各有道理**。Quill 用顺序语义（模拟用户按键），多数"一次性批量产出 ops"的 LLM 场景更适合并发语义。**选择要和 ops 的来源匹配**：来自"一次性生成"用并发，来自"多次累积"用顺序。

### 9.6 OT 与 §八 四个问题的关系

OT 作为一个统一框架，**同时**覆盖了 §八 里的三个问题：

| §八 问题 | OT 是否解决 | 怎么解决 |
|---------|-----------|---------|
| 8.1 Fan-in 合并 | ❌ 不覆盖 | OT 是单字段内的结构，不管多输入汇集 |
| 8.2 List 批量 map | ✅ | 只对 ops 里的 ADD/EDIT 位置跑 transform |
| 8.3 差分传播 | ✅ | ops 就是 diff，snapshot 是 view |
| 8.4 有序增量 | ✅ | 用 ops 序列显式表达 insert/delete |

换句话说，**OT 是另一条能同时解决 8.2 / 8.3 / 8.4 的路径**——和 §8.4 里介绍的 "stable id + fractional rank" 方案是**平行的两种选择**，不是替代关系：

| 维度 | Stable ID + Rank | OT Edit Script |
|------|------------------|----------------|
| **身份追踪** | 显式 id 字段 | 坐标空间位置（由 ops 定义） |
| **Move 表达** | 天然支持（同 id 换 rank） | 需要显式 MOVE op，否则丢失身份 |
| **LLM 负担** | 要保留 id 不变 | 要正确产出 ops |
| **diff 来源** | 事后按 id + hash 对比 | LLM 直接产出 |
| **组合复杂度** | 低（rank 独立） | 高（需要 op transform） |
| **典型实现者** | React / Yjs（rank 维度）/ Figma / CRDT | Quill / LSP / ShareJS / DiffUtil |

**适用场景的选择建议**：

- 如果 LLM 可以稳定维护 id → **Stable ID + Rank** 更简单
- 如果 LLM 自然以"编辑指令"方式思考 → **OT Edit Script** 更贴合
- 如果既要支持 move 又要支持离线合并/多端协作 → 上 **CRDT**（Automerge / Yjs）
- 如果场景足够简单（只是单向 refine） → **Simple diff-and-propagate** 就够了，不需要上 OT

### 9.7 从 §八 的反思视角看 OT

OT 是 §八.5 反思的又一个具体例证：**这是一个在 2023 年之前就已经积累了 40 年成熟理论和工程实践的领域**，但它一直在协作编辑、富文本、语言服务器这几个"非 Agent"的细分领域里发展，Agent 圈里很少被提及。

当 LLM Agent 碰到"如何增量更新一个有序的结构化状态"这件事时，它面对的问题 **99% 已经被 OT 领域解决过了**。区别只是——OT 领域是为"人类用户按键编辑文档"设计的，而 LLM Agent 里 ops 的来源从"按键"换成了"LLM 生成"。但底层的数据结构、算法、收敛性保证、测试方法论都可以直接复用。

**对应的"前人答案清单"**：

- 要解决 op 组合正确性 → 读 Ellis & Gibbs 1989 和 TP1/TP2 文献
- 要实现 op transform → 抄 ShareJS / ot.js 的 type 定义
- 要支持 move → 看 Automerge / Yjs 的 RGA 算法
- 要做离线合并 → 整个 CRDT 文献都是答案
- 要设计 LLM-facing 的 op schema → 抄 Claude Code Edit tool / Aider editblock 格式

---

## 十、2026 实证调研：开源 P&E 还剩下什么

本节基于 2026-04-08 并行派遣的 4 个研究 subagent 的发现，每个 subagent 直接 `gh api` 拉仓库元数据 + 读源码验证（不信任 marketing）。覆盖四个域：通用 P&E 项目审计、Deep Research 专题、非 coding 域（内容生成 / workflow / browser-use）、以及"为什么 P&E 输了"的 post-mortem。

### 10.1 总体结论：P&E 在开源生态里几乎是空的

**如果你在 2026 年想找一个"开箱即用的生产级开源 P&E agent"，基本找不到**。只有一个例外：

- **Microsoft Agent Framework 的 Magentic / Magentic-One** — 唯一活跃、生产级、vendor 在推的 P&E 实现（严格说是 hybrid，带 replanning + HITL plan review）。从 AutoGen 的 `_magentic_one_orchestrator` 演化而来，2024 年 11 月 paper + 2026 新的 `MagenticBuilder` API

除此之外，研究级参考实现全部冻结：

- `billxbf/ReWOO` (2023) — ReWOO 论文官方实现，最后 commit 2023-07
- `SqueezeAILab/LLMCompiler` (2024) — LLMCompiler ICML 论文实现，高质量，但最后 commit 2024-07
- `microsoft/JARVIS` (HuggingGPT) — 真 P&E，2025-07 之后未动
- `yoheinakajima/babyagi_archive` — 原版 BabyAGI 的冻结快照。作者 **明确声明归档并用非 P&E 的函数注册框架替换了主仓库**

### 10.2 最诡异的发现：主流"P&E 框架"的代码其实是 ReAct

市面上广泛被称作"P&E"的开源项目，实际代码读下来几乎全是 ReAct。这是"marketing vs reality"最严重的部分：

| 项目 | 公开宣传 | 实际代码路径验证 |
|------|---------|----------------|
| **AutoGPT** | 自主 agent / P&E | `OneShotAgentPromptStrategy` + `EpisodicActionHistory`，per-step decision loop。后转型成可视化 workflow builder |
| **MetaGPT** | 多角色 SOP | 每个 role 是 `_observe`/`_think`/`_act` 状态机，运行时是 ReAct。SOP 是**人工硬编码**的 role 间 handoff，不是 LLM 规划 |
| **CrewAI** | "planner + executor" | 执行核心 `crew_agent_executor.py` 就是 ReAct tool-calling loop；`CrewPlanner` 只是**把一段 hint 字符串拼接到 `task.description` 前**。task 列表是用户在 YAML 里写死的 |
| **Agno** (phidata) | "reasoning agent" | `ReasoningStep.next_action` 是 `CONTINUE/VALIDATE/FINAL_ANSWER/RESET` 枚举——chain-of-thought ReAct。`workflow/` 模块是确定性 Python DAG，不是 LLM planning |
| **OpenHands** (OpenDevin) | "autonomous software agent" | `codeact_agent` 是 ReAct/CodeAct，加 `task_tracker` 工具作 write_todos 助手 |
| **BabyAGI (当前仓库)** | 22k star 的"原始 P&E 经典" | **已经不是 P&E**。当前 README 写："原始 BabyAGI 已归档，这个项目现在是 function registry 框架" |

### 10.3 Deep Research 是 P&E 最健康的域，但也在向 hybrid 收敛

Agent B 专门调研了 Deep Research 域，这是所有域里 P&E 最活跃的：

**仍是严格 P&E 的**：
- **Stanford STORM** (`stanford-oval/storm`, ~28k star) — 四阶段 pipeline，知识策展 → outline → article gen → polish，**完全没有 ReAct loop**。gold-standard 参考实现，但 2025-09 之后进入维护模式
- **GPT-Researcher** (`assafelovic/gpt-researcher`, ~26k star) — `plan_research_outline()` 产出子查询列表，`asyncio.gather` 并行执行，然后 writer 合成。active (2026-03 仍有 commit)
- **Flowise Deep Research template** — 显式的 Planner Agent → Iteration (并行 subagent) → Writer Agent → Condition
- **Together.ai Open Deep Research** — 角色特化的序列 pipeline (Qwen Planner → Llama Summarizer → Llama Extractor → DeepSeek Writer)
- **LearningCircuit/local-deep-research** (~4.3k star) — 30+ 可切换策略，含 `parallel_search_strategy`，每轮并行 fan-out + 反思

**已从 P&E 切到 hybrid 的（重要信号）**：
- **LangChain `open_deep_research`** — **原版是严格 P&E，2026 版改成 "supervisor/ReAct hybrid"**。官方理由是"rigid plans produced disjoint reports"。现在架构是 3-phase 外壳 (`clarify_with_user → write_research_brief → research_supervisor → final_report_generation`) 包一个 ReAct supervisor，supervisor 通过 `ConductResearch` 工具动态 spawn parallel subagent。这是**一个 vendor 公开承认从严格 P&E 倒退的案例**
- **Anthropic Claude Research** (closed，架构公开) — 同样是 orchestrator 动态 spawn subagent，官方 blog 强调 "dynamic spawning, not pre-planning a DAG"

**名为 P&E 实为 ReAct 的**：
- **Perplexica / Vane** — 有个叫 `plan` 的工具，但它**只生成一个自然语言段落给 UI 显示**，agent 本体是 25 iteration ReAct loop
- **Dify Deep Research** — iteration loop 每次只生成一个 query，单线程反思
- **HuggingFace smolagents open_deep_research** — `planning_interval=4` 让它每 4 步 re-plan 一下，但本质是 ReAct
- **langchain-ai/local-deep-researcher** — 5 个 LangGraph 节点看起来像 pipeline，实际是线性反思 loop

**Agent B 的总结**：Deep Research 不是"P&E 例外"，而是**唯一还保留外层 Plan/Execute/Write 边界**的域。但所有新项目的 Research 阶段内部都在向"parallel-fanout from a ReAct supervisor"收敛。轨迹：**pure P&E → 发现 plan 过时 → 切到 ReAct supervisor 并行 spawn**。

### 10.4 Workflow 平台：ReAct 收敛的 smoking gun

Agent C 的最关键发现是 **n8n 的 Plan-and-Execute Agent 节点在 v1.82.0（2025 年初）被正式删除**。这是一个**大型 workflow 平台在生产中试过 P&E 然后主动移除**的公开记录。他们把所有 AI Agent 变种（ReAct / Plan-and-Execute / Conversational / OpenAI Functions / SQL）合并成单一的 "Tools Agent"，使用原生 function calling。

其他 workflow 平台同样的故事：

| 平台 | 情况 |
|------|------|
| **Dify** (~136k star) | `api/core/agent/` 只有 `cot_agent_runner.py` (ReAct/CoT) 和 `fc_agent_runner.py`。没有 planner。用户画的 visual workflow 就是"plan" |
| **Flowise** (~51k star) | GitHub 代码搜 "PlanAndExecute" 零命中。AgentFlow V2 是 LangGraph 上的手工 DCG |
| **Langflow** (~147k star) | "multi-agent orchestration" 是 marketing，实际是用户在 UI 里手画编排 |
| **n8n** | **v1.82.0 明确删除 Plan-and-Execute Agent 节点** |
| **Activepieces** | Agent 功能是在 440+ piece 库上的 tool-calling，没 planner |

**结论**：workflow 平台里的"plan"是**人画的 visual DAG，不是 LLM 规划的**。这些平台的 LLM agent 组件全部收敛到 ReAct + function calling。

### 10.5 Browser-use / 计算机使用域：也是 ReAct

| 项目 | 架构 |
|------|------|
| **browser-use** (~86k star) | 有 `enable_planning` 参数，但 plan 和 action **在同一个 `AgentOutput` schema 里一次输出**——没有单独的 planner LLM 调用 |
| **Skyvern** (~21k star) | 宣称 Planner-Actor-Validator，实际代码只有 `_speculate_next_step_plan()` 的一步前瞻，不是完整 plan |
| **Open Interpreter** (~63k star) | 纯 function-calling loop，`exec()` 是工具 |
| **OpenManus** (~55k star) | **`app/flow/planning.py` 是真 P&E**：`_create_initial_plan()` 一次 LLM 调用产出编号步骤列表，执行 loop 按步派发。但 2026-02 之后 commit 明显变慢 |

**最重要的 meta 发现**：Manus（闭源中国产品，OpenManus 模仿的对象）在官方 blog "Context Engineering for AI Agents" 里**公开承认放弃 todo.md 静态规划**，理由是"大约 1/3 的 token 花在更新 todo 列表上"。他们改成"按需触发的 planner sub-agent"——这正是 **ADaPT 论文 (arXiv:2311.05772)** 2023 年就提出的 "As-Needed Decomposition and Planning"。

### 10.6 为什么 P&E 输了：完整 post-mortem

Agent D 找到了 5 个被多个独立来源引用的原因：

**1. Plan 在执行过程中会过期，特别是在需要读取 state 的任务里。** AutoGPT 的经典失败模式是"google → 写文件 → 读文件 → 再 google"的无限循环——plan 从 step 2 就错了，但执行器无法恢复。Coding 任务尤其严重：你根本不知道下一步要做什么，直到你读完还没读的文件。

**2. 结构化输出降低 planner 的推理质量（有量化证据）。** Tam et al. 2024《Let Me Speak Freely?》实测：
- LLaMA-3-8B 在 Last Letter 任务上从 70.07% 掉到 28.00%（**38 点降幅**）
- GPT-3.5-Turbo 在 GSM8K 上从 76.60% 掉到 49.25%

**机理**：JSON mode 把 `"answer"` 字段放在 `"reason"` 字段前，导致 zero-shot 直接答题而非 zero-shot CoT。**经典 P&E 要求 planner 产出结构化 plan object，所以恰好在最需要推理的那次调用上付了这个代价**。

**3. Context 碎片化（Cognition 的 "Don't Build Multi-Agents"）。** Cognition 的著名例子：一个 subagent 生成 Mario 风格的背景，另一个 subagent 生成**不是游戏风格的鸟**——因为 planner 的隐式假设没传递给 executor。他们的两条基本原则是："**Share context, and share full agent traces, not just individual messages**" 和 "**Actions carry implicit decisions, and conflicting decisions carry bad results**"。

**4. Token / latency 开销**。Manus 的 1/3 token 花在更新 todo.md 上。一旦你必须在每次观察后 re-plan，"planner 调用少"的效率优势就蒸发了，而你还为这个结构复杂度付了税。

**5. 步骤数不可预测**。Anthropic 的 "Building Effective Agents" 原话：workflows 适合"可以 pre-define path"的任务，agents (ReAct) 适合"难以或不可能预测所需步骤数，无法 hardcode 固定 path"的任务。真实的 coding、research、客服任务**全部在后一类**。

### 10.7 "ReAct + TodoWrite" 是 2026 共识，但 TodoWrite 是 no-op

**最震撼的发现**：LangChain "Deep Agents" 博文里的原话——关于 Claude Code 的 TodoWrite 工具：

> "doesn't do anything! It's basically a no-op. It's just context engineering strategy to keep the agent on track."

**换句话说**："ReAct + 显式 TODO 是新共识"这个说法是对的，但 TODO **不是控制流意义上的 planning**——它只是个**注意力工程 hack**：把目标写回最近上下文，对抗 lost-in-the-middle。模型本身才是 planner，框架不是。

这个模式现在是 2026 年的事实标准：

| 产品 | 实现 |
|------|------|
| **Claude Code** | 单线程 `while(tool_call)` loop + `TodoWrite`（写入 `~/.claude/todos/`，system reminder 重新注入上下文） |
| **OpenHands SDK** (Nov 2025) | `TaskTrackerTool` + `FileEditorTool` 在同一个 loop 里，加 UI 面板 |
| **Cursor** | Plan Mode (Shift+Tab) 生成 Markdown plan 存 `.cursor/plans/`，但执行仍是单 agent loop；官方建议"drift 时回到 plan，refine，重跑"——**人类重规划，agent 不** |
| **LangChain 1.0** | 官方迁移路径是 `create_agent` + `TodoListMiddleware`（`write_todos` 工具）。论坛官方解释："faster than P&E because fewer LLM calls" |
| **Manus** | todo.md 作为 recitation 手段，附加在 ReAct loop 上；后来移到按需触发的 planner sub-agent |

**BabyAGI 2o 是这个趋势的终极缩影**：从经典 P&E 的完整 task list 架构**直接坍缩到 174 行代码**，作者 Yohei Nakajima 原话 "**as LLMs improved, most agent scaffolding became unnecessary — the model itself could plan**"。

**同样的终极缩影**：Karpathy 的 `autoresearch`（2026-03）—— **630 行 Python，没有 planner、没有 DAG，没有 classifier router**——自动跑了 700 个 ML 实验。他的原则是 single tight loop + Markdown spec。

### 10.8 真正还在用 P&E 的三种场景

综合所有发现，P&E 在 2026 年还能存活且合理的场景只有三种：

**场景 1：长文本 / 多页结构化内容生成**

任务天然有层级（outline → section → paragraph），"写什么"大部分独立于执行反馈。代表：Stanford STORM、GPT-Researcher、AIStoryWriter、PPTAgent、OpenManus PlanningFlow。**STORM 是 gold standard**。

**场景 2：Read-only + 并行 fan-out（Deep Research）**

动作空间是只读的搜索，世界不会在脚下改变；工作天然是 embarrassingly parallel；错 plan 的代价只是浪费几次搜索；用户愿意等几分钟。代表：ChatGPT/Gemini/Perplexity Deep Research、Magentic-One、open_deep_research（hybrid 版）。**即便如此，这些系统内部也都在向"orchestrator 持续 re-plan"收敛**。

**场景 3：多 agent orchestration 需要 HITL 或审计**

Microsoft Magentic-One 这类场景：需要显式的 plan review、合规审计、人工干预点。P&E 的 plan 是天然的 review point。

### 10.9 给 2026 年 agent 开发者的决策表

综合所有证据：

| 场景 | 推荐架构 | 原因 |
|------|---------|------|
| 任务路径可预知 | **写代码（workflow），不写 agent** | Anthropic 官方建议：workflows beat agents on cost/latency/reliability whenever path is knowable |
| 路径未知 + 动作影响 state（coding / 计算机使用 / ops） | **ReAct + no-op TodoWrite 刷上下文** | Cognition、Anthropic、BabyAGI、Manus、Cursor 全部收敛到这个 |
| 任务有天然层级结构（长文本 / 多页内容） | **真 P&E（参考 STORM 架构）** | 层级结构让 static plan 买到 coherence 而非 rigidity |
| Read-only + 并行（Deep Research 类） | **Hybrid：3-phase 外壳 + ReAct supervisor 动态 spawn parallel subagent** | LangChain open_deep_research、Anthropic Claude Research 都走到这里 |
| 需要 HITL / 合规审计 | **Magentic-One 风格的 hybrid P&E with replanning** | Microsoft Agent Framework 的 MagenticBuilder 是参考 |

### 10.10 对反思的更新

回到 §八.5 那个"独立重新发明前人答案"的反思，现在要补一条**相反方向**的观察：

**前一版反思说**："碰到工程问题时，去搜 dataflow / 流处理 / 增量计算等领域——前人已经做过"。这是对的。

**但本节的发现给出了一个补充**：**有些"前人做过的答案"其实已经被前人自己抛弃了**。P&E 就是这样——2023 年是显学，2025-2026 年在 coding / workflow / 计算机使用域**被实证抛弃**（n8n 删节点、LangChain 弃用 tutorial、BabyAGI 2o 坍缩到 174 行、AutoGPT 转型 workflow builder、Manus 公开承认放弃 todo.md、Cognition 的 "Don't build multi-agents"）。

所以反思的完整版是：

1. **碰到工程问题时**：先搜相关理论领域（dataflow / 流处理 / OT / CRDT / 增量计算），前人大概率做过
2. **但是**：也要搜**最近 12-24 个月的 post-mortem 和 deprecation 记录**，因为前人做过的东西**也会过时**。"有文献"不等于"还是好主意"
3. **特别警惕"marketing 与 code 脱节"**：2026 年开源 agent 生态里大量项目打着 "P&E" 的旗号但代码是 ReAct。去读源码，不信 README
4. **frontier 模型能力的提升会悄悄改变架构选择**：2023 年 P&E 合理是因为 GPT-3.5 planner 比 executor 强；2026 年 Sonnet 4.5 / Opus 4.6 够聪明了，"scaffold 代替模型 reasoning" 的理由消失了

**对自己场景的启示**：前面 §七 / §八 说的"选 P&E 的代价就是进入 dataflow 的问题域"依然成立，而且更尖锐了——因为**连 vendor 都在回避这个代价**，你要么接受 dataflow 工程税（像 Microsoft Magentic 那样真的把 dataflow 机制建起来），要么切到 ReAct + TodoWrite 方向。**选择 P&E 在 2026 年是个"逆流而上"的决定**，在内容生成这类天然层级化的域里是合理的，在其他域里需要非常强的理由。

---

## 参考来源

### 英文一手资料

**范式调研类**：
- [Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Workflow vs Agent 二分 + 6 种模式
- [LangChain — Planning Agents](https://blog.langchain.com/planning-agents/) — Plan-and-Execute / ReWOO / LLMCompiler
- [arxiv 2509.08646 — Plan-then-Execute](https://arxiv.org/abs/2509.08646) — P-t-E 的安全性与工程实现
- [arxiv 2404.11584 — Survey of Emerging AI Agent Architectures](https://arxiv.org/abs/2404.11584) — 单 agent vs 多 agent 综述
- [LangChain — Reflection Agents](https://blog.langchain.com/reflection-agents/) — Reflexion / Self-Refine
- [arxiv 2310.04406 — LATS](https://arxiv.org/abs/2310.04406) — Tree Search 统一 ReAct + Reflexion

**Dataflow / 增量计算 / Fan-in 相关**（§六、§八 引用）：
- [Frank McSherry — Differential Dataflow 博客](https://github.com/frankmcsherry/blog) — Timely / Differential Dataflow 作者的博文合集，入门最佳
- [Materialize 官方文档](https://materialize.com/docs/) — 工业级 differential dataflow 的产品化
- [Salsa — Rust 增量计算框架](https://salsa-rs.github.io/salsa/) — rust-analyzer 的底层，设计上最接近 LLM agent 的"重新计算复用旧结果"场景
- [React Reconciliation 文档](https://legacy.reactjs.org/docs/reconciliation.html) — `key` prop + diff 算法的官方解释
- [LangGraph — Send API / Map-Reduce](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) — LLM agent 圈的 dynamic task mapping 实现
- [Airflow — Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html) — `.expand()` 的官方文档
- Edward Lee — *Concurrent Models of Computation*（UC Berkeley 讲义 / 教材）— KPN / SDF / Petri net / dataflow 的系统教材

**2026 实证调研相关**（§十 引用）：
- [Cognition — Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — Context 碎片化问题的经典 post-mortem
- [Tam et al. 2024 — Let Me Speak Freely?](https://arxiv.org/html/2408.02442v1) — JSON mode 降低 reasoning 质量的定量证据（最多 38 点降幅）
- [LangChain — Deep Agents blog](https://blog.langchain.com/deep-agents/) — **包含 "Claude Code's TodoWrite is a no-op" 的原话**
- [LangChain — How and when to build multi-agent systems](https://blog.langchain.com/how-and-when-to-build-multi-agent-systems/)
- [LangChain — Prebuilt middleware (TodoListMiddleware / write_todos)](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [LangChain 论坛 — 1.0 里怎么实现 P&E](https://forum.langchain.com/t/what-is-the-best-way-to-implement-plan-and-execute-with-langchain-1-0-and-langgraph/2205)
- [LangChain open_deep_research 仓库](https://github.com/langchain-ai/open_deep_research) — 从严格 P&E 重写为 supervisor/ReAct hybrid
- [LangChain deepagents 仓库](https://github.com/langchain-ai/deepagents)
- [Manus — Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — 公开承认放弃 todo.md
- [Pragmatic Engineer — How Claude Code is built](https://newsletter.pragmaticengineer.com/p/how-claude-code-is-built)
- [ZenML LLMOps — Claude Code single-threaded master loop](https://www.zenml.io/llmops-database/claude-code-agent-architecture-single-threaded-master-loop-for-autonomous-coding)
- [Cursor — Best practices for coding with agents](https://cursor.com/blog/agent-best-practices)
- [Cursor — Introducing Plan Mode](https://cursor.com/blog/plan-mode)
- [BabyAGI Technical History (babyagi.wiki)](https://babyagi.wiki/) — BabyAGI 2o 的 "the LLM is the planner"
- [yoheinakajima/babyagi_archive](https://github.com/yoheinakajima/babyagi_archive) — 冻结的原版 P&E 快照
- [Taivo Pungas — Why AutoGPT fails and how to fix it](https://www.taivo.ai/__why-autogpt-fails-and-how-to-fix-it/)
- [Lorenzo Pieri — How to fix AutoGPT](https://lorenzopieri.com/autogpt_fix/)
- [Microsoft Agent Framework — Magentic Orchestration](https://learn.microsoft.com/en-us/agent-framework/workflows/orchestrations/magentic)
- [AutoGen — Magentic-One user guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
- [Magentic-One paper (arXiv 2411.04468)](https://arxiv.org/html/2411.04468v1)
- [ADaPT — As-Needed Decomposition and Planning (arXiv 2311.05772)](https://arxiv.org/abs/2311.05772) — "按需分解"的学术对应
- [Stanford STORM](https://github.com/stanford-oval/storm) — 长文本 P&E 的 gold standard
- [GPT-Researcher](https://github.com/assafelovic/gpt-researcher)
- [OpenManus PlanningFlow](https://github.com/FoundationAgents/OpenManus/blob/main/app/flow/planning.py) — 仍活跃的通用 P&E 参考
- [Anthropic — Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- [n8n — Plan and Execute Agent (legacy, 已弃用)](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/plan-execute-agent/) — v1.82.0 删除的 smoking gun
- [SqueezeAILab/LLMCompiler](https://github.com/SqueezeAILab/LLMCompiler) — ICML 论文参考实现（冻结）
- [billxbf/ReWOO](https://github.com/billxbf/ReWOO) — ReWOO 论文参考实现（冻结）

**Operational Transformation / Edit-Script 相关**（§九 引用）：
- [Quill Delta spec](https://github.com/quilljs/delta) — 最小、最易读的 OT 实现，支持 compose / transform / invert
- [Automerge 官方文档](https://automerge.org/docs/) — OT 向 CRDT 演化的产物，支持 move 和离线合并
- [Yjs 官方文档](https://docs.yjs.dev/) — 另一个主流 CRDT 实现
- [LSP `TextDocumentContentChangeEvent` 规范](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocumentContentChangeEvent) — 文本同步的工业标准
- [Android `DiffUtil` 文档](https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil) — RecyclerView 的列表差分工具
- [Daniel Spiewak — Understanding and Applying Operational Transformation](http://www.codecommit.com/blog/java/understanding-and-applying-operational-transformation) — OT transform 函数的直觉解释
- Ellis & Gibbs — *Concurrency Control in Groupware Systems* (SIGMOD 1989) — OT 的奠基论文
- Sun et al. — *Transparent Adaptation of Single-User Applications for Multi-User Real-Time Collaboration* (TOCHI 2006) — GOTO 算法，OT 的综合成果
- [Aider — editblock 格式文档](https://aider.chat/docs/more/edit-formats.html) — LLM-as-op-producer 的开源代表

### 中文社区文章（不同切法的横向参考）

国内平台上"X 种 Agent 设计模式"类文章很多，下面这些口径各有差异，可以对照看：

- [知乎 — Agent 的九种设计模式（图解+代码）](https://zhuanlan.zhihu.com/p/692971105) — 流传最广的"九种"分类
- [知乎 — AI Agent 模式全景图：从 ReAct 到 Multi-Agent 的演进之路](https://zhuanlan.zhihu.com/p/1968071452067620000) — 演进视角
- [zair.top — AI Agent 智能体四类设计模式](https://www.zair.top/post/ai-agent-design-pattern/) — Andrew Ng 的"四类"中文整理（Reflection / Tool Use / Planning / Multi-agent）
- [博客园 — ReAct vs Plan-and-Execute：LLM Agent 模式实战对比](https://www.cnblogs.com/muzinan110/p/18552824) — 两种模式的实战对比
- [博客园 — LLM 范式和多 Agent 架构（ReAct、Plan-and-Execute）](https://www.cnblogs.com/pass-ion/p/19360735) — 范式总结
- [知乎 — Build Effective Agents：Anthropic 的 Agent 年终经验总结](https://zhuanlan.zhihu.com/p/13760806591) — Anthropic 那篇的中文解读
- [知乎 — Anthropic 构建 Agent 的综合指南](https://zhuanlan.zhihu.com/p/1995926219003297892) — 同上，另一份解读
- [PPIO — 一文看懂 2025 年 Agent 六大最新趋势](https://ppio.com/blogs/post/yi-wen-kan-dong-2025nian-agentliu-da-zui-xin-qu-shi-aizhuan-lan) — 趋势视角

观察：这些文章的"X 种"分类大多可以归到上面 §一 的两条轴里（谁控制 × 拓扑形状），差异主要来自切法不同——有的按推理结构切（ReAct / Plan / Reflect），有的按论文谱系切（ReAct → Reflexion → ToT），有的按交互形态切（单 agent / 多 agent / 协作）。**没有谁错，只是视角不同。**

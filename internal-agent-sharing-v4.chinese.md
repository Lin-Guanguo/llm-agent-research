# Agent 大学习

Last Updated: 2026-06-09

Source: 飞书文档 `H94odHRkpoL6ZxxutH8cLqi2noh`, revision `1541`, fetched on 2026-06-09.

说明：这是飞书内部分享稿的本地 Markdown 版本。飞书里的图片、grid、callout 和嵌入表格在这里转成普通 Markdown；图片只保留文字占位，表格内容合并到对应章节。

## 0. 什么是 Agent

这部分开场可以先把常见产品摆出来：Claude Code、Codex、OpenClaw、豆包、千问这类产品都可以被放进 Agent 讨论里。

最简单的区分：

- 模型调用：一次输入，一次输出，模型主要负责生成。
- Agent：模型进入一个执行环境，可以观察上下文、调用工具、接收 observation，再决定下一步。

名词在 Agent 里大致对应：

- model：负责理解、选择动作、生成内容。
- tool：模型能触达的行动接口。
- observation：工具执行后的结果，被放回上下文。
- runtime / harness：真正调度工具、维护状态、做权限和错误处理的外层系统。
- plan：让复杂任务不失忆、不跑偏、不重复劳动的任务组织形式。

## 1. Agent 最小模型：边观察边行动

现代 tool-calling Agent 的底层循环其实很小：

```text
model -> tool call -> observation -> model
```

模型看到上下文，决定要不要调用工具。Runtime 执行工具，把结果放回上下文。模型再根据新的观察决定下一步。如果模型不再发起 tool call，系统通常认为这一轮可以结束。

这个思想最早可以追溯到 ReAct：Yao et al. 在 2022/2023 年提出让模型交替生成 reasoning trace 和 action，典型形式是：

```text
Thought -> Action -> Observation -> Thought ...
```

早期实现也很轻量，主要是 notebook 形式的 prompting experiments：[ysymyth/ReAct](https://github.com/ysymyth/ReAct)。

它当时还没有今天的 function call / tool call 协议，核心循环本质上是让模型输出一段文本，再由宿主程序解析 action 并执行。

这就是很多 Agent 的共同内核。Claude Code、Codex、OpenClaw、LangChain `create_agent`，内层都可以近似看成这种反馈循环。

核心结论：

> Agent 的核心循环很简单，LLM 只有两种行为：用 tool，看结果继续；或者输出结果，结束。复杂的是外层 harness。

这件事的名字也换过几轮：

```text
Prompt Engineering -> Context Engineering -> Harness
```

Harness 已经从“怎么把 prompt 写好”，进化到“怎么组织 tool、context、subagent、验证、长期任务和多 Agent 协作”。

## 2. Tool Surface：复杂设计的共同入口

ReAct 给的是最小循环，那么复杂 Agent 的下一步，是设计模型到底能做哪些 action。

Yao Shunyu 最早开发的 ReAct 主要靠文本输出 action；现在的模型已经把 tool call 作为一等公民。这里的 action 不只是普通 API 调用。文件读写、shell、计划管理、skill、subagent、长程任务，都可以被包装成模型可调用、runtime 可约束、结果可回到 observation 的 tool。

### 2.1 Claude Code：Coding Agent 的 Tool Surface

以 Claude Code 为例，它的 tool surface 不是一组扁平能力，而是把 agent 执行拆成几类可调用接口。

| 类别 | 代表 tool | 作用 |
|---|---|---|
| workspace tools | `Read` / `Edit` / `Write` / `NotebookEdit` | 读取、修改、创建代码文件和 notebook |
| search & navigation tools | `Glob` / `Grep` / `LSP` | 按文件名、内容和代码语义定位上下文 |
| execution tools | `Bash` / `PowerShell` | 运行测试、构建、脚本和诊断命令 |
| planning & progress tools | `TodoWrite` / `TaskCreate` / `TaskGet` / `TaskUpdate` / `TaskList` | 拆解任务、标记进度、维护依赖 |
| delegation tools | `Agent` / `SendMessage` / `TeamCreate` / `TeamDelete` | 启动 subagent、继续已有 agent，或组织多个 agent 协作 |
| background & scheduling tools | `TaskOutput` / `TaskStop` / `Sleep` / `CronCreate` / `CronList` / `CronDelete` / `RemoteTrigger` | 后台任务、等待、停止和未来执行 |
| external knowledge tools | `WebSearch` / `WebFetch` / `ListMcpResourcesTool` / `ReadMcpResourceTool` / `mcp__<server>__<tool>` | 访问网页、MCP 资源和外部系统 |
| mode & meta tools | `EnterPlanMode` / `ExitPlanMode` / `AskUserQuestion` / `Skill` / `ToolSearch` | 控制交互模式、请求澄清、调用技能，并按需加载 deferred tool schema |

嵌入表格里还记录了一些源码级工具名和备注：

| Tool | 备注 |
|---|---|
| `Agent` | 启动子 agent / subprocess 处理复杂任务，旧别名 `Task` |
| `TaskOutput` | 读取后台 task / agent / bash 输出，已标 deprecated，建议直接读 output file |
| `TaskStop` | 停止后台任务，旧别名 `KillShell` |
| `EnterWorktree` / `ExitWorktree` | 进入或退出隔离 git worktree |
| `REPL` | ant CLI REPL 模式；开启后部分基础工具会隐藏到 REPL VM 内 |
| `StructuredOutput` | schema 化结果的输出工具，不在普通 base tools 主列表里 |

对 Coding Agent 来说，`Bash` 几乎成了开放式软件任务的标准行动接口。软件任务的不确定性很高，模型需要不断读文件、跑命令、看结果、再决定下一步。测试、build、diff，都可以通过 Bash tool 回到循环里。

### 2.2 OpenClaw：Personal Agent 的 Tool Surface

OpenClaw 代表另一种方向。它不是把 Agent 主要放进代码仓库，而是放进长期运行的个人助理环境。它的 tool surface 更关心 memory、session、cron、heartbeat、message、browser 和后台生命周期。

Personal Agent 的难点不是单次任务计划，而是身份、记忆、渠道、异步唤醒和副作用边界。

OpenClaw 的工具面可以分成几类：

- workspace / coding tools：`read` / `write` / `edit` / `apply_patch`，以及 `exec` / `process`。
- web / memory tools：`web_search` / `web_fetch` / `x_search`，以及 `memory_search` / `memory_get`。
- session / delegation tools：`sessions_list` / `sessions_history` / `sessions_send` / `sessions_spawn` / `sessions_yield` / `subagents` / `session_status`。
- messaging / UI / device tools：`message` / `browser` / `canvas` / `nodes` / `gateway`。
- automation / lifecycle tools：`cron` / `heartbeat_respond`。
- agent state / planning tools：`agents_list` / `get_goal` / `create_goal` / `update_goal` / `update_plan` / `skill_workshop`。
- media tools：`image` / `pdf` / `image_generate` / `music_generate` / `video_generate` / `tts`。

### 2.3 异步 Tool

还要单独注意一类 tool：异步 tool。

普通 tool 更像函数调用：模型发起调用，runtime 执行，结果立刻作为 observation 回到上下文。

但 cron、heartbeat、subagent、background task 这类能力更像异步 tool：一次 tool call 会启动或登记一个长生命周期执行，真实工作在另一个生命周期里并发进行。结果再通过新的 user message、session update、poll / retrieve 或 wake event 回到 Agent 上下文。

也就是说，tool surface 不只是“函数集合”，它决定了模型能触达什么空间：文件系统、shell、记忆、会话、长期任务、未来唤醒、外部系统和其他 Agent。

## 3. 更复杂的 Agent 设计在解决什么问题

ReAct Agent 的核心结构从来没有发生太大变化。最简结构有最高灵活性和最大自由度，各个成熟通用 Agent 也基本采用这个内核。

那为什么还要投入人力做复杂 Agent 设计？

一篇有参考价值的文章是：[《从0开发大模型的17种Agent架构演进详细拆解》](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-06-09)。它讨论了几种 Agent 设计，虽然分成 17 种有些硬分类的意思，但每种设计其实都在解决特定问题。

相比记住 17 个名字，更重要的是每个设计都在回答这些问题：

1. **它的 State 是什么？** 新增了哪些字段，为什么必须存在。
2. **它的拓扑是什么？** 线性链、循环、分叉汇聚、共享黑板、树搜索还是网格涌现。
3. **它的 Router 怎么工作？** 固定边、条件边、动态调度、验证回路、人工审批。
4. **它的失败模式是什么？** 架构最容易在哪个环节坏掉。

我的理解是：这篇文章把工作流搭建过程中应对 Agent 不确定性的策略，总结成十几种方式，供我们在遇到相关问题时参考。

| 阶段 | 新增能力 | 一句话解释 | 代表架构 |
|---|---|---|---|
| 单次生成优化 | critique pass | 让模型先出一版，再自己挑毛病改掉，把生成拆成 generator + critic + refiner 三步 | Reflection |
| 与世界交互 | tool interface | 把外部 API / 函数挂成结构化工具接口，打破参数知识边界和上下文封闭性 | Tool Use |
| 基于观察持续行动 | observation loop | Thought -> Action -> Observation 滚动循环，上一步的观察决定下一步动作 | ReAct |
| 先生成控制流再执行 | explicit planning | 先让模型产出可检视步骤清单，再按清单执行，把控制流做成可审计对象 | Planning |
| 把验证接入主回路 | verification loop | 每一步执行完强制 verifier，失败就回到重规划，而不是事后检查 | PEV |
| 把认知任务拆成角色 | role decomposition | 研究员、写作、审阅等角色拆开，用流水线或图串起来 | Multi-Agent |
| 把中间状态显式共享 | shared workspace | 中间产物写到共享黑板，controller 根据黑板状态调度下一步 | Blackboard |
| 把入口做成路由系统 | entry routing | 请求入口先分类，把任务路由到合适专家子 agent | Meta-Controller |
| 用冗余换可靠性 | parallel redundancy | 同题交给多个独立 agent，再聚合、融合或投票 | Ensemble |
| 把历史状态纳入系统 | long-term memory | 对话片段进向量库，结构化事实进图或 KV，让系统记得住、查得到 | Episodic + Semantic / Graph |
| 把推理变成搜索 | search tree | 展开多条路径，边展开边打分、剪枝，把推理变成搜索 | ToT |
| 把行动前评估做成模拟 | counterfactual execution | 真正动手前，在内部世界模型里预演，评估风险收益 | Mental Loop |
| 把副作用关进闸门 | side-effect gating | 有副作用动作必须先 dry-run + 审核，再允许真实执行 | Dry-Run |
| 把自我边界建模 | self-boundary reasoning | 系统维护自我模型，知道擅长什么、不擅长什么，选择自己做、调工具或交给人 | Metacognitive |
| 把质量改进做成循环 | iterative refinement loop | editor 打分并给修改意见，writer 按意见改，高分样本沉淀到未来 | Self-Improve |
| 去中心化计算 | emergence | 没有中心 LLM 推理，每个单元只跑局部规则，全局行为从局部交互中涌现 | Cellular Automata |

这些设计本质上是在处理 LLM 进入工作流之后带来的不确定性。最小 tool loop 适合开放探索，因为模型可以根据 observation 自由调整路径；约束加得太早，可能反而限制模型发挥。

但专用场景的要求不同。面对固定业务场景和会被重复执行成千上万次的流程，系统不能只追求少数情况下的高上限，还要保证大多数情况下的质量下限。这时就需要把一些原本靠模型临场判断的东西，逐步变成显式控制流设计。

这导致两条优化路线：

1. **通用 Agent**：例如 Claude Code。这类系统不会增加太多程序化验证，因为任何约束都会限制发挥。它会持续增加更适合模型使用的 tool、tool 使用策略和评估体系，提升能力上限。优化重心是效果评测和 tool 设计。
2. **业务场景下线上运行的 ToC Agent**：它会在不断丰富能力的同时，追求稳定性和低 token 消耗。优化重心是效果评测和流程设计。

## 4. 通用 vs 专用：有没有标准 Plan

到这里可以把问题收束到一个词：Plan。

软件工程里也有类似经验。面对复杂需求，人类通常不会直接开做，而是先拆解计划；当同类需求反复出现时，又会把计划进一步沉淀成流程、模板或研发规范，用来提高需求交付的质量下限。

这里强调的是“下限”。如果执行者足够聪明，或者通用 Agent 足够强，它当然可以在更少约束下完成任务。但专用业务场景通常不能只赌上限，而是要保证大多数请求都能稳定落在可接受质量以上。

Plan 解决的不是“模型会不会想”，而是复杂过程中 Agent 还能不能稳定把事做完：不要失忆、不要跑偏、不要重复劳动。它有点像人在按 SOP 做事，未必提高能力上限，但能减少低级错误。

Plan 的取舍可以先这样看：

- 如果流程本身固定，直接写成固定 Workflow 就够了。开发者把路径、分支和校验写清楚，runtime 负责稳定执行。
- 如果任务非常开放，Plan 更适合留在 prompt 和模型注意力里。太早把它限定成标准 workflow，反而会牺牲通用 Agent 最重要的灵活性。

按这个取舍，可以把 Plan 分成三类。

### 4.1 Workflow

计划由开发者写成 graph、DAG 或代码。这里有标准流程，但动态性主要来自开发者预先设计。

优点是稳定、可审计、容易控制。缺点是动态性弱，很多分支需要提前设计。

LangGraph 官方的邮件解析案例就是固定流程 workflow 的典型用法：开发者把状态、节点和边写清楚，runtime 按图执行。这是 LangGraph 的舒适区。

参考：[LangGraph: Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph) (accessed: 2026-06-09)。

### 4.2 Context Plan

计划主要给模型看，不是 runtime 的标准调度协议。它的作用是帮助模型维持注意力、记录进度、和用户协作。经典实现是 todo list。

Superpowers 这类插件也可以放到这里理解。它把软件开发过程规范成类似：

```text
brainstorm -> write plan -> execute plan
```

本质上是在 Agent 开发需求时强制引入一个 plan 阶段，让模型先形成计划，再按计划推进。但这个 plan 主要还是工作协议和上下文载体，不是 runtime 自动调度的执行图。

这类 plan 的优点是灵活，适合开放任务。缺点是 runtime 很难直接调度它，也很难证明它真的完成了。

Harness 工程会在执行 Plan 的过程中或完成后进行 evaluation。语法层面、程序层面的事情可以程序校验；需求完成度这类目标，往往需要另一个“隔离上下文的 Agent”评估。但这个评估 Agent 是否准确，又是一个新的不确定性来源。

因为 plan 都在上下文里，一个 Agent 承载不了复杂任务的全部上下文，所以复杂任务一般会使用 subagent、AgentTeam 等设施。多个 ReAct Agent 每个负责一部分，互相之间加上通讯设施。背后的假设是：如果隔离上下文做得好，每个 Agent 可以把分内事情做好。

### 4.3 Dynamic Workflows

Dynamic Workflows 指的是：workflow 不完全由开发者预先写死，而是在运行时由模型、planner 或其他程序生成、选择或修复，再交给某种 runtime 执行。

#### Claude Code Dynamic Workflows

最近 Claude Code 的 dynamic workflows 可以归到这一类。

Claude 生成 JavaScript orchestration script，script 负责循环、分支、并行调 subagents 和保留中间结果，runtime 在后台执行。

参考：

- [Claude Code Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-09)
- [Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-09)

Bun 仓库迁移案例是一个典型例子。前段时间 Bun 被 Anthropic 收购，创始人用差不多一个月时间，把整个代码仓库语言从 Zig 切换到 Rust。公开 workflow 示例之一是：

[Bun `phase-g-mega-swarm.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-g-mega-swarm.workflow.js#L1-L25) (accessed: 2026-06-09)。

这个 workflow 做的是一个“大规模测试修复流水线”：

1. 每轮只构建一次：先等主仓库里的 build daemon 完成构建，普通修复 agent 不允许自己跑 build/test。
2. 批量扫描测试：survey agent 用 `xargs -P16` 并行跑一批测试，并和 `USE_SYSTEM_BUN=1` baseline 对比。
3. 挑出失败目标：只把真实 divergence 视为 port bug，每轮最多取 `MAX_FIX=30` 个失败测试。
4. 并发修复：每个失败测试交给一个 fix agent，它只能看诊断日志、测试文件、Rust 实现和 Zig spec。
5. 双人审核：每个 fix 交给两个 review agent 独立审核，抓 unsafe、层级 hack、Zig spec mismatch、reward hacking。
6. 必要时应用修正：review 不通过但给出 correction 时，再启动 apply agent。
7. 推送并进入下一轮：直到所有目标测试通过或达到最大轮数。

一句话：它把 Claude 从“一个人边想边修”变成了一个由 JS 控制的测试修复调度系统：集中构建、批量诊断、并发修复、双重审核、循环收敛。

这个 workflow 的脚本形态大致如下：

```js
export const meta = {
  name: "phase-g-mega-swarm",
  description:
    "ONE rebuild/round in main repo. Survey ALL tests. Fan out fix agents. 2-vote review each. Loop.",
  phases: [
    { title: "Build", detail: "builder daemon owns the build" },
    { title: "Survey", detail: "run tests in parallel and compare baseline" },
    { title: "Fix", detail: "parallel fix agents, one per failing test" },
    { title: "Review", detail: "two independent reviewers per fix" },
  ],
};

for (let round = 1; round <= MAX_ROUNDS; round++) {
  phase("Build");
  const build = await agent("Wait for builder daemon...", {
    label: `wait-build-r${round}`,
    phase: "Build",
    schema: BUILD_S,
  });

  if (!build?.ok) {
    await agent("Fix compile error, commit explicit paths only...", {
      label: `buildfix-r${round}`,
      phase: "Build",
    });
    continue;
  }

  phase("Survey");
  const survey = await agent("Survey all tests and return true divergences...", {
    label: `survey-r${round}`,
    phase: "Survey",
    schema: SURVEY_S,
  });

  if (survey.failing.length === 0 && (survey.uncovered ?? 1) === 0) {
    return { rounds: round, done: true, passing: survey.passing, total: survey.total };
  }

  const targets = survey.failing.slice(0, MAX_FIX);

  await pipeline(
    targets,
    f => agent(`Fix ${f.file} from diagnostic and source...`, { phase: "Fix", schema: FIX_S }),
    (fix, f) =>
      parallel(
        [0, 1].map(i => () =>
          agent(`Review fix for ${f.file}. Diff: git show ${fix.commit}`, {
            phase: "Review",
            schema: REVIEW_S,
          }),
        ),
      ),
    (review, f) =>
      review && !review.accepted
        ? agent(`Apply corrections for ${f.file}`, { phase: "Apply", schema: APPLY_S })
        : review,
  );
}
```

#### Comico Planned DAG Workflow

Comico 是我们设计的 Planned DAG Workflow：planner 在运行时生成确定性 DAG；runtime 负责验证、调度、执行和记录状态。

它试图同时拿到两件事：

- planner 带来的动态性；
- workflow runtime 带来的稳定性。

代价是系统复杂度上升。灵活性和稳定性的取舍一直存在。

小结：

> Workflow 是开发者写死流程；Context Plan 是相信模型能在 prompt 引导下做好；Dynamic Workflows 是运行时生成执行结构。

## 5. Comico：为什么选择 Planned DAG Workflow

Comico 的目标不是做通用 coding agent，也不是做长期个人助理，而是做业务任务自动化。

我们的约束是：

- token 成本要低；
- 一次成功率要高；
- 多步骤执行要稳定。

如果完全用 model-driven tool loop，每一步都让模型重新判断下一步，会更灵活，但成本和不确定性都更高。

所以 Comico 更接近一种 Plan-and-Execute：

```text
planner 生成 Planned DAG
-> runtime 验证并执行
-> internal subagents 完成具体步骤
```

顶层不是一个自由行动的 LLM manager。顶层是程序化 workflow runtime。LLM 参与其中，但主要产出结构化 artifacts：

- planner 产出 workflow DAG；
- 用 plan 调度 internal subagent 产出 typed output。

Planner 不是完全无状态 prompt。它会根据当前 request、历史状态、已完成步骤、planner policy、可见能力和 validation feedback 生成 workflow DAG。

Runtime 不让模型自由决定每一步，而是验证 DAG、调度 ready step、执行 subagent，并写入 state。

### 5.1 Planner 输出示例

以下是一个 planner 输出示例：

```json
{
  "goto": "coordinator_enter",
  "plan": {
    "memory_query": "",
    "reply_message": "收到，沈玄昭潜入与徐砚孺强攻宫城的双线剧情正在绘制中",
    "output_summary": "根据用户提供的沈玄昭宫城潜入与徐砚孺正面强攻的剧情，生成对应的多页漫画插画与配音。",
    "execution_steps": [
      {
        "step_id": "style_resolver_0",
        "capability_id": "style_resolver",
        "input_bindings": {
          "intent": "input.user_text",
          "ref_images": "input.reference_assets"
        }
      },
      {
        "step_id": "story_writer_0",
        "capability_id": "story_writer",
        "input_bindings": {
          "intent": "input.user_text",
          "ref_images": "input.reference_assets"
        }
      },
      {
        "step_id": "pages_enhancer_0",
        "capability_id": "pages_enhancer",
        "input_bindings": {
          "style": "style_resolver_0.style_description",
          "page_ops": "story_writer_0.page_ops",
          "style_id": "style_resolver_0.style_id",
          "characters": "story_writer_0.characters",
          "ref_images": [
            "style_resolver_0.style_ref_images",
            "input.reference_assets"
          ]
        }
      },
      {
        "step_id": "char_ref_render_0",
        "capability_id": "char_ref_render",
        "input_bindings": {
          "style": "style_resolver_0.style_description",
          "ref_pool": [
            "style_resolver_0.style_ref_images",
            "input.reference_assets"
          ],
          "style_profile": "style_resolver_0.style_profile",
          "character_specs": "pages_enhancer_0.characters",
          "is_custom_style": "style_resolver_0.is_custom_style",
          "image_generation_model": "style_resolver_0.image_generation_model"
        }
      },
      {
        "step_id": "image_render_0",
        "capability_id": "image_render",
        "input_bindings": {
          "style": "style_resolver_0.style_description",
          "ref_pool": [
            "char_ref_render_0.ref_pool",
            "style_resolver_0.style_ref_images",
            "input.reference_assets"
          ],
          "style_id": "style_resolver_0.style_id",
          "task_ops": "pages_enhancer_0.task_ops",
          "story_title": "story_writer_0.story_title",
          "is_custom_style": "style_resolver_0.is_custom_style",
          "image_generation_model": "style_resolver_0.image_generation_model"
        }
      },
      {
        "step_id": "bgm_design_0",
        "capability_id": "bgm_design",
        "input_bindings": {
          "page_ops": "story_writer_0.page_ops",
          "story_summary": "story_writer_0.summary"
        }
      },
      {
        "step_id": "tts_0",
        "capability_id": "tts",
        "input_bindings": {
          "task_ops": "pages_enhancer_0.task_ops",
          "characters": "story_writer_0.characters"
        }
      }
    ]
  },
  "memory_retriever_decision": "skipped"
}
```

这个例子说明：Plan 不是“给模型看的步骤文本”，而是 runtime 可以验证、调度和执行的 DAG contract。每个 step 都有 `step_id`、`capability_id` 和 `input_bindings`，runtime 可以检查能力是否存在、依赖是否可达、绑定字段是否合规。

### 5.2 request_type：入口决定 planner 形态

Comico 不是试图让同一个 planner prompt 覆盖所有对话场景。端上本来就有明确的入口差异，比如 onboarding、chat、story。我们把这种入口差异通过 `request_type` 传进 Agent。

`request_type` 的作用可以简单理解为：决定 planner 形态。

```text
request_type
-> planner prompt / policy
-> planner 可见的 subagent 集合
-> planner 生成对应场景下的 workflow DAG
```

这样做的原因是，不同场景的 planner 目标不同。Onboarding 更像固定开场流程，chat 更像需求收集和确认，story 更像正式生成和编辑。如果全部压进一个 prompt，prompt 会越来越大，而且模型很难稳定判断当前到底该遵守哪套规则。

目前可以先按几个场景理解：

- `chat`：需求收集、灵感引导、确认创作方向。重点不是立刻生成完整作品，而是把用户意图整理到可执行的创作表单或故事方向里。
- `story`：正式生成和后续编辑。Planner 需要面向完整创作链路，选择 story writer、style、image、tts 等能力，并处理跨轮编辑。
- `onboarding`：新用户开场链路。它更接近固定流程，需要围绕用户画像、OC seed、定制画风、首轮故事和渲染建立稳定的首轮体验。
- `oc`：未来可能独立出来的 OC 设计 / 角色设定场景。它可能更关注角色、设定、画风和素材组织，而不是直接进入完整 story 生成链路。

所以 `request_type` 不是把业务逻辑硬编码成完整 workflow，而是在 Plan-and-Execute 之前先做一次业务场景收束：让 planner 在更小、更明确的场景里生成 DAG。它牺牲了一点通用性，换来更清晰的 planner policy、更少的可选动作和更稳定的输出下限。

### 5.3 收益和代价

Comico 选择 Planned DAG Workflow，主要收益是：

- 成本更可控：不需要每一步都让模型重新决定下一步。
- 执行更稳定：runtime 负责验证、调度和状态记录。
- 问题更可定位：错误可以落到 DAG、binding、subagent output 或 validator。

代价也明确：

- Plan schema、binding 和 subagent contract 会变成核心系统协议。
- validator、repair、fallback 会变成必须维护的基础设施。
- runtime state 和 planner policy 的边界需要持续治理。

核心取舍：

> 通用 Agent 倾向把 plan 留在模型上下文，以换取开放任务的灵活性。业务型 Agent 如果更看重低 token、稳定性、审计和恢复，就会更追求 deterministic。

## 6. Plan 生产化：验证、修复、评估、限制

前面讲的是 Plan 的形态。真正落到生产系统里，问题会再往下走一层：Plan 不能只是“模型生成出来的一段结构”，它必须能被验证、修复、观测和评估。

这部分可以作为 Comico 的核心工程经验来讲：业务型 Agent 的可靠性，往往不来自更复杂的 agent 名词，而来自 plan 周围的一组程序化设施。

### 6.1 三类程序化设施

第一类是 **plan-level validation**。

Planner 产出的 workflow DAG 要先被检查：subagent 是否存在，binding 是否合法，依赖是否可达，DAG shape 是否符合当前 planner policy。它保护的是“这一轮 workflow 能不能被 runtime 执行”。

第二类是 **step-level validation and repair**。

Internal subagent 即使输出了合法 JSON，也可能在业务语义上错。这里需要 validator、deterministic repair、retry / fallback。它保护的是“单个节点的模型输出能不能被业务接受”。

第三类是 **evaluation and telemetry**。

生产化 Agent 不能只看单次 demo，而要持续记录 one-shot success、valid DAG rate、step failure、token cost、repair path、replan rate、fallback frequency。这样系统才会从“能跑”走向“长期稳定运行”。

### 6.2 例子：page index drift

一个具体例子是 page index drift。

业务里的 page index 本来就是 zero-based。本轮只要求处理：

```text
requested = [1, 2, 3, 4]
```

也就是说，page `0` 不应该被改动。但模型返回时倾向于把这个 batch 局部编号成：

```text
returned = [0, 1, 2, 3]
```

这不是类型错误。`page_index` 仍然是整数，结构也合法。错误在业务语义：模型把绝对下标改成了 batch-local 下标。

这类错误说明：structured output 只解决“可解析”，不保证“业务正确”。

可能更好的解法，是不要让模型直接输出数字，而是给每页一个代号，例如 alpha、beta、gamma。也就是说，不要让模型在它容易偏移的索引空间里工作。

### 6.3 限制：灵活性和稳定性的取舍

三种 plan 形态的限制可以这样看：

- ReAct / Context Plan 足够灵活，但如果想做硬性限制，很难做，只能在 prompt 约束模型，或者在部分 tool call 里加 hook。
- Workflow 最不灵活，稳定性最高，Agent 永远走标准流程。
- Planning / Dynamic Workflows 在两者之间做取舍。平衡很难掌握，但它确实是线上服务稳定性和灵活性之间目前比较理想的模式。

之前线上有一个 case：

用户要求“画面可以更鲜亮一点，色彩艳丽”。Planner 识别到这和画风渲染相关，而画面增强是 `pages_enhancer` 负责，于是 planner 把上一轮标注的“画风描述”改写了一下，再给到 `pages_enhancer`。这相当于 planner 为了符合用户要求，微调了一个画风。

这个行为后来被定义为 bug：灵活性超出了预计。调整方向是：planning 生成的计划不能改写画风，只能原样传递画风选择生成的画风。Planner 生成的计划可以通过 validator 限制灵活度。

这说明：Planned DAG Workflow 的难点不是“让模型输出结构化 JSON”，而是持续定义哪些自由度允许交给模型，哪些自由度必须被 runtime 收回。

## 7. 结尾

计算机长期给人的默认印象是：程序虽然可能写错，但执行本身是确定、可重复的。Agent 把这个印象打破了一部分。

LLM 进入程序主循环之后，程序开始带上人的特征：会理解、会迁移、会补全，也会误解、遗忘、过度泛化、看起来合理但实际跑偏。

所以很多 Agent 工程问题，本质上像是在把过去用来管理人的方法搬进程序系统：先写计划，执行中检查，关键动作审批，失败后复盘，经验沉淀成 SOP，高风险环节加 reviewer。

在对“下限”有要求的场合，这些方法现在不能只停留在管理语言里，而要变成 runtime 里的程序化设施：schema、validator、repair、fallback、evaluation、harness。

最后的核心判断：

> 承认模型带来的不确定性，并把模型擅长的“理解和组合”与程序擅长的“验证、调度和执行”拆开。

## 8. 延伸阅读

### Agent 基础和设计原则

- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (accessed: 2026-06-09) — workflow vs agent、何时增加复杂度、常见 workflow pattern 和 agent-computer interface。
- [OpenAI: A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) (accessed: 2026-06-09) — instructions、tools、guardrails、handoffs、orchestration 的产品化视角。
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) (accessed: 2026-06-09) — Codex 作为 model-tool loop harness 的拆解。
- [ysymyth/ReAct](https://github.com/ysymyth/ReAct) (accessed: 2026-06-09) — ReAct 早期 notebook / prompting 实现。

### Harness 和 Context Engineering

- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (accessed: 2026-06-09) — context 管理、context rot、tool result clearing、subagent 返回压缩等实践。
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) (accessed: 2026-06-09) — 长程 Agent 的 harness、进度管理、可控性和评估。
- [Claude Code Docs: How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) (accessed: 2026-06-09) — Claude Code 的 agentic loop、tools、context、sessions、permissions。
- [Claude Code Docs: Tools reference](https://code.claude.com/docs/en/tools-reference) (accessed: 2026-06-09) — Claude Code model-visible tool surface 的官方工具名和行为。

### Dynamic Workflow 和 Framework

- [Claude Code Docs: Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-09) — Claude Code workflow runtime、subagent orchestration、后台执行和恢复。
- [Claude: Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-09) — dynamic workflows 的产品发布和设计动机。
- [LangGraph: Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph) (accessed: 2026-06-09) — LangGraph workflow 设计示例。

### 复杂度和失败模式

- [从0开发大模型的17种Agent架构演进详细拆解](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-06-09) — 17 种 Agent 模式的教学型拆解，适合作为问题清单参考。
- [FareedKhan-dev/all-agentic-architectures](https://github.com/FareedKhan-dev/all-agentic-architectures) (accessed: 2026-06-09) — 17 种模式对应的代码项目来源。
- [Cognition: Don’t Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) (accessed: 2026-06-09) — 多 agent handoff、context loss 和复杂度税的工程复盘。

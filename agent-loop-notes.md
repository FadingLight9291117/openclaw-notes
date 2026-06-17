# Agent Loop 范式笔记

## 一、常见 Agent Loop 范式

### 1. ReAct（Reason + Act）
最经典的范式，每一步：**思考 → 行动 → 观察**，循环直到完成。

```
Thought: 我需要查一下天气
Action: search("北京天气")
Observation: 晴，25°C
Thought: 已有结果，可以回答了
Final Answer: ...
```

### 2. Plan-and-Execute
先整体规划，再逐步执行。规划与执行分离，适合长任务。

### 3. Reflection / Self-critique
执行后自我评估，不满意就修正再执行。代表：Reflexion 论文。

### 4. Tool-use Loop
模型输出 tool call，执行工具，结果追加到上下文，继续推理。Claude Code 本身就是这个范式。

### 5. Multi-agent（Orchestrator + Subagents）
一个 orchestrator 分解任务，派发给专门的 subagent 并行执行，汇总结果。

### 6. Tree of Thoughts（ToT）
在每个决策点展开多个分支，用评分函数剪枝，找最优路径。

### 7. LATS
ToT + Monte Carlo Tree Search，引入 UCB 策略做节点选择。

### 选型参考

| 场景 | 推荐范式 |
|------|---------|
| 通用任务、工具调用 | ReAct / Tool-use Loop |
| 长流程、复杂项目 | Plan-and-Execute |
| 代码生成/需要验证 | Reflection |
| 并行子任务 | Multi-agent |
| 数学/逻辑推理 | Tree of Thoughts |

---

## 二、ReAct 详解

ReAct 是 2022 年 Google 提出的论文（**Re**asoning + **Act**ing），核心思想是让 LLM **交替生成推理链和动作**。

每一轮由三部分组成：
```
Thought:     [模型的内部推理，分析当前情况]
Action:      [调用工具或执行操作]
Observation: [工具返回的结果，注入回上下文]
```

**为什么有效：**
- Thought 让模型显式推理，避免盲目行动
- Observation 把真实世界的反馈注入推理链，纠正幻觉
- 两者交织，形成"落地"的推理

| | Chain-of-Thought | ReAct |
|--|--|--|
| 推理 | 有 | 有 |
| 工具调用 | 无 | 有 |
| 外部信息 | 无 | 实时获取 |
| 适合场景 | 数学/逻辑 | 信息检索、操作任务 |

---

## 三、典型框架分析

### OpenClaw
开源通用 agent 平台。核心范式是**以 ReAct 为骨架的 tool-use loop**：

- **主循环**：ReAct / tool-use loop — 见 `agent-loop.md` 里的双重循环（外层 steering/follow-up 续命 + 内层"请求→执行工具→再请求"），这是已对照源码核实的实际形态。
- **可选叠加**：subagent 委派（Agent/Task 工具）属于按需扩展，不是默认的"多 agent 编排"；prompt 层的 todo/计划相当于轻量 plan 层。

> 不要把它描述成"Gateway 作 orchestrator 调度多个专职 subagent"——Gateway 是单一控制平面,agent 主体仍是单循环。下面是高层组件示意（非严格架构图）：

```
Channel → Gateway（控制平面）
              ↓
       Agent Loop（ReAct 主循环）
            ↙         ↘
 Memory & Knowledge   Plugins & Skills
            ↓
       LLM Provider
```

### OpenCode
SST 团队开源的终端原生编程 agent，bring-your-own-provider（号称 75+ 提供商）。**核心 agent 逻辑用 TypeScript 写，TUI 用 Go（Bubble Tea 风格），TUI 作为本地 agent server 的客户端**——不是"纯 Go 编写"。

- **核心**：Tool-use Loop
- **特色**：Plan/Build 双模式（Plan-and-Execute 的落地实现）

| 模式 | 权限 | 职责 |
|------|------|------|
| Plan Mode | 受限（只读为主） | 分析需求、制定方案 |
| Build Mode | 完整权限 | 执行：改代码、重构、跑测试 |

### Claude Code
Anthropic 官方的终端编程 agent，只接 Claude 模型。也是**以 ReAct 为骨架的 tool-use loop**，但把多种范式的扩展点做得最完整——值得作为"骨架 + 按需叠加"工程化形态的参照：

- **主循环（ReAct / tool-use loop）**：模型流式输出 → 检测 tool call → 执行 → 结果回写 transcript → 继续，直到 `end_turn`。
- **专用工具而非裸 bash**：`Read`/`Edit`/`Bash`/`Grep`/`Glob` 等独立工具，让 harness 能逐工具做**权限闸门、staleness 校验（编辑前文件改过就拒）、自定义渲染、并行调度**——这是"为什么要把动作提升成专用工具"的典型落地。
- **Plan 层**：Plan Mode（`EnterPlanMode`/`ExitPlanMode`）先只读分析、产出计划交人确认再执行，是 Plan-and-Execute 的轻量门控；TodoList 工具提供更细的任务跟踪。
- **Multi-agent（orchestrator-worker）**：`Task`/`Agent` 工具派发 subagent（如 Explore、general-purpose、Plan），可给 subagent 配**更便宜的模型**（Explore 用 Haiku）以省 token，并隔离主循环的上下文与 prompt cache。
- **上下文管理**：compaction（接近上限时服务端摘要）+ context editing（裁剪陈旧 tool 结果/thinking），配合 prompt cache 的确定性排序。
- **其它扩展**：Hooks（事件钩子）、Skills（按需渐进披露的领域指令）、MCP（外部工具/数据源）、文件式 Memory（`CLAUDE.md` + memory 目录跨会话持久化）、Adaptive thinking。

> 一句话：Claude Code ≈ **ReAct 主循环 + 专用工具闸门 + Plan 门控 + subagent 委派 + 上下文压缩** 的合集，正好把第一节那张"范式叠加"表落到了一个产品里。

### 对比

| | OpenCode | OpenClaw | Claude Code |
|--|--|--|--|
| 定位 | 编程专用 coding agent | 通用 agent 平台 | 官方终端编程 agent |
| 模型 | provider 无关（75+） | 多 provider | 仅 Claude |
| 核心范式 | Tool-use Loop + Plan/Build 双模式 | 以 ReAct 为骨架的 tool-use loop（subagent 按需叠加） | ReAct 骨架 + Plan 门控 + subagent 委派 |
| 特色 | provider 无关、LSP 集成、终端原生 | 插件生态、持久记忆、本地运行 | 专用工具闸门、Skills/Hooks/MCP、上下文压缩 |

---

## 四、Tool-use Loop vs ReAct

**ReAct** 是一种**提示范式**，强调把推理（Thought）显式写出来，和行动交织在一起。

**Tool-use Loop** 是一种**执行机制**，描述"调用工具 → 拿到结果 → 继续"这个循环本身，不强调推理是否显式。

```
Tool-use Loop（大概念）
    ├── ReAct 实现（Thought 显式输出）
    └── 隐式推理实现（模型内部思考，直接输出 tool call）
```

**ReAct 是 Tool-use Loop 的一种实现方式**，区别在于推理是否暴露出来。

现代 LLM 通过 Function Calling API 实现的循环，推理通常在模型内部，对外只暴露 tool call，更接近隐式的 Tool-use Loop。

---

## 五、`<thinking>` 的实现

### 两种不同的东西

**1. 扩展思考（Extended Thinking）— 模型层面**

Anthropic 在训练时赋予模型的原生能力，在给出最终回答前先生成一段草稿推理：
- 通过 API 的 `thinking` 参数开启
- thinking block 会出现在 `response.content` 里（与 `text` 块并列）；在多轮 tool use 场景中必须把它回传给模型，并非"不进上下文"
- 本质是让模型有更多 token 预算来"想清楚再说"

最新 Opus 4.8 / 4.7 只支持 **adaptive thinking**（由模型自行决定何时、想多深），固定 `budget_tokens` 已被移除——发 `{type:"enabled", budget_tokens:N}` 会返回 400。思考深度改用 `effort` 控制：

```python
response = client.messages.create(
    model="claude-opus-4-8",
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # low|medium|high|xhigh|max
    messages=[...]
)
# response.content 里会有 type="thinking" 和 type="text" 两块
# 注意：4.8/4.7 默认 display="omitted"（thinking 文本为空），
# 需 thinking={"type":"adaptive","display":"summarized"} 才能看到推理摘要
```

> 老模型（如 Sonnet 4.5）仍用旧写法 `thinking={"type":"enabled","budget_tokens":N}`（须 < `max_tokens`）。固定 token 预算这一概念整体上已被 adaptive thinking 取代。

**2. ReAct 的 Thought — 提示词层面**

通过 few-shot 示例让模型学会先写推理再写行动，是输出格式的约定，不是模型原生能力。

### 对比

| | Extended Thinking | ReAct Thought |
|--|--|--|
| 实现层 | 模型训练 + API 参数 | 提示词格式约定 |
| 推理深度 | 更深，可自我纠错 | 受限于单次输出 |
| 对上下文可见性 | 可选暴露给调用方 | 完全可见 |
| 出现时间 | Claude 3.7+ / o1 | 2022 年论文 |

**本质**：两者都是给模型"打草稿"的空间。Extended Thinking 是把这个能力内化到模型权重里，更自然、更强；ReAct Thought 是用提示词"骗"模型模拟这个行为，是工程 hack。

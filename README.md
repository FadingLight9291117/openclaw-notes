# OpenClaw 笔记

对 [OpenClaw](https://github.com/openclaw/openclaw) 代码库的架构与设计梳理笔记,
基于 2026.6.2 版本源码及根 `AGENTS.md`、`docs/` 整理而成。

## 目录

- [architecture-overview.md](./architecture-overview.md)
  整体架构概览:运行时拓扑(单 Gateway 控制平面)、核心/插件代码分层边界、
  一条消息从渠道到 agent 再到回复的完整生命周期。

- [skills-management-design.md](./skills-management-design.md)
  Skill 管理整体设计:`SKILL.md` 契约、六类来源的加载与合并管线、
  ClawHub 安装/校验生命周期、Skill Workshop 人审进化机制。

- [skill-invocation-mechanism.md](./skill-invocation-mechanism.md)
  Skill 触发机制:agent 如何判断是否使用以及使用哪个 skill —— 代码侧过滤可用集,
  模型侧基于 `description` 做语义匹配并按需 `read` 正文。

- [skills-cli-and-examples.md](./skills-cli-and-examples.md)
  Skill CLI 与真实样例:`openclaw skills` 命令族(list/info/check、
  search/install/update/verify、workshop 子命令)与内置 `SKILL.md` 的
  frontmatter 实例图谱。

- [agent-loop.md](./agent-loop.md)
  Agent loop 核心引擎：双重循环结构（steering 消息外层 + tool call 内层）、
  三个核心输入（AgentContext/AgentLoopConfig/AgentMessage[]）、
  全部 hook 一览（beforeToolCall/afterToolCall/steering/followUp 等）、
  AgentEvent 完整事件序列。

- [function-call-flow.md](./function-call-flow.md)
  Function call 完整流程:各 provider 原生格式差异（Anthropic/OpenAI/Google/Mistral）、
  内部归一化格式、从 ToolPlan 构建到 LLM 请求、流式解析、执行调度（并行/串行）、
  hook 拦截、结果写回 transcript 的端到端流程，以及 deferred tool 与 terminate 语义。

- [llm-autoregressive-kvcache.md](./llm-autoregressive-kvcache.md)
  LLM 自回归生成与 KV Cache：token 逐步生成的机制、prefill vs decode 阶段、
  首 token 慢的原因，以及服务端 Prompt Cache 的命中条件与 OpenClaw 中保证顺序确定性的做法。

完整官方文档:<https://docs.openclaw.ai>

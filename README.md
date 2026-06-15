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

- [mcp-overview.md](./mcp-overview.md)
  MCP（Model Context Protocol）概览:协议定位、OpenClaw 作为 MCP server
  与 client 的两个方向、MCP 与 tool use 的分层差异(协议层 vs 实现层、
  所有权与时机两根独立轴、plugin tool 反例)。

- [mcp-client-internals.md](./mcp-client-internals.md)
  OpenClaw 作为 MCP client 的内部实现:server 发现(用户 config + plugin
  manifest 合并)、连接时机(三种粒度的"启动"、attempt 引爆点)、
  模型视角下的扁平 tool 列表、catalog 缓存与失效、JSON Schema 兼容/
  content block 收敛/stdio env 过滤/失败退避等实现细节。

- [plugin-tools.md](./plugin-tools.md)
  Plugin tool:OpenClaw 插件给 agent 注册的工具——TypeScript 写、同进程跑、
  归插件作者拥有。`defineToolPlugin` 的 execute / factory 两种声明、
  manifest-first 发现、Plugin SDK 边界、跟 channel/provider plugin 的关系、
  跟 MCP tool 的关键对比,以及它在所有权 × 时机两根轴里"用户拥有 + 编译时确定"那一格的位置。

- [rag-overview.md](./rag-overview.md)
  RAG(检索增强生成)概念整理:它解决什么问题、跟 fine-tuning/超长 context/
  tool use 的对比、典型离线索引 + 在线检索两阶段流程,重点澄清两个常被混淆的问题
  ——向量化的结果是什么、拼进 prompt 的到底是向量还是原文(答案:数据库同时存
  向量和原文,向量只用来"找",原文才是发给 LLM 的内容);常见坑(检索失败、切块
  策略、embedding ≠ 相关性、评估),以及跟 MCP 的关系。

- [lsp-tools.md](./lsp-tools.md)
  LSP tool:先科普 Language Server Protocol 是什么(IDE 世界里 M × N 问题
  的标准解、跟 MCP 同源同构的 stdio + JSON-RPC 协议),再讲 OpenClaw 怎么
  把 hover/definition/references 三个 LSP 方法包装成 agent tool、跟 MCP 和
  plugin tool 平行的位置、以及当前只接受 plugin bundle 声明、用户 config
  还没接通的现状限制。

完整官方文档:<https://docs.openclaw.ai>

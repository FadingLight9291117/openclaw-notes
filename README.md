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

完整官方文档:<https://docs.openclaw.ai>

# Blog

面向读者的科普/解读文章,基于 [OpenClaw](https://github.com/openclaw/openclaw) 源码梳理。

## 目录

- [function-call-101.md](./function-call-101.md)
  Function Call 是什么、LLM 如何学会使用工具:从一个问题入手,
  解释模型不执行工具、只输出结构化调用请求的本质,
  并以 OpenClaw 源码为例说明工具声明、调用、执行的完整链路。

- [mcp-101.md](./mcp-101.md)
  MCP 是什么、为什么 Function Call 之后还需要它:用 LSP 解决编辑器 × 语言
  M × N 问题的类比引入,说清 MCP 跟 Function Call 是不同层(模型↔应用
  vs 应用↔工具),澄清三个常见误解,并用 OpenClaw 源码示意 MCP server
  怎么被发现、何时连接、模型如何看到扁平 tool 列表。

- [rag-101.md](./rag-101.md)
  RAG 是什么、怎么让 LLM 看着你的资料回答问题:从"AI 不知道公司内部数据"
  的真实问题出发,对比塞 prompt / fine-tuning / 自己搜 / RAG 四种解法,
  详细讲清三步骤(检索/增强/生成)和向量化的概念,澄清"向量塞 prompt"
  的常见误解,介绍切块/检索失败/embedding 不等于相关性/评估等工程坑,
  并对比 RAG 跟 MCP、Skill 的关系。

- [skills-101.md](./skills-101.md)
  Skill 是什么、怎么让 agent 学会"什么时候用什么工具":从"接了 100 个工具
  模型怎么选"的现实问题出发,讲清 progressive disclosure(prompt 里只放
  description 目录,正文按需 `read` 加载)、跟 RAG 的本质区别、"代码筛
  可用性 / 模型选用哪个"的分层、description 工程、Skill Workshop 人审进化、
  以及 Skill 跟 tool 是控制 vs 能力的关系。

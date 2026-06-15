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

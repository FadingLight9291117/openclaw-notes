# LSP Tool

基于 [OpenClaw](https://github.com/openclaw/openclaw) 2026.6.2 版本源码。

本文先讲 **LSP(Language Server Protocol)是什么**——它是 IDE 世界里
一个很重要的协议,但跟 AI 领域关系不大,所以单独介绍——再讲 OpenClaw
怎么把它包装成 agent 的 tool。

## LSP 是什么

### 它解决什么问题

写编辑器的人原来面临一个 M × N 问题:

```
  编辑器 (M 个)        语言支持 (N 种)
  ┌────────────┐      ┌──────────┐
  │ VSCode     │ ↔↔↔↔ │ Python   │
  │ Vim        │ ↔↔↔↔ │ Go       │
  │ Emacs      │ ↔↔↔↔ │ TypeScript│
  │ Sublime    │ ↔↔↔↔ │ Rust     │
  │ JetBrains  │ ↔↔↔↔ │ Java     │
  └────────────┘      └──────────┘
```

每个编辑器都得给每种语言写一遍 "hover 显示类型"、"跳转到定义"、
"查找所有引用"、"实时报错"...这些功能。M 个编辑器 × N 种语言 =
M × N 份重复实现。

LSP 是微软 2016 年提出的解法:**把"语言能力"抽出来变成一个独立的
服务器进程**,编辑器和服务器之间用一套标准协议通信。

```
  编辑器                    Language Server
  ┌────────┐    LSP        ┌──────────────┐
  │ VSCode │ ←─────JSON-RPC→│ tsserver     │  ← TypeScript
  └────────┘                └──────────────┘

  ┌────────┐                ┌──────────────┐
  │ Vim    │ ←──────────────│ gopls        │  ← Go
  └────────┘                └──────────────┘

  ┌────────┐                ┌──────────────┐
  │ Emacs  │ ←──────────────│ rust-analyzer│  ← Rust
  └────────┘                └──────────────┘
```

现在变成 M + N:每个编辑器实现一次 LSP client,每种语言实现一次
LSP server。VSCode 装 Rust 插件 = 它在 VSCode 端走 LSP client,
背后跑一个 `rust-analyzer` 进程。

### 协议形态

LSP 用 JSON-RPC 2.0,跑在 stdio 之上(也支持其他 transport,但 stdio
是默认)。消息分帧用 HTTP 风格的 Content-Length header:

```
Content-Length: 213\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"textDocument/hover","params":{...}}
```

典型方法名:

- `initialize` — 握手,client 告诉 server 自己支持什么,
  server 回 capabilities(支不支持 hover、definition 等)
- `textDocument/didOpen` / `didChange` / `didClose` — 通知文件状态
- `textDocument/hover` — 给我光标位置的类型/文档
- `textDocument/definition` — 跳转到定义
- `textDocument/references` — 找所有引用
- `textDocument/completion` — 自动补全
- `textDocument/publishDiagnostics` — server 主动推:这里有错

每个方法的参数和返回值都在 LSP 规范里固定下来,所以一个 LSP server
可以同时被 VSCode、Vim、Emacs 用。

### 跟 MCP 的对照

| 维度          | LSP                              | MCP                                  |
| ------------- | -------------------------------- | ------------------------------------ |
| 目标受众      | IDE / 编辑器                     | LLM 应用                             |
| 服务器干啥    | 提供代码分析(hover/跳转/补全)  | 提供任意工具(读文件/查搜索/...)    |
| 方法集合      | 固定一套 `textDocument/*` 等     | server 自己声明 `tools/list`         |
| 协议层        | JSON-RPC 2.0 + Content-Length    | JSON-RPC 2.0                         |
| 提出者        | 微软                             | Anthropic                            |
| 年份          | 2016                             | 2024                                 |

两个协议**长得很像**(stdio + JSON-RPC),解决的是不同领域的同一种
M × N 问题——把可复用的能力(代码分析 / agent 工具)抽到独立进程。
某种意义上 MCP 是 LSP 思路在 AI 时代的延续。

## OpenClaw 怎么用 LSP

核心实现 `src/agents/agent-bundle-lsp-runtime.ts`,第一行注释:

> Minimal LSP JSON-RPC framing over stdio (Content-Length header + JSON body).

不是用现成的 LSP client 库,而是自己在 OpenClaw 里实现了 LSP 协议的
最小子集(Content-Length 分帧 + JSON-RPC),spawn 语言服务器子进程,
跑 `initialize` 握手拿能力。

### 暴露的 tool

根据语言服务器 `initialize` 返回的 capabilities 动态生成
(`agent-bundle-lsp-runtime.ts:317` `buildLspTools`):

| Tool 名                    | LSP 方法                  | 用途              | capability 门槛       |
| -------------------------- | ------------------------- | ----------------- | --------------------- |
| `lsp_hover_<server>`       | `textDocument/hover`      | 看符号的类型/文档 | `hoverProvider`       |
| `lsp_definition_<server>`  | `textDocument/definition` | 跳到定义          | `definitionProvider`  |
| `lsp_references_<server>`  | `textDocument/references` | 找所有引用        | `referencesProvider`  |

如果语言服务器没声明对应 capability(比如 `hoverProvider: false`),
对应 tool 就不会被注册。

工具参数都长一样(`uri: string, line: number, character: number`),
因为 LSP 协议本身就是这套 `textDocument + position` 的形态。

**注意没有 completion**:agent 不需要交互式补全,它直接看完整文件就行。
OpenClaw 只挑了"看完整代码不能直接得到的语义信息"做工具——hover 拿
推断类型,definition/references 需要跨文件 AST 跳跃。

### 跟 MCP / plugin 平行

| 维度 | LSP tool | MCP tool | plugin tool |
| -------------- | ----------------------- | --------------------------- | --------------------------- |
| 协议            | LSP(`textDocument/*`)   | MCP(`tools/call`)           | OpenClaw Plugin SDK         |
| 进程            | stdio 子进程            | stdio 子进程 / HTTP         | 同进程                      |
| Tool 名         | hardcode 三个 + server 后缀 | server 自报 `tools/list`    | plugin 代码静态声明         |
| 配置来源        | bundle plugin manifest  | bundle plugin + 用户 config | plugin manifest             |
| 用户能加吗      | 现在不能                | 能(`openclaw mcp add`)    | 装新 plugin 即可            |
| Catalog 动态性  | **静态**(三个固定 tool) | **动态**(server 决定)     | 静态                        |
| 模型视角        | 跟其他 tool 没区别      | 跟其他 tool 没区别          | 跟其他 tool 没区别          |

跟 MCP 最大不同:**LSP 的 tool 集是 OpenClaw 写死的**(hover/definition
/references 三个),只是是否启用看 server capability。MCP 是 server
想暴露啥就 `tools/list` 出啥,完全运行时决定。

这是 LSP 协议本身决定的——LSP 的方法集就那么固定一套,不像 MCP
让 server 自由声明工具。OpenClaw 只是把 LSP 方法**翻译**成 agent
理解的 tool。

### 在 attempt 时序里

跟 MCP 完全平行(回看 [mcp-client-internals.md](./mcp-client-internals.md)
的 "Attempt 引爆点"):

```ts
// src/agents/embedded-agent-runner/run/attempt.ts:1502-1549
const bundleMcpRuntime = bundleMcpSessionRuntime
  ? await materializeBundleMcpToolsForRun({...})
  : undefined;
bundleLspRuntime = bundleLspEnabled
  ? await createBundleLspToolRuntime({...})     // ← LSP 在这里 spawn + initialize
  : undefined;

const allowedBundleMcpTools = applyEmbeddedAttemptToolsAllow(bundleMcpRuntime?.tools ?? [], ...);
const allowedBundleLspTools = applyEmbeddedAttemptToolsAllow(bundleLspRuntime?.tools ?? [], ...);
const allowedBundledTools = [...allowedBundleMcpTools, ...allowedBundleLspTools];
```

每次 attempt 开始时,两条 runtime 各自独立 spawn(MCP / LSP),
各自维护 session,各自 dispose;最后合到同一个扁平 tool list 给模型。

## 典型用例

模型要回答 "把 `runEmbeddedAttempt` 的所有调用方都列出来"。可能的工具组合:

1. `Grep "runEmbeddedAttempt"` ——快,但会被注释、字符串、相似名字误伤
2. `lsp_references_typescript` ——准,基于 AST 分析,但要先定位符号的
   精确 `(uri, line, character)`
3. 常见组合:先 `Grep` 找出大致位置 → `Read` 确认 → `lsp_references_typescript`
   拿精确引用列表

LSP tool 的价值在于 **精确度**——code intelligence 这件事 grep 替代不了:

- `Grep` 找的是文本匹配,会把字符串字面量、注释里的同名词都算进来
- `lsp_references` 走 AST + 类型系统,能区分 `foo.runEmbeddedAttempt`
  调的是哪个 `foo`,跨 import 跳得对

类似地 `lsp_hover` 拿 inferred type 是 grep / read 替代不了的——
TypeScript 的 `const x = foo()` 你想知道 `x` 是啥类型,只有 tsserver
能告诉你。

## 配置怎么写

跟 MCP server 一样,LSP server 从 plugin bundle manifest 读
(`src/plugins/bundle-lsp.ts`、`src/agents/embedded-agent-lsp.ts`),
plugin 在自己的 manifest 里声明 `lspServers`,字段形态跟 MCP 的
stdio server 几乎一样(`command`、`args`、`env`、`cwd`)。

OpenClaw 实际上**复用了 MCP 的 stdio launch config 结构**——
`agent-bundle-lsp-runtime.ts` 里直接 import 了 `resolveStdioMcpServerLaunchConfig`
和 `describeStdioMcpServerLaunchConfig`,因为两者都是 "spawn 一个本地
stdio 子进程跑 JSON-RPC",launch 部分可以共享。

## 现状的限制

`src/agents/embedded-agent-lsp.ts:22` 有一条注释:

> User-configured LSP servers could override bundle defaults here in the future.

意思是**当前只接受 plugin bundle 声明的 LSP server**,用户 config 还
没接通。MCP 已经走得更远(用户 config 可以加自己的 server),LSP 还
停在 plugin 预置的阶段。

退避逻辑也比 MCP 简化——没有 "连续 3 次失败冷却 60s" 这种保护,
失败就直接报错给 agent。

## 心智模型一句话

LSP 是把 "编辑器智能" 抽出来的协议,跟 MCP 是把 "agent 工具" 抽出来的
协议同源同构(stdio + JSON-RPC + 服务器自报能力)。OpenClaw 借了 LSP
这套现成的代码分析基础设施,把 hover/definition/references 三个最有
agent 价值的能力包装成 tool,放进 agent 的扁平 tool 列表里。模型层面
跟其他 tool 没任何区别。

## 相关

- [mcp-overview.md](./mcp-overview.md) — MCP 协议本身
- [mcp-client-internals.md](./mcp-client-internals.md) — MCP client
  的内部实现,LSP 在 attempt 时序里跟它平行
- [plugin-tools.md](./plugin-tools.md) — plugin tool,LSP server 的
  配置来源就是 plugin bundle

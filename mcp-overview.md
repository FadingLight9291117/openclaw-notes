# MCP（Model Context Protocol）概览

基于 [OpenClaw](https://github.com/openclaw/openclaw) 2026.6.2 版本源码。
本文讲 MCP 是什么、OpenClaw 怎么用它、它和 tool use 是什么关系。

OpenClaw 作为 MCP client 的内部实现(发现、连接、catalog、缓存等)
单独写在 [mcp-client-internals.md](./mcp-client-internals.md)。

## MCP 是什么

MCP 是一个开放协议,定义了 **LLM 应用(host)与外部工具/数据源(server)
之间的通信格式**。可以理解为"AI 应用的 USB-C 接口":一个 host 通过 MCP
可以同时接入文件系统、数据库、第三方 API 等任意 server,而不需要为每个
工具单独写 adapter。

通信用 JSON-RPC 2.0,MCP server 暴露三类能力:

- **tools** — 模型可以调用的函数(`tools/list`、`tools/call`)
- **resources** — 可读的数据源(`resources/list`、`resources/read`)
- **prompts** — 可被引用的预制 prompt 模板

## OpenClaw 里 MCP 的两个方向

OpenClaw 同时扮演 MCP server 和 MCP client(见 `docs/cli/mcp.md`):

```
                     ┌──────────────────────┐
                     │ External MCP client  │
                     │ (Codex / Claude Code)│
                     └─────────┬────────────┘
                               │ stdio JSON-RPC
                               ▼
            ┌──────────────────────────────────────┐
            │  openclaw mcp serve                  │
            │  (src/mcp/channel-server.ts)         │  ← OpenClaw as MCP server
            │   ChannelBridge → Gateway WS         │
            └──────────────────────────────────────┘

            ┌──────────────────────────────────────┐
            │  OpenClaw agent runtime              │
            │  (src/agents/agent-bundle-mcp-*.ts)  │  ← OpenClaw as MCP client
            └─────────┬────────────────────────────┘
                      │ stdio / SSE / streamable-http
            ┌─────────▼──────────┐ ┌──────────────┐
            │ third-party MCP    │ │ remote HTTP  │
            │ server (filesystem,│ │ MCP server   │
            │ memory, ...)       │ │ (OAuth)      │
            └────────────────────┘ └──────────────┘
```

### 方向 1:OpenClaw 当 server(`openclaw mcp serve`)

把 OpenClaw 的 channel 会话(Telegram/Discord/iMessage 等)通过 MCP
暴露给外部 client(Codex、Claude Code 等)。

入口在 `src/cli/mcp-cli.ts`,核心装配在 `src/mcp/channel-server.ts`:

```
McpServer(@modelcontextprotocol/sdk)
  + OpenClawChannelBridge (连到本地/远端 Gateway WebSocket)
  + registerChannelMcpTools(server, bridge)
```

`src/mcp/channel-tools.ts` 注册的 tool:

- `conversations_list` / `conversation_get`
- `messages_read` / `attachments_fetch`
- `events_poll` / `events_wait`
- `messages_send`
- `permissions_list_open` / `permissions_respond`

传输是 stdio:外部 client 启动 `openclaw mcp serve` 子进程,两边通过
stdin/stdout 跑 JSON-RPC。OpenClaw 内部再用 WebSocket 连到 Gateway 取
真实会话。

### 方向 2:OpenClaw 当 client

`openclaw mcp add/set/configure/probe/login/...` 把第三方 MCP server
保存到 `openclaw.json` 的 `mcp.servers` 下;OpenClaw 的 agent runtime
在跑的时候把这些 server spawn 起来,并把它们的 tool 合到模型可调用的
工具列表里。

这一侧的详细实现(server 怎么被发现、何时连接、catalog 怎么暴露给模型、
怎么缓存与失效、几个安全/兼容细节)单独写在
[mcp-client-internals.md](./mcp-client-internals.md)。

## MCP 与 tool use 的区别

这两个经常被混在一起,其实是 **不同层的概念**。

一句话:

- **Tool use(function calling)**:模型 ↔ 应用之间的**约定**——
  模型怎么"申请调用一个函数"
- **MCP**:应用 ↔ 工具实现之间的**协议**——
  这个函数从哪儿来、怎么真正执行

模型不知道 MCP 的存在,MCP server 也不知道是哪个模型在调它。

### 分层看

```
┌─────────────────────────────────────────┐
│  LLM (Claude / GPT / Gemini)            │
│  只认 tool_use / function_call          │ ← Tool Use 层
└─────────────────────┬───────────────────┘
                      │ provider API
┌─────────────────────▼───────────────────┐
│  OpenClaw agent runtime                 │
│  - 收集所有可用 tool                    │
│  - 打平成 provider 期望的格式           │
│  - 收到 tool_use 后分发执行             │
└──┬───────────────┬──────────────┬───────┘
   │               │              │
┌──▼──────┐  ┌─────▼─────┐  ┌─────▼──────┐
│ native  │  │ plugin    │  │ MCP server │  ← Tool 来源
│ Read/   │  │ tool      │  │ (stdio/HTTP)│
│ Bash... │  │           │  │             │
└─────────┘  └───────────┘  └─────────────┘
```

模型那一层永远是 tool use;MCP 只是 OpenClaw 拿到工具实现的一种方式。

所有 tool 最后都被归一化成同一个类型 `AnyAgentTool`
(`src/agents/tools/common.ts`):

- **Native tool**:代码里直接定义,例如 `src/agents/tools/read.ts`
- **MCP tool**:`buildBundleMcpToolsFromCatalog` 把 `Client.listTools()`
  返回的远端 catalog 包装成 `AnyAgentTool`,`execute` 里调
  `client.callTool(...)`

### 关键差异表

| 维度          | Tool use(原生 function calling) | MCP                                            |
| ------------- | -------------------------------- | ---------------------------------------------- |
| 谁定义        | 应用代码内联定义 schema          | 远端 MCP server 自己声明                       |
| 谁执行        | 应用代码同进程里调用             | 远端 server(子进程 stdio,或 HTTP)           |
| 发现方式      | 编译时静态注册                   | 运行时 `tools/list` 动态拉取                   |
| 跨语言        | 绑死宿主语言                     | JSON-RPC 任何语言都能写 server                 |
| 跨应用复用    | 每个应用各写一遍                 | 同一个 server 给 Claude Code、Codex、OpenClaw 都用 |
| 协议层        | 没有协议,就是个函数             | JSON-RPC 2.0 + 标准 schema                     |
| 能力          | 只有 function                    | tools / resources / prompts / notifications    |
| 生命周期      | 跟着进程                         | 独立进程,需要管 spawn/health/timeout/idle reclaim |

### 类比

- **Tool use** ≈ 编程语言里"函数调用"这个概念本身
- **MCP** ≈ gRPC / OpenAPI ——一种让函数能跨进程/跨语言被发现和调用的具体协议

或者:

- Tool use ≈ "我要打个电话"
- MCP ≈ "电话怎么拨通对方公司的总机,问到分机号,再接到具体的人"

### 实际后果

1. **不用 MCP 也能 tool use**:OpenClaw 的 `Read`、`Bash`、`Edit` 这些都是
   原生 tool,模型直接调,没 MCP 什么事。
2. **同一个 MCP server 给多个 host 复用**:`@modelcontextprotocol/server-filesystem`
   一次装好,Claude Code 和 OpenClaw 都能用;换成原生 tool 就得在每个
   host 里各实现一遍。
3. **MCP server 挂掉不影响其他 tool**:远端调用有网络/进程/超时故障模式,
   OpenClaw 用失败退避隔离(见 [mcp-client-internals.md](./mcp-client-internals.md)
   的 "几个实现细节")。
4. **MCP tool 对模型完全透明**:模型看到的就是 `filesystem__read_file(path)`,
   跟 native tool 长得一样,`__` 前缀只是为了避免重名,不是给模型暗示
   这是 MCP。

### 编译时 vs 运行时:典型用法不是协议本质

常见直觉是 "native tool use 是编译时静态、开发者固定;MCP 是运行时
动态、用户自定义"。方向对,但这是 **典型用法**,不是 **协议本质**。

实践中的分布确实如此:

| 维度        | Native tool use     | MCP tool                           |
| ----------- | ------------------- | ---------------------------------- |
| 谁定义      | 开发者写在代码里    | 用户在 config 里加                 |
| 何时确定    | 构建时 / 启动时     | 运行时 `tools/list` 拉来           |
| 变更成本    | 改代码、重发版      | 改 `openclaw.json` + `mcp reload`  |
| 列表稳定性  | 一个版本里固定      | session 内可变(MCP `listChanged`)|

但严格说要分两层:

- **Tool use(协议层)** 没规定 tool 必须是静态的。应用完全可以每次请求
  前动态生成 tool 列表(按用户权限、session 状态等)塞给模型。模型只看到
  一个数组,不在乎是 hardcode 还是即时拼的。
- **MCP** 协议本身**强制动态**:server 自己声明 `tools/list`,host 必须
  运行时去问——因为 tool 在另一个进程里,编译时无从知晓。

所以更准的说法:

- Tool use **允许**静态也**允许**动态,大多数应用选静态(因为简单)
- MCP **强制**动态(协议天然属性)

### 更本质的轴:所有权

真正的分水岭是 **谁拥有 tool 实现**:

- **Native tool** = 应用拥有 → 自然静态、自然由开发者掌控
- **MCP tool** = 第三方/用户拥有 → 必须动态发现、自然由用户配置

"编译时 vs 运行时"是这个所有权差异的**自然后果**,不是原因。

### 一个反例:plugin tool

OpenClaw 的 **plugin tool**(`extensions/*` 里那些)既不是 native 也不是
MCP,正好用来校准这两根轴:

- 像 native 一样:用 TypeScript 写,跟主进程同语言,**编译时**就在
- 像 MCP 一样:不在 core 里,由"插件作者"(广义的用户/第三方)拥有

它们走 `src/plugin-sdk/*`,跟 native tool 一样进 `AnyAgentTool`,跟 MCP tool
一起被打平。

这说明 **"动态/静态" 和 "开发者/用户" 是两个独立轴**,plugin tool 占了
"用户拥有 + 编译时确定" 那一格:

```
                  开发者拥有            用户/第三方拥有
              ┌──────────────────┬──────────────────┐
   编译时确定 │  native tool      │  plugin tool     │
              ├──────────────────┼──────────────────┤
   运行时发现 │  (动态生成的      │  MCP tool        │
              │   native tool)    │                  │
              └──────────────────┴──────────────────┘
```

## 心智模型一句话

Tool use 是模型层的调用约定;native / plugin / MCP 是这个约定下
**tool 实现的几种供应来源**,一部分由应用开发者编译时固化,一部分由
用户或第三方运行时供应。MCP 不替代 tool use,它给 tool use 加了一个
"工具供应链":模型继续按 tool use 协议出请求,应用按 MCP 协议去远端
把工具 *供应* 回来,放到模型看到的 tool 列表里。

## 继续阅读

- [mcp-client-internals.md](./mcp-client-internals.md) — OpenClaw 作为
  MCP client 的内部实现:server 发现、连接时机、catalog 暴露、缓存与
  失效、安全/兼容细节

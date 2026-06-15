# Plugin Tool

基于 [OpenClaw](https://github.com/openclaw/openclaw) 2026.6.2 版本源码。

本文讲 OpenClaw 的 **plugin tool**:跟 native tool、MCP tool 同列,
但归插件作者拥有、用 TypeScript 写、跑在 OpenClaw 同进程里的那一类工具。

## 定位

OpenClaw agent 看见的工具一共有三种来源(详见
[mcp-overview.md](./mcp-overview.md) 的 "MCP 与 tool use 的区别"):

| | 拥有者 | 在哪儿 | 进程 | 协议 |
| ------------- | ----------------- | -------------------------------- | -------------- | -------------------------- |
| **native tool** | OpenClaw core      | `src/agents/tools/*`             | 同进程         | 无                         |
| **plugin tool** | 插件作者          | `extensions/<name>/`(可第三方) | 同进程         | OpenClaw Plugin SDK        |
| **MCP tool**    | 第三方/用户       | 任何 MCP server                  | 子进程/远端    | MCP JSON-RPC               |

plugin tool 占的格子是:**用 TypeScript 写、走 OpenClaw 内部 SDK 接口、
不需要跑子进程**。介于 "core 写死的 native" 和 "完全解耦的 MCP" 之间,
专门用来做"深度集成 OpenClaw 内部能力,但又不属于 core" 的工具。

## 怎么写一个

入口 `src/plugin-sdk/tool-plugin.ts` 的 `defineToolPlugin`。最小例子
(`extensions/llm-task/index.ts`):

```ts
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";
import { Type } from "typebox";

export default defineToolPlugin({
  id: "llm-task",
  name: "LLM Task",
  description: "Generic JSON-only LLM tool for structured tasks.",
  configSchema: Type.Object({
    defaultProvider: Type.Optional(Type.String()),
    defaultModel:    Type.Optional(Type.String()),
    // ...
  }),
  tools: (tool) => [
    tool({
      name: "llm-task",
      description: "...",
      parameters: Type.Object({                       // TypeBox schema
        prompt: Type.String(),
        input:  Type.Optional(Type.Unknown()),
        // ...
      }),
      execute: async (params, config, context) => {   // 直接执行
        // 用 context.api 调用 host 服务(LLM 推理、存储等)
        // 返回 text 或 JSON
      },
      // 或者:
      // factory: ({ api, config, toolContext }) => createMyTool(api),
    }),
  ],
});
```

两种声明方式(`tool-plugin.ts` `ToolPluginToolDefinition` 的两个分支):

- **`execute`**:声明完就是个完整的 tool,框架直接调
- **`factory`**:返回一个或多个 `AnyAgentTool`,适合需要根据
  config / capability 动态构造工具的场景(`llm-task` 用的就是 factory)

`defineToolPlugin` 内部把每个 tool 注册到 plugin entry,在 plugin
activation 时跑 `register(api)`,把 tool 加进 agent 可用 tool 池
(`tool-plugin.ts:187-200`)。

## Manifest 让发现"不跑代码"

每个 plugin 同时有一份 `openclaw.plugin.json`
(`extensions/llm-task/openclaw.plugin.json`):

```json
{
  "id": "llm-task",
  "activation": { "onStartup": true },
  "configSchema": { "..." },
  "contracts": {
    "tools": ["llm-task"]
  },
  "toolMetadata": {
    "llm-task": { "optional": true }
  }
}
```

`contracts.tools` 把 plugin 提供的工具名列出来。这样 OpenClaw 在
**还没加载 plugin 代码**时就知道"装了这个 plugin 会多哪些工具"——
`openclaw plugins list`、setup、文档生成都不用 import 真实运行时。

这条规则在 `src/plugins/CLAUDE.md` 里写明:

> Preserve manifest-first behavior: discovery, config validation, and setup
> should work from metadata before plugin runtime executes.

`defineToolPlugin` 输出的对象上挂了一个 `[toolPluginMetadataSymbol]`,
包含同样的 metadata,这样 manifest 落地后可以做 contract 校验,
保证 manifest 跟代码不漂移(`tool-plugin.ts:109-116`)。

## Plugin SDK 边界

来自根 `AGENTS.md` 的硬规则:

> 插件 prod 代码:no core `src/**`, `src/plugin-sdk-internal/**`, other
> plugin `src/**`, or relative outside package.

插件只能通过 `openclaw/plugin-sdk/*` 跟 core 交互。`defineToolPlugin`
的 `register(api)` 拿到的 `OpenClawPluginApi` 就是 plugin 看得见的所有
host 能力(LLM 推理、存储、其他 plugin 句柄等),core 内部的具体实现
对 plugin 透明。

这条边界让 plugin tool 既能用上 host 内部能力(不像 MCP server 那样
完全隔离),又不会被 core 实现变更直接打死(只要 SDK 不破坏)。

## 仓库里有哪些 plugin tool

`grep "contracts.*tools" extensions/*/openclaw.plugin.json` 找到的:

| 插件 | 提供的能力 |
| ----------------- | ----------------------------------- |
| `browser`          | 浏览器自动化                        |
| `canvas`           | 画图                                |
| `diffs`            | 大文件 diff                         |
| `llm-task`         | "调一次 LLM 跑结构化任务"           |
| `memory-core`      | 知识/记忆存取                       |
| `memory-lancedb`   | 向量记忆                            |
| `memory-wiki`      | wiki 风格记忆                       |
| `tavily`           | Web 搜索                            |
| `firecrawl`        | Web 抓取                            |
| `feishu`           | 飞书集成                            |
| `qqbot`            | QQ 集成                             |
| `google-meet`     | Google Meet 集成                    |
| `file-transfer`    | 文件传输                            |
| `codex-supervisor` | Codex 任务编排                      |
| `lobster`          | (内部任务)                         |

`extensions/` 里**大多数其实不是 tool plugin**,而是另外两类——见下节。

## 它在更大的 plugin 生态里的位置

`extensions/` 里有三种主要 plugin,用不同的 `define*Plugin` 入口:

| 入口 | 用途 | 例子 |
| ------------------------------- | ---------------------- | ------------------------------ |
| `defineToolPlugin`              | 给 agent 注册工具      | `llm-task`, `tavily`, `browser` |
| `defineChannelPluginEntry`      | 接入消息渠道           | `telegram`, `discord`, `imessage` |
| `defineSingleProviderPluginEntry` | 接入 LLM provider    | `anthropic`, `openai`, `google` |
| `defineSetupPluginEntry`        | 安装/onboarding 流程   | 各 plugin 的 setup 部分        |

不同入口在 SDK 内部都收敛到 `definePluginEntry`(`src/plugin-sdk/plugin-entry.ts`)
拿到统一的 plugin lifecycle(activation、register、config)。

**channel / provider plugin 不直接给 agent 注册工具**——它们暴露的是
消息渠道或模型推理能力。tool plugin 是这个生态里**专门给 agent 加
function 的那一类**。

(理论上 channel plugin / provider plugin 也可以通过 `register(api)` 里
顺手注册 tool,但很少这么干;一旦这么做就既是 channel/provider plugin
又有 tool plugin 的属性。)

## 它怎么进到模型的 tool 列表

跟 MCP tool 走同一条路径(见
[mcp-client-internals.md](./mcp-client-internals.md) 的 "Attempt 引爆点"):

```
plugin 启动时(activation.onStartup 或按需)
  → tools 列表里每个 tool({...}) 被注册成 AnyAgentTool
  → 加进 agent 的可用 tool 池(在 host 进程内存里)

每次 attempt 开始
  → tools = [...native, ...plugin, ...bundleMcp, ...bundleLsp]
  → 走 provider adapter
  → 发给 LLM
```

模型看见的就是一个名字 `llm-task` 或者 `tavily_search`,没有任何
"这是 plugin"的标识。execute 时框架找到对应 tool,调它的 `execute`
函数,**这一切都在 OpenClaw 同一个 Node 进程里**,跟 MCP 走 stdio
子进程或 HTTP 完全不同。

## 跟 MCP tool 的关键对比

| 维度 | plugin tool | MCP tool |
| -------------- | ---------------------------------- | ---------------------------------- |
| 语言            | TypeScript(锁死 Node)              | 任意(JSON-RPC)                    |
| 进程            | 同进程                              | 子进程 / 远端                       |
| 启动开销        | 几乎 0(import 即可)               | spawn + 握手 + listTools            |
| 故障隔离        | 一个 plugin 崩了可能影响整个进程    | 子进程崩了不影响别人,有失败退避     |
| 访问 host 服务  | 通过 `OpenClawPluginApi`            | 完全隔离,什么都拿不到              |
| 跨应用复用      | OpenClaw 专属                       | 任何 MCP host 都能用                |
| 分发            | npm package(`@openclaw/plugin-*`)  | 任何二进制 / 服务                   |
| 适合做什么      | 深度集成 OpenClaw 内部能力的工具    | 通用、跨应用、可独立运行的工具      |

例子:`llm-task` 必须是 plugin tool,因为它要复用 OpenClaw 的
provider/auth/catalog 体系跑一次内嵌 LLM 推理;
`@modelcontextprotocol/server-filesystem` 适合做 MCP server,因为它
跟 OpenClaw 没耦合,Claude Code 也能用。

## 在"两根轴"里的位置

回顾 [mcp-overview.md](./mcp-overview.md) 的所有权 × 时机两根轴,
**plugin tool 占了"用户/第三方拥有 + 编译时确定"那一格**:

```
                  开发者拥有          用户/第三方拥有
              ┌──────────────────┬──────────────────┐
   编译时确定 │  native tool      │  plugin tool ★    │
              ├──────────────────┼──────────────────┤
   运行时发现 │  (动态 native)    │  MCP tool        │
              └──────────────────┴──────────────────┘
```

这个反例的价值在于:它证明了 "动态/静态" 跟 "开发者/用户" 是
**两个独立的轴**——不能简单说 "用户拥有的就一定是运行时发现"。
plugin tool 是用户拥有的(插件作者写、用户装),但因为还是 npm
package 同语言同进程,完全可以编译时确定。

## 相关

- [mcp-overview.md](./mcp-overview.md) — MCP 概览、与 tool use 的关系
- [mcp-client-internals.md](./mcp-client-internals.md) — MCP client
  实现细节(plugin tool 跟 MCP tool 在 attempt 里的拼装)

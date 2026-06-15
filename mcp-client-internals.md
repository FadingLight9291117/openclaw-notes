# OpenClaw 作为 MCP client:内部实现

基于 [OpenClaw](https://github.com/openclaw/openclaw) 2026.6.2 版本源码。

本文专讲 OpenClaw 作为 MCP client 那一侧的实现:server 怎么被发现、
何时连接、tool catalog 怎么暴露给模型、怎么缓存与失效、有哪些值得注意的
安全/兼容细节。

MCP 是什么、和 tool use 的关系、`openclaw mcp serve` 那一侧的实现见
[mcp-overview.md](./mcp-overview.md)。

## 整体生命周期

```
[发现]            读 config + plugin manifest → 一份 server map
                      (进程启动 / session 创建时纯 JSON 读,不 spawn)
   ↓
[连接]            attempt 开始触发 getCatalog
                      → resolveMcpTransport
                      → connectWithTimeout (stdio spawn / http 握手)
                      → client.listTools()
   ↓
[暴露]            buildBundleMcpToolsFromCatalog
                      → MCP tool 包装成 AnyAgentTool(name 加 server 前缀)
                      → 跟 native / plugin / LSP tool 合到同一个数组
                      → 发给 LLM
   ↓
[调用]            模型返回 tool_use → 找到 AnyAgentTool
                      → execute → client.callTool()
                      → 走早就建立好的 transport
   ↓
[缓存与失效]      同 session 的后续 attempt 复用 catalog;失效条件:
                      idle TTL / listChanged / 连续失败冷却
```

## MCP server 的发现

简短答案:**没有"扫描发现",全靠声明式 config**。OpenClaw 从两个来源
合并出 MCP server 列表。

### 两个来源

**1. 用户 config:`openclaw.json` 的 `mcp.servers`**

类型定义在 `src/config/types.mcp.ts`:

```ts
export type McpConfig = {
  servers?: Record<string, McpServerConfig>;
  sessionIdleTtlMs?: number;
};
```

用户通过 CLI(`openclaw mcp add/set/configure/...`)或直接编辑文件,
往 `~/.openclaw/openclaw.json` 里写入条目。CLI 实现在
`src/cli/mcp-cli.ts`,读写在 `src/config/mcp-config.ts`。

**2. 已启用的 bundled plugin 自带的 MCP 定义**

每个 plugin 在自己的 manifest 里可以声明附带的 MCP server,加载逻辑
在 `src/plugins/bundle-mcp.ts`:

- `loadEnabledBundleMcpConfig` 遍历 `manifestRegistry` 里所有
  **enabled** 的 plugin
- 对每个 plugin 调 `loadBundleMcpConfig`:
  1. 读 plugin 的 manifest(`claude` / `codex` / `cursor` 三种格式之一)
  2. 解析其中 `mcpServers` 字段声明的相对路径列表
  3. 如果 plugin 根目录存在 `.mcp.json` 也自动加入
  4. 把这些文件里的 server 定义和 manifest 里的 inline `mcpServers`
     merge 起来
- 路径里的 `${CLAUDE_PLUGIN_ROOT}` 占位符会被替换成 plugin 根目录

### 合并优先级

两个来源在 `src/agents/bundle-mcp-config.ts` 的 `loadMergedBundleMcpConfig`
里合并:

```ts
mcpServers: {
  ...enabledBundleMcp,        // plugin 自带的(底)
  ...enabledConfiguredMcp,    // 用户 config 的(覆盖在上)
}
```

规则:

1. **同名 server**,用户 config 覆盖 plugin 默认(注释里明确说
   "OpenClaw config 是 owner-managed layer")
2. 用户 config 里 `enabled: false` 不仅排除自己,还能**屏蔽掉
   同名的 bundled server**(`disabledConfiguredNames` 集合)
3. 一次性的 embedded agent run 通过 `loadEmbeddedAgentMcpConfig`
   (`src/agents/embedded-agent-mcp.ts`)拿到最终 merged 结果

### 发现链路

```
启动 / 每次 session
  │
  ├─ getRuntimeConfig()                ← 读 ~/.openclaw/openclaw.json
  │     └─ cfg.mcp.servers             ← 用户层
  │
  ├─ PluginManifestRegistry.plugins    ← 已发现的 plugin manifest
  │     └─ 每个 plugin 的 .mcp.json /
  │        manifest mcpServers         ← plugin 层
  │
  └─ loadMergedBundleMcpConfig({ cfg, manifestRegistry, workspaceDir })
        ↓
     { mcpServers: { name → config } } ← 最终 server 列表(还没 spawn)
```

### 发现性质

1. **完全声明式**:OpenClaw 不去网络扫描、不去 ServiceDiscovery、
   不去 mDNS——所有 server 必须显式写在某个 config 文件里
2. **plugin 是隐式来源**:用户装个 plugin,可能就"被"得到几个 MCP
   server,这是 plugin 作者预置的
3. **Config 是 process-stable**:根 `AGENTS.md` 的架构约定——
   MCP server 列表在进程启动后视为稳定,改了 config 要
   `openclaw mcp reload` 或重启 agent
4. **server 列表的"发现"和 tool 的"发现"是两层**:
   - server 列表:**静态**(读 config + plugin manifest)
   - 每个 server 的 tool:**动态**(运行时 `tools/list`,还支持
     `listChanged` 通知)

所以严格说,OpenClaw 作为 MCP client 对 **server** 是静态发现,
对 **tool** 是动态发现。

## 何时连接

关键澄清:**"懒"是相对"OpenClaw 启动"懒,不是相对"模型调用"懒**。
每次 agent attempt(模型一轮请求)开始时,MCP server 会**先全部
spawn + listTools**,然后 tool catalog 才被塞进发给模型的 tool 列表里。

### 三种粒度的"启动"

| 粒度                          | 何时发生                | 状态                                                  |
| ----------------------------- | ----------------------- | ----------------------------------------------------- |
| OpenClaw 进程启动             | `openclaw` 命令开始跑   | **不连接任何 MCP**                                    |
| Session 创建                  | 用户第一次说话          | **不连接**(只准备 runtime 壳子)                     |
| Attempt 开始(每轮模型请求)  | 准备给模型发请求        | **必须连接 + listTools**(否则模型不知道有哪些 tool) |
| Tool 实际调用                 | 模型返回 tool_use       | **走已建立的连接**                                    |

"懒到模型调用时再连接"是不行的——模型必须在请求里就看到全部 tool
才能决定调哪个。OpenClaw 的折衷点在 "attempt 开始" 这一步:对每个会话,
只在它真有 attempt 时才连(避免无对话的会话开销),但一旦要 attempt,
必须先把 catalog 凑齐。

### Attempt 引爆点

`src/agents/embedded-agent-runner/run/attempt.ts:1502` 这段是关键——
**模型还没被调用前**,这段已经把 MCP 全部 spawn 完了:

```ts
const bundleMcpSessionRuntime = bundleMcpEnabled
  ? await getOrCreateSessionMcpRuntime({...})    // 拿 runtime(可能复用)
  : undefined;
bundleMcpRuntime = bundleMcpSessionRuntime
  ? await materializeBundleMcpToolsForRun({      // ← 这里触发 getCatalog
      runtime: bundleMcpSessionRuntime,
      reservedToolNames: [...],
    })
  : undefined;
```

紧接着 `attempt.ts:1535` 把 MCP tool 合到 `allowedBundledTools`,后面跟
native tool 一起发给模型。

### 完整时间线

```
进程启动      → 读 config,记下"理论上有哪些 server",不 spawn
session 创建  → runtime 壳子,sessions=空,catalog=null
attempt 开始  → getCatalog():
                 ├─ for 每个 server:
                 │   - resolveMcpTransport
                 │   - connectWithTimeout (stdio spawn / http 握手)
                 │   - client.listTools()
                 └─ 拼成 catalog
                buildBundleMcpToolsFromCatalog():
                  把每个 MCP tool 包装成 AnyAgentTool
                  (name = "serverName__toolName")
组装 LLM 请求 → tools = [...native, ...plugin, ...mcp]
LLM 收到请求  → 看到 N 个 function 定义,只看 name/description/schema
LLM 返回      → tool_use: "filesystem__read_file"
执行          → 在 tool 列表里找到对应 AnyAgentTool,
                调它的 execute → runtime.callTool() → 已建立的 transport
```

## 模型视角:一个扁平的 tool 列表

**模型不知道"MCP"这个概念**。它只看到一个 tool 数组,每个 tool 形如:

```json
{
  "name": "filesystem__read_file",
  "description": "Read contents of a file. Returns text content...",
  "input_schema": { "type": "object", "properties": { "path": {...} } }
}
```

跟 `Read`、`Bash` 这种 native tool 在请求里**完全是同一形状**。区别仅在:

- **来源不同**:native 是代码里写的,MCP 是从 `client.listTools()` 拉来的
- **名字有前缀**:`buildSafeToolName` 给 MCP tool 加 `serverName__` 前缀
  避免重名,native tool 没这个前缀
- **execute 走向不同**:native 在本进程跑,MCP 走 `client.callTool()`
  到子进程/HTTP

`attempt.ts:1535-1549` 那段把所有来源 concat 起来:

```ts
const allowedBundleMcpTools = applyEmbeddedAttemptToolsAllow(
  bundleMcpRuntime?.tools ?? [], ...
);
const allowedBundleLspTools = applyEmbeddedAttemptToolsAllow(
  bundleLspRuntime?.tools ?? [], ...
);
const allowedBundledTools = [...allowedBundleMcpTools, ...allowedBundleLspTools];
```

native tool、plugin tool、MCP tool、LSP tool 全部以 `AnyAgentTool[]`
形式 concat,经过 provider adapter(Anthropic / OpenAI / Google)翻译
成各自的 tool schema,塞进同一个请求的 `tools` 字段。

模型挑哪个工具是**纯粹基于 `description` 和 schema 跟用户意图的语义
匹配**——跟它怎么挑 `Read` 还是 `Bash` 同一个机制。

### 扁平化的几个微妙后果

1. **模型可能"搞错来源"**——装了一个 `memory__write` 的 MCP 工具,
   模型在该用 native `Write` 写文件时可能调了 `memory__write`,只因
   后者 description 更像当下意图。这就是为啥 MCP server 的 `description`
   写得好不好直接影响 agent 效果。

2. **名字前缀是唯一的"出身标签"**——`buildSafeToolName` 加 `serverName__`
   前缀(`agent-bundle-mcp-materialize.ts`),既防重名,也无意中给
   模型一点"分组信号"。但模型不会被告知 `__` 前面是 server 名,它只是
   把整个名字当字符串看。

3. **工具排序影响行为**——`agent-bundle-mcp-materialize.ts` 显式
   `tools.sort(...)` 是为了让同一份 catalog 在不同 turn 里给模型呈现的
   顺序一致(对 prompt cache 命中重要,见根 `AGENTS.md` 里
   "deterministic ordering" 那条)。不排序时顺序漂移会让 prompt
   字节级别变化,KV cache 失效。

4. **tool filter 是预算工具**——`toolFilter.include/exclude` 在 catalog
   阶段就过滤掉,模型完全感知不到"被屏蔽"的工具存在。这是抑制模型
   乱调 MCP 工具的主要手段。

一句话:**对 LLM 来说,世界是扁平的一组 function;MCP 是 OpenClaw 这
一层的实现细节,跨过 provider boundary 之后就消失了。**

## Catalog 缓存与失效

`agent-bundle-mcp-runtime.ts` 的 `getCatalog` 有缓存——同一个 session 的
后续 attempt **不再重新 listTools**,直接复用 catalog。

失效条件:

- session 空闲超过 `mcp.sessionIdleTtlMs`(默认 10 分钟)被回收,
  下次 attempt 时 catalog 重新拼
- MCP server 发了 `listChanged` 通知,`onChanged` callback 里
  `catalog = null` 强制下次刷新
- 某个 server 连续 3 次失败,进入 60s 冷却期(后面 "Failure 退避" 段)

这就是"server 列表静态发现 + tool 列表动态发现 + tool catalog 缓存"
三层的实际落地。

## 几个值得注意的实现细节

**JSON Schema 兼容**(`agent-bundle-mcp-runtime.ts` `createBundleMcpJsonSchemaValidator`):
MCP 用 JSON Schema draft 2020-12,而 TypeBox/Ajv 对一些 keyword 支持
不一致,OpenClaw 用 `stripJsonSchemaFormats` + `normalizeJsonSchemaForTypeBox`
做 schema 归一化后再交给 TypeBox 编译。

**Content block 收敛**(`agent-bundle-mcp-materialize.ts` `mcpContentBlockToToolResult`):
MCP 的 `CallToolResult` 可以返回 text/image/audio/resource_link/resource 等
多种 block,但 OpenClaw 的 `AgentToolResult` 只支持 text/image。多余类型
被降级成 text,避免 provider(Anthropic)拒掉无效 image block 后整个
session 历史被污染(注释里点了 issue #90710)。

**Stdio env 安全过滤**(`docs/cli/mcp.md` "Stdio env safety filter"):
`NODE_OPTIONS`、`PYTHONSTARTUP`、`LD_PRELOAD` 这类能在 RPC 之前改变
解释器行为的 env 被禁止写入 stdio server 的 `env` 字段。

**Failure 退避**(`agent-bundle-mcp-runtime.ts` 常量 `BUNDLE_MCP_FAILURE_THRESHOLD`):
同一 server 连续 3 次失败会触发 60s 冷却,防止一个坏 server 阻塞整个
agent turn。

**Tool filter**:每个 server 可配 `toolFilter.include/exclude`(支持 `*` glob),
catalog 阶段就把不想暴露给模型的工具过滤掉。

## 相关

- [mcp-overview.md](./mcp-overview.md) — MCP 协议本身、OpenClaw 的两个
  方向、MCP 与 tool use 的对比
- `docs/cli/mcp.md` — 官方面向用户的 CLI 文档

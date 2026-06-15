# OpenClaw Function Call 完整流程

> 本文基于 2026.6.2 版本源码分析，关键文件：
> `packages/agent-core/src/agent-loop.ts`、`src/tools/planner.ts`、
> `src/tools/types.ts`、`src/llm/providers/anthropic.ts`、
> `src/llm/providers/openai-completions.ts`、`src/llm/providers/openai-responses-tools.ts`、
> `src/llm/providers/google-shared.ts`。

---

## Tool 如何传递给 Provider

### 传递路径

```
AgentTool（注册在 AgentContext.tools）
       │
       │  streamAssistantResponse()  packages/agent-core/src/agent-loop.ts:416
       ▼
Context.tools: Tool[]          ← llm-core 层通用接口
  { name, description, parameters }   （TypeBox / JSON Schema）
       │
       │  每个 provider 的 convertTools()
       ▼
各 Provider 原生格式（见下节）
       │
       │  放入请求 body（params.tools）
       ▼
LLM API
```

`AgentTool` 继承自 `Tool`，provider 只使用三个字段：`name` / `description` / `parameters`。

### 各 Provider 的 `convertTools()` 差异

**Anthropic**（`src/llm/providers/anthropic.ts:1480`）
```ts
{
  name: tool.wireName,           // OAuth 模式下做名称转换
  description: tool.description,
  input_schema: tool.inputSchema,   // 字段名是 input_schema
  cache_control: ...,               // 最后一个 tool 加 cache_control（prompt cache）
  eager_input_streaming: true,      // 部分模型支持，提前流式返回 arguments
}
```

**OpenAI Completions**（`src/llm/providers/openai-completions.ts:1174`）
```ts
{
  type: "function",
  function: {
    name: tool.name,
    description: tool.description,
    parameters: tool.parameters,   // 字段名是 parameters
    strict: false,                 // 可选，部分 provider 不接受此字段
  }
}
```

**OpenAI Responses API**（`src/llm/providers/openai-responses-tools.ts:60`）

比 Completions 多两步：按 `name` 排序（保证 prompt cache 字节确定性）+ `normalizeOpenAIStrictToolParameters()` 处理 strict 模式兼容性。
```ts
{
  type: "function",
  name: tool.name,
  description: tool.description,
  parameters: normalizeOpenAIStrictToolParameters(...),
  strict: true | false | null,
}
```

**Google**（`src/llm/providers/google-shared.ts:367`）

格式差异最大：tool 列表包在 `functionDeclarations` 数组里再套一层对象；有新旧两个 schema 字段，`useParameters` 参数控制选哪个（Cloud Code Assist 等场景需要旧字段）。
```ts
[{
  functionDeclarations: tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    parametersJsonSchema: tool.parameters,   // 新字段，完整 JSON Schema
    // 或 parameters: sanitizeForOpenApi(...)  // 旧字段，OpenAPI 3.0 格式
  }))
}]
```

### 值得注意的细节

**Prompt cache 排序：** OpenAI Responses 和 Anthropic 都要求 tool 列表顺序固定才能命中 prompt cache。Responses API 在 `convertResponsesToolPayload()` 里显式按 name 排序；`src/tools/planner.ts` 的 `buildToolPlan()` 也用 `sortKey ?? name` 保证 planner 输出有序。

**Tool name projection：** Anthropic OAuth 模式下，内部 tool 名称用 `toClaudeCodeName()` 转成 wire name，响应里再用 `resolveOriginalAnthropicToolName()` 反查回来，避免与 Claude.ai 原生工具名冲突。OpenAI 侧同样有 `projectOpenAITools()` 做类似映射。

---

## 各 Provider 原生格式对比

### Anthropic

**发出（assistant message）：**
```json
{ "type": "tool_use", "id": "toolu_xxx", "name": "tool_name", "input": { ... } }
```
**结果（tool result）：**
```json
{ "type": "tool_result", "tool_use_id": "toolu_xxx", "content": [...], "is_error": false }
```
**流式：** `content_block_start`（`tool_use` 类型）+ `content_block_delta`（`input_json_delta`，流式 JSON 片段），arguments 需要边收边拼接解析。

### OpenAI Completions

**发出：**
```json
{ "type": "function", "function": { "name": "tool_name", "arguments": "{\"key\":\"val\"}" } }
```
`arguments` 是**字符串**，不是对象。

**结果：** 消息角色为 `tool`，带 `tool_call_id`：
```json
{ "role": "tool", "tool_call_id": "call_xxx", "content": "..." }
```
**流式：** `delta.tool_calls[]` 数组，按 `index` 对齐，`function.arguments` 是流式字符串片段。

### Google Generative AI

**发出：**
```json
{ "functionCall": { "name": "tool_name", "args": { ... } } }
```
`args` 是对象（非字符串）；工具声明用 `function_declarations`。

**结果：**
```json
{ "functionResponse": { "name": "tool_name", "response": { ... } } }
```
注意：Google 不保证 tool call 有稳定 id，代码会在缺失时自动生成（`needsNewId` 逻辑）。

### Mistral

格式最接近 OpenAI Completions，差异：
- 字段名为 camelCase：`delta.toolCalls`（非 `tool_calls`）
- id 可能是字符串 `"null"` 而非真 null，代码有专门判断
- `arguments` 可能已经是对象，代码做 `typeof === "string"` 判断后再决定是否 parse

### 内部统一格式

所有 provider adapter 最终都归一化为：
```ts
{ type: "toolCall", id: string, name: string, arguments: Record<string, unknown> }
```
adapter 的主要工作：把流式碎片 JSON 字符串拼好、把各家字段（`input`/`args`/`arguments`）统一解析成对象、补全缺失 id。

---

## 完整执行流程

```
用户消息
   │
   ▼
① 构建 tool 列表（ToolPlan）
   src/tools/planner.ts → buildToolPlan()
   根据当前 auth/config/插件状态过滤出 visible tools
   → toToolProtocolDescriptors() 转成各 provider 格式
   │
   ▼
② LLM 流式请求
   packages/agent-core/src/agent-loop.ts → streamAssistantResponse()
   把 messages + tools 发给 provider
   │
   ▼
③ Provider 层解析流式响应（各家格式不同，见上节）
   全部归一化为内部 { type: "toolCall", id, name, arguments } 格式
   │
   ▼
④ agent-loop 检测 tool call
   message.content.filter(c => c.type === "toolCall")
   有 toolCall → 进入执行；没有 → 结束本轮
   │
   ▼
⑤ resolveToolCallTool()
   先在 context.tools 里按 name 查找
   找不到 → 调 config.resolveDeferredTool()（动态加载）
   仍找不到 → 返回 "Tool not found" error result
   │
   ▼
⑥ prepareToolCall()
   a. prepareArguments()      —— tool 可预处理入参
   b. validateToolArguments() —— 按 JSON Schema 校验
   c. beforeToolCall() hook   —— 可返回 { block: true } 拦截执行
   │
   ▼
⑦ 执行模式判断
   ┌─ 串行（sequential）：tool 标记了 executionMode="sequential"，或全局配置强制
   └─ 并行（parallel）：默认，批次内所有 tool call 同时 Promise.all
   │
   ▼
⑧ executePreparedToolCall()
   调 tool.execute(id, args, signal, onPartialResult)
   执行中可发 tool_execution_update 事件（流式进度回调）
   │
   ▼
⑨ finalizeExecutedToolCall()
   afterToolCall() hook —— 可覆盖 content / isError / details
   可返回 stopAfterBatch: true 让 loop 在这批后停止
   │
   ▼
⑩ 结果写回 transcript
   createToolResultMessage()
   → { role: "toolResult", toolCallId, content, isError }
   push 到 context.messages
   │
   ▼
⑪ 下一轮 LLM 请求（回到步骤②）
   带着 tool results 再请求，直到：
   - 没有新的 toolCall（模型给出最终回复）
   - terminate = true（所有 tool 都返回 terminate: true）
   - AbortSignal 触发
   - stopAfterBatch 标记
```

---

## 关键设计点

**双重循环**
外层等待用户 steering 消息，内层处理 tool call batch。模型可以一次返回多个 tool call，执行完全部后再请求一次 LLM。

**Deferred tool**
不在初始 tool 列表里的 tool 可在执行时动态 resolve（`resolveDeferredTool`），用于权限控制或懒加载场景。recover 后会追加到 `currentContext.tools`，让本轮后续 provider continuation 可见。

**terminate 语义**
tool result 可带 `terminate: true`。当批次内所有 call 都 terminate 时 loop 停止，不再请求 LLM。只要有一个 call 不 terminate，loop 继续。

**串行降级**
并行模式下只要发现任意一个 tool 的 `executionMode === "sequential"`，整批立即降为串行执行。

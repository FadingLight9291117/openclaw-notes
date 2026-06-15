# Function Call 是什么？LLM 如何学会使用工具

> 本文以 [OpenClaw](https://github.com/openclaw/openclaw) 开源源码为例（基于 2026.6.2 版本），结合实际代码说明 Function Call 的工作原理。涉及源码路径均可在仓库中直接查阅。

如果你用过 ChatGPT 的联网功能，或者让 AI 帮你查股票、操作文件，你已经在使用 Function Call 的成果了。但它背后是怎么运作的？模型是怎么"调用"工具的？本文从头说清楚。

---

## 从一个问题开始

> "今天北京天气怎么样？"

把这个问题丢给 LLM，它大概率会回答：

> "抱歉，我的训练数据有截止日期，无法获取实时天气信息。"

这暴露了 LLM 的本质局限：**它只能生成文本，没有任何与外部世界交互的能力**。它不能查数据库、不能调 API、不能执行代码——所有"知识"都封存在训练时的权重里。

Function Call（也叫 Tool Use）就是为了解决这个问题而生的。

---

## Function Call 是什么

一句话：**让开发者提前告诉模型"有哪些工具可以用"，模型在需要时决定调用哪个工具、传什么参数，再由代码来真正执行它。**

有一个关键细节值得提前说清楚：**模型本身不执行任何工具**。它只是输出一段结构化的"调用请求"，真正的执行发生在你的代码里。这个区别很重要，后面会反复提到。

---

## 一个完整的例子

还是那个天气问题，看看有了 Function Call 之后发生了什么。

**第一步：开发者注册工具**

在调用 LLM 之前，把可用的工具告诉模型：

```json
{
  "name": "get_weather",
  "description": "获取指定城市的实时天气",
  "parameters": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "城市名称" }
    },
    "required": ["location"]
  }
}
```

**第二步：用户提问，模型决定调用工具**

模型收到"北京今天天气怎么样"后，意识到自己没有实时数据，但有 `get_weather` 工具可以用。于是它不直接回答，而是输出一个调用请求：

```json
{
  "type": "tool_use",
  "name": "get_weather",
  "input": { "location": "北京" }
}
```

**第三步：代码执行工具，结果还给模型**

你的程序接收到这个请求，真正去调天气 API，拿到结果后告诉模型：

```json
{
  "type": "tool_result",
  "content": "北京今天晴，气温 28℃，东南风 3 级"
}
```

**第四步：模型拿到结果，给出最终回答**

> "北京今天天气不错，晴天，气温 28℃，东南风 3 级，适合出门。"

整个过程对用户是透明的，感觉就像模型"知道"天气一样。

---

## 工作原理

理解了"是什么"，再看"怎么做到的"。

### 工具的定义与注册

在 OpenClaw 里，一个工具用 TypeBox schema 描述参数，并提供一个 `execute` 函数负责实际执行（`packages/agent-core/src/types.ts`）：

```ts
const weatherTool: AgentTool = {
  name: "get_weather",
  label: "获取天气",
  description: "获取指定城市的实时天气信息",
  parameters: Type.Object({
    location: Type.String({ description: "城市名称，如「北京」" }),
  }),
  execute: async (toolCallId, args) => {
    const data = await fetchWeatherApi(args.location);
    return { content: [{ type: "text", text: data }] };
  },
};
```

三个核心字段：`name` 是模型调用时用的名字，`description` 是**模型决定"要不要用它"的唯一依据**，`parameters` 是模型生成参数时要遵守的 JSON Schema。

工具注册到 `AgentContext.tools` 后，每次请求 LLM 前经过 provider 的 `convertTools()` 转换，写入请求 body 的 `tools` 字段。

### 各家 API 的格式差异

不同厂商的格式不统一，是接入多个 provider 时最头疼的地方：

| | 工具声明字段 | 调用响应格式 | 结果角色 |
|---|---|---|---|
| **Anthropic** | `input_schema` | `{ type: "tool_use", input: {...} }` | `tool_result`（role: user） |
| **OpenAI** | `parameters` | `{ function: { arguments: "字符串" } }` | role: `tool` |
| **Google** | `parametersJsonSchema` | `{ functionCall: { args: {...} } }` | `functionResponse` |
| **Mistral** | `parameters` | 同 OpenAI，但字段名 camelCase | role: `tool` |

几个容易踩的坑：

- **OpenAI 的 `arguments` 是字符串**，不是对象，需要 `JSON.parse()` 才能用
- **Anthropic 的工具结果 role 是 `user`**，不是独立角色，多个工具结果会合并进同一条 user 消息
- **Google 不保证 tool call 有稳定 id**，需要在缺失时自动生成

### 完整请求体长什么样

把上面这些拼在一起，实际发给 Anthropic 的请求体是这样的（`src/llm/providers/anthropic.ts`）：

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 8096,
  "stream": true,
  "system": [
    {
      "type": "text",
      "text": "You are a helpful assistant...",
      "cache_control": { "type": "ephemeral" }
    }
  ],
  "messages": [
    { "role": "user", "content": "北京今天天气怎么样？" }
  ],
  "tools": [
    {
      "name": "get_weather",
      "description": "获取指定城市的实时天气信息",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": { "type": "string", "description": "城市名称" }
        },
        "required": ["location"]
      },
      "cache_control": { "type": "ephemeral" }
    }
  ]
}
```

注意 `cache_control` 只加在 tools 数组的**最后一个工具**上——这是 Anthropic 的 prompt cache 标记方式，缓存整个 tools 块，下次相同前缀的请求直接命中。

### Provider 返回体与 transcript 拼接

请求发出后，Anthropic 以 SSE 流式返回。一次含 tool call 的响应，事件序列大致如下：

```
// 1. 消息开始，返回 token 用量（含 prompt cache 命中数）
event: message_start
data: { "message": { "id": "msg_01xxx", "usage": { "input_tokens": 312, "cache_read_input_tokens": 280 } } }

// 2. 文本块流式输出
event: content_block_start  → data: { "index": 0, "content_block": { "type": "text" } }
event: content_block_delta  → data: { "index": 0, "delta": { "type": "text_delta", "text": "我来查一下。" } }
event: content_block_stop   → data: { "index": 0 }

// 3. tool_use 块开始（携带 id 和 name，input 为空）
event: content_block_start
data: { "index": 1, "content_block": { "type": "tool_use", "id": "toolu_01xxx", "name": "get_weather", "input": {} } }

// 4. 参数以 JSON 字符串碎片流式输出，需要本地拼接
event: content_block_delta  → data: { "index": 1, "delta": { "type": "input_json_delta", "partial_json": "{\"loc" } }
event: content_block_delta  → data: { "index": 1, "delta": { "type": "input_json_delta", "partial_json": "ation\": \"北京\"}" } }
event: content_block_stop   → data: { "index": 1 }

// 5. 消息结束，stop_reason 说明停止原因
event: message_delta  → data: { "delta": { "stop_reason": "tool_use" }, "usage": { "output_tokens": 42 } }
event: message_stop
```

三个值得注意的细节：

- **tool 参数是字符串碎片**，需要在本地逐片拼接后解析，解析失败就暂时返回 `{}`，等待更多碎片
- **`stop_reason: "tool_use"`** 告诉框架这轮因调工具而停止，agent loop 据此进入执行流程
- **`cache_read_input_tokens: 280`** 说明 system + tools 部分命中了 prompt cache，跳过了 prefill 计算

工具执行完毕后，结果带着原始的 `tool_use_id` 写回 transcript，下一轮整段重发给 LLM：

```json
[
  { "role": "user", "content": "北京今天天气怎么样？" },
  {
    "role": "assistant",
    "content": [
      { "type": "text", "text": "我来查一下。" },
      { "type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": { "location": "北京" } }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_01",
        "content": [{ "type": "text", "text": "北京今天晴，28℃，东南风 3 级" }],
        "is_error": false
      }
    ]
  }
]
```

`tool_use_id` 的作用是让模型把"这个结果"和"那次调用"对应起来——一次响应里可能有多个 tool call，每个 result 都要通过 id 说明自己是哪次调用的答复。

### 模型实际"看到"的是什么

API 接收到结构化 JSON 后，服务端会把它展平成一段带特殊 token 的连续文本序列，才是真正喂进 transformer 的内容。各家格式没有完整公开，但大致规律如下。

**Anthropic**：tools 被序列化成 XML 追加在 system 末尾，含 tool call 的完整对话在 token 层面长这样：

```
[SYSTEM]
You are a helpful assistant...
<tools><tool_description>...</tool_description></tools>
[/SYSTEM]

[HUMAN]北京今天天气怎么样？[/HUMAN]

[ASSISTANT]我来查一下。
<tool_use>{"name": "get_weather", "input": {"location": "北京"}}</tool_use>
[/ASSISTANT]                     ← 模型生成到这里停止

[HUMAN]
<tool_result tool_use_id="toolu_01">北京今天晴，28℃，东南风 3 级</tool_result>
[/HUMAN]

[ASSISTANT]                      ← 服务端追加，模型从这里继续生成
```

**OpenAI**：使用 ChatML 格式（`<|im_start|>` / `<|im_end|>`），tools 序列化为类 JSON 文本追加进 system，tool result 作为 `tool` 角色注入。

从模型视角看，它每次看到的都是一段完整历史，**完全不知道中间发生过"执行工具"这件事**——工具执行是框架层的行为，对模型透明。

这也解释了两件事：**`description` 写得好不好直接影响模型判断**（它就是模型读到的那段文字）；以及对话越长首 token 延迟越高（每次都要从头 prefill 整段历史），prompt cache 缓存不变的前缀正是为了解决这个问题。

### Agent Loop 与执行调度

真实场景中，模型往往不是调一次工具就结束，而是经历多轮。OpenClaw 的 agent loop 核心是一个双层循环（`packages/agent-core/src/agent-loop.ts`）：

```ts
while (true) {                              // 外层：等待用户追加消息
  let hasMoreToolCalls = true;

  while (hasMoreToolCalls) {                // 内层：处理 tool call 批次
    const message = await streamAssistantResponse(context);

    const toolCalls = message.content.filter(c => c.type === "toolCall");
    if (toolCalls.length === 0) {
      hasMoreToolCalls = false;             // 没有 tool call，本轮结束
      break;
    }

    const results = await executeToolCalls(toolCalls);
    context.messages.push(...results);      // 带着结果进入下一轮请求
  }
}
```

模型可以**一次输出多个 tool call**，框架默认并行执行以节省时间。比如"帮我查北京和上海的天气"，模型同时发出两个 `get_weather` 调用，结果都回来后再统一回答。

**并行还是串行，由开发者在定义工具时决定，不是 LLM 控制的：**

```ts
const weatherTool: AgentTool = {
  name: "get_weather",
  executionMode: "parallel",    // 查询类，并发没问题
};

const writeFileTool: AgentTool = {
  name: "write_file",
  executionMode: "sequential",  // 有副作用，强制串行
};
```

框架降级规则：同一批里**只要有一个**工具标了 `sequential`，整批立即降为串行。

| 工具类型 | 推荐模式 | 原因 |
|---|---|---|
| 查询、读取、幂等操作 | `parallel` | 并发安全，节省时间 |
| 写文件、发消息、调支付 | `sequential` | 有副作用，避免竞态 |
| 有依赖关系的操作 | LLM 跨轮次自然处理 | 模型看不到上一步结果就不会发下一步调用 |

---

## 如果模型"没调对"怎么办

Function Call 依赖模型生成结构化输出，但模型并不总是完美的，实际上有三种失败情形。

**情形一：调用了工具，但参数不符合 schema**

OpenClaw 的处理分两步：先尝试自动类型转换（比如 schema 要求数字，模型给了字符串 `"28"`，会尝试自动转成 `28`）；转换后仍然不通过，才把详细错误信息作为 tool result 还给模型：

```
Validation failed for tool "get_weather":
  - /location: Expected string
Received arguments: { "city": "北京" }
```

模型通常能读懂并修正参数重新调用，整个过程用户无感知。

**情形二：模型把 tool call 写成了普通文本**

部分模型（尤其是较早版本或上下文过长时）会退化，不输出结构化的 `tool_use`，而是直接在回复文本里写：

```
[get_weather]
{"location": "北京"}
```

或者 XML 风格：

```
<function=get_weather>
<parameter=location>北京</parameter>
</function>
```

OpenClaw 有专门的 `tool-call-repair` 模块识别这些格式，解析成功后提升为正式的结构化调用继续执行，同时把这段文本从用户可见的回复里抹掉。

**情形三：模型完全没有识别到需要调工具**

没有任何 tool call，agent loop 当普通文本回复处理，直接返回给用户。没有报错，但用户可能拿到的是一个不准确的答案——这正是为什么工具的 `description` 要写清楚。

| 情形 | 处理方式 |
|---|---|
| 参数类型小错误 | 自动类型转换，尽量容忍 |
| 参数不符合 schema | 错误信息还给模型，让模型重试 |
| 模型输出纯文本 tool call | 解析并提升为结构化调用 |
| 模型完全没调工具 | 当普通文本回复，loop 结束 |

---

## 几个常见误区

**误区一：`description` 随便写就行**

`description` 是模型决定"要不要用这个工具"的唯一依据，也是它真实读到的文本。写得模糊，模型就可能在不该调用时调用，或者该调用时放弃。工具描述和参数说明值得认真打磨。

**误区二：一次只能调一个工具**

现代模型支持在一次响应里输出多个 tool call，框架可以并行执行。这对"分头查询再汇总"的任务（比如同时搜索多个关键词）效率提升很大。

**误区三：有顺序依赖的操作需要开发者干预**

"先创建文件再写入"这类有依赖的操作，LLM 会自然地分轮次处理——它看不到上一步的结果，就不会发出下一步的调用。开发者只需要处理**同批次内的并发安全**（用 `sequential`），跨轮次的逻辑顺序不需要干预。

---

## 小结

Function Call 的本质是一个**协议**：开发者用 schema 描述工具，模型用结构化输出表达"想调哪个"，代码执行并返回结果，模型用结果继续生成。这个循环让 LLM 从"知识库"变成了"能行动的 agent"。

你现在看到的各种 AI 助手能查天气、写代码、操作文件、发邮件，背后都是这套机制在支撑。如果你对 agent 如何管理多轮工具调用的完整循环感兴趣，可以进一步了解 agent loop 的设计；如果想知道工具列表如何按需过滤，可以看 tool planner 的实现。

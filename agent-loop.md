# OpenClaw Agent Loop

> 本文基于 2026.6.2 版本源码，关键文件：
> `packages/agent-core/src/agent-loop.ts`、`packages/agent-core/src/types.ts`。

---

## 是什么

Agent loop 是 OpenClaw 的核心驱动引擎，控制一次 agent 运行的完整生命周期。

一次用户请求进来后，不是只请求一次 LLM 就结束——模型可能要多次调用工具、每次看完工具结果再继续思考，直到给出最终答复。Agent loop 就是控制这个"请求 → 工具执行 → 再请求"反复循环的引擎。

**两个入口：**
- `agentLoop(prompts, context, config)` — 带新消息启动，返回 `EventStream`
- `agentLoopContinue(context, config)` — 从现有 context 继续（用于重试）

两者都是异步流，调用方通过监听 `AgentEvent` 来驱动 UI 更新。

---

## 双重循环结构（`runLoop`）

```
外层 while(true)
│   等待 steering 消息（用户在 agent 工作时插入的消息）
│   没有新消息且无 tool call → 检查 getFollowUpMessages()
│   还是空 → 退出
│
└── 内层 while(hasMoreToolCalls || pendingMessages.length > 0)
        ① 注入 pending steering 消息
        ② streamAssistantResponse()  —— 一次 LLM 流式请求
        ③ 检测 toolCall 块
        ④ executeToolCalls()         —— 执行这批 tool，结果写回 transcript
        ⑤ prepareNextTurn()          —— 可替换下一轮的 model/context
        ⑥ shouldStopAfterTurn()      —— 可优雅退出
```

---

## 三个核心输入

| 输入 | 类型 | 说明 |
|------|------|------|
| `AgentContext` | 当前状态 | `systemPrompt` + `messages`（完整 transcript）+ 可用 `tools` |
| `AgentLoopConfig` | 行为配置 | model、各种 hook、转换函数 |
| `AgentMessage[]` | 新消息 | 这轮用户输入 |

---

## AgentLoopConfig 关键 hook

| hook | 作用 |
|------|------|
| `convertToLlm` | AgentMessage[] → LLM Message[]（必填，过滤 UI-only 消息） |
| `transformContext` | 发给 LLM 前压缩/裁剪 context window |
| `beforeToolCall` | 可拦截 tool 执行（返回 `{ block: true }`） |
| `afterToolCall` | 可修改 tool result，或设 `stopAfterBatch` 终止本批 |
| `resolveDeferredTool` | 动态加载初始列表之外的 tool |
| `getSteeringMessages` | 每轮工具执行完后注入的插队消息 |
| `getFollowUpMessages` | agent 即将退出时追加的后续消息 |
| `prepareNextTurn` | 每轮结束后替换 model 或 context |
| `shouldStopAfterTurn` | 优雅退出（如 context 快满时） |

---

## AgentEvent 事件序列

调用方（UI、channel、测试）订阅事件流来渲染界面或做断言，不需要关心内部循环细节。

```
agent_start
  turn_start
    message_start  (user)
    message_end    (user)
    message_start  (assistant, 流式开始)
      message_update × N  (流式 delta)
    message_end    (assistant)
    tool_execution_start
      tool_execution_update × N  (进度回调)
    tool_execution_end
    message_start  (toolResult)
    message_end    (toolResult)
  turn_end
  turn_start       ← 有 tool call 时进入下一轮
  ...
agent_end
```

tool call 的具体执行流程见 [function-call-flow.md](./function-call-flow.md)。

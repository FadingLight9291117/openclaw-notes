# LLM 自回归生成与 KV Cache

> 通用 LLM 原理，与 OpenClaw 代码无关，但解释了 streaming 延迟和 prompt cache 的底层依据。

---

## 自回归生成（Autoregressive Generation）

LLM 每次只生成一个 token，然后把它追加到输入序列末尾，重新跑一遍前向传播，生成下一个 token，循环直到 `<|EOS|>`：

```
输入: [token1, token2, ... tokenN]    →  Transformer  →  tokenN+1
输入: [token1, token2, ... tokenN+1]  →  Transformer  →  tokenN+2
...
```

这就是为什么 LLM 的输出是"流式逐字出现"的——每个 token 生成后立即可以返回给调用方，不需要等全部生成完。

---

## KV Cache

如果每次都把整个序列从头重算，代价随序列长度线性增长。实际上 transformer 每层 attention 会把历史 token 的 Key/Value 矩阵缓存下来，新 token 只需计算自己的 Q 与历史 KV 的 attention，不重算历史。

```
第1步（prefill）：完整计算所有输入 token 的 KV，存入缓存
第2步以后（decode）：只算新 token，复用缓存中的历史 KV
```

**这解释了两个常见现象：**

- **首 token 慢、后续 token 快**：prefill 是全量计算，耗时与输入长度成正比；decode 是增量计算，每步耗时基本固定
- **长 system prompt 影响首 token 延迟**：system prompt 越长，prefill 越慢，但后续 decode 速度不受影响

---

## Prompt Cache（服务端 KV Cache）

Anthropic、OpenAI 等服务商把 KV Cache 做到了服务端——把 system prompt + tools 的 KV 矩阵持久化存储，下次相同前缀的请求直接命中缓存，跳过 prefill 阶段。

**命中条件：前缀必须完全一致（逐 token 匹配）。**

这就是 OpenClaw 里多处保证顺序确定性的原因：
- `src/tools/planner.ts` 的 `buildToolPlan()` 用 `sortKey ?? name` 对 tool 列表排序
- `src/llm/providers/openai-responses-tools.ts` 在 `convertResponsesToolPayload()` 里再次按 name 排序
- Anthropic 的 `cache_control` 打在 tools 数组最后一个 tool 上，标记缓存边界

顺序一旦变化，前缀不匹配，缓存失效，重新 prefill。

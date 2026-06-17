# OpenCode Plan Mode 是怎么实现的

> 基于 sibling 仓库 `../opencode` 源码实读。关键文件:
> `packages/opencode/src/agent/agent.ts`、`.../permission/index.ts`、
> `.../session/llm/request.ts`、`.../tool/plan.ts`、`.../tool/edit.ts`。

## 一句话

OpenCode **没有专门的「plan mode」状态机**。Plan Mode 就是一个叫 `plan` 的 **agent**,
本质是一套 **permission 规则集(Ruleset)**;它复用了 OpenCode 给每次工具调用做权限
闸门的同一套机制。「进/出 plan」= 切换这一回合归属的 agent。

---

## 1. Mode 即 Agent:plan 是预置 agent 之一

`agent.ts:138` 起定义了四个 native agent,每个就是一个 permission Ruleset:

| agent | mode | 作用 |
|-------|------|------|
| `build` | primary | 默认 agent,按配置权限执行(可编辑) |
| `plan` | primary | Plan mode,禁所有编辑工具 |
| `general` | subagent | 通用并行子任务 |
| `explore` | subagent | 只读探索代码库 |

`plan` 的定义(`agent.ts:154`):

```ts
plan: {
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(
    defaults,                    // "*": allow;plan_enter/plan_exit/question 默认 deny
    Permission.fromConfig({
      question: "allow",
      plan_exit: "allow",        // 只有 plan agent 能退出
      task: { general: "deny" }, // 禁止派 general subagent(否则可绕道写文件)
      edit: {
        "*": "deny",             // ★ 核心:禁所有编辑
        [".opencode/plans/*.md"]: "allow",   // 但允许写计划 md
        [globalPlansDir + "/*.md"]: "allow",
      },
    }),
    user,                        // 用户 config 最后合并,可覆盖
  ),
  mode: "primary", native: true,
}
```

对照 `build`(`agent.ts:139`):`plan_enter: "allow"` + `question: "allow"`,继承
`"*": "allow"`,所以能编辑、也能进入 plan。

`defaults`(`agent.ts:117`)是所有 agent 的底:`"*": "allow"`、`doom_loop: "ask"`、
`question: "deny"`、`plan_enter/plan_exit: "deny"`、`read` 对 `*.env` 类 `ask`、
外部目录 `ask`。

---

## 2. 规则集怎么被执行:last-match-wins + 两层闸门

### 规则求值

`evaluate()`(`permission/index.ts:39`)在合并后的 ruleset 里 `findLast` 命中
`{permission, pattern}` 的规则——**后写覆盖先写**(所以 `user` 放在 merge 最后能
覆盖默认),无命中则默认 `ask`。

### 第一层 · 整工具移除(发请求前)

`resolveTools()`(`session/llm/request.ts:198`)调 `Permission.disabled()`
(`index.ts:215`):

```ts
export function disabled(tools, ruleset) {
  const edits = ["edit", "write", "apply_patch"]
  return new Set(tools.filter((tool) => {
    const permission = edits.includes(tool) ? "edit" : tool
    const rule = ruleset.findLast((r) => Wildcard.match(permission, r.permission))
    return rule?.pattern === "*" && rule.action === "deny"   // 仅当 "*" 全禁才删
  }))
}
```

一个工具若其**最后命中**规则是 `pattern === "*" && action === "deny"`,就整个从
模型可见工具表里删掉(`edit/write/apply_patch` 都映射到 `edit` 这个 key)。

### 第二层 · 调用时逐次 ask(运行时)

工具执行时自己调 `ctx.ask(...)`。例如 `tool/edit.ts:102,145`:

```ts
yield* ctx.ask({ permission: "edit", patterns: [文件路径], ... })
```

`Permission.ask()`(`index.ts:78`)对每个 pattern 跑 `evaluate`:

- `deny` → 抛 `DeniedError`,该 tool call 被拒,错误回写给模型;
- `allow` → 静默放行;
- `ask` → 发 `Asked` 事件、阻塞在 Deferred 上,等用户回 once / always / reject。

### 为什么 plan 能「只读但能写计划文件」

关键细节:plan 给 `.opencode/plans/*.md` 留了 `allow`,所以 `disabled()` 的
`findLast("edit")` 命中的是那条 **path 级 allow**(`pattern` 不是 `"*"`)——
于是 **edit 工具不会被整个删掉**,而是保留下来、在运行时按路径逐个判:

- 写普通文件 → 命中 `edit "*": deny` → `DeniedError` 挡掉;
- 写 `.opencode/plans/foo.md` → 命中 path allow → 放行。

这就是「只读分析、但能把计划落地写进 md」的实现方式。

---

## 3. 进/出 Plan:`plan_enter` / `plan_exit` 是受权限管的伪工具

切换不是 UI flag,而是两个权限动作:

- `build`:`plan_enter: allow`、`plan_exit: deny`(继承)→ 只能进
- `plan`:`plan_exit: allow`→ 只能出

退出由 `plan_exit` 工具实现(`tool/plan.ts:15`):

1. `question.ask(...)` 弹确认:「计划已完成,要切到 build agent 开始实现吗?」
   (`plan.ts:30`)
2. 选 **No** → 抛 `Question.RejectedError`,留在 plan(`plan.ts:46`)。
3. 选 **Yes** → 注入一条 **synthetic user 消息**,把 `agent: "build"`
   (`plan.ts:53-61`)+ 合成文本「计划已批准,你现在可以编辑文件,执行计划」
   (`plan.ts:67`)。这一回合的归属 agent 翻成 `build`,其 ruleset 放开编辑——
   权限随之解锁。

```
[plan agent]  edit "*": deny  ──plan_exit(用户确认 Yes)──▶  注入 agent="build" 的 user 消息
                                                            └─ [build agent] "*": allow,可编辑
```

---

## 设计要点

OpenCode 把 **plan-mode、只读 explore、各种 subagent、用户自定义 agent 全部统一成
「命名的 permission ruleset」**。同一套:

- `evaluate`(last-match-wins)
- `disabled`(整工具删除)
- `ask`(运行时 deny / allow / 询问)

既做单次工具闸门,也实现了 Plan Mode。「换模式」= 换这一回合归属的 agent,
经 `plan_enter` / `plan_exit` 且需用户确认。

> 对比 Claude Code:Claude Code 的 Plan Mode 是独立的 `EnterPlanMode`/`ExitPlanMode`
> 门控;OpenCode 则把它压进通用 permission 体系里,代价更小、可被用户 config 复用同
> 一套 allow/ask/deny 规则。

> 注:`../opencode/packages/core/src/plugin/agent.ts:117` 有一份 V1 core 等价镜像
> 定义(`{action:"plan_enter", effect:"deny"}` 等),结构一致;上文以
> `packages/opencode/src` 下现行实现为准。

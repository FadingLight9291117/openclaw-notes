# Skill 是什么?让 agent 学会"什么时候用什么工具"

> 本文以 [OpenClaw](https://github.com/openclaw/openclaw) 开源源码为例
> (基于 2026.6.2 版本),结合实际代码说明 agent skill 系统的设计和工作原理。
> 涉及源码路径均可在仓库中直接查阅。

如果你跟着这个系列读到这儿:
[Function Call](./function-call-101.md) 讲了模型怎么调工具,
[MCP](./mcp-101.md) 讲了工具怎么从哪儿来,
[RAG](./rag-101.md) 讲了怎么让模型看着资料回答。

但还有一个问题没解:**给 agent 接了 100 个工具,它怎么知道
什么时候该用哪个?** 这次不是 "知识不知道",而是 "方法不知道"。

本文回答这个问题。答案叫 **Skill**(技能)——它会借用一些 RAG 的
设计思想,但走出一个有意思的变体。

---

## 一个真实的麻烦

假设你正在做一个 AI 编程助手,接了一堆工具:`Read`、`Bash`、`Grep`、
`Edit`、`gh`(GitHub CLI 包装)、`git`、`docker`、`kubectl`、`npm`、
`pytest`、再加上 20 个 MCP server 暴露的工具。模型看到的 tool 列表有
80 个。

用户说:**"帮我看看这个 PR 的评论"**。

模型该用哪个工具?是 `gh pr view --comments`?还是 `git log --grep`?
还是某个 MCP 工具?光看工具名和参数 schema,**模型其实不知道你的团队
习惯用哪种方式**。

你可以这么做:

### 选项 A:把"怎么做"全塞进 system prompt

```
You are a coding assistant. When the user asks about PR comments,
use `gh pr view <pr-number> --comments --json`. When asked about
recent commits, use `git log --oneline -20`. When deploying, use
kubectl with our cluster context "prod-east"...
```

这能解决问题——但 10 条指令还行,100 条就崩了:**system prompt 越来越长,
所有任务都付全套 token 成本,而且每条 instruction 都会跟其他 instruction
互相干扰**。

### 选项 B:让模型自己看 README

把所有工作流写成 markdown 文档,告诉模型 "需要时去读"。问题是:它
怎么知道现在该读哪份?80 个工具加 50 份操作文档,**索引到底在哪儿**?

### 选项 C:用 RAG 查相关文档

把所有文档向量化,每次任务先 retrieval([RAG](./rag-101.md) 那一套)。
可行,但有两个问题:

- 要维护一套向量库——切块、embedding、向量数据库,基础设施成本不小
- 程序操作类的精确指令上,**embedding 经常翻车**——"PR 评论怎么查"
  跟 "GitHub gh CLI 文档" 字面差很远,语义匹配可能拉到错的段落

对"知识检索"RAG 很合适,但对"流程指令"它有点重。

Skill 是第四个选项,而且很巧妙——**它借了 RAG 的"先找再用"思路,
但把"找"这一步做得更轻**。

---

## Skill 是什么

一句话:**Skill 是一份 markdown 文档,描述"什么时候、怎么完成某类
任务",存在文件系统里,模型按需读取。**

每个 skill 是一个目录,核心是带 YAML frontmatter 的 `SKILL.md`:

```markdown
---
name: github
description: Interact with GitHub via gh CLI - view PRs, issues, comments, etc.
metadata: {"openclaw": {"requires": {"bins": ["gh"]}}}
---

When the user asks about a PR's comments, run:
  gh pr view <pr-number> --comments --json

When asked about recent commits in a repo, use:
  git log --oneline -20

If the user is asking about a closed PR, also check:
  gh pr view <pr-number> --json mergedAt,closedAt
...
```

就这。整个"能力"就是这份 markdown 加上目录结构。OpenClaw 仓库里有约
50 个内置 skill,每个都长这样——`github`、`pytest`、`docker`、
`image-lab` 等等。

---

## 核心机制:渐进式披露(progressive disclosure)

最聪明的部分在这里。

**模型的 prompt 里不放 skill 的正文,只放一个目录**:

```xml
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.

<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub via gh CLI - view PRs, issues...</description>
    <location>/path/to/skills/github/SKILL.md</location>
    <version>3</version>
  </skill>
  <skill>
    <name>pytest</name>
    <description>Run and interpret Python tests...</description>
    <location>/path/to/skills/pytest/SKILL.md</location>
    <version>1</version>
  </skill>
  ...50 个 skill...
</available_skills>
```

每条只有几十个 token。模型看到这个目录后:

1. 用户问 **"帮我看看这个 PR 的评论"**
2. 模型扫一遍 description,**"github" 这条匹配**
3. 模型调用 `read` 工具,读取那个 `<location>`
4. 拿到 SKILL.md 正文,按指令执行 `gh pr view ... --comments`

**只有匹配上的 skill 才付正文 token 成本**。50 个 skill 在目录里大概几千
token,如果今天用了 github 和 pytest 两个,实际加载的正文也就 1000 tok。
对比"全塞进 system prompt"那种动辄几万 token 的方案,差距巨大。

这个设计有个名字叫 **progressive disclosure**——一开始只给最少必要
信息,需要细节时再加载。Anthropic 2025 年初提出的 [Agent Skills 规范](https://agentskills.io)
就是基于这个思路。

---

## OpenClaw 里是怎么落地的

讲完概念,看看 OpenClaw 源码里一次完整的 skill 调用从头到尾发生了什么。
这一节回答几个具体问题:**skill 是怎么被找到的、prompt 长什么样、
模型怎么"决定"用某个 skill、它返回的内容里怎么标识这次调用**。

### 1. 加载:六类来源,优先级合并

agent session 启动时,OpenClaw 扫描六类目录(`src/skills/loading/`):

```
优先级 1(最高) │ <workspace>/skills          仅该 workspace
优先级 2       │ <workspace>/.agents/skills  仅该 agent
优先级 3       │ ~/.agents/skills            本机所有 agent
优先级 4       │ ~/.openclaw/skills          本机所有 agent
优先级 5       │ 内置 bundled(约 50 个)
优先级 6(最低) │ extraDirs + 插件携带
```

同名时高优先级覆盖低优先级——你可以在 workspace 里写一个 `github`
覆盖内置的 `github`,本地定制不动上游。

扫到的每个 SKILL.md 解析 YAML frontmatter,经过门控过滤(缺二进制
`requires.bins`、缺环境变量 `requires.env`、不在 agent allowlist 等)
被剔除掉。

活下来的合到一份 **SkillSnapshot**(`src/skills/runtime/session-snapshot.ts`)
固化在 session 上。**会话中途用户改 skill 不影响进行中的对话**——这跟
[MCP catalog 缓存](./mcp-101.md) 是同样的"热路径不轮询"思路。

### 2. Prompt:实际的 XML 长什么样

prompt 由 `formatSkillsForPrompt`(`src/skills/loading/skill-contract.ts:53`)
生成。**真实输出就是下面这段** XML(下面是从源码原文摘的):

```
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
If a skill's <version> differs from a previous turn, re-read its SKILL.md before using it.
When a skill file references a relative path, resolve it against the skill directory ...

<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub via gh CLI - view PRs, issues, comments, releases</description>
    <location>/home/clz/.openclaw/skills/github/SKILL.md</location>
    <version>3</version>
  </skill>
  <skill>
    <name>pytest</name>
    <description>Run and interpret Python tests, debug failures</description>
    <location>/home/clz/.openclaw/skills/pytest/SKILL.md</location>
    <version>1</version>
  </skill>
  ...
</available_skills>
```

这段拼到 system prompt 里,跟 system instruction、工具列表(`tools`
字段)一起发给模型。这就是模型看到 skill 的**全部**——50 个 skill
也就几千 token,跟"全塞 skill 正文"动辄几万 token 差好几个数量级。

注意三点:

1. **没有正文**,只有 `<location>`(SKILL.md 绝对路径)
2. **`<version>` 用于会话内缓存失效**——上轮和这轮版本不同时,模型
   被指令要求重读
3. **prompt 里有句明确指令**:"Use the read tool to load a skill's
   file when the task matches its description"。**这一句就是全部
   触发机制**,没有别的代码逻辑

### 3. LLM 怎么"决定"使用某个 skill

**完全是 LLM 自己的语义判断,没有任何代码做匹配**。

OpenClaw 这一侧:

- 不算 embedding
- 不做关键词索引
- 不调用任何路由算法
- 不告诉模型"现在该用 X skill"

模型这一侧拿到 prompt 之后:

1. 看到用户问题 "帮我看看 PR 123 的评论"
2. 扫 `<available_skills>` 里几十条 `<description>`
3. 用语义判断:**"github" 这条 description 提到了 PR 和 comments,匹配**
4. 决定调 read 工具去读 `<location>` 指向的文件

整个"路由智能"完全是 LLM 推理出来的。OpenClaw 只负责把候选目录端
端正正放好。

**推论:`description` 的写法直接决定 skill 会不会被触发**——它是
skill 作者手里最重要的"路由表项"。写得含糊(`"GitHub stuff"`)模型
就匹配不到;写得准确(`"Interact with GitHub via gh CLI - view PRs,
issues, comments"`)模型几乎必中。

### 4. LLM 返回的内容里有什么

**没有"特殊的 skill 调用类型"**。模型返回的就是普通的 `tool_use`——
一个 `read` 工具调用,参数是 SKILL.md 路径:

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "text",
      "text": "我用 github skill 来查 PR 评论。"
    },
    {
      "type": "tool_use",
      "id": "toolu_01abc...",
      "name": "Read",
      "input": {
        "file_path": "/home/clz/.openclaw/skills/github/SKILL.md"
      }
    }
  ]
}
```

注意 `name` 是 `"Read"`,**不是 `"skill_github"` 或 `"invoke_skill"`
之类的特殊名字**。Skill 触发本质上就是一次 file read,跟模型读其他
文件没有任何机制上的区别。

OpenClaw 这一侧收到 tool_use 后:

1. 在 tool 列表里找到 `Read`,调它的 `execute`
2. 返回 SKILL.md 全文给模型
3. 模型在下一轮看到正文,按指令开始执行真正的工作
   (`gh pr view 123 --comments --json`)

整个流程**完全复用了 Function Call 的标准机制**——这是 progressive
disclosure 设计的另一个优点:**不需要新协议、不需要 provider 支持
特殊字段、模型不需要任何专门的训练。**

### 5. 走完一遍完整时序

```
Session 启动
  │
  ├─ 扫六类目录,加载 SKILL.md frontmatter
  ├─ 门控过滤,剔除不满足 requires 的
  ├─ 固化 SkillSnapshot
  └─ formatSkillsForPrompt() → <available_skills> XML 片段

用户发问:"帮我看看 PR 123 的评论"
  │
  ├─ OpenClaw 组装请求:
  │     system: "...<available_skills>...</available_skills>"
  │     tools:  [Read, Write, Bash, ...]
  │     user:   "帮我看看 PR 123 的评论"
  │
  └─ 发给 LLM

LLM 第 1 轮返回:
  │
  └─ tool_use: Read(file_path="/path/to/skills/github/SKILL.md")

OpenClaw 执行 Read,返回 SKILL.md 正文

LLM 第 2 轮返回:
  │
  ├─ "我用 github skill,执行 gh pr view"
  └─ tool_use: Bash(command="gh pr view 123 --comments --json")

OpenClaw 执行 Bash,返回 JSON 结果

LLM 第 3 轮返回:
  │
  └─ 用自然语言把 PR 评论整理给用户
```

整个机制非常简洁:**两次额外 tool_use 往返(一次 read SKILL.md、
一次执行 SKILL.md 里指的命令),换来零基础设施 + 任意可扩展的能力库**。

### 6. 一个有意思的细节:为啥不直接塞 SKILL.md 内容

读到这里你可能会想:"既然要 read 一次拿全文,为啥不在 prompt 里
直接给正文,省一次往返?"

OpenClaw 的设计就是为了避这个开销:

- 50 个 skill 的正文加起来可能上万 token,**每轮都付这个钱**(prompt
  cache 命中也得算 hash + 重新走 prefill)
- 用户这一轮可能根本不用任何 skill(普通问候/闲聊),正文白付
- 多一次 read 往返,对比"每轮付全部 token",在大多数场景下都更划算

进一步,**OpenClaw 在 prompt 预算紧张时还会更激进地压缩目录**
(`applySkillsPromptLimits`)——只保留 name + location,丢掉 description。
即使在这种最差情况下,模型仍然知道"有这些 skill 存在",可以试着读
进来看看。

---

## 跟 RAG 有什么不一样

上一篇刚讲完 [RAG](./rag-101.md),你可能觉得这俩思路很像——都是
"先找相关的、再让模型用"。**核心思想确实是 RAG 那一套 progressive
disclosure 的延伸**,但 Skill 在关键一步上换了实现:

| | RAG | Skill |
| ---- | --------------------------------- | ----------------------------------- |
| 索引 | embedding 向量库 | 目录 + 文本 description |
| 匹配 | cosine similarity(数学) | 模型读 description 做语义判断(LLM) |
| 加载 | 检索器自动选 top-k | 模型自己决定调 `read` 工具 |
| 基础设施 | 向量数据库 + embedding 模型 | 文件系统 + 一个 read 工具 |
| 适合什么 | 大规模、模糊查询、长尾知识 | 中等数量、明确分类、操作指令 |

关键差别在 **"找" 这一步谁来做**:

- **RAG** 让向量库来找——准确但要基础设施(embedding 服务 + 向量数据库),
  适合海量、模糊、跨文档的知识检索
- **Skill** 让模型自己来找——把目录直接塞进 prompt,让模型读 description
  做语义匹配,**省掉向量化整套链路**

为什么 Skill 敢省掉向量化?因为它的场景跟 RAG 不一样:

- RAG 要在几万段文档里找几个相关段——向量化是必要的,模型扫不过来
- Skill 通常就 50~100 条——目录占几千 token,模型完全扫得过来

简言之:**RAG 适合"知识检索",Skill 适合"流程路由"**。规模小、目标
明确的场景,直接让 LLM 充当"路由器"比搭一套向量库更轻、更准。

这是个有意思的工程权衡:**当 LLM 自己够聪明时,有些中间层就可以
省掉**。

---

## "路由智能"留给模型,"可用性控制"留给代码

OpenClaw 的实现(`src/skills/`)有一个清晰的分工:

**代码这一层管"哪些可用"**——会被加载进目录的 skill 必须通过这些过滤:

- 缺少必需的二进制(`requires.bins: ["gh"]` 但本机没装 gh)→ 排除
- 缺少必需的环境变量(`requires.env: ["GITHUB_TOKEN"]`)→ 排除
- 平台不符(`requires.os: ["macos"]` 但跑在 Linux 上)→ 排除
- 不在当前 agent 的 allowlist 里 → 排除

**模型这一层管"用不用、用哪个"**——目录里活下来的 skill,选哪个完全是
模型的语义判断。代码不去做关键词匹配、不去算 embedding、不去做任何
"推荐"。

这个分层很重要:

- 代码擅长**确定性检查**(二进制装没装、env 设没设)
- 模型擅长**语义判断**("用户说的'PR 评论'对应 github 这个 skill")

让各自做自己擅长的事。

---

## description 是 skill 作者最重要的产物

既然模型靠 description 做匹配,**description 写得好不好直接决定 skill
会不会被触发**。

好的 description:

```yaml
description: Interact with GitHub via gh CLI - view PRs, issues,
comments, releases, run workflows, manage labels.
```

涵盖了关键名词("PR"、"issue"、"comment")和动词("view"、"run"、
"manage"),用户问任何相关问题模型都能匹配上。

差的 description:

```yaml
description: GitHub stuff
```

模型看到这个根本不知道你 skill 里写了什么。用户问 "看 PR 评论" 时模型
不会想到调它。

这是 skill 工程的核心技能——不是写代码,而是 **写出让模型能精确匹配的
英文描述**。

---

## 版本失效:会话中如何感知 skill 变化

`<available_skills>` 里每条带一个 `<version>`,prompt 里有句指令:

> If a skill's `<version>` differs from a previous turn, re-read its SKILL.md before using it.

会话开始时,OpenClaw 固化一份 SkillSnapshot(`src/skills/runtime/`),
中途用户改了 `SKILL.md`,新的 version 会进到目录里,模型看到版本变了
就重读正文——**缓存失效靠版本号传导,而不是每轮都重读**。

这跟我们之前在 [MCP 内部实现](../mcp-client-internals.md) 里看到的
"catalog 缓存 + listChanged 失效" 是同一个套路:**热路径缓存,显式信号
触发刷新**。

---

## 还有几个有意思的设计

### 1. prompt 预算超限时怎么办

50 个 skill 写得详细的话,目录可能上万 token。OpenClaw 有梯度退化
(`applySkillsPromptLimits`):

| 梯度 | 行为 | 代价 |
| --- | --- | --- |
| 正常 | name + description + location + version | 无 |
| 超 `maxSkillsInPrompt` | 截断技能数量 | 后面的不可见 |
| 超字符预算 | 降级 compact:只剩 name + location | 丢 description,模型只能按名字猜 |
| 再超 | 截断 + 注入警告 | 提示用户 `openclaw skills check` |

哪怕最差情况,模型也至少**知道有这个 skill 存在**,可以试着用 read
工具看看。这是 graceful degradation 的好例子——情况越糟,体验越糟,
但永远不彻底崩。

### 2. 绕过模型的两条路径

有时候你不想让模型自己决定:

- `disable-model-invocation: true` → 不进目录,模型自主用不到;**只能
  用户敲 `/skill-name` 触发**
- `command-dispatch: tool` → 斜杠命令**连模型都不经过**,直接分发到
  注册工具(确定性操作)

这是给敏感操作或者高频操作准备的——比如 `/deploy` 你绝对不希望模型
"自己判断时机",必须人触发。

### 3. Skill Workshop:agent 提案 + 人审

agent 在工作中发现"这个流程值得做成 skill"时,**不直接写 SKILL.md**,
而是产出一份提案进队列。用户审批(`openclaw skills workshop apply <id>`)
后才写进 skill 目录。

这是 "agent 可以建议、人类拥有最终所有权" 原则的体现——你不会希望模型
偷偷改自己的指令集。

### 4. 安装是供应链问题

skill 可以从多个来源安装:ClawHub 注册表(类似 npm)、Git 仓库、本地
目录、zip 上传。每个来源都被视为 **不可信代码**:

- ClawHub 上架前过安全扫描(VirusTotal / ClawScan / 静态分析)
- 安装时本地可以配 `security.installPolicy` 跑自定义策略命令,fail-closed
- `openclaw skills verify` 拿信任信封做校验,适合接入 CI 做供应链门禁

**skill 不是"代码"但也不只是"文档"**——它能让模型执行任意命令,
所以安全模型按代码标准来。

---

## Skill 跟 native tool / MCP tool 是什么关系

你可能会想:"模型已经有 tool 了,skill 是不是又一层抽象?"

**不是的**。skill 不是 tool,它**控制模型怎么用现有的 tool**。

- **Tool**(包括 native、plugin、MCP)是"能做什么"——读文件、跑命令、调 API
- **Skill** 是"什么时候做、怎么做"——遇到这种用户请求,该按什么顺序调哪些工具

类比一下:

- Tool 是厨房里的刀、锅、铲——基础能力
- Skill 是菜谱——告诉你做番茄炒蛋要先打蛋还是先炒西红柿

你完全可以只有 tool 没有 skill,模型靠常识也能用。但如果有领域特定的
习惯(我们公司 PR 用 squash 合并 / 我们的数据库用这个连接字符串 /
这个 API 错误码 200 其实是失败)——这些是模型训练数据里没有的东西,
就需要 skill 来教。

---

## 总结:为什么 Skill 是个好主意

回到开头那个问题:**给 agent 接了 100 个工具,它怎么知道什么时候该
用哪个?**

```
            塞进 system prompt              全部 token 成本
                  ↑                              ↓
  Skill ←─────────┴─ progressive disclosure ─────┘  + 模型语义匹配
                  ↓                              ↑
            按需 read 加载                只付匹配上的成本
```

Skill 在三个轴上找到了平衡:

| 轴 | 极端 1 | 极端 2 | Skill 在哪儿 |
| --- | --- | --- | --- |
| 信息密度 | 全部塞进 prompt(贵) | 全部不给(模型不知道有) | 只给目录,正文按需加载 |
| 路由智能 | 代码决定(僵硬) | 模型决定(可能瞎选) | 代码筛可用性,模型选用哪个 |
| 演化方式 | 硬编码(改要发版) | 模型自改(危险) | 文件系统 + workshop 人审 |

最关键的洞察是:**目录这个抽象足够便宜,可以放大量条目;模型的语义
匹配能力足够强,可以按 description 准确路由;文本格式足够灵活,可以
表达任意流程指令**。三者结合,得到了一套零基础设施、可扩展、可治理
的"agent 能力库"。

这跟 [LSP](../lsp-tools.md) 把"语言能力"抽出来变成独立服务、跟
[MCP](./mcp-101.md) 把"工具能力"抽出来变成独立 server 是同样的思路——
**把可复用的智能抽到通用的接口背后**。Skill 抽的是"流程指令"。

---

## 想深入了解?

- AgentSkills 规范:<https://agentskills.io>
- OpenClaw skills 系统源码:`src/skills/`
- 配套的技术笔记(更细):
  - 整体设计:[../skills-management-design.md](../skills-management-design.md)
  - 触发机制:[../skill-invocation-mechanism.md](../skill-invocation-mechanism.md)
  - CLI 与样例:[../skills-cli-and-examples.md](../skills-cli-and-examples.md)
- 系列前文:[function-call-101.md](./function-call-101.md)、
  [mcp-101.md](./mcp-101.md)、[rag-101.md](./rag-101.md)

# 第 2 章 —— 提示词加载流程：一条消息如何变成一次 API 请求

*你输入的每一条消息，在抵达模型之前都要穿过若干个组装阶段 —— 而最终送达的内容里，大部分根本不是你打出来的。*

## 为什么重要

当你让 Claude Code「修复 auth 模块里的 bug」时，模型收到的远不止这一句话。它收到的是：一段角色定义、一组行为规则、若干工具描述、一份 git 快照、你的 `CLAUDE.md`、一份可用技能列表、一条 token 用量警告，以及十几个其他上下文块 —— 每一轮都按固定顺序重新组装。如果你在构建智能体，这个顺序和缓存边界，决定了一个系统究竟是每轮只花几分钱，还是每次都把整块静态指令重新计费一遍。本章从可观察到的行为出发追踪这趟旅程，让你可以照搬这套架构。

## 三个层

对所观察行为的一种合理解读是：请求是分三个概念层组装出来的：

1. **静态系统提示词** —— 每会话构建一次，之后从缓存提供。
2. **上下文注入** —— 系统上下文追加在后，用户上下文前置在前。
3. **逐轮附件** —— 每一轮都重新收集并重新注入。

最后两步把这一切黏合起来：每个工具的描述被生成，然后整体被组装进向外发送的请求。

```
            "fix the bug in the auth module"
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 1  Static system prompt                             |
  | (built ONCE per session, then served from cache)          |
  |                                                           |
  |   (1) role + identity line                               |
  |   (2) reminder-tag explainer                             |
  |   (3) core task rules (prohibitions + behaviors)         |
  |   (4) act-with-care safety block                         |
  |   (5) tool-usage policy (prefer dedicated tools > Bash)  |
  |   (6) tone, style, and output efficiency                 |
  |  =============  cache boundary (static | dynamic)  ====== |
  |   (7) memory section                                     |
  |   (8) environment info (OS / shell / git)                |
  |   (9) language preference                                |
  |  (10) connected-server instructions   [uncached]         |
  |  (11) scratchpad path                                    |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 2  Context injection                                |
  |                                                           |
  |  choose WHICH base prompt to use (by priority)           |
  |  append system context  -> git snapshot + cache breaker  |
  |  prepend user context   -> CLAUDE.md + date, wrapped in   |
  |                            a reminder-tagged USER message |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 3  Per-turn attachments                             |
  | (re-collected EVERY turn, event-driven)                   |
  |                                                           |
  |   skill listing | agent listing | changed files |        |
  |   todo reminders | new diagnostics | nested memory | ...  |
  |   each -> wrapped in a reminder tag                       |
  +-----------------------------------------------------------+
                        |
            per-tool descriptions  ->  tools: [...]
                        |
                        v
            final assembly  ->  POST /v1/messages
```

## 第 1 层：静态提示词与缓存边界

静态系统提示词的行为表现得就像一个有序的文本块数组。开头的几项实际上是恒定的：角色定义那一行、对智能体用来注入上下文的提醒标签的说明、核心的「做任务」规则（一组禁令加上几条行为）、一个关于谨慎操作的安全块、工具使用策略，以及语气/输出效率块。

接着是那条承重的分界线：一条把静态那一半与动态那一半隔开的**缓存边界**。它之前的一切都是稳定的；它之后的内容 —— 记忆、环境信息、语言偏好、已连接服务器的指令、scratchpad 路径 —— 可能在轮次之间发生变化。

这条边界存在只有一个原因：**提示词缓存是按前缀匹配的。**缓存命中要求前缀完全一致。把所有易变内容放在稳定内容*之后*，智能体就能让前缀跨轮次保持恒定，命中缓存，从而在静态块上只付输入 token 成本的一小部分，而不是每轮都付全价。已连接服务器的指令那一段不做缓存，因为只要有服务器连接或断开它就会变化。

## 第 2 层：上下文注入 —— 以及为什么 CLAUDE.md 不在系统提示词里

在注入之前，智能体先决定*用哪个*基础系统提示词，按优先级从高到低：

| 优先级 | 来源 | 何时胜出 |
|----------|--------|-----------|
| 1 | 覆盖提示词 | 某个模式（如 loop）替换掉一切 |
| 2 | 协调器提示词 | 运行在协调器模式下 |
| 3 | 智能体提示词 | 某个子智能体提供了自己的系统提示词 |
| 4 | 自定义提示词标志 | 用户在命令行上传入了自定义提示词 |
| 5 | 默认 | 回退到第 1 层的结果 |

然后两个注入器依次运行。第一个**追加系统上下文**：一份只读的 `git status --short` 加 `git log --oneline -5` 快照，后跟一个缓存断路器，追加到系统提示词末尾。值得注意的是，这份 git 快照只采集一次，对话进行中*不会*刷新。

第二个**前置用户上下文**，它做的事更微妙。它把你的 `CLAUDE.md`（从工作目录沿目录树向上遍历发现）和当前日期取出来，包进一个提醒块，然后作为一条**用户消息**注入 —— 而不是注入到系统提示词里：

```
<system-reminder>
As you answer the user's questions,
you can use the following context:
[CLAUDE.md contents + current date]
</system-reminder>
```

为什么是一条用户消息？因为 `CLAUDE.md` 经常变动。如果它待在系统提示词里，每改一次都会让缓存前缀失效，把整个静态块重新计费一遍。把它放进一条用户消息，就能把这种频繁变动与系统提示词的缓存隔离开。这是整条流水线中最重要、却也最不显眼的架构决策之一。

## 第 3 层：附件是事件驱动的，不是一次性加载的

附件收集器**每一轮都运行**，根据刚刚发生的事情重新收集上下文：

| 附件 | 触发条件 |
|-----------|---------|
| 技能列表 | 始终注入（技能 + 描述） |
| 智能体列表 | 始终注入（智能体 + 何时使用） |
| 已连接服务器的指令 | 有服务器连接或断开 |
| 动态技能 | 某次文件操作触及了新路径 |
| 待办提醒 | 已有一段时间没用过待办工具 |
| 变更文件 | 上一轮修改了文件 |
| 新诊断 | 编译器/类型检查器报出了新错误 |
| 嵌套记忆 | 加载了一个嵌套的记忆文件 |
| token 用量 | token 消耗发生了变化 |

每个附件都会被包进一个提醒标签，以一个 `<system-reminder>` 块的形式发出。这正是你可能听说过的「许多个提示词文件」背后的机制：它们并非一次性全部加载。它们由会话中的事件驱动，按需触发。大多数轮次只注入其中寥寥几个。

## 最终组装

工具描述是在代码里生成的，不是从文件里读的。每个活跃工具都暴露一个方法，由它产出自己的 `description` 字段，并从许多个小的约束片段组合而成 —— 单是 Bash 工具的描述就拼接了其中几十个。并不存在独立的「工具描述」文件；每个工具的提示词都是以编程方式组装出来的。

最后一步则构建出向外发送的请求：

```jsonc
POST /v1/messages
{
  "model": "claude-sonnet-...",
  "system": [
    { "type": "text", "text": "You are Claude Code...",
      "cache_control": { "type": "ephemeral" } },   // cache breakpoint
    { "type": "text", "text": "...dynamic sections..." }
  ],
  "messages": [
    { "role": "user", "content": [
      "<system-reminder>CLAUDE.md...</system-reminder>",
      "<system-reminder>skill listing...</system-reminder>",
      "<system-reminder>token usage...</system-reminder>",
      "fix the bug in the auth module"            // your actual input, last
    ]},
    { "role": "assistant", "content": [ ... ] }     // prior turns
  ],
  "tools": [ /* generated tool schemas */ ],
  "max_tokens": 16384,
  "thinking": { "type": "enabled", "budget_tokens": 31999 }
}
```

system 数组在静态/动态的分割处带着一个 `cache_control: ephemeral` 断点。你打出来的那条消息落在用户 content 数组的*最后*，排在所有注入的提醒之后 —— 模型先读它的上下文，最后才读你的请求。

## 拿去用

把你自己智能体的提示词组装也架构成同样这三层：

- **第 1 层 —— 一段静态的、可缓存的前缀。**把角色定义、行为规则、工具策略和语气放进一个会话内永不变化的块里，在它末尾标一个缓存断点。所有稳定的东西放在断点*之前*，所有易变的东西放在*之后*。仅这一个决策，就能把指令块的输入成本大幅削减。
- **第 2 层 —— 把易变的项目上下文隔离进用户消息。**任何经常变动的东西（你那份相当于 `CLAUDE.md` 的内容、日期、git 快照）都*不该*放进系统提示词 —— 它会让缓存前缀失效。用一个分隔符（`<system-reminder>` 或你自己的标签）把它包起来，改成作为用户消息注入。
- **第 3 层 —— 让上下文事件驱动，而非常驻。**别每一轮都加载所有可能用到的指令。根据发生了什么变化逐轮收集附件：触及了哪些文件、抛出了什么错误、哪些工具闲置了。只注入*这一轮*真正相关的内容，每条都放进一个界限清晰的块里。
- **顺序很重要：上下文在前，请求在后。**把用户真正的诉求放在 content 数组的最末尾，排在所有注入的上下文之后，这样模型在动手任务之前先读完了它的简报。
- **工具描述用代码生成，不用文件。**每个工具一个产出描述的方法，让你能动态组合约束，并保持唯一的事实来源。

**用户敲下一句话；架构决定模型实际读到什么 —— 而你把缓存边界划在哪里，决定了这要花多少钱。**

---

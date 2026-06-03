# Prompt 加载流程——一条消息如何变成一次 API 请求

*你输入的每一条消息，在抵达 Anthropic 之前都要穿过六个入口函数和三个组装层——而最终送达的内容里，大部分根本不是你打出来的。*

## 为什么重要

当你让 Claude Code「修复 auth 模块里的 bug」时,模型收到的远不止这一句话。它收到的是:一段角色定义、一组行为规则、若干工具描述、一份 git 快照、你的 `CLAUDE.md`、一份可用 skill 列表、一条 token 用量警告,以及十几个其他上下文块——每一轮都按固定顺序重新组装。如果你在构建 agent,这个顺序和缓存边界,决定了一个系统究竟是每轮只花几分钱,还是每次都把 187K token 的静态指令重新计费一遍。本章从源码层面追踪这趟旅程,让你可以照搬这套架构。

## 三个层

Claude Code 把一次请求分三个概念层组装,对应六个源码层面的入口函数:

1. **静态 system prompt**——每会话构建一次,缓存复用。(`getSystemPrompt`)
2. **上下文注入**——system 上下文追加在后,user 上下文前置在前。(`buildEffectiveSystemPrompt`、`appendSystemContext`、`prependUserContext`)
3. **逐轮 attachment**——每一轮都重新收集并重新注入。(`getAttachments`)

最后两步把这一切黏合起来:工具描述(`toolToAPISchema`)和最终组装(`queryModel`)。

```
            "fix the bug in the auth module"
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 1  Static system prompt   getSystemPrompt()          |
  | (built ONCE per session, then served from cache)          |
  |                                                           |
  |   (1) "You are Claude Code, Anthropic's official CLI..."  |
  |   (2) <system-reminder> tag explainer                     |
  |   (3) doing-tasks rules (8x no-* + 2 behavior)            |
  |   (4) executing-actions-with-care                         |
  |   (5) tool-usage policy (prefer dedicated tools > Bash)   |
  |   (6) tone-and-style + output-efficiency                  |
  |  ===== SYSTEM_PROMPT_DYNAMIC_BOUNDARY (cache split) =====  |
  |   (8) memory      <- loadMemoryPrompt()                   |
  |   (9) env_info    (OS / shell / git)                      |
  |  (10) language preference                                 |
  |  (11) mcp_instructions   [uncached]                       |
  |  (12) scratchpad path                                     |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 2  Context injection                                |
  |                                                           |
  |  buildEffectiveSystemPrompt()  -> picks WHICH prompt      |
  |  appendSystemContext()  -> git status snapshot + breaker  |
  |  prependUserContext()   -> CLAUDE.md + date, wrapped in   |
  |                            <system-reminder> USER message |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 3  Per-turn attachments   getAttachments()          |
  | (re-collected EVERY turn, event-driven)                   |
  |                                                           |
  |   skill_listing | agent_listing | changed_files |         |
  |   todo_reminders | new_diagnostics | nested_memory | ...  |
  |   each -> wrapInSystemReminder()                          |
  +-----------------------------------------------------------+
                        |
            toolToAPISchema()  ->  tools: [...]
                        |
                        v
            queryModel()  ->  POST /v1/messages
```

## Layer 1:静态 prompt 与缓存边界

`getSystemPrompt()` 返回一个字符串数组。前六项基本是常量:角色定义那一行、`<system-reminder>` 标签说明、「doing tasks」规则(八条 `no-*` 禁止项加两条行为)、谨慎操作的安全块、工具使用策略,以及语气/输出效率块。

接着是那条承重的分界线:`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`。它之前的一切都是稳定的;它之后的内容——memory、环境信息、语言偏好、MCP 指令、scratchpad 路径——可能在轮次之间发生变化。动态那一半由 `resolveSystemPromptSections()` 填充。

这条边界存在只有一个原因:**Anthropic 的 prompt caching 是按前缀匹配的。** 缓存命中要求前缀完全一致。把所有易变内容放在稳定内容*之后*,Claude Code 就能让前缀跨轮次保持恒定,命中缓存,从而在静态块上只付大约 10% 的输入 token 费用,而不是每轮都付全价。MCP 指令那一段被显式标记为不缓存,因为只要有 server 连接或断开它就会变化。

## Layer 2:上下文注入——以及为什么 CLAUDE.md 不在 system prompt 里

`buildEffectiveSystemPrompt()` 决定*用哪个* system prompt,按优先级从高到低:

| 优先级 | 来源 | 何时胜出 |
|----------|--------|-----------|
| 1 | 覆盖 prompt | 某个模式(如 loop)替换掉一切 |
| 2 | 协调器 prompt | 运行在协调器模式下 |
| 3 | Agent prompt | 子 agent 提供了自己的 `systemPrompt` |
| 4 | `--system-prompt` 参数 | 用户传入了自定义 prompt |
| 5 | 默认 | 回退到 Layer 1 的结果 |

然后两个注入器依次运行。`appendSystemContext()` 把一份只读的 `git status --short` 加 `git log --oneline -5` 快照和一个 `cacheBreaker` 追加到 system prompt 末尾。注意:这份 git 快照只采集一次,对话进行中*不会*刷新。

`prependUserContext()` 做的事更微妙。它把你的 `CLAUDE.md`(从工作目录沿目录树向上遍历发现)和当前日期取出来,包进一个 `<system-reminder>` 块,然后作为一条 **user message** 注入——而不是注入到 system prompt 里:

```
<system-reminder>
As you answer the user's questions,
you can use the following context:
[CLAUDE.md contents + current date]
</system-reminder>
```

为什么是一条 user message?因为 `CLAUDE.md` 经常变动。如果它待在 system prompt 里,每改一次都会让缓存前缀失效,把整个静态块重新计费一遍。把它放进一条 user message,就能把这种频繁变动与 system prompt 的缓存隔离开。这是整条流水线中最重要、却也最不显眼的架构决策之一。

## Layer 3:attachment 是事件驱动的,不是一次性加载的

`getAttachments()` **每一轮都运行**,根据刚刚发生的事情重新收集上下文:

| Attachment | 触发条件 |
|-----------|---------|
| `skill_listing` | 始终注入(skill 列表 + 描述) |
| `agent_listing` | 始终注入(agent 列表 + whenToUse) |
| `mcp_instructions` | 有 MCP server 连接或断开 |
| `dynamic_skill` | 某次文件操作触及了新路径 |
| `todo_reminders` | 已有一段时间没用过 TodoWrite |
| `changed_files` | 上一轮修改了文件 |
| `new_diagnostics` | 编译器/类型检查器报出了新错误 |
| `nested_memory` | 加载了一个嵌套的 memory 文件 |
| `token_usage` | token 消耗发生了变化 |

每个 attachment 都会经过 `wrapInSystemReminder()` 处理,以一个 `<system-reminder>` 块的形式发出。这正是你可能听说过的「110+ 个提示词文件」背后的机制:它们并非一次性全部加载。它们由会话中的事件驱动,按需触发。大多数轮次只注入其中寥寥几个。

## 最终组装

`toolToAPISchema()` 遍历每个活跃工具,调用它的 `.prompt()` 方法——`BashTool.prompt()`、`ReadTool.prompt()`、`EditTool.prompt()` 等等——来生成 `description` 字段。并不存在独立的「工具描述」文件;每个工具的 prompt 都是在代码里组装出来的(单是 `BashTool.prompt()` 就拼接了 30 多个约束片段)。

然后 `queryModel()` 构建出向外发送的请求:

```jsonc
POST /v1/messages
{
  "model": "claude-sonnet-4-6",
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
  "tools": [ /* Phase-5 schemas */ ],
  "max_tokens": 16384,
  "thinking": { "type": "enabled", "budget_tokens": 31999 }
}
```

system 数组在静态/动态的分割处带着一个 `cache_control: ephemeral` 断点。你打出来的那条消息落在 user content 数组的*最后*,排在所有注入的 reminder 之后——模型先读它的上下文,最后才读你的请求。

## 拿去用

把你自己 agent 的 prompt 组装也架构成同样这三层:

- **Layer 1——一段静态的、可缓存的前缀。** 把角色定义、行为规则、工具策略和语气放进一个会话内永不变化的块里,在它末尾标一个缓存断点。所有稳定的东西放在断点*之前*,所有易变的东西放在*之后*。仅这一个决策,就能把指令块的输入成本砍掉约 90%。
- **Layer 2——把易变的项目上下文隔离进 user message。** 任何经常变动的东西(你那份相当于 `CLAUDE.md` 的内容、日期、git 快照)都*不该*放进 system prompt——它会让缓存前缀失效。用一个分隔符(`<system-reminder>` 或你自己的标签)把它包起来,改成作为 user message 注入。
- **Layer 3——让上下文事件驱动,而非常驻。** 别每一轮都加载所有可能用到的指令。根据发生了什么变化逐轮收集 attachment:触及了哪些文件、抛出了什么错误、哪些工具闲置了。只注入*这一轮*真正相关的内容,每条都放进一个界限清晰的块里。
- **顺序很重要:上下文在前,请求在后。** 把用户真正的诉求放在 content 数组的最末尾,排在所有注入的上下文之后,这样模型在动手任务之前先读完了它的简报。
- **工具描述用代码生成,不用文件。** 每个工具一个 `.prompt()` 方法,让你能动态组合约束,并保持唯一的事实来源。

**用户敲下一句话;架构决定模型实际读到什么——而你把缓存边界划在哪里,决定了这要花多少钱。**

---

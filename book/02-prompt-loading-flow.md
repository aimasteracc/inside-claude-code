# The Prompt Loading Flow — How One Message Becomes an API Request

*Every message you type travels through six entry functions and three assembly layers before it ever reaches Anthropic — and most of what arrives wasn't typed by you.*

## Why it matters

When you ask Claude Code to "fix the bug in the auth module," the model receives far more than that sentence. It receives a role definition, behavior rules, tool descriptions, a git snapshot, your `CLAUDE.md`, a list of available skills, a token-usage warning, and a dozen other context blocks — assembled fresh, in a specific order, on every turn. If you build agents, the order and the caching boundaries are the difference between a system that costs pennies per turn and one that re-bills 187K tokens of static instructions every time. This chapter traces that journey at the source level so you can copy the architecture.

## The three layers

Claude Code assembles a request in three conceptual layers, mapped onto six source-level entry functions:

1. **Static system prompt** — built once per session, cached. (`getSystemPrompt`)
2. **Context injection** — system context appended, user context prepended. (`buildEffectiveSystemPrompt`, `appendSystemContext`, `prependUserContext`)
3. **Per-turn attachments** — collected and re-injected on every single turn. (`getAttachments`)

Then two final steps glue it together: tool descriptions (`toolToAPISchema`) and final assembly (`queryModel`).

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

## Layer 1: the static prompt and the cache boundary

`getSystemPrompt()` returns an array of strings. The first six are effectively constant: the role line, the `<system-reminder>` tag explainer, the "doing tasks" rules (eight `no-*` prohibitions plus two behaviors), the careful-actions safety block, the tool-usage policy, and the tone/output-efficiency block.

Then comes the load-bearing line: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`. Everything before it is stable; everything after — memory, environment info, language preference, MCP instructions, scratchpad path — can change between turns. `resolveSystemPromptSections()` fills in the dynamic half.

This boundary exists for one reason: **Anthropic's prompt caching does prefix matching.** A cache hit requires an identical prefix. By putting all the volatile content *after* the stable content, Claude Code keeps the prefix constant across turns, hits the cache, and pays roughly 10% of the input-token cost on the static block instead of full price every turn. The MCP instructions section is explicitly marked uncached because it changes whenever a server connects or disconnects.

## Layer 2: context injection — and why CLAUDE.md is not in the system prompt

`buildEffectiveSystemPrompt()` decides *which* system prompt to use, by priority, highest first:

| Priority | Source | Wins when |
|----------|--------|-----------|
| 1 | Override prompt | A mode (e.g. loop) replaces everything |
| 2 | Coordinator prompt | Running in coordinator mode |
| 3 | Agent prompt | A subagent supplies its own `systemPrompt` |
| 4 | `--system-prompt` flag | User passed a custom prompt |
| 5 | Default | Falls back to the Layer 1 result |

Then two injectors run. `appendSystemContext()` appends a read-only `git status --short` plus `git log --oneline -5` snapshot and a `cacheBreaker` to the end of the system prompt. Note: this git snapshot is captured once and does *not* refresh mid-conversation.

`prependUserContext()` does something subtler. It takes your `CLAUDE.md` (discovered by walking up the directory tree from the working directory) and the current date, wraps them in a `<system-reminder>` block, and injects them as a **user message** — not into the system prompt:

```
<system-reminder>
As you answer the user's questions,
you can use the following context:
[CLAUDE.md contents + current date]
</system-reminder>
```

Why a user message? Because `CLAUDE.md` changes often. If it lived in the system prompt, every edit would bust the cached prefix and re-bill the entire static block. Keeping it in a user message isolates that churn from the system-prompt cache. This is one of the most important — and least obvious — architectural decisions in the whole pipeline.

## Layer 3: attachments are event-driven, not loaded

`getAttachments()` runs **on every turn** and re-collects context based on what just happened:

| Attachment | Trigger |
|-----------|---------|
| `skill_listing` | Always injected (skills + descriptions) |
| `agent_listing` | Always injected (agents + whenToUse) |
| `mcp_instructions` | An MCP server connected or disconnected |
| `dynamic_skill` | A file operation touched a new path |
| `todo_reminders` | TodoWrite hasn't been used in a while |
| `changed_files` | The previous turn modified files |
| `new_diagnostics` | The compiler/type-checker reported new errors |
| `nested_memory` | A nested memory file was loaded |
| `token_usage` | Token consumption changed |

Each attachment is passed through `wrapInSystemReminder()` and emitted as a `<system-reminder>` block. This is the mechanism behind the "110+ prompt files" you may have heard about: they are not all loaded at once. They fire on demand, driven by events in the session. Most turns inject only a handful.

## Final assembly

`toolToAPISchema()` walks every active tool and calls its `.prompt()` method — `BashTool.prompt()`, `ReadTool.prompt()`, `EditTool.prompt()`, and so on — to produce the `description` field. There are no standalone "tool description" files; each tool's prompt is assembled in code (`BashTool.prompt()` alone concatenates 30+ constraint fragments).

`queryModel()` then builds the outgoing request:

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

The system array carries a `cache_control: ephemeral` breakpoint at the static/dynamic split. Your typed message lands *last* in the user content array, after all the injected reminders — the model reads its context first, your request last.

## Steal this

Architect your own agent's prompt assembly as these same three layers:

- **Layer 1 — a static, cached prefix.** Put your role definition, behavior rules, tool policy, and tone in one block that never changes within a session. Mark a cache breakpoint at its end. Everything stable goes *before* the breakpoint; everything volatile goes *after*. This single decision can cut your input cost by ~90% on the instruction block.
- **Layer 2 — isolate volatile project context into user messages.** Anything that changes often (your equivalent of `CLAUDE.md`, the date, a git snapshot) does *not* belong in the system prompt — it busts the cache prefix. Wrap it in a delimiter (`<system-reminder>` or your own tag) and inject it as a user message instead.
- **Layer 3 — make context event-driven, not always-on.** Don't load every possible instruction every turn. Collect attachments per turn based on what changed: files touched, errors raised, tools idle. Inject only what's relevant *this* turn, each in a clearly delimited block.
- **Order matters: context first, request last.** Put the user's actual ask at the very end of the content array, after all injected context, so the model reads its briefing before the task.
- **Generate tool descriptions in code, not files.** A `.prompt()` method per tool lets you compose constraints dynamically and keep one source of truth.

**The user types one sentence; the architecture decides what the model actually reads — and where you draw the cache boundary decides what it costs.**

---

# The Prompt Loading Flow — How One Message Becomes an API Request

*Every message you type passes through several assembly stages before it ever reaches the model — and most of what arrives wasn't typed by you.*

## Why it matters

When you ask Claude Code to "fix the bug in the auth module," the model receives far more than that sentence. It receives a role definition, behavior rules, tool descriptions, a git snapshot, your `CLAUDE.md`, a list of available skills, a token-usage warning, and a dozen other context blocks — assembled fresh, in a specific order, on every turn. If you build agents, the order and the caching boundaries are the difference between a system that costs pennies per turn and one that re-bills its entire static instruction block every time. This chapter traces that journey from observed behavior so you can copy the architecture.

## The three layers

A reasonable reading of the observed behavior is that the request is assembled in three conceptual layers:

1. **Static system prompt** — built once per session, then served from cache.
2. **Context injection** — system context appended to it, user context prepended around it.
3. **Per-turn attachments** — collected and re-injected on every single turn.

Two final steps glue it together: each tool's description is generated, and the whole thing is assembled into the outgoing request.

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

## Layer 1: the static prompt and the cache boundary

The static system prompt behaves like an ordered array of text blocks. The first several are effectively constant: the role line, an explainer for the reminder tag the agent uses to inject context, the core "doing tasks" rules (a set of prohibitions plus a couple of behaviors), a safety block about acting carefully, the tool-usage policy, and the tone/output efficiency block.

Then comes the load-bearing line: a **cache boundary** that separates the static half from the dynamic half. Everything before it is stable; everything after — memory, environment info, language preference, connected-server instructions, scratchpad path — can change between turns.

This boundary exists for one reason: **prompt caching does prefix matching.** A cache hit requires an identical prefix. By putting all the volatile content *after* the stable content, the agent keeps the prefix constant across turns, hits the cache, and pays a small fraction of the input-token cost on the static block instead of full price every turn. The connected-server instructions section is left uncached, because it changes whenever a server connects or disconnects.

## Layer 2: context injection — and why CLAUDE.md is not in the system prompt

Before injection, the agent decides *which* base system prompt to use, by priority, highest first:

| Priority | Source | Wins when |
|----------|--------|-----------|
| 1 | Override prompt | A mode (e.g. loop) replaces everything |
| 2 | Coordinator prompt | Running in coordinator mode |
| 3 | Agent prompt | A subagent supplies its own system prompt |
| 4 | Custom-prompt flag | User passed a custom prompt on the command line |
| 5 | Default | Falls back to the Layer 1 result |

Then two injectors run. The first **appends system context**: a read-only snapshot of `git status --short` plus `git log --oneline -5`, followed by a cache breaker, added to the end of the system prompt. Notably, this git snapshot is captured once and does *not* refresh mid-conversation.

The second **prepends user context**, and it does something subtler. It takes your `CLAUDE.md` (discovered by walking up the directory tree from the working directory) and the current date, wraps them in a reminder block, and injects them as a **user message** — not into the system prompt:

```
<system-reminder>
As you answer the user's questions,
you can use the following context:
[CLAUDE.md contents + current date]
</system-reminder>
```

Why a user message? Because `CLAUDE.md` changes often. If it lived in the system prompt, every edit would bust the cached prefix and re-bill the entire static block. Keeping it in a user message isolates that churn from the system-prompt cache. This is one of the most important — and least obvious — architectural decisions in the whole pipeline.

## Layer 3: attachments are event-driven, not loaded

The attachment collector runs **on every turn** and re-collects context based on what just happened:

| Attachment | Trigger |
|-----------|---------|
| Skill listing | Always injected (skills + descriptions) |
| Agent listing | Always injected (agents + when-to-use) |
| Connected-server instructions | A server connected or disconnected |
| Dynamic skill | A file operation touched a new path |
| Todo reminders | The todo tool hasn't been used in a while |
| Changed files | The previous turn modified files |
| New diagnostics | The compiler/type-checker reported new errors |
| Nested memory | A nested memory file was loaded |
| Token usage | Token consumption changed |

Each attachment is wrapped in a reminder tag and emitted as a `<system-reminder>` block. This is the mechanism behind the "many prompt files" you may have heard about: they are not all loaded at once. They fire on demand, driven by events in the session. Most turns inject only a handful.

## Final assembly

Tool descriptions are generated in code, not read from files. Each active tool exposes a method that produces its `description` field, composing it from many small constraint fragments — the Bash tool's description alone concatenates dozens of them. There are no standalone "tool description" files; each tool's prompt is assembled programmatically.

The final step then builds the outgoing request:

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

The system array carries a `cache_control: ephemeral` breakpoint at the static/dynamic split. Your typed message lands *last* in the user content array, after all the injected reminders — the model reads its context first, your request last.

## Steal this

Architect your own agent's prompt assembly as these same three layers:

- **Layer 1 — a static, cached prefix.** Put your role definition, behavior rules, tool policy, and tone in one block that never changes within a session. Mark a cache breakpoint at its end. Everything stable goes *before* the breakpoint; everything volatile goes *after*. This single decision can cut your input cost dramatically on the instruction block.
- **Layer 2 — isolate volatile project context into user messages.** Anything that changes often (your equivalent of `CLAUDE.md`, the date, a git snapshot) does *not* belong in the system prompt — it busts the cache prefix. Wrap it in a delimiter (`<system-reminder>` or your own tag) and inject it as a user message instead.
- **Layer 3 — make context event-driven, not always-on.** Don't load every possible instruction every turn. Collect attachments per turn based on what changed: files touched, errors raised, tools idle. Inject only what's relevant *this* turn, each in a clearly delimited block.
- **Order matters: context first, request last.** Put the user's actual ask at the very end of the content array, after all injected context, so the model reads its briefing before the task.
- **Generate tool descriptions in code, not files.** A description-producing method per tool lets you compose constraints dynamically and keep one source of truth.

**The user types one sentence; the architecture decides what the model actually reads — and where you draw the cache boundary decides what it costs.**

---

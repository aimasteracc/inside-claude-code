# Chapter 1 — The Five-Layer Prompt Architecture

*A top-tier coding agent's "system prompt" isn't a string. It's a library of many small, versioned units, each doing exactly one job.*

## Why it matters

Most people imagine an agent's system prompt as one heroic block of text — a few thousand words of "You are a helpful assistant who writes clean code." That mental model is why most homegrown agents plateau: a monolithic prompt can't be versioned, can't be A/B tested, can't be selectively loaded, and rots the moment it grows past a screen.

Claude Code's prompt layer behaves as though it were built the opposite way. Observed behavior is consistent with a system decomposed into **dozens of independent units**, grouped into roughly **six categories**, each unit carrying its own name, short description, and a version tag. Understanding this decomposition is the foundation for everything else in this book — because every pattern in Part II is a consequence of this one architectural decision.

## The six categories

Watching the agent across many sessions, you can infer a clean separation of concerns into about six families of prompt unit:

| Category | Job | One-line memory |
|----------|-----|-----------------|
| **System prompts** | Core persona + behavioral rules | "Who the AI is, how it should act" |
| **System reminders** | Conditionally-triggered event notices | "Something happened — pay attention" |
| **Tool descriptions** | The cognitive boundary of each tool | "What the AI can use, and how" |
| **Agent prompts** | Personas for sub-agents | "Each sub-agent's job" |
| **Data** | SDK/API reference knowledge injection | "The AI's reference manual" |
| **Skills** | Built-in capability packs for specific tasks | "One-trigger ability bundles" |

```
many small prompt units = ~6 categories x separation of concerns
┌─────────────────────────────────────────────────────┐
│  System prompts   ← persona + rules ("who I am")      │
│  System reminders ← event notices ("watch this")      │
│  Tool descriptions← capability edges ("what I do")    │
│  Agent prompts    ← sub-agent personas ("divide")     │
│  Data             ← knowledge injection ("manual")    │
│  Skills           ← built-in abilities ("1-click")    │
└─────────────────────────────────────────────────────┘
```

Read that table again with an engineer's eye. This isn't prompt *writing*. It's prompt *architecture* — the same instinct that splits a codebase into modules with single responsibilities. There are many such units, and individually each is tiny — often just a few dozen tokens — which is exactly what lets them be composed, versioned, and loaded on demand.

## Layer 1: System prompts — and the surprising weight of "no"

The system-prompt layer holds the agent's persona and its hard rules. The most revealing artifacts are the **constraints** — and there are a lot of them. Each is small, and most are framed as a prohibition. Among the most important:

| Constraint | Rule |
|------------|------|
| Don't over-abstract | Don't build an abstraction for a one-off |
| Don't over-build | Don't improve beyond what was asked |
| No compatibility shims | Delete unused code fully; no shims |
| No time estimates | Don't promise how long work will take |
| No defensive over-handling | Don't guard against impossible states |
| Read before modifying | Read the code before changing it |
| Minimize new files | Prefer editing existing files |
| Security baseline | Avoid injection, XSS, and the OWASP set |

Notice the framing. Most of these are stated as things *not* to do. This is the single most important observation in the whole prompt set, and we devote Chapter 3 to it: **the majority of a great agent's instructions define what it must *not* do.** Capability is the easy part — the model already wants to write code. The engineering is in the guardrails.

## Layer 2: Tool descriptions — capability is what the AI can *see*

A tool the model can't see doesn't exist. The tool-description layer is where the agent's *cognitive map* of its own abilities is drawn. The system prompt even biases the agent toward purpose-built tools over raw shell:

| Operation | Dedicated tool | Not |
|-----------|----------------|-----|
| Read a file | Read | cat/head/tail/sed |
| Edit a file | Edit | sed/awk |
| Create a file | Write | echo / heredoc |
| Find files | Glob | find/ls |
| Search content | Grep | grep/rg |

And the riskiest tool gets the most containment. Of all the tools, the raw shell carries by far the densest set of dedicated constraints — many distinct rules, clustered into families:

| Constraint family | Key rules |
|-------------------|-----------|
| Sandbox | Sandbox by default; evidence-based escape; retry policy |
| Sleep | Keep waits short; never poll; prefer a check |
| Git | No hook-skipping; no force-push; prefer new commits |
| Path/command | Absolute paths; quote spaces; `&&` to chain |

The principle generalizes: **the more powerful the tool, the denser the containment net around it.** Each rule reads like it traces back to a real incident — otherwise it wouldn't have earned its own versioned entry.

## Layer 3: System reminders — perception, not instruction

The system reminders are the agent's *interrupt handlers*. They are not always-on; they fire on conditions:

| Class | Examples | Fires when |
|-------|----------|-----------|
| Filesystem | File was modified, truncated, or is now empty | IDE or external edits |
| Plan mode | Planning is active; planning was exited | Entering/leaving planning |
| Hook | A hook succeeded; a hook blocked an action | A hook returns |
| Security | A safety check after reading a file | After reading a file |
| Session | Token usage; budget spend | Session state changes |

This is event-driven prompting: context is injected *when relevant*, not crammed in permanently. It is how the agent stays aware of a changing world without paying the token cost of describing that whole world on every turn.

## Layers 4–6: Delegation, knowledge, and skills

- **Agent prompts** give each sub-agent its own persona. The most substantial one is a security watcher — a dedicated monitor for autonomous runs and noticeably larger than the rest. Others include a verification specialist (build + test + check) and background routines that consolidate memory between sessions.
- **Data** injects reference knowledge — Claude API examples across Python, TypeScript, Java, Go, C#, and more — loaded on demand rather than parked in context (the "static RAG" idea we revisit in Chapter 4).
- **Skills** are one-trigger capability packs; the heaviest of them — a guide to building LLM-powered apps — is among the largest single prompt units in the whole system.

## From source to theory

One more thing the observation makes clear — there's a path from product behavior to teachable theory, in three steps:

```
product behavior        ← the agent as it actually runs
    │ observed and distilled into ↓
a structured prompt model ← many small units, organized and version-aware
    │ interpreted into ↓
the 5-layer theory        ← the framework explaining *why* it works
```

The lesson: the people who understand this system best didn't stop at a blog post. They watched the behavior closely, distilled a structured model of it, and *then* built a theory. This book hands you the result of that work.

## Steal this

Port the architecture, not just the rules:

- **Decompose your system prompt into files**, one responsibility each. Even three files (`persona.md`, `constraints.md`, `tools.md`) beats one monolith. You gain versioning, diffing, and selective loading immediately.
- **Lead with constraints.** Before writing what your agent *does*, write the 8–10 things it must never do — phrased as `Don't X. Because Y. Except when Z.` (the exact template from Chapter 3).
- **Version every rule.** Add a date or version tag the day a rule is born. That tag is your incident log: six months in, the diff between versions *is* your agent's accumulated wisdom.
- **Make context event-driven.** Anything you currently prepend on every turn — ask whether it could instead be injected only when its triggering condition occurs. Your token bill and your signal-to-noise both improve.
- **Wrap your most dangerous tool the tightest.** Whatever your "Bash" is — a shell, a DB write, a payment call — give it the densest, most specific guardrails in your stack.

**Stop writing a prompt. Start architecting a prompt system — modular, versioned, mostly made of "no."**

---

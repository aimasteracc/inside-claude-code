# Chapter 1 — The Five-Layer Prompt Architecture

*A top-tier coding agent's "system prompt" isn't a string. It's a filesystem of 110+ versioned files, each doing exactly one job.*

## Why it matters

Most people imagine an agent's system prompt as one heroic block of text — a few thousand words of "You are a helpful assistant who writes clean code." That mental model is why most homegrown agents plateau: a monolithic prompt can't be versioned, can't be A/B tested, can't be selectively loaded, and rots the moment it grows past a screen.

Claude Code's prompt layer is built the opposite way. It is decomposed into **110+ independent files**, grouped into **six categories**, each file carrying its own `name`, `description`, and `ccVersion`. Understanding this decomposition is the foundation for everything else in this book — because every pattern in Part II is a consequence of this one architectural decision.

## The six categories

Reverse-engineering the prompt set (tracked against release `v2.1.97`) reveals a clean separation of concerns:

| Category | Count | Prefix | Job | One-line memory |
|----------|-------|--------|-----|-----------------|
| **System prompts** | 67+ | `system-prompt-` | Core persona + behavioral rules | "Who the AI is, how it should act" |
| **System reminders** | 37+ | `system-reminder-` | Conditionally-triggered event notices | "Something happened — pay attention" |
| **Tool descriptions** | 73+ | `tool-description-` | The cognitive boundary of each tool | "What the AI can use, and how" |
| **Agent prompts** | 32+ | `agent-prompt-` | Personas for sub-agents | "Each sub-agent's job" |
| **Data** | 25+ | `data-` | SDK/API reference knowledge injection | "The AI's reference manual" |
| **Skills** | 16+ | `skill-` | Built-in capability packs for specific tasks | "One-trigger ability bundles" |

```
110+ prompt files = 6 categories x separation of concerns
┌─────────────────────────────────────────────────────┐
│  System prompts (67)  ← persona + rules ("who I am")  │
│  System reminders (37)← event notices ("watch this")  │
│  Tool descriptions(73)← capability edges ("what I do")│
│  Agent prompts (32)   ← sub-agent personas ("divide") │
│  Data (25)            ← knowledge injection ("manual")│
│  Skills (16)          ← built-in abilities ("1-click")│
└─────────────────────────────────────────────────────┘
```

Read that table again with an engineer's eye. This isn't prompt *writing*. It's prompt *architecture* — the same instinct that splits a codebase into modules with single responsibilities.

## Layer 1: System prompts — and the surprising weight of "no"

The system-prompt layer holds the agent's persona and its hard rules. The most revealing artifacts are the **constraints** — and there are a lot of them. Eight of the most important, with their actual token costs:

| File | Tokens | Rule |
|------|--------|------|
| `no-premature-abstractions` | 72 | Don't build an abstraction for a one-off |
| `no-unnecessary-additions` | 78 | Don't improve beyond what was asked |
| `no-compatibility-hacks` | 52 | Delete unused code fully; no shims |
| `no-time-estimates` | 47 | Don't give time estimates |
| `no-unnecessary-error-handling` | 64 | Don't guard against impossible states |
| `read-before-modifying` | 46 | Read the code before changing it |
| `minimize-file-creation` | 47 | Prefer editing existing files |
| `security` | 67 | Avoid injection, XSS, and the OWASP set |

Notice the naming. Six of the eight start with `no-`. This is the single most important observation in the whole prompt set, and we devote Chapter 3 to it: **the majority of a great agent's instructions define what it must *not* do.** Capability is the easy part — the model already wants to write code. The engineering is in the guardrails.

## Layer 2: Tool descriptions — capability is what the AI can *see*

A tool the model can't see doesn't exist. The tool-description layer — 73+ files — is where the agent's *cognitive map* of its own abilities is drawn. The system prompt even biases the agent toward purpose-built tools over raw shell:

| Operation | Dedicated tool | Not |
|-----------|----------------|-----|
| Read a file | Read | cat/head/tail/sed |
| Edit a file | Edit | sed/awk |
| Create a file | Write | echo / heredoc |
| Find files | Glob | find/ls |
| Search content | Grep | grep/rg |

And the riskiest tool gets the most containment. **Bash alone carries 30+ dedicated constraint files**:

| Constraint family | Count | Key rules |
|-------------------|-------|-----------|
| Sandbox | 12 | Sandbox by default; evidence-based escape; retry policy |
| Sleep | 4 | Keep 1–5s; never poll; prefer a check |
| Git | 4 | No hook-skipping; no force-push; prefer new commits |
| Path/command | 10+ | Absolute paths; quote spaces; `&&` to chain |

The principle generalizes: **the more powerful the tool, the denser the containment net around it.** Each rule traces back to a real incident — otherwise it wouldn't have earned a version number.

## Layer 3: System reminders — perception, not instruction

The 37+ system reminders are the agent's *interrupt handlers*. They are not always-on; they fire on conditions:

| Class | Examples | Fires when |
|-------|----------|-----------|
| Filesystem | `file-modified`, `file-truncated`, `file-exists-but-empty` | IDE or external edits |
| Plan mode | `plan-mode-is-active`, `exited-plan-mode` | Entering/leaving planning |
| Hook | `hook-success`, `hook-blocking-error` | A hook returns |
| Security | `malware-analysis-after-read` | After reading a file |
| Session | `token-usage`, `usd-budget` | Session state changes |

This is event-driven prompting: context is injected *when relevant*, not crammed in permanently. It is how the agent stays aware of a changing world without paying the token cost of describing that whole world on every turn.

## Layers 4–6: Delegation, knowledge, and skills

- **Agent prompts (32+)** give each sub-agent its own persona. The largest is the `security-monitor` at ~6,426 tokens — a dedicated watcher for autonomous runs. Others include a `verification-specialist` (build + test + check) and memory routines with names like `dream-memory-consolidation`.
- **Data (25+)** injects reference knowledge — Claude API examples across Python, TypeScript, Java, Go, C#, and more — loaded on demand rather than parked in context (the "static RAG" idea we revisit in Chapter 4).
- **Skills (16+)** are one-trigger capability packs; the largest, `build-llm-powered-apps`, is ~7,556 tokens.

## The three-source hierarchy

One more thing the reverse-engineering makes clear — there are three layers of artifact, bottom-up:

```
cc-source/src/              ← product source (the prompts are hardcoded here)
    │ extracted into ↓
claude-code-system-prompts/ ← 110+ prompts, organized, version-tracked
    │ interpreted by ↓
prompt-mastery curriculum   ← the 5-layer theory explaining *why*
```

The lesson: the people who understand this system best didn't read a blog post. They read the source, extracted the prompts, and *then* built a theory. This book hands you the result of that work.

## Steal this

Port the architecture, not just the rules:

- **Decompose your system prompt into files**, one responsibility each. Even three files (`persona.md`, `constraints.md`, `tools.md`) beats one monolith. You gain versioning, diffing, and selective loading immediately.
- **Lead with constraints.** Before writing what your agent *does*, write the 8–10 things it must never do — phrased as `Don't X. Because Y. Except when Z.` (the exact template from Chapter 3).
- **Version every rule.** Add a date or version tag the day a rule is born. That tag is your incident log: six months in, the diff between versions *is* your agent's accumulated wisdom.
- **Make context event-driven.** Anything you currently prepend on every turn — ask whether it could instead be injected only when its triggering condition occurs. Your token bill and your signal-to-noise both improve.
- **Wrap your most dangerous tool the tightest.** Whatever your "Bash" is — a shell, a DB write, a payment call — give it the densest, most specific guardrails in your stack.

**Stop writing a prompt. Start architecting a prompt system — modular, versioned, mostly made of "no."**

---

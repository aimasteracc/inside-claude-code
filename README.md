<div align="center">

# Inside Claude Code
### A Reverse-Engineering Field Guide to Agent Engineering

**12 prompt-design patterns · 7 agent failure modes · the source-level mechanics behind the world's leading agentic coding tool.**

[**📘 Get the full book + toolkit → $29**](https://gumroad.com/l/inside-claude-code)  ·  [Read 2 free chapters ↓](#-read-free)

*Launch price $19 for the first 72 hours — use code `SHIPIT`.*

</div>

> 🚧 **Pre-launch.** The two free chapters below are live now. The full book + checkout go live within 48h — ⭐ **star this repo** to get notified.

---

## Why this exists

Everyone is building agents. Almost no one has taken apart one that actually works.

Claude Code is the reference implementation of the autonomous coding agent — it runs for hours, edits dozens of files, tests its own work, and ships. Underneath it is a body of hard-won engineering: **110+ versioned prompt files**, a precise request-assembly pipeline, a context engine that survives 200K-token sessions, and a set of guardrails where **~60% of the instructions describe what the agent must *not* do.**

This book reverse-engineers that machine and hands you the patterns — organized, translated into plain English, and turned into things you can paste into your own agent today.

> If you've ever watched an autonomous agent confidently declare a half-finished task "complete," there is a known fix for that. It's in Chapter 6.

## Who it's for

- Engineers **building agents** on the Claude or OpenAI APIs who want production-grade patterns, not blog-post platitudes.
- **Power users** of Claude Code / Cursor who want to understand — and bend — the tool they live in.
- Anyone shipping LLM features who is tired of agents that stub out work, inflate their own grades, or panic near the context limit.

## What's inside

**40+ pages · ~14,000 words · 10 chapters · a copy-paste toolkit.**

| # | Chapter | You walk away with |
|---|---------|--------------------|
| 1 | The Five-Layer Prompt Architecture | Why a great agent's prompt is a *filesystem*, not a string |
| 2 | The Prompt Loading Flow | How one message becomes an API request — and where the cache boundary saves you 187K tokens/turn |
| 3 | Twelve Patterns I — Behavior & Architecture | Constraint-first, single-responsibility, event-driven, cognitive boundaries, delegation, progressive safety |
| 4 | Twelve Patterns II — Runtime & Knowledge | Compaction, output efficiency, mode switching, the memory trinity, skillification, observability |
| 5 | Print Mode & Autonomous Loops | Signal handling, the ~187K compaction trigger, designing a headless loop that won't wedge |
| 6 | The Seven Agent Failure Modes | Each failure, its root cause, and the *verified* fix |
| 7 | Sub-agents & Layered Delegation | The isolation contract — and the messaging asymmetry that silently kills most multi-agent setups |
| 8 | The Autonomous Development Pipeline | A full prompt-to-verified-build loop with an expert-review panel |
| 9 | Designing Your Own Harness | The whole book reduced to five buildable subsystems |
| 10 | The Checklist & Templates | A pre-flight checklist + paste-ready templates |

### The toolkit (ships with the book)
- 🗂️ **`CLAUDE.md.template`** — a constraint-first operating-rules file for any repo
- 🤖 **`subagent-prompt.template.md`** — brief sub-agents so they don't silently fail
- 🔄 **`context-handoff.template.md`** — survive a context reset without losing the thread
- 📄 **Patterns cheat sheet** (1 page) + **Failure-modes checklist** (1 page) — print and pin

## <a name="-read-free"></a>📖 Read free

No email required. Read the first two chapters right here:

- [**Introduction — Why Take a Coding Agent Apart?**](book/00-introduction.md)
- [**Chapter 1 — The Five-Layer Prompt Architecture**](book/01-five-layer-architecture.md)
- [**Chapter 2 — The Prompt Loading Flow**](book/02-prompt-loading-flow.md)

If the mechanism-level depth is what you've been looking for, the other eight chapters and the toolkit are one click away.

<div align="center">

### [📘 Get the full book + toolkit → $29](https://gumroad.com/l/inside-claude-code)
*First 72 hours: $19 with code `SHIPIT`. 30-day money-back guarantee — no questions.*

</div>

## Why trust it

These patterns were distilled from **public reverse-engineering of Claude Code's prompt set** (tracked to release `v2.1.97`), source-level traces of its request pipeline, and Anthropic's own published guidance on long-running agent harnesses. Every claim is anchored to a concrete artifact — a file name, a version number, a token count — not vibes. This is analysis of a public system, not leaked proprietary code.

## FAQ

**Is this just the Claude Code docs?** No. The docs tell you how to *use* the tool. This book reverse-engineers how it's *built*, and turns that into patterns for your own agents.

**I don't use Claude Code — is it still useful?** Yes. The patterns are model- and tool-agnostic. Chapters 3, 4, 6, 7, and 9 apply to any agent on any API.

**What format?** A PDF plus the raw Markdown, and the toolkit files. Yours forever, free updates.

**Refunds?** 30 days, no questions asked.

## License

The two free preview chapters and the README are shared for evaluation. The full book and toolkit are commercial — please don't redistribute. © 2026. All rights reserved.

<div align="center">

*Study the machine that works, and you stop guessing at the one you're building.*

</div>

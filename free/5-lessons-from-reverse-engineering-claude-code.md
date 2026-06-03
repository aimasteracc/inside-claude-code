# I reverse-engineered Claude Code's prompt architecture. Here's what it taught me about building agents.

Everyone is building AI agents right now. Almost no one has taken apart one that already works.

So I did. I spent weeks studying how Claude Code behaves — its prompt layer, its request pipeline, its context engine, its failure modes — and inferring the architecture underneath. I came away with a stack of patterns I now use in every agent I build. Here are the five that changed how I work.

## 1. The "system prompt" is a filesystem, not a string

The first surprise: Claude Code's system prompt doesn't behave like a heroic block of text. It behaves like **many small, independent units**, grouped into roughly six categories — system prompts, system reminders, tool descriptions, sub-agent personas, data, and skills. Each unit reads as if it carries its own name, short description, and a version tag.

That structure is the whole game. A monolithic megaprompt can't be versioned, can't be A/B tested, can't be selectively loaded, and rots the moment it outgrows a screen. A *filesystem* of single-responsibility prompt files can do all four. It's the same instinct that splits a codebase into modules — applied to instructions.

If you take one thing from this post: stop writing a prompt. Start architecting a prompt system. Even three files — `persona.md`, `constraints.md`, `tools.md` — beats one wall of text.

## 2. The majority of a great agent's prompt is "no"

Look at the agent's constraints and a pattern jumps out — there's a dedicated, individually-versioned rule for each prohibition, things like:

- don't build an abstraction for a one-off use
- don't improve beyond what was asked
- don't guard against impossible states
- read the code before you change it

Most of the instruction budget goes to defining what the agent must **not** do. And it makes sense: capability is the easy part — the model already wants to write code. The engineering is in the guardrails. Each prohibition reads like a line in an incident log, because that's effectively what it is: a mistake someone watched the agent make, turned into a rule.

The reusable shape: `Don't <X>. <why>. <exception>.`

When I started writing my agents' prohibitions *before* their capabilities, the quality jump was immediate.

## 3. There's a cache boundary in the request pipeline (and it's worth real money)

Trace how Claude Code assembles a request and you find three layers: a static system prompt built once per session, context injected around it, and per-turn attachments collected fresh every single turn. The static layer sits above an explicit cache boundary; the dynamic state sits below it.

Miss this in your own agent and you re-bill your entire static instruction set — easily 100K+ tokens of rules and tool schemas — on *every turn*. Put a hard line between "static, cache this" and "dynamic, re-send this," and your cost per turn can drop by an order of magnitude with zero behavior change. This is the cheapest performance win in agent engineering and almost nobody does it on their first build.

## 4. Autonomous agents fail in seven predictable ways

This is the part I wish I'd had a year ago. Long-running agents don't fail randomly — they fail in a small set of recognizable modes:

1. **One-shot impulse** — tries to do everything at once, exhausts context, leaves fragments.
2. **Premature "done"** — declares victory with most of the work unfinished.
3. **Context anxiety** — senses it's near the context limit and starts rushing to wrap up, even with capacity left.
4. **Self-evaluation inflation** — grades its own broken work 9/10 (LLMs are generous to LLM output).
5. **Skipping E2E** — unit tests pass, the actual button does nothing.
6. **Stub-ification** — the UI looks complete; the interactions are hollow.
7. **Spec cascade** — a planner's one wrong detail poisons everything downstream.

Each has a verified fix. My favorite, because it's so counterintuitive until you've been burned: **the thing that generates cannot be the thing that grades.** Split the generator and the evaluator into separate roles, calibrate the evaluator to be harsh, and the self-inflation problem largely evaporates. Same with "premature done" — gate completion behind a machine-checkable contract (a JSON checklist whose items can only flip from `false` to `true`, never be deleted) and the agent can't talk its way out of the remaining work.

## 5. Sub-agents have a messaging asymmetry that silently kills multi-agent systems

When you scale to multiple agents, there's a trap nobody warns you about. A lead agent can message a sub-agent. But sub-agents generally **cannot reliably message each other** — they're one-shot, isolated workers with no inbox and no ability to wait.

So any design that assumes peer-to-peer messaging fails *silently*: the agent that was told to "wait for a message from the other agent" has no mechanism to wait, so it either aborts or runs on stale assumptions. The pattern that actually works is **memory-as-bus, lead-orchestrated phases**: all shared state lives in a common store, the lead spawns workers and verifies their outputs before spawning the next phase, and no worker is ever told to wait on a peer.

I lost a week to this before I understood it. You don't have to.

---

## Why this matters

None of these patterns are Claude-specific. They're what production-grade agent engineering looks like, surfaced by studying the best public example of it. Whether you're on the Claude API, the OpenAI API, or rolling your own harness, the same five ideas apply: architect your prompts as files, lead with constraints, set a cache boundary, defend against the seven failure modes, and orchestrate sub-agents through a lead.

I wrote the full teardown up as a field guide — the twelve patterns, all seven failure modes with their fixes, the request pipeline traced at the source level, sub-agent orchestration, a build-your-own-harness chapter, and paste-ready templates. **Two chapters are free, no email gate**, here:

👉 **https://github.com/aimasteracc/inside-claude-code**

If you build agents, I'd genuinely love to hear which of these five matches your own scars. The failure-mode taxonomy especially — I suspect everyone who's run an agent unattended has met at least three of the seven.

*Tags: #ai #llm #claude #agents #promptengineering*

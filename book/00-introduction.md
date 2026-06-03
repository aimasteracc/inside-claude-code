# Introduction — Why Take a Coding Agent Apart?

*The best way to learn agent engineering is to study the best agent that exists — closely, mechanically, line by line.*

In late 2025, "AI coding tool" stopped meaning autocomplete and started meaning **an autonomous engineer that runs for hours**: reading your repo, planning, editing dozens of files, running tests, fixing its own mistakes, and shipping. Claude Code is the reference implementation of that idea. Whatever you're building — a coding agent, a research agent, a customer-support agent, an internal automation — Claude Code has already solved, in production, the hard problems you are about to hit.

So this book does something unusual. It takes a working, best-in-class agent apart and shows you the mechanism underneath: how its prompts are structured, how a single user message becomes an API request, how it stays sane across a 200K-token session, why 60% of its instructions describe what *not* to do, how it delegates to sub-agents without corrupting its own context, and the seven specific ways autonomous agents fail in the wild — each with a fix that has been verified in real harnesses.

## What this is

A **field guide**, not a tutorial. Every chapter follows the same shape:

1. **The mechanism** — what the system actually does, with concrete artifacts: file names, version numbers, token counts, trigger conditions. No hand-waving.
2. **Steal this** — how to port that mechanism into *your* agent, your `CLAUDE.md`, your prompt stack.

The patterns here were distilled from public reverse-engineering of Claude Code's 110+ prompt files, source-level traces of its request pipeline, and Anthropic's own published guidance on long-running agent harnesses. They are organized into something you can act on.

## Who it's for

- Engineers **building agents** on top of the Claude or OpenAI APIs who want production-grade patterns instead of reinventing them.
- **Power users** of Claude Code, Cursor, or similar tools who want to understand — and bend — the machine they rely on every day.
- Anyone who has watched an autonomous agent confidently declare victory on a half-finished task and thought: *there has to be a known fix for this.* (There is. Chapter 6.)

This is not an introduction to LLMs. You should know what a system prompt and a tool call are. Everything past that, we build up.

## The map

| Part | Chapters | What you walk away with |
|------|----------|--------------------------|
| **I — The Architecture** | 1–2 | How a top-tier agent's prompts and request pipeline are actually structured |
| **II — The Patterns** | 3–4 | Twelve reusable prompt-design patterns you can apply tomorrow |
| **III — Running Loose** | 5–8 | Headless loops, the 7 failure modes, sub-agent delegation, a full autonomous pipeline |
| **IV — Build Your Own** | 9–10 | A synthesis chapter plus copy-paste templates and a shippable checklist |

Chapters 1 and 2 are free — read them, and if the mechanism-level depth is what you've been looking for, the rest is waiting.

You don't need to read in order after Part I. If you're debugging a flaky autonomous agent *right now*, skip to Chapter 6. If you're designing a multi-agent system, Chapter 7 will save you a week of silent failures.

Let's open it up.

**Study the machine that works, and you stop guessing at the one you're building.**

---

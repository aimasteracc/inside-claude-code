# The 12 Patterns — One-Page Cheat Sheet

> From *Inside Claude Code*. Print it, pin it, apply it.

```
┌────────────┬────────────┬─────────────┬──────────────────┐
│ BEHAVIOR   │ ARCHITECTURE│ RUNTIME     │ KNOWLEDGE         │
├────────────┼────────────┼─────────────┼──────────────────┤
│ 1 Constraint│ 4 Cognitive│ 7 Context   │ 10 Memory         │
│   -First    │   Boundary │   Compaction│    Trinity        │
│ 2 Single    │ 5 Layered  │ 8 Output    │ 11 Skillification │
│   Respons.  │   Delegation│   Efficiency│                  │
│ 3 Event     │ 6 Progress.│ 9 Mode      │ 12 Observability  │
│   -Driven   │   Safety   │   Switching │                  │
└────────────┴────────────┴─────────────┴──────────────────┘
```

| # | Pattern | In one line | How to apply |
|---|---------|-------------|--------------|
| 1 | **Constraint-First** | ~60% of a great agent's prompt says what NOT to do | List prohibitions before capabilities |
| 2 | **Single Responsibility** | One rule per file, each versioned | Split the monolith; tag every rule |
| 3 | **Event-Driven** | Inject context on a trigger, not always-on | Turn "always remind" into "remind when X" |
| 4 | **Cognitive Boundary** | A tool the model can't see doesn't exist | Densest containment on the riskiest tool |
| 5 | **Layered Delegation** | Fork / Sub-agent / Worker, by isolation need | Research → Fork; independent task → Sub-agent |
| 6 | **Progressive Safety** | 4 layers: behavior → tool → monitor → confirm | Don't ban; contain in depth |
| 7 | **Context Compaction** | Summarize, don't truncate | 5-section summary: Task/State/Discoveries/Next/Preserve |
| 8 | **Output Efficiency** | Max info density, min text | Answer first, reasoning after; skip filler |
| 9 | **Mode Switching** | Normal/Auto/Plan/Learning/Minimal | Plan mode = read-only; Auto = act-first |
| 10 | **Memory Trinity** | User / Feedback / Project memory | Memory expires — trust observation on conflict |
| 11 | **Skillification** | Turn a winning session into a reusable skill | 4-step interview → SKILL.md |
| 12 | **Observability** | Analyze usage to drive improvement | 5-dimension JSON insights |

**The reusable constraint shape:** `Don't <X>. <why>. <exception>.`

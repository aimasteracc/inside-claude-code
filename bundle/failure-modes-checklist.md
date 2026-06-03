# The 7 Failure Modes — Defensive Checklist

> From *Inside Claude Code*. Tape it above your monitor before you run an agent unattended.

| # | Failure mode | Recognize it by | The fix (harness component) |
|---|--------------|-----------------|------------------------------|
| 1 | **One-shot impulse** | Context exhausted, half-built fragments everywhere | Force ONE feature per run |
| 2 | **Premature "done"** | "Looks complete!" with 180 items left | JSON feature list; flip `passes` only, never delete |
| 3 | **Context anxiety** | Rushes to wrap up at ~70% context | Context Reset + structured handoff (not just compaction) |
| 4 | **Self-eval inflation** | "9/10, excellent work" — but it's broken | Separate Generator from Evaluator |
| 5 | **Skips E2E** | Unit tests pass; the button does nothing | Give it Playwright/Puppeteer; force human-like operation |
| 6 | **Stub-ification** | UI looks done; interactions are hollow | Sprint Contract + per-criterion verification |
| 7 | **Spec cascade** | Planner's wrong detail poisons the whole build | Planner does high-level design only; constrain outcomes, not path |

## Model note
- **Sonnet-class:** context anxiety is severe — use Context Reset.
- **Opus-class:** can usually ride compaction without a full reset.

## The session-diagnosis flow (run at the start of every session)

```
New session
  ├─ pwd                       confirm working directory
  ├─ read progress notes       history (defends #1, #2)
  ├─ read feature_list.json    pick next feature (defends #1)
  ├─ git log --oneline -20     recent change history
  ├─ run init / dev env        (defends #5)
  ├─ baseline E2E              verify env isn't broken
  │
  ├─ implement ONE feature
  │
  ├─ E2E the new feature       real, not stub (defends #6)
  ├─ flip `passes` only        never delete tests (defends #2)
  ├─ commit + descriptive msg  leave a trail for next session
  └─ update progress notes     structured handoff (defends #3)
```

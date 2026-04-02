---
description: "Show current harness status — active sessions, build progress, eval results"
---

# /dkh:status — Harness Status

Show the current state of the harness pipeline.

## Execution

1. Check for active dkod sessions via `dk_status`
2. Check for harness artifacts:
   - Plan/spec files
   - Work unit definitions
   - Eval reports from previous rounds
3. Display a summary:

```
Harness Status
══════════════
Round:        2 / 3
Phase:        BUILD (generators running)

Plan:         ✓ 6 work units, 3 waves
Generators:   4 / 6 complete
Landing:      pending
Evaluation:   Round 1: 4/6 units passed

Active Sessions:
  generator-unit-5  "Task filtering UI"     in progress
  generator-unit-6  "Dashboard charts"      in progress

Previous Failures:
  Unit 3: "Auth middleware" — criterion "JWT expiry returns 401" scored 4/10
  Unit 4: "Task form"      — criterion "validation errors display" scored 5/10
```

## Output

If no harness is running, display:
```
No active harness session. Run /dkh <prompt> to start a build.
```

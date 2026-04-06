# Design: Eval Verdicts, Auto-Discovery, Per-Unit Attempts, Stale Detection

**Date:** 2026-04-04
**Status:** Approved

## Overview

Four improvements inspired by pi-messenger patterns, adapted for our dkod harness.

## 1. Three-Tier Eval Verdicts

Replace binary pass/fail with PASS / RETRY / REPLAN.

| Verdict | Meaning | Action |
|---------|---------|--------|
| PASS | All criteria met | dk_push(PR) |
| RETRY | Implementation bugs, plan is sound | Re-dispatch failed units with feedback |
| REPLAN | Structural flaw, plan itself is wrong | Re-run planner with eval report |

Evaluator chooses:
- PASS: every criterion >= 7/10
- RETRY: failures are implementation bugs (wrong logic, missing handler)
- REPLAN: failures indicate wrong data model, missing feature, impossible criteria

Orchestrator Phase 5:
- PASS → ship
- RETRY (round < 3) → fix round
- RETRY (round 3) → forced ship with documented issues
- REPLAN (max 1 per build) → back to Phase 1, round resets to 1

### Files changed
- agents/evaluator.md — add verdict selection guidance + output format
- agents/orchestrator.md — update Phase 5 with REPLAN path
- skills/dkh/SKILL.md — update Phase 5 in flow diagram and gate rules

## 2. Auto-Discovery of Spec Files

Planner searches for existing docs before generating from scratch.

Discovery order (first match wins):
```
PRD.md, prd.md, SPEC.md, spec.md, REQUIREMENTS.md, requirements.md,
DESIGN.md, design.md, docs/PRD.md, docs/prd.md, docs/SPEC.md,
docs/spec.md, docs/DESIGN.md, docs/design.md, docs/REQUIREMENTS.md,
docs/requirements.md
```

Behavior:
- Found → planner reads and augments with user prompt
- Not found → generate full spec from scratch
- Multiple → first match only
- Content cap: 100KB

### Files changed
- agents/planner.md — add Step 0: Discover existing specs
- skills/dkh/SKILL.md — mention auto-discovery in Phase 1

## 3. Per-Unit Attempt Tracking

Track retries per work unit. Auto-block after 3 failures.

New orchestrator state:
```
unit_attempts: {}       # { "unit-id": attempt_count }
blocked_units: []       # Units that exceeded MAX_UNIT_ATTEMPTS (3)
```

Behavior:
- Each re-dispatch increments unit's attempt count
- At MAX_UNIT_ATTEMPTS (3) → move to blocked_units, remove from active_units
- Blocked units documented in PR, not retried
- Build continues with remaining units
- If all units blocked → forced ship

### Files changed
- agents/orchestrator.md — add unit_attempts + blocked_units to state, update
  Phase 5 round transition, update Phase 2 to skip blocked units

## 4. Stale Detection

Timeout agents that don't report back.

| Agent | Timeout | Action |
|-------|---------|--------|
| Planner | 30 minutes | Re-dispatch (up to 2 retries) |
| Generator | 45 minutes | Record as failed, re-dispatch in fix round |
| Evaluator | 30 minutes | Re-dispatch evaluator |

Implementation: orchestrator tells each agent its time budget in the dispatch
prompt. If agent hasn't returned when orchestrator checks, treat as crash.

### Files changed
- agents/orchestrator.md — add Stale Detection section, add timeout to dispatch examples
- agents/planner.md — mention time budget awareness
- agents/generator.md — mention time budget awareness
- agents/evaluator.md — mention time budget awareness

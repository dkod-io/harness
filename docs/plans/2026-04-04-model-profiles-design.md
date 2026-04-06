# Design: Model Profiles for Token Optimization

**Date:** 2026-04-04
**Status:** Approved

## Problem

Every agent in the harness runs on Opus — the most expensive model. For a build with
1 planner + 6 generators + 6 evaluators, that's 13 Opus agents. Most of this work
(structured implementation, checklist evaluation) can run on cheaper models without
sacrificing quality.

## Solution: Approach C — Profile table in SKILL.md, no agent frontmatter

Single source of truth in SKILL.md. Remove `model:` from all agent frontmatter.
The orchestrator reads the active profile and passes `model:` on every Agent dispatch.

## Profiles

| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| Orchestrator | opus | opus | sonnet |
| Planner | opus | opus | sonnet |
| Generator | opus | sonnet | sonnet |
| Evaluator | opus | sonnet | haiku |

Default: balanced

Rationale:
- Planner stays opus in balanced — bad plans waste all downstream tokens
- Generators drop to sonnet — they follow a detailed plan with specific symbols
- Evaluators drop to sonnet/haiku — they follow a checklist
- Orchestrator stays opus in balanced — autonomous decision-making

## File Changes

### skills/dkh/SKILL.md
- Add Model Profiles section after Prerequisites
- Update dispatch examples to include `model:` parameter

### agents/orchestrator.md
- Remove `model: opus` from frontmatter
- Add instruction to read active profile and pass `model:` on every Agent call
- Update Phase 1, 2, 4 dispatch examples with `model:` parameter

### agents/planner.md
- Remove `model: opus` from frontmatter

### agents/generator.md
- Remove `model: opus` from frontmatter

### agents/evaluator.md
- Remove `model: opus` from frontmatter

### commands/plan.md
- Reference active profile when spawning planner standalone

### commands/eval.md
- Reference active profile when spawning evaluator standalone

## Profile Switching

Edit SKILL.md: `Active profile: balanced` -> `Active profile: quality`
No runtime config files. SKILL.md is the single source of truth.

# Design: Eliminate Waves, Maximize Parallelism

**Date:** 2026-04-04
**Status:** Approved

## Problem

The harness carries serialization assumptions from pre-dkod thinking. Waves, dependency
graphs, `depends_on` fields, and "Files touched" in work units all imply that file overlap
matters for planning. It doesn't — dkod merges at the AST level. Multiple agents can write
to the same files, same modules, even overlapping areas simultaneously.

10 specific issues across 7 files where serialization language or patterns persist:

1. `planning-guide.md` Pattern 4 shows 3-wave sequential flow
2. `planning-guide.md` Pattern 2 (Layer-Based) creates schema->API->frontend chain
3. `planning-guide.md` "What Creates a Dependency" lists dkod-handled things as deps
4. `planning-guide.md` "Minimize Dependencies" frames workarounds instead of elimination
5. `dkod-patterns.md` "Greenfield Setup" says scaffolding must merge before features
6. `dkod-patterns.md` "Shared Types Pattern" recommends Wave 1 types -> Wave 2 features
7. `dkod-patterns.md` Landing Pipeline references Wave 3
8. Work Unit schema has `Files touched` and `Depends on` fields
9. `evaluator.md` says evaluators may run simultaneously (they're sequential)
10. `dkod-patterns.md` "Config File Merging" implies special handling needed

## Solution: Approach A — Purge Waves, Keep Symbol Ownership

Remove the wave concept entirely. All units dispatch simultaneously in one blast. Keep
`OWNS` to prevent true symbol conflicts (two generators writing the same function), but
file overlap is irrelevant.

## Core Model Change

**Before:** Units have `depends_on` fields and wave assignments. Orchestrator groups by
wave, dispatches Wave 1, waits, dispatches Wave 2.

**After:** All units dispatch simultaneously. No waves, no `depends_on`, no dependency
graphs. Planner's only structural constraint: symbol ownership.

### Removed from model
- `depends_on` field from work unit schema
- Wave assignments and dependency graph section in plans
- Parallelism score (always N/N = 1.0)
- `waves`, `current_wave` from orchestrator state
- Per-wave Build->Land loop (replaced by single Build->Land pass)
- All Gate 1 checks for waves and parallelism score

### Kept
- `OWNS (exclusive)` — prevents true symbol conflicts
- Gate 1 duplicate ownership check
- Symbol-level decomposition principle
- Inline types pattern

## File-by-File Changes

### agents/planner.md
- Remove "Maximum 2 waves" hard rule and all wave text
- Remove `depends_on` from work unit output schema
- Remove dependency graph + parallelism score from output
- Kill dependency justification table
- Kill Pattern 2 (Layer-Based) and Pattern 4 (Setup + Feature Waves) references
- Reframe: planner assigns symbol ownership + acceptance criteria, nothing structural

### agents/orchestrator.md
- Remove `waves`, `current_wave` from state
- Replace per-wave Build->Land loop with single pass: dispatch all -> land all -> eval -> ship
- Gate 1: remove wave/parallelism checks, keep duplicate ownership
- Gate 2: "all generators reported with changeset_ids"
- Gate 3: "all changesets merged or failed"
- Remove wave-related decision table entries

### skills/dkh/SKILL.md
- Remove wave references from flow diagram
- Remove Gate 1 wave/parallelism checks
- Simplify Phase 2+3 to single Build->Land pass
- Update gate enforcement rules (remove wave rules, keep dk_push + autonomy)

### skills/dkh/references/planning-guide.md
- Kill Pattern 2 (Layer-Based) — inherently sequential
- Kill Pattern 4 (Setup + Feature Waves) — waves don't exist
- Rewrite "What Creates a Dependency" -> "What is NOT a Dependency"
- Rewrite "Minimize Dependencies" -> "There Are No Dependencies"
- Add prominent "dkod means no serialization" section

### skills/dkh/references/dkod-patterns.md
- Kill "Shared Types Pattern"
- Rewrite "Greenfield Setup" — scaffolding runs in parallel with features
- Remove Wave 3 references from landing pipeline
- Simplify "Config File Merging" — dkod auto-merges, no special handling
- Simplify landing pipeline ordering — no wave ordering

### agents/evaluator.md
- Fix line 18: evaluators run sequentially (shared browser), not simultaneously

### agents/generator.md
- Remove "if this is a later wave" language

## Work Unit Schema

**Before:**
```
## Work Unit: <id>
**OWNS (exclusive):** <symbols>
**Symbols to create:** <new symbols>
**Files touched:** <files>
**Depends on:** none | [other unit IDs]
**Acceptance criteria:** ...
**Complexity:** low | medium | high
```

**After:**
```
## Work Unit: <id>
**OWNS (exclusive):** <symbols this unit is the sole owner of>
**Creates:** <new symbols with file paths>
**Acceptance criteria:** ...
**Complexity:** low | medium | high
```

Removed: `Depends on` (doesn't exist), `Files touched` (irrelevant with dkod).

## Plan Output Format

**Before:**
```
## Dependency Graph
Wave 1: [Unit 1-6]
Wave 2: [Unit 7: integration]
## Parallelism Score: 6/7
```

**After:**
```
## Dispatch
All units dispatch simultaneously: [Unit 1, Unit 2, ..., Unit N]
```

## Gate Changes

### Gate 1 (Plan) — simplified
- Removed: wave count, parallelism score, dependency graph
- Kept: spec exists, work units exist, criteria exist, no duplicate symbol ownership, design direction

### Gate 2 (Build) — simplified
- Removed: per-wave check
- Now: all generators reported back with changeset_ids

### Gate 3 (Land) — simplified
- Removed: per-wave check
- Now: all changesets merged or failed, at least one merged

### Gates 4 + 5 — unchanged

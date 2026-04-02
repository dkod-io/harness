---
name: harness
description: >
  Autonomous harness for building complete applications from a single prompt. Uses dkod for
  parallel agent execution with AST-level semantic merging. Orchestrates a Planner that decomposes
  work by symbol into parallel units, N Generator agents that implement simultaneously via isolated
  dkod sessions, and a skeptical Evaluator that tests the live result via chrome-devtools and
  dk_verify. Fully autonomous — zero user interaction from prompt to working, tested PR.
  Use this skill whenever the user provides a build prompt ("build a...", "create a...",
  "make a...") or invokes /dkh.
compatibility: >
  Requires dkod MCP server (claude mcp add --transport http dkod https://api.dkod.io/mcp)
  and chrome-devtools MCP for evaluation. Works with Claude Code and Opus 4.6.
---

# dkod Harness — Autonomous Parallel Build System

## What This Is

A fully autonomous build harness. The user provides a single prompt ("build a webapp that...").
The harness does everything else — planning, parallel implementation, testing, fixing, and
shipping — without any further user interaction.

This is an implementation of Anthropic's Planner → Generator → Evaluator harness pattern,
purpose-built for dkod's parallel execution capabilities. Where Anthropic's reference
architecture runs generators sequentially, this harness runs N generators simultaneously
because dkod's AST-level merge eliminates false conflicts.

## When This Skill Activates

- User says "build a...", "create a...", "make a..."
- User invokes `/dkh <prompt>`
- User describes a complete application or feature set to build from scratch
- Any task complex enough to benefit from parallel decomposition + evaluation

## Prerequisites Check

Before starting, verify both are available:

1. **dkod MCP tools**: `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit`, `dk_verify`,
   `dk_approve`, `dk_merge`, `dk_push`, `dk_status`
2. **chrome-devtools MCP**: `navigate_page`, `take_screenshot`, `click`, `evaluate_script`,
   `list_console_messages`, `lighthouse_audit`

If dkod is missing, guide installation:
```bash
claude mcp add --transport http dkod https://api.dkod.io/mcp
```

If chrome-devtools is missing, note that evaluation will be limited to `dk_verify` + code
review (no live UI testing).

## The Autonomous Loop

```
USER PROMPT
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 1: PLAN                                      │
│  Planner agent: prompt → spec → work units          │
│  • dk_connect + dk_context (read codebase)          │
│  • Decompose by SYMBOL, not file                    │
│  • Define acceptance criteria per unit              │
│  • Map dependencies between units                   │
│  • Output: spec + work_units + criteria             │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 2: BUILD (parallel)                          │
│  N Generator agents dispatched simultaneously       │
│  Each agent:                                        │
│  • dk_connect (own session, own overlay)            │
│  • dk_context (understand target symbols)           │
│  • dk_file_read → dk_file_write (implement)         │
│  • dk_submit (changeset)                            │
│  • Report completion                                │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 3: LAND                                      │
│  Orchestrator merges all changesets:                 │
│  • dk_verify each (lint, type-check, test)          │
│  • dk_resolve if conflicts (auto: keep_yours for    │
│    non-overlapping, surface true conflicts)         │
│  • dk_approve each                                  │
│  • dk_merge each (sequential — dkod AST merge)      │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 4: EVAL                                      │
│  Evaluator agent tests the merged result:           │
│  • Start dev server (detect framework)              │
│  • chrome-devtools: navigate, screenshot, click     │
│  • Test each acceptance criterion                   │
│  • dk_verify for code quality                       │
│  • Grade: PASS/FAIL per criterion with evidence     │
│  • Produce structured eval report                   │
└────────────────────┬────────────────────────────────┘
                     │
              ┌──────┴──────┐
              ▼             ▼
           ALL PASS      FAILURES
              │             │
              ▼             ▼
┌──────────────────┐  ┌──────────────────────────────┐
│  PHASE 5: SHIP   │  │  PHASE 2b: FIX (parallel)    │
│  dk_push(PR)     │  │  Re-dispatch generators with  │
│  Done.           │  │  specific eval feedback.      │
└──────────────────┘  │  Max 3 rounds total.          │
                      │  Then back to LAND → EVAL.    │
                      └──────────────────────────────┘
```

## Orchestrator Behavior

The orchestrator (you, when this skill is active) drives the entire loop autonomously:

### Phase 1: Plan
1. Spawn the **planner** agent (subagent_type: not specified — use general-purpose with the
   planner.md prompt injected)
2. Wait for the plan to complete
3. Validate: does the plan have work units? Do units have acceptance criteria? Are dependencies
   mapped?
4. If the plan is too vague, re-run the planner with more specific instructions

### Phase 2: Build
1. Read the work units from the plan
2. Identify which units can run in parallel (no dependency on each other)
3. Dispatch **generator** agents — one per work unit — all at once using multiple Agent tool
   calls in a single message
4. Each generator receives: the full spec, their specific work unit, and the acceptance criteria
   they must satisfy
5. Wait for all generators to complete

### Phase 3: Land
1. For each completed changeset, in sequence:
   - `dk_verify` — if verification fails, note the failures
   - `dk_approve` — approve the changeset
   - `dk_merge` — merge into dkod main
2. If `dk_merge` returns a conflict:
   - For non-overlapping symbol changes: `dk_resolve` with `proceed` and retry
   - For true conflicts: `dk_resolve` with `keep_yours` (trust the most recent generator's
     intent), then re-verify
3. If verification fails: add failures to the eval feedback for Phase 2b

### Phase 4: Eval
1. Spawn the **evaluator** agent
2. Evaluator starts the dev server, tests via chrome-devtools, grades each criterion
3. Read the eval report
4. Count PASS vs FAIL

### Phase 5: Ship or Fix
- **All criteria PASS**: `dk_push(mode: "pr")` — create the PR. Done. Report the PR URL.
- **Some criteria FAIL** (round < 3):
  - Extract the specific failures and which work units they map to
  - Re-dispatch only the generators whose units failed, with the eval feedback injected
  - Back to Phase 3 (Land) → Phase 4 (Eval)
- **Still failing after 3 rounds**: `dk_push(mode: "pr")` anyway, note remaining issues in
  the PR description. Report to user what passed and what didn't.

## Critical Design Principles

### 1. Decompose by symbol, not file
The Planner MUST decompose work into units that target specific **functions, classes, and
modules** — not files. Two generators editing different functions in the same file is not a
conflict. dkod merges at the AST level.

### 2. One dkod session per generator
Each generator calls `dk_connect` with its own descriptive `agent_name` and `intent`. Sessions
are isolated — one generator's writes are invisible to all others until merge.

### 3. The Evaluator is standalone and skeptical
From Anthropic's research: "tuning a standalone evaluator to be skeptical turns out to be far
more tractable than making a generator critical of its own work." The evaluator MUST:
- Actually test the live app (not just read code)
- Use chrome-devtools to navigate, click, fill forms, check console
- Grade each criterion with evidence (screenshots, console output)
- Default to FAIL unless proven PASS

### 4. File-based communication
The plan (spec + work units + criteria) and eval reports are structured artifacts. They survive
context resets and provide clear contracts between agents.

### 5. Autonomy is non-negotiable
The harness NEVER asks the user for input after receiving the initial prompt. Every decision
is made autonomously:
- Conflicts → auto-resolve with sensible defaults
- Eval failures → auto-fix with targeted re-dispatch
- Framework choice → infer from the prompt
- Port numbers → use defaults (5173 for Vite, 3000 for Next, etc.)
- Package manager → detect from lockfiles or default to npm

### 6. Max 3 eval rounds
Prevents infinite loops. After 3 rounds, ship whatever works and document what doesn't.

## Work Unit Schema

The Planner produces work units in this structure (embedded in the plan artifact):

```
## Work Unit: <id>
**Title:** <descriptive title>
**Symbols to modify:** <list of qualified symbol names>
**Symbols to create:** <list of new symbols with file paths>
**Files touched:** <list of files>
**Depends on:** <list of other unit IDs, or "none">
**Acceptance criteria:**
- <testable criterion 1>
- <testable criterion 2>
**Complexity:** low | medium | high
```

Units with `depends_on: none` run in the first wave. Units with dependencies run after their
dependencies complete. The orchestrator handles wave scheduling automatically.

## Agent Definitions

- **Planner**: `agents/planner.md` — expands prompt into spec + parallel work units
- **Generator**: `agents/generator.md` — implements a single work unit via dkod session
- **Evaluator**: `agents/evaluator.md` — tests merged result via chrome-devtools + dk_verify
- **Orchestrator**: `agents/orchestrator.md` — drives the autonomous loop (this is you)

## Reference Guides

- `references/planning-guide.md` — deep guide for symbol-level decomposition
- `references/evaluation-guide.md` — skeptical evaluation techniques and chrome-devtools patterns
- `references/dkod-patterns.md` — dkod session lifecycle and merge patterns

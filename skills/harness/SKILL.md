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

Before starting, verify these are available:

1. **dkod MCP tools**: `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit`, `dk_verify`,
   `dk_approve`, `dk_merge`, `dk_push`, `dk_status`
2. **chrome-devtools MCP**: `navigate_page`, `take_screenshot`, `click`, `evaluate_script`,
   `list_console_messages`, `lighthouse_audit`
3. **frontend-design skill**: Required for any project with UI. Generators MUST invoke
   `Skill(skill: "frontend-design")` before implementing UI components. The planner MUST
   include a Design Direction section in the spec. The evaluator MUST score design quality.

If dkod is missing, guide installation:
```bash
claude mcp add --transport http dkod https://api.dkod.io/mcp
```

If chrome-devtools is missing, note that evaluation will be limited to `dk_verify` + code
review (no live UI testing).

If frontend-design skill is missing, generators MUST still follow the Design Direction section
from the spec manually. The planner's Design Direction section provides all the creative
direction needed — generators should treat it as their design brief and apply it directly
without invoking the skill.

## The Autonomous Loop — STRICT GATES

Each phase produces a required artifact. The next phase CANNOT start until the gate check
confirms the artifact exists. **Skipping a phase is a harness violation.**

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
│                                                     │
│  GATE 1 — Required output:                          │
│  ✓ Specification with stack, features, data model   │
│  ✓ Work units with symbols + acceptance criteria    │
│  ✓ Dependency graph with wave assignments           │
│  BLOCKED until all three exist.                     │
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
│  • Report with changeset_id                         │
│                                                     │
│  GATE 2 — Required output:                          │
│  ✓ Every generator reported back                    │
│  ✓ Every report includes a changeset_id             │
│  ✓ List of all changeset IDs collected              │
│  BLOCKED until all generators have submitted.       │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 3: LAND                                      │
│  Orchestrator merges all changesets:                 │
│  • dk_verify ALL changesets in PARALLEL              │
│  • dk_approve each verified changeset               │
│  • dk_merge each sequentially (only merge is serial)│
│  • dk_resolve if conflicts (auto-resolve)           │
│                                                     │
│  GATE 3 — Required output:                          │
│  ✓ Every changeset merged OR recorded as failed     │
│  ✓ At least one merged (commit hash exists)         │
│  BLOCKED until all changesets are resolved.          │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 4: EVAL (sequential) — MANDATORY              │
│  *** YOU CANNOT SKIP THIS PHASE ***                  │
│  *** dk_push IS BLOCKED UNTIL EVAL COMPLETES ***     │
│                                                     │
│  Orchestrator starts dev server ONCE, then:         │
│  Evaluators run ONE AT A TIME (shared browser):     │
│  • One evaluator per work unit (sequential)         │
│  • Each: connect to shared dev server, test via     │
│    chrome-devtools, grade criteria with evidence     │
│  • One final evaluator for overall/integration      │
│                                                     │
│  GATE 4 — Required output:                          │
│  ✓ Eval report exists for EVERY work unit           │
│  ✓ Each report has scores + evidence per criterion  │
│  ✓ Overall integration report exists                │
│  ✓ Every criterion scored (no unscored criteria)    │
│  BLOCKED until all eval reports collected.           │
└────────────────────┬────────────────────────────────┘
                     │
              ┌──────┴──────┐
              ▼             ▼
           ALL PASS      FAILURES
              │             │
              ▼             ▼
┌──────────────────┐  ┌──────────────────────────────┐
│  PHASE 5: SHIP   │  │  PHASE 2b: FIX (parallel)    │
│                  │  │  Re-dispatch generators with  │
│  GATE 5:         │  │  specific eval feedback.      │
│  ✓ Eval report   │  │  Max 3 rounds total.          │
│    shows ALL     │  │  Then back to BUILD → LAND →  │
│    PASS, OR      │  └──────────────────────────────┘
│  ✓ Round 3       │
│    exhausted     │
│                  │
│  dk_push(PR)     │
│  Done.           │
└──────────────────┘
```

### GATE ENFORCEMENT RULES

**These are not guidelines. They are hard blocks.**

1. **You CANNOT call `dk_push` without a completed eval report.** The eval report must contain
   scores for every acceptance criterion. If you find yourself about to call `dk_push` and
   you have not dispatched evaluator agents, STOP. You are skipping Phase 4.

2. **You CANNOT dispatch evaluators without landed code.** All changesets must be merged first.
   If you find yourself dispatching evaluators before `dk_merge`, STOP.

3. **You CANNOT dispatch generators without a plan.** The plan must have work units with
   acceptance criteria. If you find yourself writing code without a plan artifact, STOP.

4. **Each phase checks the previous phase's gate.** Before starting Phase N, explicitly verify
   that Phase N-1's required output exists:
   - Phase 2 starts: "Do I have a plan with work units and criteria? YES → proceed"
   - Phase 3 starts: "Do I have changeset IDs from all dispatched generators? YES → proceed"
   - Phase 4 starts: "Did at least one changeset merge? Do I have a commit hash? YES → proceed"
   - Phase 5 starts: "Do I have eval reports for every unit? YES → proceed"

5. **dk_verify is NOT evaluation.** `dk_verify` runs lint/type-check/test. It does NOT start
   the application, test the UI, check acceptance criteria, or produce scores. It is a code
   quality gate in Phase 3. Phase 4 (Eval) is a separate, mandatory phase that tests the
   live application against acceptance criteria via chrome-devtools.

## Orchestrator Behavior — Phase-by-Phase with Gate Checks

The orchestrator (you, when this skill is active) drives the entire loop autonomously.
**Each phase has a gate check at entry and exit. Do not skip gates.**

### Phase 1: Plan
1. Spawn the **planner** agent
2. Wait for the plan to complete

**GATE 1 CHECK** — Before proceeding, verify ALL of:
- [ ] Plan contains a specification (stack, features, data model)
- [ ] Plan contains work units with symbol-level decomposition
- [ ] Every work unit has acceptance criteria (5+ testable criteria each)
- [ ] Dependency graph exists with wave assignments
- [ ] Overall acceptance criteria exist
- [ ] **For UI projects**: Spec includes a `## Design Direction` section with a concrete
  aesthetic tone (not "modern and clean"), hex color values, and named font choices
  (not "Arial", "Inter", "Roboto", or system defaults)

If any check fails → re-run the planner with specific feedback. Do not proceed.

### Phase 2: Build
**GATE 1 ENTRY CHECK**: "Do I have a validated plan? YES → proceed."

1. Read the work units. Group by dependency wave.
2. Dispatch ALL generators in the wave simultaneously (one Agent call per unit, all in a
   single message)
3. Wait for all generators in the wave to complete before starting the next wave

**GATE 2 CHECK** — Before proceeding, verify ALL of:
- [ ] Every generator in the current dispatch (all units in round 1, only failed units
  in rounds 2+) has reported back
- [ ] Every report includes a changeset_id
- [ ] I have a complete list of changeset IDs for this round's units

If a generator crashed → re-dispatch that single generator. Do not proceed until all
dispatched generators have submitted changesets.

### Phase 3: Land
**GATE 2 ENTRY CHECK**: "Do I have changeset IDs from every dispatched generator? YES → proceed."

1. **Verify in parallel**: `dk_verify` ALL changesets simultaneously
2. **Approve all verified**: `dk_approve` each
3. **Merge sequentially**: `dk_merge` each in dependency order
4. Handle conflicts: `dk_resolve` → retry

**GATE 3 CHECK** — Before proceeding, verify ALL of:
- [ ] Every changeset is either merged OR explicitly recorded as failed with reason
- [ ] At least one changeset merged successfully (commit hash exists)
- [ ] Merge/verification failures are recorded for the eval phase

Partial merge failures are tolerable — the evaluator will catch missing functionality
from unmerged units. But if zero changesets merged, that's a hard block — re-dispatch.

### Phase 4: Eval — MANDATORY, NEVER SKIP
**GATE 3 ENTRY CHECK**: "Did at least one changeset merge? Do I have a commit hash? YES → proceed."

⚠️ **THIS PHASE IS NOT OPTIONAL. dk_verify IS NOT A SUBSTITUTE FOR EVALUATION.**
⚠️ **YOU MUST DISPATCH EVALUATOR AGENTS BEFORE YOU CAN CALL dk_push.**

1. **Start the dev server ONCE** as the orchestrator (install deps, run dev command,
   wait for the port to be ready). Record the server URL.
2. **Dispatch evaluators sequentially** (one at a time): One evaluator per work unit,
   then one for overall integration. Pass the already-running server URL. Evaluators
   run sequentially because they share a single chrome-devtools browser session —
   parallel evaluators would race on navigate/screenshot/click calls, corrupting evidence.
3. Each evaluator MUST:
   - Connect to the already-running dev server (do NOT start another one)
   - Test via chrome-devtools (navigate, screenshot, click, fill forms)
   - Score every criterion with evidence
   - It has exclusive browser access — no other evaluator runs concurrently
4. Wait for each evaluator to complete before dispatching the next
5. After the final (integration) evaluator completes, stop the dev server
6. Collect all eval reports into a unified result

**GATE 4 CHECK** — Before proceeding, verify ALL of:
- [ ] I have an eval report for EVERY work unit
- [ ] I have an overall/integration eval report
- [ ] Every acceptance criterion has a score (no unscored criteria)
- [ ] Every score has evidence (screenshots, console output, test results)
- [ ] Pass/fail counts are calculated

If an evaluator crashed → re-dispatch that evaluator. Do not proceed without complete reports.

### Phase 5: Ship or Fix
**GATE 4 ENTRY CHECK**: "Do I have complete eval reports with scores for every criterion?
YES → proceed. NO → GO BACK TO PHASE 4."

- **All criteria PASS** → `dk_push(mode: "pr")`. Create the PR. Done.
- **Some criteria FAIL** (round < 3) → Execute Round Transition state reset, then
  re-enter Phase 2 with only the failed units. Proceed through Phase 2 → 3 → 4 → 5.
- **Still failing after round 3** → `dk_push(mode: "pr")` with issues documented in PR.

**FINAL GATE**: The PR description MUST include the eval results (scores, pass rate, evidence
summary). If the PR description doesn't reference eval results, you skipped Phase 4.

## Critical Design Principles

### 0. MAXIMIZE PARALLELISM — THE PRIME DIRECTIVE

This principle overrides all others. Every agent, at every phase, must default to parallel
execution and only serialize when there is a hard dependency that makes it impossible.

**You have two parallelism superpowers. Use both aggressively:**

1. **Claude Code agent teams** — The `Agent` tool can dispatch multiple agents simultaneously
   in a single message. Every time you have 2+ independent tasks, you MUST dispatch them as
   parallel agents in one message. Never serialize independent work.

2. **dkod session isolation** — Each agent gets its own `dk_connect` session with a
   copy-on-write overlay. N agents can edit the same files, the same modules, even overlapping
   areas of code — all at the same time. dkod's AST-level merge handles it.

**The combination is the unlock:** Claude Code agent teams give you N parallel workers.
dkod gives each worker an isolated workspace that merges cleanly. Together, they turn a
60-minute serial build into a 10-minute parallel build.

**This applies to EVERY phase:**
- **Plan**: The planner designs for maximum Wave 1 coverage — flatten the dependency graph
  so the most units run simultaneously.
- **Build**: ALL generators in a wave dispatch in a single message as parallel agents.
  Use `run_in_background: true` when you have other work to do while waiting.
- **Land**: Run `dk_verify` on ALL changesets in parallel (each verify is independent).
  Only `dk_merge` must be sequential (each merge advances HEAD).
- **Eval**: **Exception to parallel dispatch.** Evaluators run sequentially (one at a
  time) because they share a single chrome-devtools browser session. Parallel evaluators
  would race on navigate/screenshot/click calls, corrupting evidence. Dispatch one
  evaluator per work unit, wait for it to complete, then dispatch the next. Final
  evaluator runs overall integration criteria.
- **Fix**: Re-dispatch ALL failed generators simultaneously, not one at a time.

**Anti-pattern: serializing independent work.** If you find yourself waiting for Agent A
to finish before dispatching Agent B, and B doesn't depend on A's output — you are wasting
time. Dispatch both in the same message.

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

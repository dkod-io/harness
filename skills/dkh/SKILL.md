---
name: dkh
version: 0.1.28
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

## Handling `/dkh continue`

When the user sends `/dkh continue` (or just "continue"):

1. **If an active harness session exists** (the agent has state from a prior turn):
   - Output the current harness state and phase
   - Show which agents are active and what they're doing
   - Resume the harness loop from where it left off
   - Example: "Resuming harness — currently in Phase 2: Build. 4/7 generators complete, 3 still running."

2. **If no active harness session exists** (fresh context after app restart):
   - Acknowledge the command: "No active harness session found in this context."
   - Check for any active dkod sessions via `dk_status`
   - If active sessions found, report their state
   - If no sessions found, tell the user to start a new build with `/dkh <prompt>`

**Never ignore a `/dkh continue` silently.** Always acknowledge with current status.

## Prerequisites Check

Before starting, verify these are available:

1. **dkod MCP tools**: `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit`, `dk_verify`,
   `dk_review`, `dk_approve`, `dk_merge`, `dk_push`, `dk_status`, `dk_watch`
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

## Model Profiles

**Active profile: quality**

Each agent runs on a model appropriate to its task. The orchestrator reads the active
profile and passes `model:` AND `effort:` on every Agent dispatch call.

| Agent | quality | balanced | budget | effort |
|-------|---------|----------|--------|--------|
| **Orchestrator**\* | opus | opus | sonnet | high |
| **Planner** | opus | opus | sonnet | max |
| **Generator** | opus | sonnet | sonnet | high |
| **Evaluator** | opus | sonnet | haiku | max |

\* The orchestrator model is set by the invoking Claude Code session, not by this table. This row is a recommendation for the session model, not enforced by the harness.

**Effort levels are mandatory.** Planner and Evaluator use `max` (complex reasoning —
decomposition, scoring). Generator uses `high` (fast execution — file writes, not deep
analysis). Always pass `effort:` when dispatching agents.

- **quality** — All Opus. Maximum capability. Use for complex or high-stakes builds.
- **balanced** (default) — Opus for planning and orchestration, Sonnet for implementation
  and evaluation. Best cost/quality trade-off.
- **budget** — Sonnet for planning and implementation, Haiku for evaluation. Fastest and
  cheapest. Use for simple builds or iteration.

To switch profiles, change `Active profile:` above. The orchestrator reads this value
at the start of each run.

## The Autonomous Loop

The harness runs a strict phase-gated loop: **PLAN → BUILD → LAND → File Sync → Smoke
Test → EVAL → SHIP/FIX/REPLAN**. Each phase has entry/exit gates that block progression
until artifacts exist.

**`agents/orchestrator.md` is the single source of truth** for all phase details, gate
checks, state tracking, dispatch templates, and transition logic. Read it before starting.

**Key constraints:**
- No `dk_push(mode:"pr")` without completed eval reports (Phase 5 only)
- No evaluators without landed + smoke-tested code
- No generators without a validated plan
- `dk_verify` is NOT evaluation — Phase 4 tests the live app via chrome-devtools
- All code changes go through dkod — never use Write/Edit/Bash on source files
- Never ask the user anything — every decision is autonomous
- Max 3 eval rounds, then ship with documented issues

## Critical Design Principles

### 0. Maximize parallelism — THE PRIME DIRECTIVE

Default to parallel execution; only serialize when there is a hard data dependency.
Use Claude Code agent teams (multiple Agent calls in one message) + dkod session isolation
(each agent gets its own overlay). Together they turn serial builds into parallel builds.

**Applies to every phase:** generators dispatch simultaneously, dk_verify runs in parallel,
failed generators re-dispatch in parallel. The ONE exception: evaluators run sequentially
(shared chrome-devtools browser session).

### 1. Decompose by symbol, not file
dkod merges at the AST level. Two generators editing different functions in the same file
is not a conflict.

### 2. One dkod session per generator
Each generator calls `dk_connect` once. Sessions are isolated until merge.

### 3. The Evaluator is standalone and skeptical
From Anthropic's research: "tuning a standalone evaluator to be skeptical turns out to be far
more tractable than making a generator critical of its own work." Default to FAIL unless
proven PASS with evidence.

### 4. File-based communication
The plan and eval reports are structured artifacts that survive context resets.

### 5. Autonomy is non-negotiable
The harness NEVER asks the user for input. Conflicts → auto-resolve. Eval failures →
auto-fix. Framework → infer from prompt. Package manager → bun.

### 6. Max 3 eval rounds
Prevents infinite loops. After 3 rounds, ship whatever works and document what doesn't.

## Work Unit Schema

The Planner produces work units in this structure (embedded in the plan artifact):

```
## Work Unit: <id>
**Title:** <descriptive title>
**OWNS (exclusive):** <list of qualified symbol names this unit solely owns>
**Creates:** <list of new symbols with file paths>
**Acceptance criteria:**
- <testable criterion 1>
- <testable criterion 2>
**Complexity:** low | medium | high
```

## Agent Definitions

- **Planner**: `agents/planner.md` — expands prompt into spec + parallel work units
- **Generator**: `agents/generator.md` — implements a single work unit via dkod session
- **Evaluator**: `agents/evaluator.md` — tests merged result via chrome-devtools + dk_verify
- **Orchestrator**: `agents/orchestrator.md` — drives the autonomous loop (this is you)

## Reference Guides

- `references/planning-guide.md` — deep guide for symbol-level decomposition
- `references/evaluation-guide.md` — skeptical evaluation techniques and chrome-devtools patterns
- `references/dkod-patterns.md` — dkod session lifecycle and merge patterns

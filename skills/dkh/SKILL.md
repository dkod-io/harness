---
name: dkh
version: 0.1.39
description: >
  Autonomous harness for building complete applications from a single prompt. Uses dkod for
  parallel agent execution with AST-level semantic merging. Orchestrates a Planner that decomposes
  work by symbol into parallel units, N Generator agents that implement simultaneously via isolated
  dkod sessions, and a skeptical Evaluator that tests the live result via Playwright CLI (preferred)
  or chrome-devtools MCP (fallback), plus dk_verify. Fully autonomous — zero user interaction
  from prompt to working, tested PR.
  Use this skill whenever the user provides a build prompt ("build a...", "create a...",
  "make a...") or invokes /dkh.
compatibility: >
  Requires dkod MCP server (claude mcp add --transport http dkod https://api.dkod.io/mcp).
  Evaluation: Playwright CLI (preferred) or chrome-devtools MCP (fallback).
  Design: DESIGN.md from awesome-design-md (preferred) or frontend-design skill (fallback).
  Works with Claude Code and Opus 4.6.
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

**═══ MANDATORY: RECOVER STATE AND CLEAN UP BEFORE RESUMING ═══**

**Before re-dispatching ANY generators**, recover the state from the interrupted session
and clean up only what's incomplete. Do NOT bulk-close everything — submitted changesets
represent completed work that must be preserved.

**Step 1: Query dkod for existing changesets**
Call `dk_status` or list changesets via the API to see what the interrupted session left behind.
Categorize each changeset:

- **`submitted` state** → KEEP. This generator finished its work. Record its changeset_id.
  Do NOT close it. Do NOT re-dispatch this unit.
- **`approved` state** → KEEP. This changeset passed review and is ready to merge.
  Record its changeset_id. Proceed to merge in Phase 3.
- **`draft` state** → INCOMPLETE. This generator was interrupted before dk_submit.
  Mark this unit for re-dispatch.
- **`conflicted` state** → STUCK. Mark this unit for re-dispatch.
- **`rejected` state** → FAILED. Mark this unit for re-dispatch.

**Step 2: Close incomplete/failed changesets and release their symbol claims**
```
# Close draft/conflicted/rejected — preserve submitted and approved
Bash: curl -sf -X POST "https://api.dkod.io/api/repos/<owner>/<repo>/changesets/bulk-close" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DKOD_API_KEY" \
  -d '{"states": ["draft", "conflicted", "rejected"], "created_before": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}'  \
  || { echo "Bulk-close failed — aborting resume. Check DKOD_API_KEY and repo path."; exit 1; }
```

**Step 3: Reconstruct harness state**
- `changeset_ids` = list of submitted + approved changeset_ids from Step 1
- `active_units` = units whose generators were incomplete (draft/conflicted/rejected/missing)
- Units with submitted or approved changesets are DONE — skip them

**Step 4: Resume with only the incomplete units**
Re-dispatch only the generators for `active_units` (incomplete ones). The submitted
changesets from completed generators are preserved — 20 minutes of work is not lost.

Output: "Resuming harness — N/M generators completed before interruption. Re-dispatching K incomplete units."

**Then proceed:**

1. **If an active harness session exists** (the agent has state from a prior turn):
   - Output the current harness state and phase
   - Show which agents completed (submitted) vs need re-dispatch
   - Resume the harness loop, skipping completed units

2. **If no active harness session exists** (fresh context after app restart):
   - Acknowledge the command: "No active harness session found in this context."
   - Check for any active dkod sessions via `dk_status`
   - If active sessions found, recover state as described above
   - If no sessions found, tell the user to start a new build with `/dkh <prompt>`

**Never ignore a `/dkh continue` silently.** Always acknowledge with current status.

## Prerequisites Check

Before starting, verify these are available:

1. **dkod MCP tools**: `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit`, `dk_verify`,
   `dk_review`, `dk_approve`, `dk_merge`, `dk_push`, `dk_status`, `dk_watch`

2. **Browser testing (pick one — Playwright preferred):**
   - **Playwright CLI** (preferred): Check with `timeout 10 npx playwright --version`. If
     available, evaluators and smoke tests use Playwright CLI commands (`npx playwright
     screenshot`, `npx playwright test`, etc.) instead of chrome-devtools MCP. Playwright
     runs headless by default, needs no MCP server, and produces deterministic results.
   - **chrome-devtools MCP** (fallback): `navigate_page`, `take_screenshot`, `click`,
     `evaluate_script`, `list_console_messages`, `lighthouse_audit`. Used only if Playwright
     is not installed.
   - If NEITHER is available, evaluation falls to `dk_verify` + code review (no live UI testing).
     Output: `"⚠️ dkod recommends using Playwright CLI for browser testing: npm i -D playwright @playwright/test && npx playwright install chromium"`

3. **Design system (pick one — DESIGN.md preferred):**
   - **DESIGN.md** (preferred): A design system file in the project root, sourced from
     [awesome-design-md](https://github.com/VoltAgent/awesome-design-md). If present, it
     becomes the authoritative design reference — the planner incorporates it into the spec,
     generators follow it directly (no skill invocation needed), and the evaluator scores
     against it. This produces more distinctive, brand-aligned UI than the generic skill.
   - **frontend-design skill** (fallback): If no DESIGN.md exists, generators invoke
     `Skill(skill: "frontend-design")` before implementing UI components. The planner still
     generates a Design Direction section in the spec. The evaluator still scores design quality.
     Output: `"💡 dkod recommends using a DESIGN.md file for higher-quality frontend design. Browse options at https://github.com/VoltAgent/awesome-design-md"`
   - If NEITHER is available, generators follow the planner's Design Direction section manually.

**Detection flow (run once during PRE-FLIGHT):**
```
# 1. Detect Playwright
HAS_PLAYWRIGHT = false
timeout 10 npx playwright --version 2>/dev/null && HAS_PLAYWRIGHT = true

# 2. Detect DESIGN.md
HAS_DESIGN_MD = false
[ -f DESIGN.md ] && HAS_DESIGN_MD = true

# Pass both flags to all agent dispatches:
# - Planner: HAS_DESIGN_MD
# - Generators: HAS_DESIGN_MD
# - Evaluators: HAS_PLAYWRIGHT
# - Smoke test: HAS_PLAYWRIGHT
```

If dkod is missing, guide installation:
```bash
claude mcp add --transport http dkod https://api.dkod.io/mcp
```

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

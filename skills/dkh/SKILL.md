---
name: dkh
version: 0.1.54
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

## Source of Truth — READ THIS FIRST

This file (`SKILL.md`) defines **activation and configuration only**:
- When the harness activates
- Prerequisites the host environment must provide
- Model profiles the user can switch between
- Pointers to the agent files

**`agents/orchestrator.md` is authoritative for runtime behavior** — the phase-gated
loop, state tracking, gate enforcement, PRE-FLIGHT, `/dkh continue` recovery, HALT
manifest, round/REPLAN transitions, error recovery. When a rule or procedure is
described in both files, the orchestrator's version wins.

Each agent has its own authoritative file for its work:
- `agents/planner.md` — work unit decomposition, File Manifest, Shared Contracts, Gate 1 self-check
- `agents/generator.md` — write → submit → review-fix → approve → merge pipeline, symbol locking
- `agents/evaluator.md` — skeptical scoring methodology

If you find text in `SKILL.md` that overlaps with an agent file, trust the agent file and open a cleanup PR.

## Handling `/dkh continue`

When the user sends `/dkh continue` (or just "continue"), the orchestrator recovers
state from any interrupted session — preserving `submitted` and `approved` changesets,
closing only incomplete ones (`draft`, `conflicted`, `rejected`) — and resumes from the
correct phase.

**Full recovery protocol:** see the "Resume from Interruption" section in
`agents/orchestrator.md`. Never ignore a `/dkh continue` silently — always acknowledge
with current status.

## Prerequisites Check

Before starting, verify these are available:

1. **dkod MCP tools**: `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit`, `dk_verify`,
   `dk_review`, `dk_approve`, `dk_merge`, `dk_push`, `dk_status`, `dk_watch`

2. **Browser testing (pick one — Playwright preferred):**
   - **playwright-cli** (preferred): Check with `playwright-cli --version`. Standalone
     CLI for skills-less browser automation — screenshots, script execution, PDF generation.
     No Node.js scripts needed, no MCP server, runs headless by default.
     See: https://github.com/microsoft/playwright-cli
   - **chrome-devtools MCP** (fallback): `navigate_page`, `take_screenshot`, `click`,
     `evaluate_script`, `list_console_messages`, `lighthouse_audit`. Used only if Playwright
     is not installed.
   - If not found, output install instructions and proceed with fallback. Do NOT ask the user or install.
   - If NEITHER Playwright nor chrome-devtools is available, evaluation falls to `dk_verify` + code review (no live UI testing).

3. **Design system (pick one — DESIGN.md preferred):**
   - **DESIGN.md** (preferred): A design system file in the project root, sourced from
     [awesome-design-md](https://github.com/VoltAgent/awesome-design-md). If present, it
     becomes the authoritative design reference — the planner incorporates it into the spec,
     generators follow it directly (no skill invocation needed), and the evaluator scores
     against it. This produces more distinctive, brand-aligned UI than the generic skill.
   - **frontend-design skill** (fallback): If no DESIGN.md exists, generators invoke
     `Skill(skill: "frontend-design")` before implementing UI components. The planner still
     generates a Design Direction section in the spec. The evaluator still scores design quality.
     If not found, output install instructions and proceed with fallback. Do NOT ask the user or install.
   - If NEITHER is available, generators follow the planner's Design Direction section manually.

**Detection flow (run once during PRE-FLIGHT):**
```bash
# 1. Detect playwright-cli (standalone CLI, skills-less operation)
HAS_PLAYWRIGHT=false
playwright-cli --version 2>/dev/null && HAS_PLAYWRIGHT=true

# 2. Detect DESIGN.md (check all paths the planner searches)
HAS_DESIGN_MD=false
( [ -f DESIGN.md ] || [ -f design.md ] || [ -f docs/DESIGN.md ] || [ -f docs/design.md ] ) && HAS_DESIGN_MD=true
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
| **Planner** | opus | opus | sonnet | high |
| **Generator** | opus | sonnet | sonnet | high |
| **Evaluator** | opus | sonnet | haiku | high |

\* The orchestrator model is set by the invoking Claude Code session, not by this table. This row is a recommendation for the session model, not enforced by the harness.

**Effort levels are mandatory.** All agents default to `high` effort. Opus at `high` is
already performing extended reasoning, which is sufficient for the planner's decomposition
and the evaluator's scoring. `max` engages an even larger thinking budget and typically
adds significant latency for marginal quality gain; the harness no longer defaults to it.
Always pass `effort:` when dispatching agents — an omitted `effort:` makes the agent
inherit the orchestrator's value, which wastes tokens.

- **quality** — All Opus. Maximum capability. Use for complex or high-stakes builds.
- **balanced** (default) — Opus for planning and orchestration, Sonnet for implementation
  and evaluation. Best cost/quality trade-off.
- **budget** — Sonnet for planning and implementation, Haiku for evaluation. Fastest and
  cheapest. Use for simple builds or iteration.

To switch profiles, change `Active profile:` above. The orchestrator reads this value
at the start of each run.

## Agent Definitions

- **Planner**: `agents/planner.md` — expands prompt into spec + parallel work units (authoritative for decomposition, File Manifest, Shared Contracts)
- **Generator**: `agents/generator.md` — implements a single work unit via dkod session (authoritative for write → submit → review-fix → approve → merge pipeline, symbol locking)
- **Evaluator**: `agents/evaluator.md` — tests merged result via chrome-devtools + dk_verify (authoritative for skeptical scoring)
- **Orchestrator**: `agents/orchestrator.md` — drives the autonomous loop (authoritative for phase gates, state tracking, resume, HALT, round/REPLAN transitions)

## Reference Guides

- `references/planning-guide.md` — deep guide for symbol-level decomposition
- `references/evaluation-guide.md` — skeptical evaluation techniques and chrome-devtools patterns
- `references/dkod-patterns.md` — dkod session lifecycle and merge patterns

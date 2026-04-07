---
name: dkh
version: 0.1.9
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

**Active profile: balanced**

Each agent runs on a model appropriate to its task. The orchestrator reads the active
profile and passes `model:` on every Agent dispatch call.

| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| **Orchestrator**\* | opus | opus | sonnet |
| **Planner** | opus | opus | sonnet |
| **Generator** | opus | sonnet | sonnet |
| **Evaluator** | opus | sonnet | haiku |

\* The orchestrator model is set by the invoking Claude Code session, not by this table. This row is a recommendation for the session model, not enforced by the harness.

- **quality** — All Opus. Maximum capability. Use for complex or high-stakes builds.
- **balanced** (default) — Opus for planning and orchestration, Sonnet for implementation
  and evaluation. Best cost/quality trade-off.
- **budget** — Sonnet for planning and implementation, Haiku for evaluation. Fastest and
  cheapest. Use for simple builds or iteration.

To switch profiles, change `Active profile:` above. The orchestrator reads this value
at the start of each run.

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
│  • Auto-discover PRD.md / SPEC.md / DESIGN.md       │
│  • Decompose by SYMBOL, not file                    │
│  • Define acceptance criteria per unit              │
│                                                     │
│  GATE 1 — Required output:                          │
│  ✓ Specification with stack, features, data model   │
│  ✓ Work units with symbols + acceptance criteria    │
│  ✓ No duplicate symbol ownership                    │
│  ✓ Aggregation symbols identified with single owners│
│  BLOCKED until all four exist.                      │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 2: BUILD                                     │
│  ALL N generators dispatched simultaneously         │
│  Each agent:                                        │
│  • dk_connect (own session, own overlay)            │
│  • dk_context (understand target symbols)           │
│  • dk_file_read → dk_file_write (check warnings!)   │
│  • dk_submit (changeset)                            │
│  • REVIEW-FIX LOOP (max 3 rounds):                  │
│    - Fix local review findings (inline w/ submit)   │
│    - dk_watch for deep review completion            │
│    - dk_review → fix deep findings → re-submit      │
│    - Exit when score ≥ 4 or 3 rounds exhausted     │
│                                                     │
│  GATE 2 — Required output:                          │
│  ✓ Every generator exited with changeset_id +       │
│    final review score                               │
│  BLOCKED until all generators have completed.       │
├─────────────────────────────────────────────────────┤
│  PHASE 3: LAND                                      │
│  Orchestrator has changeset_ids from generator exits │
│  • Log any generators with score < 3 (warn only)   │
│  • dk_verify ALL changesets in PARALLEL              │
│  • dk_approve ALL verified changesets               │
│  • dk_merge each sequentially                       │
│                                                     │
│  ⚠️  DO NOT dk_push(mode:"pr") after landing!       │
│  ⚠️  DO NOT ask the user what to do next!           │
│  Proceed to FILE SYNC then EVAL.                    │
│                                                     │
│  GATE 3 — Required output:                          │
│  ✓ Every changeset merged OR recorded as failed     │
│  ✓ At least one changeset merged (commit hash       │
│    exists)                                          │
│  BLOCKED until all changesets fully resolve.        │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  FILE SYNC — get merged code locally                 │
│                                                     │
│  dk_push(mode:"branch", branch_name:"dkh/sync-<repo>")  │
│  Then: git fetch && git checkout dkh/sync-<repo>    │
│  This is a temp branch — NOT a PR.                  │
│  Cleanup happens in Phase 5.                        │
│                                                     │
│  ⚠️  NEVER use dk_file_read to sync files!          │
│  ⚠️  One push + one checkout vs 100+ reads.         │
├─────────────────────────────────────────────────────┤
│  SMOKE TEST — MANDATORY BEFORE EVAL                  │
│  *** App MUST start and load before eval ***         │
│                                                     │
│  • Install deps, start dev server                   │
│  • Navigate to app in browser (chrome-devtools)     │
│  • Take screenshot — must show real content          │
│  • Check console — no fatal errors                  │
│                                                     │
│  FAIL? → Fix round (build failure, not eval)        │
│  PASS? → Proceed to EVAL with server running        │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  PHASE 4: EVAL (sequential) — MANDATORY              │
│  *** YOU CANNOT SKIP THIS PHASE ***                  │
│  *** dk_push IS BLOCKED UNTIL EVAL COMPLETES ***     │
│                                                     │
│  Dev server already running from smoke test.        │
│  Evaluators run ONE AT A TIME (shared browser):     │
│  • One evaluator per work unit (sequential)         │
│  • Each: test via chrome-devtools, grade criteria   │
│  • One final evaluator for overall/integration      │
│                                                     │
│  GATE 4 — Required output:                          │
│  ✓ Eval report exists for EVERY work unit           │
│  ✓ Each report has scores + evidence per criterion  │
│  ✓ At least one screenshot in eval evidence         │
│  ✓ Overall integration report exists                │
│  ✓ Every criterion scored (no unscored criteria)    │
│  BLOCKED until all eval reports collected.           │
└────────────────────┬────────────────────────────────┘
                     │
              ┌──────┴───────┬──────────────┐
              ▼              ▼              ▼
           PASS           RETRY          REPLAN
              │              │              │
              ▼              ▼              ▼
┌──────────────────┐  ┌────────────────┐  ┌────────────────────┐
│  PHASE 5: SHIP   │  │  FIX (parallel)│  │  RE-PLAN           │
│                  │  │  Re-dispatch   │  │  Re-run planner    │
│  dk_push(PR)     │  │  failed units  │  │  with eval report  │
│  Done.           │  │  Max 3 rounds  │  │  Max 1 replan/build│
│                  │  │  Auto-block    │  │  Then BUILD → LAND │
│                  │  │  after 3 unit  │  │  → EVAL again      │
│                  │  │  attempts      │  │                    │
└──────────────────┘  └────────────────┘  └────────────────────┘
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
   - Phase 4 starts: "Did the smoke test pass? Is the dev server running? YES → proceed"
   - Phase 5 starts: "Do I have eval reports for every unit? YES → proceed"

5. **dk_verify is NOT evaluation.** `dk_verify` runs lint/type-check/test. It does NOT start
   the application, test the UI, check acceptance criteria, or produce scores. It is a code
   quality gate in Phase 3. Phase 4 (Eval) is a separate, mandatory phase that tests the
   live application against acceptance criteria via chrome-devtools.

6. **dk_push(mode:"pr") is ONLY allowed in Phase 5.** Not after Phase 3. Not "just to save
   progress." The one exception: `dk_push(mode:"branch", branch_name:"dkh/sync-*")` is required
   after landing to sync merged code locally for the smoke test. This is a temp branch, not
   a PR — it gets cleaned up in Phase 5. If you catch yourself calling `dk_push(mode:"pr")`
   before `eval_reports` is populated, STOP. `dk_merge` commits code to the dkod session
   locally — that is LANDING, not shipping. Landing is Phase 3. Shipping is Phase 5.

7. **NEVER ask the user anything.** Not "should I proceed?" Not "what's your preference?"
   Not "option A or B?" The user gave you a prompt and walked away. Every decision is yours.
   If you are composing a question to the user, STOP. Pick the best option and proceed.

8. **The app MUST start and load before dk_push.** After landing code, you MUST run a smoke
   test: start the dev server, navigate to the app in the browser, take a screenshot, and
   confirm it renders real content (not an error overlay, not a blank page, not a crash).
   If the smoke test fails, that is a build failure — enter a fix round. Do NOT dispatch
   evaluators on a broken app. Do NOT produce "degraded eval reports" to bypass this gate.
   **No screenshot in eval evidence = the app was never tested = gate violation.**

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
- [ ] No duplicate symbol ownership across work units
- [ ] Aggregation symbols identified with single owners (no entry point conflicts)
- [ ] Overall acceptance criteria exist
- [ ] **For UI projects**: Spec includes a `## Design Direction` section with a concrete
  aesthetic tone (not "modern and clean"), hex color values, and named font choices
  (not "Arial", "Inter", "Roboto", or system defaults)

If any check fails → re-run the planner with specific feedback. Do not proceed.

### Phase 2: Build
**GATE 1 ENTRY CHECK**: "Do I have a validated plan? YES → proceed."

1. Dispatch ALL generators simultaneously (one Agent call per unit)
2. Each generator implements its unit, submits, then runs a **review-fix loop**:
   - Fix local review findings (returned inline with `dk_submit` response)
   - Wait for deep review via `dk_watch` (filter: `changeset.review.completed`)
   - Fetch findings via `dk_review(changeset_id)`
   - If score < 4 OR any `severity:"error"` findings → fix files → `dk_submit` again
   - Max **3 review-fix rounds**, then exit regardless of score
   - `dk_watch` blocks at tool level — zero LLM inference while waiting
3. Wait for all generators to complete and return their final state

**GATE 2 CHECK:**
- [ ] Every generator exited with a changeset_id and final review score
- [ ] All changeset_ids collected
- [ ] Log any generators that exhausted 3 rounds with score < 3 (warning only, don't block)

If a generator crashed → re-dispatch it. Do not proceed until all have completed.

### Phase 3: Land
Generators already handled review-fix loops. Orchestrator has changeset_ids from their exits.
No session-to-changeset mapping needed. No review score checking needed.

1. **Check generator exit states**: Log any generators that exhausted 3 rounds with low scores (warning only)
2. **Verify in parallel**: `dk_verify` ALL changesets simultaneously
3. **Approve all verified**: `dk_approve` each
4. **Merge sequentially**: `dk_merge` each one at a time
5. Handle conflicts: `dk_resolve` → retry

**DO NOT dk_push(mode:"pr"). PRs are Phase 5 only.** The only allowed push after
landing is `dk_push(mode:"branch")` for the file sync step — this creates a temporary
branch, not a PR.

**GATE 3 CHECK:**
- [ ] Every changeset merged OR recorded as failed with reason
- [ ] At least one changeset merged (commit hash exists)

Partial merge failures are tolerable — the evaluator will catch missing functionality.
Zero merges is a hard block — re-dispatch generators before advancing.

### File Sync — Get Merged Code Locally
**GATE 3 ENTRY CHECK**: "Did at least one changeset merge? Do I have a commit hash? YES → proceed."

After all merges are complete, sync the merged code to the local filesystem for smoke
testing and evaluation. **Do NOT use dk_file_read** to sync files one by one — that wastes
100+ tool calls and can exceed turn limits.

1. Push merged code to a temporary branch:
   `dk_push(mode: "branch", branch_name: "dkh/sync-<repo-name>")`
   This is NOT a PR — just a sync branch for local checkout.
2. Fetch and checkout locally:
   `git fetch origin && git checkout dkh/sync-<repo-name>`
3. Verify the checkout succeeded (files exist on disk)

The temp branch `dkh/sync-*` is cleaned up in Phase 5 after the final PR push.

### Smoke Test — MANDATORY BEFORE EVAL

Before dispatching evaluators, verify the app actually starts and loads:
1. Install deps (`bun install`), start dev server, wait for port
2. Navigate to the app with chrome-devtools, take a screenshot
3. Confirm the screenshot shows real content (not error overlay, not blank page)
4. Check console for fatal errors

**If smoke test FAILS** → build failure. Enter fix round with crash error as feedback.
DO NOT dispatch evaluators on a broken app.
**If smoke test PASSES** → proceed to Phase 4 with the server already running.

### Phase 4: Eval — MANDATORY, NEVER SKIP

⚠️ **THIS PHASE IS NOT OPTIONAL. dk_verify IS NOT A SUBSTITUTE FOR EVALUATION.**
⚠️ **YOU MUST DISPATCH EVALUATOR AGENTS BEFORE YOU CAN CALL dk_push.**

Dev server is already running from the smoke test.
1. **Dispatch evaluators sequentially** (one at a time): One evaluator per work unit,
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
- [ ] **At least one screenshot exists in the eval evidence** — if zero screenshots,
  the evaluator did not actually test the live app. That is a gate failure.
- [ ] Pass/fail counts are calculated

If an evaluator crashed → re-dispatch that evaluator. Do not proceed without complete reports.

### Phase 5: Ship, Fix, or Replan
**GATE 4 ENTRY CHECK**: "Do I have complete eval reports with scores for every criterion?
YES → proceed. NO → GO BACK TO PHASE 4."

Read the evaluator's **verdict**:
- **PASS** → `dk_push(mode: "pr")`. Create the PR. Clean up temp branch. Done.
- **RETRY** (round < 3) → Increment per-unit attempt counts. Auto-block units with 3+
  attempts. Re-dispatch remaining failed units with feedback. Phase 2 → 3 → 4 → 5.
- **RETRY** (round 3) → `dk_push(mode: "pr")` with issues documented. Clean up temp branch.
- **REPLAN** (max 1 per build) → Re-run planner with eval report. Reset round to 1.
  Back to Phase 1 gate check.

**Temp branch cleanup:** After `dk_push(mode: "pr")` completes, delete the sync branch:
```
git push origin --delete dkh/sync-<repo-name>
git branch -d dkh/sync-<repo-name>
git checkout main
```
This keeps the remote clean — only the PR branch remains.

**FINAL GATE**: The PR description MUST include the eval results (scores, pass rate,
verdict, evidence summary). If the PR description doesn't reference eval results, you
skipped Phase 4.

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
- **Plan**: The planner produces N units with non-overlapping symbol ownership.
  All dispatch at once.
- **Build**: ALL generators dispatch in a single message as parallel agents.
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


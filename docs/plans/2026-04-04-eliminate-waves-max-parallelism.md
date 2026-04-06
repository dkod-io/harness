# Eliminate Waves, Maximize Parallelism — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Remove the wave concept from the harness entirely. All work units dispatch simultaneously in a single blast. Symbol ownership is the only structural constraint.

**Architecture:** Purge `depends_on`, wave assignments, dependency graphs, parallelism scores, per-wave loops, and "Files touched" from all 7 harness files. Replace with single-blast dispatch model where the planner assigns symbol ownership and the orchestrator dispatches all generators at once.

**Tech Stack:** Markdown files — agents/planner.md, agents/orchestrator.md, agents/generator.md, agents/evaluator.md, skills/dkh/SKILL.md, skills/dkh/references/planning-guide.md, skills/dkh/references/dkod-patterns.md

---

### Task 1: Rewrite `agents/planner.md` — Remove waves and dependencies

**Files:**
- Modify: `agents/planner.md`

**Step 1: Replace the PRIME DIRECTIVE section (lines 17-52)**

Remove all wave/dependency language. Replace with:

```markdown
## THE PRIME DIRECTIVE: ALL UNITS DISPATCH SIMULTANEOUSLY

Your primary objective is to produce a plan where ALL generators run AT THE SAME TIME.
Not "most in Wave 1" — ALL of them. There are no waves. There are no dependencies.
Every unit dispatches simultaneously in a single blast.

This is possible because:

1. **dkod session isolation** — Each generator gets its own `dk_connect` session. N generators
   can edit the same files at the same time because dkod merges at the AST level. Two
   generators touching the same file is NOT a conflict. Two generators touching different
   SYMBOLS in the same file run in parallel with zero conflicts.

2. **Claude Code agent teams** — The orchestrator dispatches ALL generators in a SINGLE
   message. They run simultaneously as parallel agents. If the plan has 8 units, 8 agents
   run at once.

3. **Generators inline what they need.** A generator building "Task UI" does NOT need
   to wait for "Task API" to be merged. It defines its own TypeScript interfaces for the
   API response shape and builds against those.

**Your success metric: unit count.** More units = more parallel agents = faster build.
Target 5-8 units for a typical application.

**The ONLY structural constraint is symbol ownership.** No two units may create or modify
the same function, component, class, or type. This prevents true AST conflicts. Everything
else — file overlap, import dependencies, config files — is handled by dkod automatically.
```

**Step 2: Remove the dependency justification table (lines 200-209)**

Replace the entire "Step 4: Assign Symbol Ownership + Eliminate Dependencies" section. Remove the "I need a dependency because..." table. Replace with:

```markdown
### Step 4: Assign Symbol Ownership

**Every symbol has exactly one owner.** This is the planner's only structural rule.

| Rule | Why |
|------|-----|
| Every function/component/class has exactly one owner | Prevents true AST conflicts at merge |
| Hub files (App.tsx, router.ts, package.json) belong to the scaffolding unit | Avoids N generators all writing the root component |
| If two units need the same type, each defines it locally | Type duplication is cheap — it eliminates the need for any coordination |
| Inline/local types are NOT listed in `OWNS:` | Only globally-unique exported symbols need ownership tracking |

**There are no dependencies. There are no waves.** Every unit is independent. dkod handles
file overlap, config merging, and import deduplication automatically. If a generator needs
something another generator creates, it defines its own local version and builds against that.

**The result:**
```
Unit 1: scaffolding + app shell
Unit 2: auth API + types
Unit 3: task CRUD API + types  
Unit 4: auth UI (login/signup pages)
Unit 5: task list UI
Unit 6: task detail + editing UI
Unit 7: dashboard page

ALL dispatch simultaneously → 7 agents at once
```
```

**Step 3: Update the Work Unit schema in Output Format (lines 271-297)**

Replace the work unit schema. Remove `Depends on`, `Files touched`, `Symbols to modify`. New schema:

```markdown
### Unit 1: <title>
**OWNS (exclusive):** <symbols this unit is the sole owner of>
**Creates:** <new symbols with file paths>
**Acceptance criteria:**
- <criterion 1>
- <criterion 2>
**Complexity:** low | medium | high
```

**Step 4: Replace the Dependency Graph section in Output Format (lines 299-312)**

Replace with:

```markdown
## Dispatch
All units dispatch simultaneously: [Unit 1, Unit 2, ..., Unit N]
```

Remove `## Parallelism Score` entirely.

**Step 5: Update the Rules section (lines 314-330)**

Replace rule 1 from "Maximum 2 waves" to:

```markdown
1. **All units dispatch simultaneously.** There are no waves, no dependencies, no
   `depends_on`. Every unit runs in parallel. dkod handles file overlap and merge conflicts.
```

Remove rule 7 ("Include a parallelism score") — it's always N/N.

**Step 6: Remove "Good decomposition" wave annotations (lines 146-182)**

Remove all `← Wave 1` annotations and the final line `ALL units: depends_on: none → ALL run in Wave 1 → 6 agents simultaneously`. Replace with `ALL units dispatch simultaneously → 6 agents at once`.

Remove `depends_on: none` from all example units — the field no longer exists.

**Step 7: Commit**

```bash
git add agents/planner.md
git commit -m "Remove waves and dependencies from planner — all units dispatch simultaneously"
```

---

### Task 2: Rewrite `agents/orchestrator.md` — Single-blast dispatch

**Files:**
- Modify: `agents/orchestrator.md`

**Step 1: Replace state tracking block (lines 47-58)**

Remove `waves`, `current_wave`, `pushed`. New state:

```
round: 1                    # Current round (1, 2, or 3)
plan: null                  # Set after Phase 1
active_units: []            # All units in round 1; only failed units in rounds 2+
changeset_ids: []           # Set after Phase 2 — one per unit in active_units
merged_commit: null         # Set after Phase 3 — latest commit hash
merge_failures: []          # Changesets that failed to merge
eval_reports: []            # Set after Phase 4 — MUST EXIST before dk_push
```

**Step 2: Simplify Gate 1 (lines 76-99)**

Remove wave checks (lines 81-90: max 2 waves, parallelism score). Keep:
- Plan has spec, work units, criteria (5+ per unit), overall criteria
- No duplicate symbol ownership
- Design direction for UI projects

Remove the `set waves = ...` and `set current_wave = 0` from gate pass action. Just:
```
If gate passes → set `plan = <the plan>`, set `active_units = plan.work_units`. Proceed to Phase 2.
```

**Step 3: Replace Phases 2+3 per-wave loop (lines 101-178) with single-blast**

Replace the entire "PHASES 2 + 3 — BUILD AND LAND (per-wave loop)" with:

```markdown
---

### PHASE 2 — BUILD

**Entry check**: `plan` must be set. If null → STOP, go back to Phase 1.

Dispatch ALL generators simultaneously in a single message:

\```
// Single message with multiple Agent tool calls — ALL units at once:
Agent(
  subagent_type: "general-purpose",
  prompt: <inject generator.md instructions + spec + this unit>,
  description: "Build: <unit title>",
  name: "generator-<unit-id>"
)
// ... one per unit in active_units — ALL of them, not batched by wave
\```

**═══ GATE 2 CHECK ═══**
Before proceeding, verify:
- [ ] Every generator in `active_units` has reported back
- [ ] Every report includes a changeset_id
- [ ] `changeset_ids` has one entry per unit

**If gate fails** → re-dispatch crashed generators. Do NOT proceed until all have submitted.
**If gate passes** → set `changeset_ids = [...]`. Proceed to Phase 3.

---

### PHASE 3 — LAND

**Entry check**: `changeset_ids` must be non-empty. If empty → STOP, go back to Phase 2.

1. **Verify in PARALLEL** — `dk_verify` ALL changesets simultaneously
2. **Approve** — `dk_approve` each verified changeset
3. **Merge sequentially** — `dk_merge` each (merge order doesn't matter — no wave dependencies)

Handle conflicts: dkod auto-resolves AST-level conflicts. For true symbol conflicts,
use `keep_yours` for the changeset being merged — the next changeset will auto-rebase.

⚠️ **DO NOT call `dk_push` after landing. Shipping is Phase 5 only.**

**═══ GATE 3 CHECK ═══**
Before proceeding, verify:
- [ ] Every changeset is either merged OR recorded in `merge_failures` with reason
- [ ] At least one changeset merged successfully (a `merged_commit` hash exists)
- [ ] Verification/merge failures are recorded for the eval phase

**If zero merges** → all generators failed. Re-run planner or re-dispatch generators.
**If some merged** → set `merged_commit = <hash>`, record `merge_failures`. Proceed to Phase 4.

---
```

**Step 4: Simplify Round Transition (lines 272-294)**

Remove `waves`, `current_wave`, `pushed` from reset block:

```
# ROUND TRANSITION — execute this before re-entering Phase 2:
round += 1
active_units = [only the units whose criteria failed in eval]
changeset_ids = []          # wiped — new generators will repopulate
merged_commit = null        # wiped — new merges will set this
merge_failures = []         # wiped
eval_reports = []           # wiped — new evaluators will repopulate
# plan remains unchanged
```

**Step 5: Update decision table (lines 316-327)**

Remove the "Multiple waves exist" row entirely.

**Step 6: Update bottom state block (lines 379-398)**

Remove `waves`, `current_wave`, `pushed` from the final state block. Simplify self-check:

```
**Self-check before dk_push** (run this EVERY time before calling dk_push):
1. "Is `eval_reports` populated with scores for every criterion? If NO → STOP. Phase 4 incomplete."
2. "Am I in Phase 5? If NO → STOP. dk_push is only allowed in Phase 5."
```

**Step 7: Commit**

```bash
git add agents/orchestrator.md
git commit -m "Replace per-wave loop with single-blast dispatch in orchestrator"
```

---

### Task 3: Rewrite `skills/dkh/SKILL.md` — Remove waves from flow diagram and gates

**Files:**
- Modify: `skills/dkh/SKILL.md`

**Step 1: Update Phase 1 in flow diagram (lines 70-83)**

Remove line 76 "Map dependencies between units" and line 81 "Dependency graph with wave assignments". Replace line 81 with:
```
│  ✓ No duplicate symbol ownership                  │
```

**Step 2: Replace Phase 2+3 box in flow diagram (lines 86-116)**

Replace with single-blast Build + Land (no wave loop):

```
┌─────────────────────────────────────────────────────┐
│  PHASE 2: BUILD (all units simultaneously)          │
│  ALL N generators dispatched in a single blast      │
│  Each agent:                                        │
│  • dk_connect (own session, own overlay)            │
│  • dk_context (understand target symbols)           │
│  • dk_file_read → dk_file_write (implement)         │
│  • dk_submit (changeset)                            │
│                                                     │
│  GATE 2 — Required output:                          │
│  ✓ Every generator reported back with changeset_id  │
│  BLOCKED until all generators have submitted.       │
├─────────────────────────────────────────────────────┤
│  PHASE 3: LAND                                      │
│  Orchestrator merges all changesets:                 │
│  • dk_verify ALL changesets in PARALLEL              │
│  • dk_approve each verified changeset               │
│  • dk_merge each sequentially                       │
│  • dkod auto-resolves AST-level conflicts           │
│                                                     │
│  ⚠️  DO NOT dk_push after landing!                  │
│                                                     │
│  GATE 3 — Required output:                          │
│  ✓ Every changeset merged OR recorded as failed     │
│  ✓ At least one merged (commit hash exists)         │
│  BLOCKED until all changesets are resolved.          │
└────────────────────┬────────────────────────────────┘
```

**Step 3: Update Gate 1 CHECK in Orchestrator Behavior section (lines 204-213)**

Remove "Dependency graph exists with wave assignments" check. Replace with "No duplicate symbol ownership across work units". Keep all other checks.

**Step 4: Replace Phases 2+3 Build and Land section (lines 216-252)**

Replace with non-wave version:

```markdown
### Phase 2: Build
**GATE 1 ENTRY CHECK**: "Do I have a validated plan? YES → proceed."

Dispatch ALL generators simultaneously in a single message (one Agent call per unit).
Wait for all to complete.

**GATE 2 CHECK:**
- [ ] Every generator reported back with a changeset_id
- [ ] changeset_ids collected for all units

If a generator crashed → re-dispatch it. Do not proceed until all have submitted.

### Phase 3: Land
1. **Verify in parallel**: `dk_verify` ALL changesets simultaneously
2. **Approve all verified**: `dk_approve` each
3. **Merge sequentially**: `dk_merge` each
4. Handle conflicts: dkod auto-resolves. For true conflicts, `dk_resolve` → retry.

⚠️ **DO NOT dk_push after landing. Shipping is Phase 5 only.**

**GATE 3 CHECK:**
- [ ] Every changeset merged OR recorded as failed with reason
- [ ] At least one changeset merged (commit hash exists)

Zero merges is a hard block — re-dispatch generators.
```

**Step 5: Update gate enforcement rule 6 (lines 185-189)**

Remove "Not between waves." — waves don't exist. Simplify to:

```markdown
6. **dk_push is ONLY allowed in Phase 5.** Not after Phase 3. Not "just to save progress."
   `dk_push` creates a branch or PR on GitHub — that is SHIPPING. Shipping happens after
   eval. `dk_merge` commits code locally — that is LANDING. Landing is Phase 3. Shipping
   is Phase 5.
```

**Step 6: Update Prime Directive section (lines 296-333)**

Remove wave references:
- Line 318: "The planner designs for maximum Wave 1 coverage — flatten the dependency graph so the most units run simultaneously." → "The planner produces N units with non-overlapping symbol ownership. All dispatch at once."
- Line 320: "ALL generators in a wave dispatch in a single message" → "ALL generators dispatch in a single message"

**Step 7: Replace Work Unit Schema (lines 368-386)**

Remove `Files touched`, `Depends on`, wave references:

```markdown
## Work Unit Schema

The Planner produces work units in this structure (embedded in the plan artifact):

\```
## Work Unit: <id>
**Title:** <descriptive title>
**OWNS (exclusive):** <symbols this unit is the sole owner of>
**Creates:** <new symbols with file paths>
**Acceptance criteria:**
- <testable criterion 1>
- <testable criterion 2>
**Complexity:** low | medium | high
\```

All units dispatch simultaneously. The orchestrator sends one Agent call per unit
in a single message — N generators run at once.
```

**Step 8: Commit**

```bash
git add skills/dkh/SKILL.md
git commit -m "Remove waves from SKILL.md flow diagram, gates, and work unit schema"
```

---

### Task 4: Rewrite `skills/dkh/references/planning-guide.md` — Kill sequential patterns

**Files:**
- Modify: `skills/dkh/references/planning-guide.md`

**Step 1: Add "dkod means no serialization" section at top (after line 6)**

Insert after "Deep reference for the Planner agent...":

```markdown
## dkod Eliminates Serialization

With dkod, there is NO reason to serialize work between generators:

- **Same file?** dkod merges at the AST level. Two generators editing different functions
  in the same file → zero conflicts, auto-merged.
- **Same config file?** dkod merges JSON at the key level, code at the symbol level.
- **Import dependencies?** Generators inline their own types. No waiting for another unit.
- **Scaffolding needed first?** No — scaffolding runs in parallel with features. dkod
  merges the scaffolding generator's `package.json` with the feature generator's code.

**ALL units dispatch simultaneously. There are no waves. There are no dependencies.**
The planner's only structural constraint is symbol ownership — no two units may create
or modify the same function, component, or class.
```

**Step 2: Kill Pattern 2 "Layer-Based Decomposition" (lines 38-57)**

Delete the entire section. It creates a sequential schema → API → frontend chain.

**Step 3: Kill Pattern 4 "Setup + Feature Waves" (lines 84-99)**

Delete the entire section including the 3-wave example. Waves don't exist.

**Step 4: Rewrite "What Creates a Dependency" → "What is NOT a Dependency" (lines 103-119)**

Replace both "What Creates a Dependency" and "What Does NOT Create a Dependency" with:

```markdown
## What is NOT a Dependency

With dkod, almost nothing creates a real dependency between work units:

| Situation | Why it's NOT a dependency |
|-----------|--------------------------|
| Two units touch the same file | dkod merges at the AST level — different symbols auto-merge |
| Unit B needs types from Unit A | Unit B defines its own types inline. No import needed. |
| Unit B calls an API that Unit A builds | Unit B uses mock data or hardcoded responses. |
| Unit B renders a component Unit A creates | Unit B creates its own version. dkod deduplicates. |
| Both units add to the same config file | dkod merges config keys automatically |
| Both units import the same utility | Both import independently — no coordination needed |
| Unit B needs a database table Unit A defines | Unit B defines its own schema inline |
| Scaffolding must exist before features | No — scaffolding runs in parallel. dkod merges. |

**The only real constraint is symbol ownership.** Two generators writing the same function
body creates a true AST conflict. Assign each symbol to exactly one unit.
```

**Step 5: Rewrite "Minimize Dependencies" → "There Are No Dependencies" (lines 123-133)**

Replace with:

```markdown
## There Are No Dependencies

Every unit is independent. The planner does not create dependency graphs, wave assignments,
or `depends_on` fields. All units dispatch simultaneously.

If you catch yourself thinking "Unit B should run after Unit A because..." — stop. Ask:
- Can Unit B inline the types it needs? → Yes. Do that.
- Can Unit B use mock data for Unit A's API? → Yes. Do that.
- Can Unit B create its own version of what it needs? → Yes. Do that.

The integration evaluator will catch mismatches after all units merge. Fix rounds handle
any wiring issues.
```

**Step 6: Remove wave references from any remaining sections**

Scan for "Wave 1", "Wave 2", "wave" and remove all references.

**Step 7: Commit**

```bash
git add skills/dkh/references/planning-guide.md
git commit -m "Kill sequential patterns from planning guide — no dependencies, no waves"
```

---

### Task 5: Rewrite `skills/dkh/references/dkod-patterns.md` — Kill stale patterns

**Files:**
- Modify: `skills/dkh/references/dkod-patterns.md`

**Step 1: Rewrite "Greenfield Project Setup" (lines 229-240)**

Replace "The first generator should scaffold..." with:

```markdown
### Greenfield Project Setup

Scaffolding runs in PARALLEL with feature generators. The scaffolding generator creates
`package.json`, `tsconfig.json`, `vite.config.ts`, `src/main.tsx`, `src/App.tsx`, and
`index.html` — while feature generators create their own files simultaneously.

dkod merges the scaffolding generator's output with all feature generators' output at
the AST level. There is no need to "scaffold first, then build features."

```
dk_connect(agent_name: "generator-scaffolding", intent: "Project scaffolding")
dk_file_write("package.json", "...")     # ← runs at the same time as feature generators
dk_file_write("src/App.tsx", "...")       # ← other generators create their own files
dk_submit(intent: "Project scaffolding with Vite + React + TypeScript")
```
```

**Step 2: Kill "Shared Types Pattern" (lines 247-258)**

Delete the entire section. Replace with:

```markdown
### Inline Types Pattern

Generators define their own types locally instead of sharing a central type file.
This eliminates coordination between generators.

```
// Generator for Task UI defines its own Task type:
interface Task { id: string; title: string; status: 'todo' | 'done'; }

// Generator for Task API also defines Task — that's fine.
// If the types diverge, the evaluator catches it. Fix rounds align them.
```

Type duplication is cheap. Coordination between generators is expensive.
```

**Step 3: Simplify Landing Pipeline ordering (lines 147-165)**

Remove wave ordering references. Replace:

```markdown
### Sequential Landing

```
for each changeset:
  1. dk_verify(changeset_id)
     → if FAIL: record failures, skip to next
  2. dk_approve()
  3. dk_merge(commit_message)
     → if MergeConflict: resolve and retry
     → if OverwriteWarning: merge with force: true
     → if MergeSuccess: continue
```

**Why sequential?** Each `dk_merge` creates a new commit. The next changeset's merge must
rebase against the new HEAD. dkod handles this automatically (AST-level rebase), but the
merges must happen one at a time.

**Ordering:** Merge order doesn't matter — all units are independent. dkod auto-rebases
each changeset against the latest HEAD.
```

**Step 4: Simplify "Config File Merging" (lines 262-272)**

Replace with:

```markdown
### Config File Merging

dkod handles config file merging automatically:
- **Code files** (`.ts`, `.tsx`, `.py`): AST-level merge — different symbols auto-merge
- **JSON files** (`package.json`): key-level merge — different keys auto-merge, same key conflicts
- **Other config** (`.env`, `.gitignore`): line-level merge

Multiple generators can add dependencies to `package.json`, routes to `router.ts`, or
entries to `.env` — dkod merges them all. No special handling needed.
```

**Step 5: Commit**

```bash
git add skills/dkh/references/dkod-patterns.md
git commit -m "Kill stale sequential patterns from dkod-patterns reference guide"
```

---

### Task 6: Fix `agents/evaluator.md` and `agents/generator.md` — Minor corrections

**Files:**
- Modify: `agents/evaluator.md:15-18`
- Modify: `agents/generator.md:53-54`

**Step 1: Fix evaluator.md line 15-18**

Replace:
```
You may be one of N evaluators running simultaneously as a Claude Code agent team — each
evaluator testing a different work unit's criteria in parallel. Or you may be the integration
evaluator testing overall criteria. Either way, your scope is defined by the criteria you
receive.
```

With:
```
You are one of several evaluators that run SEQUENTIALLY (one at a time). Each evaluator
tests a different work unit's criteria. You may also be the integration evaluator testing
overall criteria. Evaluators run sequentially because they share a single chrome-devtools
browser session — you have exclusive access while you run. Your scope is defined by the
criteria you receive.
```

**Step 2: Fix generator.md lines 53-54**

Replace:
```
If this is a later wave (your dependencies have already merged), read the files that earlier
generators created — they're now in the base codebase.
```

With:
```
Your dkod session sees the base codebase snapshot at connection time. Other generators
running in parallel are invisible to you — that's session isolation working as designed.
```

**Step 3: Commit**

```bash
git add agents/evaluator.md agents/generator.md
git commit -m "Fix evaluator sequential description and remove wave reference from generator"
```

---

### Task 7: Update `README.md` — Remove wave references

**Files:**
- Modify: `README.md`

**Step 1: Scan README for wave/dependency language**

Look for "Wave", "wave", "dependency", "sequentially" in the context of generators. Update the architecture diagram description and the "autonomous loop" section to reflect single-blast dispatch.

**Step 2: Commit**

```bash
git add README.md
git commit -m "Update README to reflect single-blast dispatch model"
```

---

### Task 8: Final verification

**Step 1: Grep for stale references**

```bash
grep -rn -i "wave\|depends.on\|dependency.graph\|parallelism.score\|files.touched" agents/ skills/ --include="*.md" | grep -v "node_modules"
```

Expected: zero results (or only "wave" in legitimate non-harness contexts like "sound wave").

**Step 2: Grep for `depends_on` field references**

```bash
grep -rn "depends_on\|Depends on" agents/ skills/ --include="*.md"
```

Expected: zero results.

**Step 3: Verify no file references "Wave 1" or "Wave 2"**

```bash
grep -rn "Wave [0-9]" agents/ skills/ --include="*.md"
```

Expected: zero results.

**Step 4: Commit any stragglers**

If any wave references remain, fix them and commit.

```bash
git add -A
git commit -m "Remove remaining wave references found by verification grep"
```

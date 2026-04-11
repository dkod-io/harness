# Symbol-Level Locking + Streaming Merge

**Date:** 2026-04-11
**Status:** Approved
**Scope:** dkod-engine, dk-mcp, dkod-harness

## Problem

The current dkod platform uses advisory conflict detection — `dk_file_write` returns
`conflict_warnings` that agents can ignore. Conflicts only surface at merge time, after
agents have spent minutes and thousands of tokens writing code. The harness tries to
coordinate concurrency through planning (symbol ownership), but:

1. The harness is not the only client — multiple developers with independent agents need
   the engine itself to enforce safe concurrent access.
2. Batch-then-merge (all generators submit, then orchestrator merges sequentially) delays
   conflict detection to the latest possible moment.
3. Advisory warnings are routinely ignored, causing expensive merge conflicts and
   post-merge integration fixes (46K+ tokens observed).

## Solution: Symbol-Level Blocking Locks + Streaming Merge

Two coupled changes that make the engine the source of truth for concurrency:

### 1. Symbol-Level Locking (Engine)

Make `SymbolClaimTracker` blocking instead of advisory. When an agent writes a symbol,
it acquires a lock. Other agents writing the SAME symbol get blocked. Different symbols
in the same file proceed freely — preserving dkod's core differentiator.

### 2. Streaming Merge (Harness)

Each generator owns its full pipeline: write → submit → verify → review → approve → merge.
No batch phase. Locks release on merge, unblocking other generators immediately. The
orchestrator simplifies from 6 phases to 5 — Phase 3 (LAND) collapses into Phase 2 (BUILD).

## Detailed Design

### Engine: SymbolClaimTracker Changes

**File:** `crates/dk-engine/src/conflict/claim_tracker.rs`

Current `record_claim` becomes `acquire_lock`:
- Returns `Ok(())` if no other session holds the symbol
- Returns `Err(SymbolLocked { symbol, locked_by, session_id, locked_since })` if another
  session holds it
- Same-session re-acquisition always succeeds (idempotent)

Current `check_conflicts` becomes `check_locks` (terminology shift, same logic).

Add `release_locks(session_id)`:
- Removes all claims for the given session
- Called on: `dk_merge` success, `dk_close`, session timeout (30 min)
- Emits `symbol.lock.released` event for each released symbol

### MCP: dk_file_write Response Changes

**File:** `crates/dk-mcp/src/server.rs`

After AST symbol extraction, call `acquire_lock` instead of `record_claim`. New response
status when lock is held by another session:

```
{
  status: "locked",
  locked_symbols: [
    {
      symbol: "useTaskStore",
      file_path: "src/stores/useTaskStore.ts",
      locked_by: "generator-unit-2",
      session_id: "...",
      locked_since: "2026-04-11T06:30:00Z"
    }
  ],
  message: "Symbols locked by another agent. Call dk_watch(filter: 'symbol.lock.released') to wait, then dk_file_read and retry."
}
```

This is a status response, not an error. The write did not happen. The agent should watch
and retry.

### MCP: New Watch Event

**Event type:** `symbol.lock.released`

```
{
  event_type: "symbol.lock.released",
  symbols: ["useTaskStore", "Task"],
  file_path: "src/stores/useTaskStore.ts",
  released_by: "generator-unit-2",
  session_id: "...",
  repo_id: "..."
}
```

Emitted when `release_locks` is called (merge, close, timeout). Blocked agents watching
for this event wake up and retry their `dk_file_write`.

### MCP: dk_merge Lock Release

After successful merge in `dk_merge` handler:
1. Call `release_locks(session_id)` to free all symbol locks
2. Emit `symbol.lock.released` events for all released symbols
3. Return `MergeSuccess` as today

Same release in `dk_close` handler (already cleans up, just needs event emission).

### Lock Lifecycle

```
dk_file_write(symbol X)
  → acquire_lock(session_id, symbol X)
  → LOCKED until one of:
    1. dk_merge succeeds → release_locks → symbol.lock.released event
    2. dk_close called → release_locks → symbol.lock.released event
    3. Session timeout (30 min) → release_locks → symbol.lock.released event
```

### Generator: Self-Merging Pipeline

**File:** `harness/skills/dkh/agents/generator.md`

Current flow:
```
write → submit → report(changeset_id) → DONE (orchestrator handles the rest)
```

New flow:
```
write → submit → verify → review-fix loop → approve → merge → report(merged_commit)
```

Tool constraints update:
- ALLOWED (new): `dk_verify`, `dk_approve`, `dk_merge`, `dk_resolve`
- These were previously FORBIDDEN (orchestrator-only)

Blocked generator flow:
```
dk_file_write("src/stores.ts", useProjectStore)
  → SYMBOL_LOCKED: useTaskStore locked by generator-unit-2
  → dk_watch(filter: "symbol.lock.released")    ← blocks until unit-2 merges
  → dk_file_read("src/stores.ts")               ← sees unit-2's merged code
  → dk_file_write("src/stores.ts", content)     ← writes alongside unit-2's code
  → proceeds normally
```

Generator report changes:
```
## Generator Report
**Status:** merged                     ← was "submitted"
**Merged Commit:** abc123              ← new field
**Session ID:** ...
**Changeset ID:** ...
**Final review score:** local X/5, deep Y/5
**Rounds used:** N
```

New status values:
- `merged` — successfully merged, lock released
- `blocked_timeout` — couldn't acquire lock within time budget
- `review_failed` — couldn't pass review (local < 4/5 or deep < 4/5) after max rounds
- `conflict_unresolved` — dk_merge conflict couldn't be self-resolved

### Orchestrator: Phase Simplification

**File:** `harness/skills/dkh/agents/orchestrator.md`

Current phases:
```
1. PLAN
2. BUILD (dispatch generators, wait for submit)
3. LAND (verify → review → approve → merge sequentially)
4. FILE SYNC + SMOKE TEST
5. EVAL
6. SHIP or FIX
```

New phases:
```
1. PLAN
2. BUILD + LAND (dispatch generators, each self-merges)
3. FILE SYNC + SMOKE TEST
4. EVAL
5. SHIP or FIX
```

Phase 3 (LAND) is removed entirely. The orchestrator no longer calls `dk_verify`,
`dk_review`, `dk_approve`, or `dk_merge` — generators do this themselves.

State simplification:
```
round: 1
plan: null
active_units: []
merged_units: []           ← replaces changeset_ids
merge_failures: []
eval_reports: []
unit_attempts: {}
blocked_units: []
replan_count: 0
```

Removed state:
- `changeset_ids` → generators merge themselves, report merged_commit
- `session_map` → generators close their own sessions
- `review_round` → generators track their own review rounds

Gate 2 check simplifies to:
- Every generator reported back
- Count `merged` vs `blocked_timeout` / `review_failed` / `conflict_unresolved`
- At least one generator merged successfully
- Failed units → increment `unit_attempts`, re-dispatch or block

### Review Quality Gates

Unchanged targets (applied by generators internally):
- **Local review:** must be ≥ 4/5 with no severity:"error"
- **Deep review:** must be ≥ 4/5 with no severity:"error"
- **Max rounds:** 10 per generator
- **Max-rounds fallback:** local ≥ 4/5 AND deep ≥ 3/5 → approve with warning. Otherwise skip.

## What Does NOT Change

- `dk_connect` / `dk_close` / `dk_context` / `dk_file_read` / `dk_file_list` — unchanged
- `dk_submit` — unchanged (locks already acquired during writes)
- `dk_push` — unchanged (orchestrator-only, Phase 5)
- `dk_watch` — unchanged (new event type added, existing filter mechanism works)
- AST-level merge semantics — unchanged (different symbols = no conflict)
- Planner — unchanged (symbol ownership + file manifest still valuable for reducing contention)
- Evaluator — unchanged
- Session overlay isolation — unchanged (writes still go to overlay, invisible to others)

## New MCP Tools Required

**Zero.** Everything works through existing tools with new response statuses and event types.

## Benefits

1. **Engine enforces concurrency** — no harness required for safety
2. **Works for any number of developers/agents** — no coordination needed
3. **Earlier conflict detection** — at write time, not merge time
4. **Less token waste** — blocked agents wait instead of writing code that will conflict
5. **Faster unblocking** — streaming merge releases locks as soon as each unit merges
6. **Simpler harness** — orchestrator drops ~150 lines (Phase 3 + state tracking)
7. **Greenfield improvement** — early-merging generators make files available to later ones

## Implementation Order

1. **Engine: SymbolClaimTracker** — `acquire_lock` / `release_locks` / `symbol.lock.released` event
2. **MCP: dk_file_write** — return `SYMBOL_LOCKED` status instead of `conflict_warnings`
3. **MCP: dk_merge / dk_close** — call `release_locks`, emit events
4. **Harness: generator.md** — add verify/approve/merge to pipeline, handle SYMBOL_LOCKED
5. **Harness: orchestrator.md** — remove Phase 3, simplify state
6. **Harness: dkod-patterns.md** — update reference docs

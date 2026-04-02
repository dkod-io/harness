# dkod Patterns: Session Lifecycle and Merge Strategies

Deep reference for all agents on dkod-specific patterns — session management, merge
sequencing, conflict resolution, and the landing pipeline.

## Session Lifecycle

### Connect Phase

Every agent (planner, generator, evaluator) creates its own session:

```
dk_connect(
  agent_name: "generator-unit-3",
  intent: "Implement task CRUD API",
  codebase: "org/repo"
)
→ session_id, changeset_id, codebase_summary
```

**Rules:**
- One session per agent. Never share sessions.
- Use descriptive `agent_name` — it appears in conflict reports and event streams.
- Use descriptive `intent` — it's recorded in the changeset for traceability.
- The session sees a consistent snapshot at connection time. Other agents' uncommitted
  work is invisible.

### Read Phase

Use `dk_context` for semantic understanding:
```
dk_context(query: "createTask", depth: "FULL")
→ symbol source, callers, callees, dependencies
```

Use `dk_file_read` for raw file content:
```
dk_file_read(path: "src/api/tasks.ts")
→ file content (overlay version if modified, base version otherwise)
```

Use `dk_file_list` for directory structure:
```
dk_file_list(prefix: "src/", only_modified: false)
→ all files under src/ with modification metadata
```

### Write Phase

```
dk_file_write(path: "src/api/tasks.ts", content: "<full file content>")
→ new_hash, detected_changes, conflict_warnings
```

**Key behaviors:**
- Writes go to the session overlay only — invisible to other agents.
- Always write the COMPLETE file content, not patches.
- The response includes `detected_changes` — which symbols dkod detected as added/modified.
- The response may include `conflict_warnings` if another agent has touched the same file.
  Warnings are informational — they don't block the write.

### Submit Phase

```
dk_submit(intent: "Implement task CRUD API")
→ status: ACCEPTED | CONFLICT, changeset_id
```

**What happens on submit:**
1. dkod diffs the overlay against the base snapshot
2. If the base has moved (another agent merged), dkod auto-rebases
3. If auto-rebase succeeds, changeset is accepted
4. If auto-rebase finds symbol-level conflicts, returns CONFLICT with details

**On CONFLICT:** The response includes `conflicting_symbols[]` with:
- `file_path` — which file
- `qualified_name` — which symbol (e.g., "src/api/tasks.ts::createTask")
- `conflicting_agent` — who else touched it
- `their_change` / `your_change` / `base_version` — three-way context

### Verify Phase

```
dk_verify(changeset_id: "...")
→ stream of VerifyStepResult (lint, type-check, test, semantic)
```

**Steps run in order:**
1. **Lint** — ESLint, Clippy, etc.
2. **Type-check** — TypeScript, Rust compiler, etc.
3. **Test** — Only tests affected by changed symbols
4. **Semantic** — API compatibility, safety analysis

Each step returns: status (pass/fail), output, findings[], suggestions[].

### Approve Phase

```
dk_approve()
→ changeset transitions to "approved" state
```

Pre-condition: changeset must be in "submitted" state with no unresolved conflicts.

### Merge Phase

```
dk_merge(commit_message: "feat: implement task CRUD API")
→ MergeSuccess | MergeConflict | OverwriteWarning
```

**MergeSuccess:**
```
{ commit_hash, auto_rebased: true/false, auto_rebased_files: [...] }
```

**MergeConflict:**
```
{ conflicts: [{ file_path, symbols, your_agent, their_agent, conflict_type }] }
```

**OverwriteWarning:**
```
{ recently_merged_symbols: [{ symbol, merged_by, merged_at }] }
→ Can proceed with force: true
```

### Push Phase

```
dk_push(mode: "pr", branch_name: "feat/task-management", pr_title: "...")
→ pr_url, branch_name, commit_hash
```

**Modes:**
- `"branch"` — push to a branch only
- `"pr"` — push to branch + open GitHub PR

## The Landing Pipeline

After all generators have submitted, the orchestrator runs the landing pipeline.
Order matters — each merge may advance the HEAD, affecting subsequent merges.

### Sequential Landing

```
for each changeset (in dependency order):
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

**Ordering strategy:**
1. Merge Wave 1 units first (no dependencies)
2. Then Wave 2 units (depend on Wave 1)
3. Then Wave 3 units (depend on Wave 2)

Within a wave, order doesn't matter — the units are independent by design.

### Conflict Resolution Strategies

**Auto-merge (no action needed):**
- Different functions in the same file → dkod auto-merges
- Different fields added to the same type → dkod auto-merges
- Same import added by both agents → dkod deduplicates

**Non-overlapping conflict (auto-resolve):**
```
dk_resolve(resolution: "proceed")
→ reconnect and re-submit with updated base
```

**True conflict (orchestrator decides):**

For the autonomous harness, use this decision tree:

1. Is the conflicting symbol in a later-wave unit?
   → `keep_yours` (the later wave should take precedence — it was designed to work with
   the earlier wave's output)

2. Is it in the same wave?
   → `keep_yours` for the changeset being merged. The other changeset will auto-rebase
   on its turn. If it can't, the evaluator will catch the integration issue.

3. Is it a setup/scaffolding conflict (package.json, config files)?
   → `keep_yours` with `force: true`. These are usually additive (both agents adding
   dependencies or config entries) and dkod will merge them.

### Post-Landing Verification

After all changesets are merged, run one final `dk_verify` on the complete codebase:
```
dk_connect(agent_name: "harness-verifier", intent: "Final verification")
dk_verify()
```

This catches integration issues that per-changeset verification might miss:
- Missing imports between merged modules
- Type mismatches at integration boundaries
- Test failures from symbol interactions

## Event Monitoring

The orchestrator can monitor generator progress via `dk_watch`:

```
dk_watch(filter: "changeset.*")
→ stream of events:
  - changeset.submitted (agent-3 submitted)
  - changeset.verify_started (agent-3 verification running)
  - changeset.merged (agent-1 merged successfully)
```

Use this to:
- Track which generators have finished
- Detect conflicts early (before the landing phase)
- Monitor verification progress

## Common Patterns

### Greenfield Project Setup

The first generator should scaffold the project:
```
dk_connect(agent_name: "generator-scaffolding", intent: "Project scaffolding")
dk_file_write("package.json", "...")
dk_file_write("tsconfig.json", "...")
dk_file_write("vite.config.ts", "...")
dk_file_write("src/main.tsx", "...")
dk_file_write("src/App.tsx", "...")
dk_file_write("index.html", "...")
dk_submit(intent: "Project scaffolding with Vite + React + TypeScript")
```

This must be in Wave 1 and merged before feature generators start.

### Shared Types Pattern

If multiple generators need the same TypeScript interfaces:
```
// Wave 1 unit: "Shared types and interfaces"
dk_file_write("src/types/index.ts", `
  export interface User { id: string; email: string; name: string; }
  export interface Task { id: string; title: string; userId: string; status: TaskStatus; }
  export type TaskStatus = 'todo' | 'in_progress' | 'done';
`)
dk_submit(intent: "Shared type definitions")
```

All Wave 2 generators can then import from `src/types/index.ts`.

### Config File Merging

Multiple generators may need to add to the same config:
- Routes in `src/router.ts`
- Dependencies in `package.json`
- Environment variables in `.env`

dkod handles these via AST merge for code files. For JSON files (package.json), dkod
merges at the key level — different keys auto-merge, same key conflicts.

**Best practice:** Have the scaffolding unit create the config skeleton, and feature
generators add only their specific entries.

### Test Alongside Implementation

A generator can write tests as part of its unit:
```
dk_file_write("src/api/tasks.ts", "... implementation ...")
dk_file_write("src/api/__tests__/tasks.test.ts", "... tests ...")
dk_submit(intent: "Task API with tests")
```

Then `dk_verify` runs those tests as part of verification.

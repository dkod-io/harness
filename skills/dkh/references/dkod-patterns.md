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
- **Symbol locking:** Each symbol you write acquires a lock. If another agent already holds
  the lock on the same symbol, the write returns `status: "locked"` with details about
  the lock holder. Your write DID NOT happen — you must wait and retry.
- Different symbols in the same file do NOT contend — only same-symbol writes are blocked.

**If `status: "locked"` is returned:**
```
dk_watch(filter: "symbol.lock.released", wait: true)   # blocks until lock releases
dk_file_read(path)                                      # read their merged code
dk_file_write(path, adapted_content)                    # write alongside their code
```

**Lock lifecycle:** Acquired on `dk_file_write`, held through submit/review/merge,
released on `dk_merge` success, `dk_close`, or session timeout (30 min).

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

## Streaming Merge Pipeline

Each generator owns its full pipeline: write → submit → verify → review → approve → merge.
There is no batch phase — generators merge as soon as they pass review. Symbol locks
ensure safe concurrent access at the engine level.

### Generator Self-Merge Flow

```
# Each generator runs this autonomously:
dk_connect(...)
dk_file_write(...)         # acquires symbol locks
dk_submit(intent)
dk_verify(changeset_id)
# review-fix loop (max 10 rounds)
dk_approve(changeset_id)
dk_merge(changeset_id)     # releases symbol locks → unblocks other generators
dk_close(session_id)
```

**Why streaming?** Symbol locks are held until merge. If generators submit-and-wait
for batch merge, locks block other generators indefinitely (deadlock). Streaming merge
releases locks as soon as each generator finishes, maximizing parallelism.

### Conflict Resolution (by generators)

**Auto-merge (no action needed):**
- Different functions in the same file → dkod auto-merges
- Different fields added to the same type → dkod auto-merges
- Same import added by both agents → dkod deduplicates

**MergeConflict:**
```
dk_resolve(resolution: "proceed")   # accept your changes
dk_merge(changeset_id)              # retry
```

**OverwriteWarning:**
```
dk_merge(changeset_id, force: true)  # your version is authoritative
```

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

Scaffolding runs in parallel with feature generators -- no need to scaffold first. dkod
merges the scaffolding output with all feature generator outputs at the AST level.

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

This runs concurrently with all other generators. During landing, dkod auto-merges the
scaffolding files with feature code regardless of merge order.

### Inline Types Pattern

Generators define their own types locally instead of sharing a central type file.
Type duplication is cheap; coordination between generators is expensive.

```
// generator-tasks defines its own types
dk_file_write("src/api/tasks.ts", `
  interface Task { id: string; title: string; userId: string; status: TaskStatus; }
  type TaskStatus = 'todo' | 'in_progress' | 'done';
  // ... implementation using these types
`)

// generator-users defines its own types independently
dk_file_write("src/api/users.ts", `
  interface User { id: string; email: string; name: string; }
  // ... implementation using these types
`)
```

Each generator is self-contained. If types need to be shared later, the evaluator or a
post-landing cleanup pass can consolidate them.

### Config File Merging

dkod handles config file merging automatically. Multiple generators can add to the same
files simultaneously -- no special handling needed.

- **Code files** (router.ts, etc.) -- merged at the AST level
- **JSON files** (package.json) -- merged at the key level
- **Env files** (.env) -- merged line by line

Multiple generators can add to `package.json`, `router.ts`, and `.env` at the same time.
Different keys/routes/variables auto-merge; only true conflicts (same key, different value)
require resolution.

### Test Alongside Implementation

A generator can write tests as part of its unit:
```
dk_file_write("src/api/tasks.ts", "... implementation ...")
dk_file_write("src/api/__tests__/tasks.test.ts", "... tests ...")
dk_submit(intent: "Task API with tests")
```

Then `dk_verify` runs those tests as part of verification.

## Code Review

Code review runs automatically after every `dk_submit`. Two tiers:

**Local review** (synchronous, included in submit response):
- Pattern-based checks: security issues, test gaps, convention violations
- Score returned inline: `review_summary.score` (1-5)
- Full findings via `dk_review(changeset_id)`

**Deep review** (asynchronous, requires LLM API key):
- Multi-pass LLM analysis: security, logic, architecture, conventions
- Notification via `dk_watch`: `changeset.review.completed` event
- Full findings via `dk_review(changeset_id)`

**Score guide:**

| Score | Meaning | Generator action |
|-------|---------|------------------|
| 5 | No issues | Exit — changeset is clean |
| 4 | Minor warnings | Exit — unless `severity:"error"` finding present, then fix + re-submit |
| 3 | Test gaps or conventions | Fix + re-submit |
| 2 | Errors found | Fix + re-submit |
| 1 | Security issues | Fix + re-submit |

### Generator Review-Fix Loop

**Generators own the review-fix lifecycle.** After `dk_submit`, each generator stays alive
to receive both local and deep review feedback and fix issues before exiting.

```
round = 1
dk_submit(intent)  → response includes local review (score + findings)

LOOP while round ≤ 3:

  # 1. Check LOCAL review (inline with the dk_submit that just ran)
  if local review has severity:"error" findings:
    fix files based on local findings
    round += 1
    if round > 3 → break
    dk_submit(intent)  → new local review
    continue           → re-check local on the new submission

  # 2. Local is clean — wait for DEEP review
  dk_watch(filter: "changeset.review.completed", wait: true)  — blocks, zero LLM cost
  dk_review(changeset_id) → deep findings + score

  # 3. Check deep review
  if score ≥ 4 AND no severity:"error" findings:
    break  → changeset is clean

  # 4. Deep found issues — fix and re-submit
  fix files based on deep findings
  round += 1
  if round > 3 → break
  dk_submit(intent)  → new local review
  # loop continues — re-check local before waiting for deep again

EXIT: return { changeset_id, final_score, rounds_used }
```

**Round counting:** every `dk_submit` (including the initial one) is a round. A local-error
re-submit consumes a round. This prevents local↔deep ping-pong from exceeding 3 total
submissions.

**Key behaviors:**
- After every `dk_submit`, local review is re-checked before proceeding to `dk_watch` —
  no silent carrying of unfixed local errors into the deep-review wait
- `dk_watch` blocks at tool level — zero LLM inference while waiting for deep review
- Max 3 rounds (= 3 submissions), then exit regardless of score (review is advisory)
- The orchestrator collects changeset_ids + scores from generator exits — no dk_status
  parsing or session-to-changeset mapping needed in LAND phase
- All generators run their review-fix loops in parallel (isolated sessions)

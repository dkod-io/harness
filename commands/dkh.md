---
description: "Autonomous build — one prompt in, working tested PR out. Full Planner → Generator → Evaluator pipeline powered by dkod parallel execution."
---

# /dkh — Autonomous Parallel Build

You received a build prompt. Execute the full autonomous harness pipeline.

**This is fully autonomous. Do not ask the user for clarification or input at any point.**

## Input

`$ARGUMENTS` contains the user's build prompt (e.g., "build a task management webapp with
authentication, categories, and a dashboard").

If `$ARGUMENTS` is empty, check if there's a `prompt.md` or `PROMPT.md` file in the current
directory and read it.

If both are empty, tell the user: "Usage: `/dkh <build prompt>` — describe what you want built."

## Execution

Load and follow the harness skill (`skills/harness/SKILL.md`) exactly:

1. **Prerequisites** — verify dkod MCP and chrome-devtools MCP are available
2. **Phase 1: Plan** — spawn planner agent with the prompt
3. **Phase 2: Build** — dispatch parallel generators (one per work unit)
4. **Phase 3: Land** — verify, approve, merge all changesets
5. **Phase 4: Eval** — spawn evaluator to test the merged application
6. **Phase 5: Ship or Fix** — push PR if passing, re-dispatch if failing (max 3 rounds)

## Output

When complete, report to the user:
- The PR URL
- What was built (feature summary)
- How many rounds it took
- How many generators ran in parallel
- Any criteria that still fail (if round 3 didn't fix everything)

## Decision Defaults

Do not ask the user. Use these defaults:

| Decision | Default |
|----------|---------|
| Frontend stack | React + Vite + TypeScript + Tailwind CSS |
| Backend stack | FastAPI + Python + SQLite |
| Package manager | npm |
| Test framework | Vitest (frontend), pytest (backend) |
| Styling | Tailwind CSS |
| Git branch | feat/<slug-from-prompt> |
| PR target | main |

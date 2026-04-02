---
description: "Run only the evaluation phase — test the current state of the application against acceptance criteria"
---

# /dkh:eval — Evaluate Only

Run only the evaluation phase of the harness. Tests the current application state against
provided or inferred acceptance criteria.

## Input

`$ARGUMENTS` is optional. If provided, it's treated as the spec or criteria reference.

If empty, the evaluator will:
1. Read any existing spec or plan files in the workspace
2. Infer criteria from the codebase structure
3. Run a general quality evaluation

## Execution

1. Verify chrome-devtools MCP is available (fall back to Bash-based testing if not)
2. Spawn the evaluator agent with:
   - Current codebase state
   - Acceptance criteria (from arguments, plan files, or inferred)
3. The evaluator:
   - Starts the dev server
   - Tests via chrome-devtools
   - Runs dk_verify
   - Scores each criterion
   - Kills background processes
4. Display the evaluation report

## Output

The structured eval report with:
- Overall PASS/FAIL
- Per-criterion scores with evidence
- Screenshots of key states
- Console error summary
- Specific fix hints for failures

This command is useful for:
- Evaluating work done outside the harness
- Re-evaluating after manual fixes
- Getting a quality assessment of existing code

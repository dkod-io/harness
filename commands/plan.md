---
description: "Run only the planning phase — produce a spec with parallel work units without building anything"
---

# /dkh:plan — Plan Only

Run only the planning phase of the harness. Produces a specification with parallel work
units but does not dispatch generators or build anything.

## Input

`$ARGUMENTS` contains the build prompt.

If empty: "Usage: `/dkh:plan <build prompt>` — describe what you want planned."

## Execution

1. Verify dkod MCP is available
2. Spawn the planner agent with the prompt
3. Display the resulting plan:
   - Specification summary
   - Work units with their acceptance criteria
   - Dependency graph (which units run in parallel, which depend on others)
   - Wave breakdown (Wave 1, Wave 2, etc.)

## Output

Present the plan in a clear format. The user can review it before running `/dkh` to
execute the full pipeline, or they can modify the prompt and re-plan.

This command is useful for:
- Reviewing the plan before committing to a full build
- Iterating on the decomposition strategy
- Understanding how the harness would approach a project

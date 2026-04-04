# Planning Guide: Symbol-Level Decomposition

Deep reference for the Planner agent on how to decompose work for parallel execution via dkod.

## The Core Principle

**Decompose by symbol, not by file.**

dkod merges code at the AST (Abstract Syntax Tree) level. It understands functions, classes,
types, and imports as independent units. Two generators editing different functions in the same
file produces zero conflicts — dkod auto-merges them in under 50ms.

This means your decomposition should target **symbols** (functions, classes, modules), not
files. A single file can be safely touched by multiple generators as long as they're working on
different symbols within it.

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

## Decomposition Patterns

### Pattern 1: Feature Vertical Slices

Split by feature, where each feature spans multiple layers:

```
Unit: "User Authentication"
  Symbols: loginHandler(), signupHandler(), AuthMiddleware, LoginPage, SignupForm
  Files: src/api/auth.ts, src/middleware/auth.ts, src/pages/Login.tsx, src/components/SignupForm.tsx

Unit: "Task Management"
  Symbols: createTask(), getTask(), updateTask(), TaskList, TaskCard, useTaskQuery()
  Files: src/api/tasks.ts, src/pages/Tasks.tsx, src/components/TaskCard.tsx, src/hooks/useTaskQuery.ts
```

Both units may touch `src/api/index.ts` (to register routes) — different symbols, auto-merged.

**When to use:** Features with low coupling. Each feature is mostly self-contained.

### Pattern 2: Batch Operation

Apply the same pattern across many modules:

```
Unit: "Add input validation to user endpoints"
  Symbols: createUserHandler(), updateUserHandler()
  Files: src/api/users.ts

Unit: "Add input validation to task endpoints"
  Symbols: createTaskHandler(), updateTaskHandler()
  Files: src/api/tasks.ts

Unit: "Add input validation to tag endpoints"
  Symbols: createTagHandler(), updateTagHandler()
  Files: src/api/tags.ts
```

All three run in parallel. Same pattern, different targets.

**When to use:** Repetitive changes across similar modules.

## What is NOT a Dependency

With dkod's AST-level merging, almost nothing that feels like a dependency actually is one:

| Situation | Why it is NOT a dependency |
|---|---|
| Two units edit the same file | dkod merges at the symbol level — different functions in the same file auto-merge |
| Unit B imports a type that Unit A defines | Each generator inlines its own types. No waiting. |
| Both units add to `package.json` | dkod merges JSON at the key level — additions from both units coexist |
| Both units add routes to a router | dkod deduplicates and merges route registrations |
| Unit B needs scaffolding that Unit A creates | Scaffolding runs in parallel. dkod merges the scaffolding output with feature code. |
| Both units create test files for different modules | Completely independent — no conflict possible |
| Unit B renders a component that Unit A creates | Unit B inlines a stub or its own version. The merge pass reconciles. |
| Both units use the same utility library | Both import independently — no coordination needed |

**The only real constraint is symbol ownership.** No two units may create or modify the
same function, component, or class. Everything else merges automatically.

### There Are No Dependencies

Every unit is independent. There are no dependency graphs. No wave assignments. No
`depends_on` fields.

If you catch yourself thinking "Unit B should run after Unit A" — stop. Inline what Unit B
needs. Let dkod merge the results. The planner never sequences units; it dispatches them
all at once and lets the merge engine handle the rest.

## Sizing Work Units

### Too Small (avoid)
```
Unit: "Add export to User model"
  1 line of code. This doesn't warrant a generator agent.
```

### Too Large (avoid)
```
Unit: "Implement the entire backend"
  This will take 60+ minutes and produces a massive changeset.
```

### Right Size (target)
```
Unit: "User authentication API with JWT"
  3-5 files, 10-20 functions, 200-500 lines of implementation.
  Takes a generator 10-20 minutes.
```

**Rule of thumb:** 5-15 acceptance criteria per unit. If you have fewer than 5, the unit is
too small — merge it with a related unit. If you have more than 15, split it.

## Acceptance Criteria Authoring

### Good Criteria (testable, specific)
```
- POST /api/tasks with valid body returns 201 and the created task object
- POST /api/tasks with missing title returns 400 with error message
- GET /api/tasks?status=completed returns only completed tasks
- Task list page shows loading spinner while fetching
- Clicking "Delete" on a task shows confirmation dialog before deleting
- Task card displays title, due date, and priority badge
```

### Bad Criteria (vague, untestable)
```
- API works correctly
- UI looks good
- Tasks can be managed
- Error handling is implemented
```

### Criteria Categories

Include criteria across these dimensions:

1. **Functionality** — Does the feature do what it's supposed to?
2. **Error handling** — Does it handle invalid input, empty states, failures?
3. **Integration** — Does it work with other features (auth, navigation)?
4. **UI/UX** — Does it render correctly, respond to interaction, show feedback?
5. **Edge cases** — Empty lists, long strings, special characters, concurrent actions

### Overall Criteria (always include)

Every plan should include these overall criteria:

```
- Application installs dependencies without errors
- Application starts dev server without errors
- Home/landing page loads and renders within 5 seconds
- No unhandled JavaScript errors in the console on any page
- Navigation between pages works without full page reloads
- Application is responsive at 375px and 1440px widths
```

## Contract Negotiation

After producing the plan, the orchestrator may run a **contract negotiation** step where
the evaluator reviews your criteria and tightens them. This is inspired by the adversarial-dev
pattern:

1. You produce work units with criteria
2. The evaluator reviews the criteria
3. The evaluator may add edge cases, increase specificity, or raise the bar
4. The negotiated criteria become the final contract

This prevents the "but that's not what I meant" problem. Both sides agree on what "done"
means before any code is written.

## Aggregation Symbol Pattern

Entry points are the most common source of true conflicts in parallel builds. Every
generator that adds a feature wants to register it in the entry point — but only one
generator can own that symbol.

### The Pattern

1. **Identify all aggregation symbols** — functions/components that wire the app together
2. **Assign each to exactly one unit** (usually scaffolding)
3. **The owner writes the FINAL version** with all imports pre-included
4. **Other units write only their implementations** in separate files

### Example: Tauri App

**Wrong (causes 5 conflicts):**
- WU-01 writes `lib.rs::run()` with its own commands
- WU-02 writes `lib.rs::run()` to add dkod commands → CONFLICT
- WU-03 writes `lib.rs::run()` to add repo commands → CONFLICT

**Right (zero conflicts):**
- WU-01 owns `lib.rs::run()` and writes it with ALL commands registered:
  `commands::dkod_connect, commands::repo_list, commands::file_open, ...`
- WU-02 writes `src/commands/dkod.rs` (its own file, no conflict)
- WU-03 writes `src/commands/repo.rs` (its own file, no conflict)

### Example: React App

**Wrong:**
- WU-01 writes `App.tsx` with layout shell
- WU-05 writes `App.tsx` to add routes → CONFLICT

**Right:**
- WU-01 owns `App.tsx` and writes it with ALL routes pre-imported
- WU-05 writes `src/pages/Dashboard.tsx` (its own file, no conflict)

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

### Pattern 2: Layer-Based Decomposition

Split by architectural layer when features are heavily interconnected:

```
Unit: "Database Schema and Models"
  Symbols: createSchema(), User, Task, Tag, migrate()
  Files: src/db/schema.ts, src/models/*.ts

Unit: "API Handlers"
  Symbols: all *Handler() functions, validation schemas
  Files: src/api/*.ts

Unit: "Frontend Components"
  Symbols: all React components, hooks, utility functions
  Files: src/components/*.tsx, src/hooks/*.ts, src/pages/*.tsx
```

**When to use:** Features share many dependencies (e.g., all API handlers use the same models).
Layer units have clear dependency order: schema → API → frontend.

### Pattern 3: Batch Operation

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

### Pattern 4: Setup + Feature Waves

First wave establishes infrastructure, subsequent waves build on it:

```
Wave 1 (parallel):
  Unit: "Project scaffolding" — package.json, tsconfig, vite.config, main entry
  Unit: "Design system" — colors, typography, component primitives, Tailwind config
  Unit: "Database setup" — schema, migrations, connection, seed data

Wave 2 (parallel, after Wave 1):
  Unit: "Auth API + Auth UI"
  Unit: "Dashboard page + data fetching"
  Unit: "Settings page + user preferences"

Wave 3 (parallel, after Wave 2):
  Unit: "Integration tests"
  Unit: "Error handling and loading states"
  Unit: "Performance optimization"
```

**When to use:** Greenfield projects that need scaffolding before features.

## Dependency Analysis

### What Creates a Dependency

A unit depends on another when:
- It imports a symbol that the other unit creates (e.g., importing a model from the schema unit)
- It extends or implements a type from the other unit
- It renders a component that the other unit creates
- It reads data from a database table that the other unit defines
- It calls an API endpoint that the other unit implements

### What Does NOT Create a Dependency

- Touching the same file (dkod handles this via AST merge)
- Using the same utility library (both can import independently)
- Adding to the same configuration file (e.g., both adding routes to a router — dkod deduplicates)
- Creating parallel test files for different modules

### Minimize Dependencies

Every dependency serializes work. Strategies to reduce them:

1. **Define interfaces early.** If Unit B needs a type from Unit A, can you put the interface
   in a shared types file in a Wave 1 unit? Then both A and B can run in Wave 2.

2. **Use dependency injection.** If Unit B needs Unit A's database connection, can Unit B
   accept it as a parameter instead of importing it directly? Then both can be created in
   parallel with a later integration unit.

3. **Stub external dependencies.** If Unit B needs an API that Unit A builds, can Unit B use
   mock data initially? Then a fix round can wire up the real connection.

4. **Merge setup into fewer units.** If 5 units all depend on the same scaffolding, put all
   scaffolding in one Wave 1 unit.

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

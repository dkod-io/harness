<p align="center">
  <a href="https://dkod.io">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/dkod-io/dkod-engine/main/.github/assets/banner-dark.svg">
      <img alt="dkod — Agent-native code platform" src="https://raw.githubusercontent.com/dkod-io/dkod-engine/main/.github/assets/banner-dark.svg" width="100%">
    </picture>
  </a>
</p>

# dkod harness

Autonomous harness for building complete applications from a single prompt.

One prompt in. Working, tested PR out. Zero human interaction in between.

## What is this?

An implementation of [Anthropic's Planner → Generator → Evaluator harness pattern](https://www.anthropic.com/engineering/harness-design-long-running-apps), purpose-built for [dkod](https://dkod.io)'s parallel agent execution.

Where Anthropic's reference architecture runs generators **sequentially** (one sprint at a time), this harness runs **N generators simultaneously** — because dkod's AST-level semantic merge eliminates false conflicts between agents editing the same files.

```
"Build a task management webapp with auth, categories, and a dashboard"
    │
    ▼
┌─────────────────┐
│     PLANNER      │  Expands prompt → spec → parallel work units
└────────┬────────┘
         │
         ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│GENERATOR │ │GENERATOR │ │GENERATOR │ │GENERATOR │  All running
│  Auth    │ │  Tasks   │ │  UI      │ │  Dashboard│  simultaneously
└────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
     └──────┬──────┴──────┬──────┴──────┬──────┘
            ▼ dkod AST merge (zero conflicts)
┌─────────────────────────────────────────────────┐
│              EVALUATOR                           │
│  Starts app → chrome-devtools → tests everything │
│  PASS? → ship PR    FAIL? → re-dispatch fixes    │
└─────────────────────────────────────────────────┘
```

## How it works

### The agents

| Agent | Job | Tools |
|-------|-----|-------|
| **Planner** | Expands a brief prompt into a full spec with parallel work units decomposed by symbol | `dk_connect`, `dk_context` |
| **Generator** (×N) | Each implements one work unit in its own isolated dkod session | `dk_connect`, `dk_context`, `dk_file_write`, `dk_submit` |
| **Evaluator** | Tests the merged application — adversarial, skeptical, evidence-based | `chrome-devtools`, `dk_verify`, `Bash` |
| **Orchestrator** | Drives the autonomous loop — plan, dispatch, land, eval, fix, ship | All of the above |

### The key insight: decompose by symbol, not file

Traditional parallel agents must carefully avoid touching the same files. With dkod, two generators editing **different functions** in the **same file** is not a conflict — dkod merges at the AST level. This means the planner can decompose work by features and symbols, producing far more parallelism than file-based decomposition allows.

### The autonomous loop

1. **Plan** — Planner reads the codebase, expands the prompt into a spec, decomposes into parallel work units with testable acceptance criteria
2. **Build** — ALL generators dispatched simultaneously in a single blast, each with its own dkod session
3. **Land** — Orchestrator verifies, approves, and merges all changesets (dkod AST merge)
4. **Eval** — Evaluator starts the app, tests via chrome-devtools, grades every criterion
5. **Ship or Fix** — All pass? Push PR. Failures? Re-dispatch generators with feedback. Max 3 rounds.

## Install

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with Opus 4.6
- [dkod MCP server](https://dkod.io)
- Chrome DevTools MCP (for live UI testing)

### Setup

```bash
# Install the harness plugin
npx skills add dkod-io/harness

# Or add dkod MCP directly (if not already installed)
claude mcp add --transport http dkod https://api.dkod.io/mcp
```

## Usage

### Full autonomous build

```
/dkh Build a project management webapp with kanban boards, team collaboration, file attachments, and real-time updates
```

That's it. The harness does the rest.

### Plan only (review before building)

```
/dkh:plan Build a recipe sharing platform with user profiles, collections, and ingredient search
```

### Evaluate existing code

```
/dkh:eval
```

### Check harness progress

```
/dkh:status
```

## Architecture

```
harness/
├── skills/dkh/
│   ├── SKILL.md                    # Core harness behavior and autonomous loop
│   ├── agents/
│   │   ├── orchestrator.md         # Drives the full autonomous pipeline
│   │   ├── planner.md              # Prompt → spec → parallel work units
│   │   ├── generator.md            # Implements one work unit per dkod session
│   │   └── evaluator.md            # Adversarial testing via chrome-devtools
│   └── references/
│       ├── planning-guide.md       # Symbol-level decomposition patterns
│       ├── evaluation-guide.md     # Chrome DevTools testing + scoring calibration
│       └── dkod-patterns.md        # Session lifecycle and merge strategies
├── commands/
│   ├── plan.md                     # /dkh:plan — planning only
│   ├── eval.md                     # /dkh:eval — evaluation only
│   └── status.md                   # /dkh:status — progress check
└── hooks/
    └── hooks.json                  # SubagentStart session reminders
```

## Design principles

**From [Anthropic's harness research](https://www.anthropic.com/engineering/harness-design-long-running-apps):**

1. **Separate generation from evaluation.** Generators can't honestly grade their own work. A standalone adversarial evaluator produces better signal.

2. **Criteria before code.** Define what "done" looks like before writing a line. Acceptance criteria are contracts between planner, generators, and evaluator.

3. **Stress-test your assumptions.** Every harness component encodes an assumption about what the model can't do alone. As models improve, strip away scaffolding that's no longer load-bearing.

**From dkod:**

4. **Decompose by symbol, not file.** AST-level merge means two agents editing different functions in the same file is safe. Plan for maximum parallelism.

5. **One session per agent.** Isolation is the foundation. Each generator's `dk_connect` creates a copy-on-write overlay. Writes are invisible to others until merge.

6. **Sequential landing, parallel building.** Generators run in parallel. Merges happen sequentially (each advances the HEAD). dkod's AST rebase handles the rest.

## Inspired by

- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [coleam00/adversarial-dev](https://github.com/coleam00/adversarial-dev) — GAN-inspired adversarial evaluation
- [dkod: Agent-native code platform](https://dkod.io)

## License

MIT

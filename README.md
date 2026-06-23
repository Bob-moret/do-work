# do-work

A task queue skill for Claude Code. Capture requests fast, process them later.

## Installation

```bash
npx skills add Bob-moret/do-work
```

## Welcome to your new work loop

This skill gives you a two-phase workflow:

1. **Capture**: Throw ideas, bugs, and feature requests at your assistant as they come up. Each one becomes a structured request file in `do-work/`.

2. **Process**: When you're ready, tell the assistant to work. It picks up pending requests one by one, triages complexity, and builds until the queue is empty.

The idea: separate *thinking of things* from *doing things*. Capture is instant. Processing is thorough.

## Quick usage

**Add a request:**
```
do work add dark mode to the settings page
```
Creates `do-work/REQ-001-dark-mode.md`

**Add multiple at once:**
```
do work the search is slow, also add an export button, and fix the header alignment
```
Creates three separate request files.

**Process the queue:**
```
do work run
```
Starts the work loop. The assistant triages each request by complexity:
- **Simple** (config changes, small fixes) → straight to implementation
- **Medium** (clear goal, unknown location) → explore codebase first
- **Complex** (new features, architectural) → plan, explore, then build

Each completed request gets archived with its implementation notes and a git commit.

## How it works

```
do-work/
├── REQ-018-pending.md       # Queue (pending requests)
├── REQ-019-pending.md
├── user-requests/            # Verbatim input + assets per user request
│   └── UR-003/
│       ├── input.md          # Original user input (source of truth)
│       └── assets/
├── working/                  # Currently being processed
│   └── REQ-020-in-progress.md
└── archive/                  # Completed work (self-contained units)
    ├── UR-001/               # UR folder with its completed REQs inside
    │   ├── input.md
    │   └── REQ-013-done.md
    └── REQ-010-legacy.md     # Legacy REQs archive directly
```

Every `do` invocation creates a User Request (UR) folder preserving the verbatim input. REQ files in the queue reference their UR. When all REQs from a UR are completed, the UR folder moves to archive as a self-contained unit.

Legacy REQs (created before the UR system) work the same as before — they archive directly without a UR folder.

## Built for Claude Code

This skill is a Claude Code command-skill. It uses Claude Code's native agent types
(`Plan`, `Explore`, `general-purpose`) via the Agent tool, `AskUserQuestion` for the rare
clarifying question, and git for per-request commits.

### Agents scale to the work

The work loop assigns each request a complexity route and **only spawns the agents that route
needs** — so a typo doesn't pay for a five-agent pipeline:

| Route | Agents | What runs |
| ----- | ------ | --------- |
| **A** Simple  | 0–1 | Implement only (often inline) |
| **B** Medium  | 3   | Plan → Explore → Implement |
| **C** Complex | 5   | Plan → Verify → Explore → Implement → Test |

Before spawning anything, the orchestrator **announces the budget** (e.g.
`Route B → 3 agents: plan, explore, implement`) and reports a running tally in the status log,
so it's always clear how many agents a request is using.

## The three actions

### Do (capture)

Invoked when you provide descriptive content. Optimized for speed:
- Minimal questions - capture what was said, don't interrogate
- Handles simple one-liners and complex multi-feature specs
- Always creates a UR folder preserving the full verbatim input
- Checks for duplicates against existing requests

See [actions/do.md](./actions/do.md) for the full capture logic.

### Work (process)

Invoked when you say "run", "go", "start", or just confirm the prompt. Runs the build loop:
- Triages each request into a route (A/B/C) and spawns only the agents that route needs
  (see [Agents scale to the work](#agents-scale-to-the-work)) — announcing the budget up front
- Passes each agent only the REQ file path and collects short status tokens, so request
  content stays on disk instead of filling the main context — long queues stay runnable
- Archives completed work with implementation notes
- Creates atomic git commits per request

See [actions/work.md](./actions/work.md) for the full processing logic.

### Verify (evaluate)

Invoked when you say "verify", "check", "evaluate", or "review requests". Quality gate:
- Reads the original user input from the UR folder
- Compares against extracted REQ files for completeness
- Scores coverage, UX detail capture, intent signal preservation
- Optionally fixes identified gaps

See [actions/verify.md](./actions/verify.md) for the full evaluation logic.

### Cleanup (consolidate)

Invoked when you say "cleanup", "tidy", or "consolidate". Also runs automatically at the end of every work loop. Keeps the archive organized:
- Closes UR folders in `user-requests/` when all their REQs are complete
- Moves loose REQ files from `archive/` root into their UR folders
- Moves legacy REQs (no UR reference) into `archive/legacy/`
- Fixes misplaced folders (e.g., `archive/user-requests/UR-NNN` → `archive/UR-NNN`)

See [actions/cleanup.md](./actions/cleanup.md) for the full consolidation logic.

## License

MIT

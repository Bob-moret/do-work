# Work Orchestration Reference

> Reference document extracted from the [work action](../actions/work.md). Contains agent prompt templates, orchestrator checklists, request file schema, folder structure, and error handling.

---

## Table of Contents

1. [Agent Prompt Templates](#agent-prompt-templates)
2. [Orchestrator Checklists](#orchestrator-checklists)
3. [Request File Schema](#request-file-schema)
4. [Folder Structure](#folder-structure)
5. [Error Handling](#error-handling)

---

## Agent Prompt Templates

### Plan Agent Prompt (Step 4)

```
You are planning the implementation for this request:

---
[Full content of the request file]
---

Complexity assessment: Route [A/B/C] - [Simple/Medium/Complex]

Project context:
- This is a [describe project type based on package.json/Cargo.toml]
- Key directories: [list from exploring project structure]

Create an implementation plan proportional to the complexity:
- Route A (Simple): 1-3 lines. Name the file(s) and what changes. That's it.
- Route B (Medium): Identify patterns to match or locations to find, plus implementation steps.
- Route C (Complex): Detailed plan with files, ordering, dependencies, architecture decisions, and testing approach.

Be specific about file paths and function names where possible.
If your plan includes steps that require finding unknown files or discovering
existing patterns, note them clearly — these signal that exploration is needed.
```

### Explore Agent Prompt (Step 5)

```
Based on this implementation plan:

---
[Plan from previous phase]
---

For this request:
[Brief summary of the request]

Find and read the relevant files that will need to be modified or that contain patterns we should follow. Focus on:
1. Files mentioned or implied by the plan
2. Similar existing implementations we should match
3. Type definitions and interfaces we'll need
4. Test files that show testing patterns

Return a summary of what you found, including:
- Key code patterns to follow
- Existing types/interfaces to use
- Any concerns or blockers discovered
```

### Implementation Agent Prompt -- With Exploration Context (Step 6)

```
You are implementing this request:

## Request
[Full content of request file]

## Implementation Plan
[Plan from Step 4]

## Codebase Context
[Output from Explore agent]

## Instructions

Implement the changes according to the plan, using the patterns and locations
identified in the codebase context. You have full access to edit files and
run shell commands.

Key guidelines:
- Follow existing code patterns identified in the codebase context
- Make minimal, focused changes
- If you encounter blockers, document them clearly
- If you deviate from the plan, note why

Testing requirements:
- Identify existing tests related to the changes
- Write new tests for new functionality, following patterns from the codebase context
- For bug fixes, add regression tests that would have caught the bug
- Update existing tests if behavior intentionally changed

When complete, provide a summary of:
1. What files were changed/created
2. Any deviations from the plan and why
3. What tests exist and what new tests were written
4. Any follow-up items needed
```

### Implementation Agent Prompt -- Without Exploration Context (Step 6)

```
You are implementing this request:

## Request
[Full content of request file]

## Implementation Plan
[Plan from Step 4]

## Instructions

Implement the changes according to the plan. You have full access to edit
files and run shell commands, including exploring the codebase if you need
additional context.

Key guidelines:
- Make minimal, focused changes
- If you encounter blockers, document them clearly
- If you deviate from the plan, note why

Testing requirements:
- If the project has tests, identify tests related to your changes
- Write new tests for new functionality or bug fixes (regression tests)
- Update existing tests if behavior intentionally changed

When complete, provide a summary of:
1. What files were changed/created
2. Any deviations from the plan and why
3. What tests exist and what new tests were written
4. Any follow-up items needed
```

### Test-Writing Agent Prompt (Step 6.5)

```
The following changes were made for this request:

## Request
[Brief summary]

## Changes Made
[Files created/modified from implementation summary]

## Task
Write appropriate tests for these changes. Follow the existing testing patterns in the codebase.

Guidelines:
- Match the testing style/framework already in use
- Cover the happy path and key edge cases
- For bug fixes, add a regression test that would have caught the bug
- Keep tests focused and readable
- Place test files according to project conventions

After writing tests, run them to verify they pass.
```

---

## Orchestrator Checklists

### Step 0: Work Order Checklist

Before touching any files, write out the following checklist for the request you are about to process. Treat it as a live work order -- check items off as you complete each step. Do not start Step 1 until you have written this out.

```
Work order — [REQ-NNN]:
[ ] Step 1:   Identify next REQ-*.md in do-work/
[ ] Step 2:   Claim → move to working/, update frontmatter (status: claimed)
[ ] Step 3:   Triage → assign route A/B/C, append ## Triage section
[ ] Step 4:   Plan → spawn Plan agent, append ## Plan section
[ ] Step 4.5: *** MANDATORY *** Verify Plan → enumerate requirements, map to plan steps, fix gaps, append ## Plan Verification section
[ ] Step 5:   Explore → spawn Explore agent OR append "Exploration: Not needed"
[ ] Step 6:   Implement → spawn implementation agent, capture summary
[ ] Step 6.5: Run tests → append ## Testing section
[ ] Step 7:   Archive → update frontmatter, move file per UR/legacy logic
[ ] Step 8:   Commit → git add -A && git commit (git repos only)
[ ] Step 9:   Loop or exit
```

Marking Step 4.5 as done means `## Plan Verification` is written to the request file. Marking Step 6.5 as done means `## Testing` is written to the request file. Checking a box without the artifact does not count.

### Per-Request Orchestrator Checklist

```
□ Step 1: List REQ-*.md files in do-work/, pick first one
□ Step 2: mkdir -p do-work/working && mv do-work/REQ-XXX.md do-work/working/
□ Step 2: Update frontmatter: status: claimed, claimed_at: <timestamp>
□ Step 3: Read request, decide route (A/B/C), update frontmatter with route
□ Step 3: Append ## Triage section to request file (including Planning status)
□ Step 4: Spawn Plan agent (depth scales to route), append ## Plan section
□ Step 4.5: Run verify-plan — enumerate, map, fix plan, store coverage
□ Step 5: If plan indicates exploration needed: Spawn Explore agent, append ## Exploration section
□ Step 5: If plan is fully specified: Append "Exploration: Not needed" section
□ Step 6: Spawn implementation agent
□ Step 6.5: Run tests, append ## Testing section
□ Step 7: Update frontmatter: status: completed, completed_at: <timestamp>
□ Step 7: Append ## Implementation Summary section
□ Step 7: Archive REQ (see UR vs legacy archival logic)
□ Step 7: If user_request exists → check if all UR's REQs complete → move UR folder to archive/
□ Step 7: If context_ref exists (legacy) → check if all related REQs archived → move CONTEXT to archive/
□ Step 7: If neither → mv do-work/working/REQ-XXX.md do-work/archive/
□ Step 8: git add -A && git commit (if git repo)
□ Step 9: Check for more requests, loop or exit
□ Step 9: If exiting: Run cleanup action (close completed URs, consolidate loose REQs)
```

**Common mistakes to avoid:**
- Spawning implementation agent without first moving file to `working/`
- Completing implementation without moving file to `archive/`
- Forgetting to update status in frontmatter
- Letting agents handle file management (they shouldn't)
- Skipping planning for simple requests (all routes get planned -- the plan just scales down)
- Skipping verify-plan (it's mandatory for all routes unless user said "skip verification")
- Forgetting to check/archive related UR folders or legacy context documents
- Archiving a UR folder before all its REQs are complete

---

## Request File Schema

Request files use YAML frontmatter for tracking. Fields are added progressively by the do and work actions.

```yaml
---
# Set by do action when created
id: REQ-001
title: Short descriptive title
status: pending
created_at: 2025-01-26T10:00:00Z
user_request: UR-001  # Links to originating user request (may be absent on legacy REQs)

# Set by work action when claimed
claimed_at: 2025-01-26T10:30:00Z
route: A | B | C

# Set by work action when finished
completed_at: 2025-01-26T10:45:00Z
status: completed | failed
commit: abc1234  # Git commit hash (if git repo)
error: "Description of failure"  # Only if failed
---
```

### Field Reference

| Field | Set By | When | Description |
|-------|--------|------|-------------|
| `id` | do | Creation | Request identifier (REQ-NNN) |
| `title` | do | Creation | Short title for the request |
| `status` | Both | Throughout | Current state (see flow below) |
| `created_at` | do | Creation | ISO timestamp when request was captured |
| `user_request` | do | Creation | UR identifier linking to originating user request (absent on legacy REQs) |
| `claimed_at` | work | Claim | ISO timestamp when work began |
| `route` | work | Triage | Complexity route (A/B/C) |
| `completed_at` | work | Completion | ISO timestamp when work finished |
| `commit` | work | Commit | Git commit hash (omitted if not a git repo) |
| `error` | work | Failure | Error description if status is `failed` |

### Status Flow

```
pending (in do-work/, set by do action)
    → claimed (moved to working/, set by work action)
    → [planning] → [exploring] → implementing → testing
    → completed (moved to archive/)
    ↘ failed (moved to archive/ with error)
```

Brackets indicate optional phases based on route.

---

## Folder Structure

```
do-work/
├── REQ-018-pending-task.md       # Pending (root = queue)
├── REQ-019-another-task.md
├── user-requests/                 # User Request folders (verbatim input + assets)
│   ├── UR-003/
│   │   ├── input.md              # Verbatim original input
│   │   └── assets/
│   │       └── REQ-017-screenshot.png
│   └── UR-004/
│       └── input.md
├── assets/                        # Legacy assets (pre-UR system)
│   └── CONTEXT-002-old-batch.md   # Legacy context docs
├── working/                       # Currently being processed
│   └── REQ-020-in-progress.md
└── archive/                       # Completed work
    ├── UR-001/                    # Archived as self-contained unit
    │   ├── input.md
    │   ├── REQ-013-feature.md
    │   └── assets/
    │       └── screenshot.png
    ├── UR-002/
    │   ├── input.md
    │   ├── REQ-014-feature.md
    │   └── REQ-015-feature.md
    ├── REQ-010-legacy-task.md     # Legacy REQs (no UR) archive directly
    └── CONTEXT-001-auth-system.md # Legacy context docs archive directly
```

- **Root `do-work/`**: The queue - ONLY pending `REQ-*.md` files live here
- **`user-requests/`**: UR folders with verbatim input and assets per user request
- **`working/`**: File moves here when claimed, prevents double-processing
- **`archive/`**: Completed UR folders (self-contained units) AND legacy REQs/CONTEXT docs
- **`assets/`**: Legacy screenshots and context documents (pre-UR system)

**Immutability rule:** Files in `working/` and `archive/` are locked. Only the work action's own processing pipeline may modify files in `working/` (updating frontmatter, appending workflow sections). The do action and all other external processes must never reach into these folders. If a change request comes in for something already claimed or completed, the do action creates a new addendum REQ in the queue -- it never touches the original. See the do action docs for the `addendum_to` field.

**User Request (UR) lifecycle:**
- Created in `do-work/user-requests/` by the do action
- Referenced by REQ files via `user_request: UR-NNN` frontmatter
- When ALL related REQs are complete: UR folder moves from `user-requests/` to `archive/`, with completed REQ files pulled into the UR folder

**Backward compatibility:**
- REQs without `user_request` field (legacy) archive directly to `do-work/archive/` as before
- REQs with `context_ref` (legacy) trigger CONTEXT doc archival when all related REQs complete
- The `do-work/assets/` folder is only used for legacy items -- new assets go into UR folders

---

## Error Handling

### Plan agent fails
- Mark request as `failed` with error
- Continue to next request (don't block the queue)

### Explore agent fails
- Proceed to implementation anyway with reduced context
- Note the limitation in the implementation prompt
- Builder can explore on its own if needed

### Implementation agent fails
- Mark request as `failed`
- Preserve any plan and exploration outputs in the request file for retry

### Tests fail repeatedly
- After 3 attempts to fix failing tests, mark request as `failed`
- Include the test failure details in the error field
- Preserve the implementation work done (it may be correct, tests may need adjustment)
- Note in the request file what tests failed and why fixes didn't work

### Commit fails
- Report the error to the user (usually pre-commit hook failure)
- Do NOT retry with `--no-verify` or alternative strategies
- Continue to next request - changes remain uncommitted but are archived
- User can manually fix and commit later

### Unrecoverable error
- Stop the loop
- Report clearly what happened
- Leave queue state intact for manual recovery

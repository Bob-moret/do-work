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

> **The REQ file is the data bus.** The orchestrator does NOT paste request content,
> plans, or exploration output into these prompts. It passes only the **path to the
> claimed request file** (`do-work/working/REQ-XXX.md`). Each agent **reads** the inputs
> it needs from that file and **returns only a short status token** to the orchestrator.
> This keeps the heavy content on disk and out of the orchestrator's context — the
> per-phase fan-out (a fresh agent per phase) is unchanged, so output quality is identical
> to passing content inline.
>
> **The universal invariant:** the orchestrator **never pastes content forward** into a
> later phase's prompt — the next agent always reads the file. This kills the biggest cost
> (the implementation prompt used to re-paste the full request + plan + exploration).
>
> **Who writes the section (Claude Code agent types):**
> - **`general-purpose` agents** (Implement, Verify, Test) can write files, so they write their
>   own output section directly into the REQ file. The orchestrator never sees it.
> - **`Plan` and `Explore` agents** are read-only and cannot write files. They **return** their
>   output text and the orchestrator writes that one section into the file. The orchestrator
>   holds that text once, but still never forwards it.
>
> The templates below are written for the write-capable case ("write the section"). For the
> read-only `Plan`/`Explore` types, read that as "return the section text for the orchestrator
> to write." Substitute `REQ-XXX.md` with the actual claimed filename when spawning each agent.
>
> **Not every phase runs every time.** The route assigned at triage decides the agent budget:
> Route A = Implement only (0–1 agents), Route B = Plan → Explore → Implement (3), Route C =
> Plan → Verify → Explore → Implement → Test (5). See
> [Agent Budget Per Route](../actions/work.md#agent-budget-per-route) in the work action.

### Plan Agent Prompt (Step 4 — Routes B and C only)

> Triage is done by the orchestrator (Step 3), so the route is already known and passed into
> this prompt. The Plan agent only plans. Route A does not use this agent — the orchestrator
> writes a 1–3 line `## Plan` inline.

```
Plan the implementation for the request in this file:

  do-work/working/REQ-XXX.md

This request was triaged as Route <B|C>. Read the file (the request body is under the
original heading; ignore any frontmatter timestamps), then create an implementation plan
proportional to the route. Return it as a `## Plan` section:
- Route B: Identify patterns to match or locations to find, plus implementation steps.
- Route C: Detailed plan with files, ordering, dependencies, architecture decisions,
  and testing approach.
Be specific about file paths and function names where possible. If the plan requires
finding unknown files or discovering existing patterns, say so — that signals that
exploration is needed.

(The read-only Plan agent returns this `## Plan` text for the orchestrator to write into
the file.)

Do NOT return anything but a single status line:
  exploration_needed=<true|false>
```

### Explore Agent Prompt (Step 5)

```
Read the `## Plan` and the request body in this file:

  do-work/working/REQ-XXX.md

Find and read the relevant codebase files the plan implies. Focus on:
1. Files mentioned or implied by the plan
2. Similar existing implementations we should match
3. Type definitions and interfaces we'll need
4. Test files that show testing patterns

Append an `## Exploration` section to the SAME file capturing:
- Key code patterns to follow
- Existing types/interfaces to use
- Any concerns or blockers discovered

Do NOT return the findings. Return ONLY a single status line:
  explored=<N> files=<comma-separated key paths> blockers=<none|short note>
```

### Implementation Agent Prompt (Step 6)

```
You are implementing the request in this file:

  do-work/working/REQ-XXX.md

Read it in full first. It contains:
- the request body (what to build),
- a `## Plan` section (how to build it),
- and, if present, an `## Exploration` section (codebase patterns/types to follow).
If there is no `## Exploration` section, explore the codebase yourself as needed.

Implement the changes according to the plan. You have full access to edit files and
run shell commands.

Key guidelines:
- Follow existing code patterns (those in `## Exploration` if present)
- Make minimal, focused changes
- If you encounter blockers, document them clearly
- If you deviate from the plan, note why

Testing requirements:
- Identify existing tests related to the changes
- Write new tests for new functionality, following the project's testing patterns
- For bug fixes, add regression tests that would have caught the bug
- Update existing tests if behavior intentionally changed

When done, append an `## Implementation Summary` section to the SAME file covering:
1. What files were changed/created
2. Any deviations from the plan and why
3. What tests exist and what new tests were written
4. Any follow-up items needed

Do NOT return the summary. Return ONLY a single status line:
  files=<N> tests=<N> status=<ok|blocked> note=<short note if blocked>
```

### Test-Writing Agent Prompt (Step 6.5)

```
Changes were made for the request in this file:

  do-work/working/REQ-XXX.md

Read its `## Implementation Summary` to see which files changed, then write appropriate
tests for those changes. Follow the existing testing patterns in the codebase.

Guidelines:
- Match the testing style/framework already in use
- Cover the happy path and key edge cases
- For bug fixes, add a regression test that would have caught the bug
- Keep tests focused and readable
- Place test files according to project conventions

After writing tests, run them. Record results in the `## Testing` section of the SAME
file. Do NOT return test output. Return ONLY a single status line:
  added=<N> status=<pass|fail> note=<short note if fail>
```

---

## Orchestrator Checklists

### Step 0: Work Order Checklist

Before touching any files, write out the following checklist for the request you are about to process. Treat it as a live work order -- check items off as you complete each step. Do not start Step 1 until you have written this out.

This is the full Route C work order. After triage (Step 3), drop the steps your route does not
use: **Route A** keeps only Triage → Implement → Test-run → Archive → Commit → Log;
**Route B** drops the dedicated Verify (Step 4.5, unless under-specified) and the dedicated
Test agent (Step 6.5 — Implement writes its own tests).

```
Work order — [REQ-NNN]:    (pass agents the REQ path only; collect status tokens)
[ ] Step 1:   Identify next REQ-*.md in do-work/ (filenames only)
[ ] Step 2:   Claim → move to working/, update frontmatter (status: claimed)
[ ] Step 3:   Triage → read REQ once, assign route, write ## Triage, ANNOUNCE budget
              (e.g. "Route B → 3 agents: plan, explore, implement")
[ ] Step 4:   Plan → (B/C) spawn Plan agent (path only); ## Plan produced; record
              exploration_needed from token. (Route A: write 1-3 line ## Plan inline)
[ ] Step 4.5: Verify Plan → MANDATORY (C) / optional (B) / skip (A): spawn Verify agent
              (path only); it enumerates/maps/fixes and writes ## Plan Verification
[ ] Step 5:   Explore → (B/C, if exploration_needed) spawn Explore agent;
              ## Exploration produced. Else note "Not needed"
[ ] Step 6:   Implement → spawn implementation agent (path only); it writes
              ## Implementation Summary (A/B: also writes+runs its own tests)
[ ] Step 6.5: Test → orchestrator runs the suite; (C) spawn dedicated Test agent for new
              tests; ## Testing written; record pass/fail from token
[ ] Step 7:   Archive → update frontmatter, move file per UR/legacy logic
[ ] Step 8:   Commit → git add -A && git commit (git repos only)
[ ] Step 9:   Log compact status block (incl. "Route X · N agents" tally), loop or exit
```

When Step 4.5 runs (always for Route C), marking it done means `## Plan Verification` is written to the request file. Marking Step 6.5 as done means `## Testing` is written to the request file (by the Route C Test agent, or by the orchestrator after running the suite). Checking a box without the artifact does not count. ("Produced" means a write-capable `general-purpose` agent wrote the section itself, or the orchestrator wrote it from a read-only agent's returned text — never forwarded to a later phase.)

### Per-Request Orchestrator Checklist

```
□ Step 1: List REQ-*.md files in do-work/, pick first one
□ Step 2: mkdir -p do-work/working && mv do-work/REQ-XXX.md do-work/working/
□ Step 2: Update frontmatter: status: claimed, claimed_at: <timestamp>
□ Step 3: Triage: read REQ once, assign route, write ## Triage, ANNOUNCE budget ("Route X → N agents")
□ Step 4: (B/C) Spawn Plan agent (path only); ## Plan produced; record exploration_needed. (A: write 1-3 line ## Plan inline)
□ Step 4.5: (C mandatory / B optional / A skip) Spawn Verify agent (path only); writes ## Plan Verification; record coverage
□ Step 5: (B/C) If exploration_needed: Spawn Explore agent (path only); ## Exploration produced. Else: note "Not needed"
□ Step 6: Spawn implementation agent (path only); it writes ## Implementation Summary (A/B: also writes+runs its own tests)
□ Step 6.5: Run the suite; (C) spawn dedicated Test agent for new tests; ## Testing written; record pass/fail from token
□ Step 7: Update frontmatter: status: completed, completed_at: <timestamp>
□ Step 7: Verify ## Implementation Summary is present (agent wrote it; missing → treat as failure)
□ Step 7: Archive REQ (see UR vs legacy archival logic)
□ Step 7: If user_request exists → check if all UR's REQs complete → move UR folder to archive/
□ Step 7: If context_ref exists (legacy) → check if all related REQs archived → move CONTEXT to archive/
□ Step 7: If neither → mv do-work/working/REQ-XXX.md do-work/archive/
□ Step 8: git add -A && git commit (if git repo); file list from `git diff --cached --name-only`
□ Step 9: Log compact status block from collected tokens; check for more requests, loop or exit
□ Step 9: If exiting: Run cleanup action (close completed URs, consolidate loose REQs)
```

**Common mistakes to avoid:**
- Spawning any agent without first moving the file to `working/` (agents read it from there)
- Pasting request/plan/exploration content into a later phase's prompt (pass the path; the agent reads the file)
- Reading the plan/summary/test output back into the orchestrator's context (you only need the status tokens)
- Completing implementation without moving file to `archive/`
- Forgetting to update status in frontmatter
- Letting agents handle the file *lifecycle* — moves, archival, commits (those stay with the orchestrator; agents only write their content sections)
- Skipping planning entirely (every route gets a plan -- Route A's is a 1-3 line inline plan, B/C get a Plan agent)
- Skipping verify-plan on Route C (it's mandatory there unless the user said "skip verification"; optional on B, skip on A)
- Spawning a Plan/Verify/Explore agent for a Route A request (those are B/C only -- Route A is Implement only)
- Spawning a dedicated Test agent on Route A/B (Implement writes its own tests there; dedicated Test agent is Route C only)
- Forgetting to announce the agent budget at Step 3, or logging an agent count that doesn't match it
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

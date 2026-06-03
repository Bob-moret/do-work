# Work Action

> **Part of the do-work skill.** Invoked when routing determines the user wants to process the queue. Processes requests from the `do-work/` folder in your project.

An orchestrated build system that processes request files created by the do action. Every request gets planned, verified, and implemented — the plan's depth scales to the task, from a single line for config changes to a multi-step strategy for new features.

## Request Files as Living Logs

**Each request file becomes a complete historical record of the work performed.** As you process a request, you append sections documenting each phase: Triage, Plan, Plan Verification, Exploration, Implementation Summary, Testing.

## Architecture

```
work action (orchestrator - lightweight, stays in loop)
  |
  +-- For each pending request:
  |     |   (the orchestrator passes ONLY the REQ file path to each agent and
  |     |    receives ONLY a short status token back -- the REQ file is the data bus)
  |     |
  |     +-- PLAN (+triage): agent reads REQ; ## Triage + ## Plan produced -> route token
  |     +-- VERIFY PLAN: agent reads REQ+plan, writes ## Plan Verification -> coverage token
  |     +-- EXPLORE: agent reads plan; ## Exploration produced -> files token (when needed)
  |     +-- IMPLEMENT: agent reads REQ+plan+exploration, writes ## Implementation Summary -> files token
  |     +-- TEST: agent/orchestrator runs tests, writes ## Testing -> pass/fail token
  |
  +-- Loop continues until queue empty
```

### The REQ file is the data bus

To keep the orchestrator's context small over a long queue, **content flows agent → disk →
agent, never agent → orchestrator → agent.** The orchestrator never pastes request bodies,
plans, or exploration output into agent prompts and never reads them back into its own
context. Instead:

- The orchestrator spawns each agent with **only the claimed REQ file path**.
- Each agent **reads** the inputs it needs from that file and **writes** its output section
  back to the file (the "living log").
- Each agent **returns only a short status token** (e.g. `route=B exploration_needed=true`).

The per-phase fan-out is unchanged — every phase is still a fresh agent in its own clean
context — so output quality is identical to passing content inline. Only the bookkeeping
moves off the orchestrator's context and onto disk.

## Sub-agent Compatibility

This document uses "spawn agent" language. Use your platform's subagent or multi-agent mechanism when available. If your tool does not support subagents, run the phases sequentially in the same session — the protocol is identical, you just lose the per-phase context isolation.

**The return convention:** every spawned phase agent receives **only the REQ file path** and
returns **only a short status token**. The orchestrator **never pastes content forward** into a
later phase's prompt — the next agent reads the file. See
[The REQ file is the data bus](#the-req-file-is-the-data-bus).

**Who writes each section** depends on whether the agent can write files:
- **Write-capable agents write their own section** directly into the REQ file (the orchestrator
  never sees it).
- **Read-only agents return their text** and the orchestrator writes that one section — same as
  the skill has always done. Quality is identical because the same specialized agent runs.

**Claude Code hint**: native agent types, with the quality-safe default:
- **Plan phase** → `subagent_type: Plan` (read-only, optimized for planning). It returns the
  triage + plan text; the orchestrator writes `## Triage` and `## Plan`. Identical to today.
- **Explore phase** → `subagent_type: Explore` (read-only, optimized for codebase search). It
  returns findings; the orchestrator writes `## Exploration`. Identical to today.
- **Verify / Implement / Test phases** → `subagent_type: general-purpose` (full tool access).
  These write their own sections (`## Plan Verification`, `## Implementation Summary`,
  `## Testing`) directly — this is where the orchestrator stops re-pasting content and the
  context savings come from.

> **Even-leaner option (optional):** run the Plan and Explore phases with `general-purpose`
> too, so they write their own sections and their text never touches the orchestrator. This
> maximizes context savings but swaps the read-only specialized planners for `general-purpose`
> — a minor change to those two phases. Keep the default above if you want plan/exploration
> quality byte-for-byte unchanged.

For reference files in this skill, use `${CLAUDE_SKILL_DIR}` to build paths (e.g., `${CLAUDE_SKILL_DIR}/references/work-orchestration.md`).

## Complexity Assessment

Before spawning any agents, quickly assess the request's complexity. This produces a **route label** (A/B/C) that tells the planner how deep to go.

### Route A: Simple

Plan should be 1-3 lines. Exploration unlikely needed.

**Indicators:** Bug fix with clear steps, config change, single UI element, styling, request names specific files, well-specified with obvious scope, copy/text changes, feature flag toggle.

### Route B: Medium

Plan should include steps for discovering patterns/locations. Exploration likely needed.

**Indicators:** Clear outcome but unspecified location, "like the existing X", need to find similar implementations, modifying something at unknown location, matching existing conventions.

### Route C: Complex

Plan should be detailed with multiple steps, dependencies, and testing approach. Exploration almost certainly needed.

**Indicators:** New multi-component feature, architectural changes, ambiguous scope, touches multiple systems, external service integration, 100+ word request, introduces new concepts.

**When uncertain, prefer Route B.**

### Assessment Decision Flow

```
Read the request file
     |
     +-- Names specific files AND clear changes? -> Route A
     +-- Bug fix with clear reproduction? -> Route A
     +-- Simple value/config/copy change? -> Route A
     +-- Clear outcome but unknown location? -> Route B
     +-- Ambiguous, multi-system, or architectural? -> Route C
     +-- Default: Route B
```

## Folder Structure & Schema

For the complete folder structure, request file schema, frontmatter fields, and status flow, see [references/work-orchestration.md](../references/work-orchestration.md).

Key points:
- **Root `do-work/`**: The queue — ONLY pending `REQ-*.md` files
- **`working/`**: File moves here when claimed, prevents double-processing
- **`archive/`**: Completed UR folders (self-contained) AND legacy REQs
- **Immutability rule**: Files in `working/` and `archive/` are locked

## Workflow

**CRITICAL: Orchestrator Responsibilities**

The orchestrator owns the **loop and the file lifecycle** but stays out of the per-request
content to keep its context small.

**You (orchestrator) must do yourself:** move files to `working/`, update frontmatter
(status, route, timestamps, commit hash), move files to `archive/`, create git commits, and
keep the loop running. These touch the file's metadata and location — not its content.

**Agents do (reading their inputs from the REQ file directly):** triage + planning, plan
verification, exploring, implementation, writing/running tests. They **produce** the
`## Triage`, `## Plan`, `## Plan Verification`, `## Exploration`, `## Implementation Summary`,
and `## Testing` sections — write-capable agents (Verify, Implement, Test) write them into the
file themselves; read-only agents (the default `Plan`/`Explore` types) return their text and
the orchestrator writes that one section. Either way the content is **never pasted forward**
into a later phase's prompt — the next agent reads the file.

> **Why this changed:** previously the orchestrator assembled each phase's prompt by pasting
> the full request + plan + exploration, pulling all that content through its context every
> request. Now agents read the file directly and return only short status tokens, so the
> orchestrator's context stays small over long queues. The fan-out (a fresh agent per phase)
> is unchanged, so quality is unaffected.

---

### Step 0: Write Your Work Order

**Before touching any files**, write out the checklist. See [references/work-orchestration.md](../references/work-orchestration.md) for the full checklist template.

```
Work order -- [REQ-NNN]:
[ ] Step 1:   Identify next REQ-*.md in do-work/ (filenames only -- don't read content)
[ ] Step 2:   Claim -> move to working/, update frontmatter
[ ] Step 3:   (triage is folded into the Plan agent -- no separate orchestrator step)
[ ] Step 4:   Plan -> spawn Plan agent (path only); ## Triage + ## Plan produced;
              record route from token
[ ] Step 4.5: *** MANDATORY *** Verify Plan -> spawn Verify agent; agent writes
              ## Plan Verification; record coverage from token
[ ] Step 5:   Explore -> if token said exploration_needed, spawn Explore agent;
              ## Exploration produced; else note "Not needed"
[ ] Step 6:   Implement -> spawn implementation agent (path only); agent writes
              ## Implementation Summary
[ ] Step 6.5: Test -> run/spawn tests; ## Testing written; record pass/fail from token
[ ] Step 7:   Archive -> update frontmatter, move file
[ ] Step 8:   Commit -> git add -A && git commit (git repos only)
[ ] Step 9:   Log compact status block, loop or exit
```

### Step 1: Find Next Request

**[Orchestrator action]**

1. **List** (don't read) `REQ-*.md` filenames in `do-work/`
2. Sort by filename (REQ-001 before REQ-002)
3. Pick the first one

If no request files found, report completion and exit.

**Stale claim check:** Check `do-work/working/`. If a file has `claimed_at` older than 1 hour, unclaim it and move back to `do-work/`. `do work resume` forces this regardless of age.

### Step 2: Claim the Request

**[Orchestrator action — BEFORE spawning any agents]**

1. Create `do-work/working/` if needed
2. Move request file from `do-work/` to `do-work/working/`
3. Update frontmatter: `status: claimed`, `claimed_at: <timestamp>`

### Step 3: Triage

**[Folded into the Plan agent — no separate orchestrator step]**

Triage no longer runs as its own orchestrator step. The Plan agent (Step 4) reads the REQ,
assigns the route using the [Complexity Assessment](#complexity-assessment) rubric, and writes
the `## Triage` section itself. This keeps the request content out of the orchestrator's
context. The orchestrator only records the resulting `route` in the frontmatter from the
agent's status token.

### Step 4: Planning Phase (All routes)

**[Spawn agent — pass the REQ file path only]**

Spawn the Plan agent with **only the claimed REQ file path** (e.g. `do-work/working/REQ-007.md`).
The agent reads the request and triages it. It then produces both the `## Triage` and `## Plan`
sections — writing them into the file itself if write-capable, or returning the text for you to
write if you used the read-only `Plan` type (the default). Either way you never paste this
content forward. Plan depth scales to the route:
- **Route A**: 1-3 lines. Name the file(s) and the change.
- **Route B**: Identify what to find/match, plus implementation steps.
- **Route C**: Detailed — files, ordering, dependencies, architecture, testing approach.

For the full Plan agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

The agent returns **only a status token**: `route=<A|B|C> exploration_needed=<true|false>`.
Do NOT read the plan into your context — it is already in the file.

**Orchestrator action after the token returns:** update the frontmatter with `route` and
report briefly ("Simple request", "Medium complexity", or "Complex request"). Remember
`exploration_needed` for Step 5.

### Step 4.5: Verify Plan (All routes)

**[Mandatory step — spawn a Verify agent (pass the REQ file path only)]**

Spawn a Verify agent that reads the REQ + plan, ensures every requirement is addressed, fixes
gaps directly, and writes the `## Plan Verification` section to the file. The agent runs this
protocol:

1. **Enumerate source items** from REQ + UR input.md
2. **Map each item to plan**: Full / Partial / Missing
3. **Calculate coverage**: (full + 0.5 x partial) / total x 100
4. **Auto-fix the plan** — add missing steps, expand partial coverage
5. **Store results** — write `## Plan Verification` to the request file

See [verify-plan action](./verify-plan.md) for the full protocol. The agent returns **only a
status token**: `coverage=<N>% items=<N> fixed=<N>`. Do NOT read the coverage map into your
context — it is in the file.

> **Gate:** `## Plan Verification` must be written before starting Step 5.

### Step 5: Exploration Phase (When plan indicates)

**[Spawn agent only if Step 4's token said `exploration_needed=true`]**

Branch on the `exploration_needed` value from the Plan agent's token — you do not need to read
the plan yourself to decide:

- **`exploration_needed=true`:** spawn the Explore agent with **only the REQ file path**. The
  agent reads the plan and explores the codebase, then produces the `## Exploration` section —
  writing it itself if write-capable, or returning findings for you to write if you used the
  read-only `Explore` type (the default). It returns **only a status token**:
  `explored=<N> files=<paths> blockers=<...>`.
- **`exploration_needed=false`:** skip the agent. Write a short `## Exploration` section to the
  file noting "Not needed — plan names specific files and exact changes."

For the full Explore agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

### Step 6: Implementation Phase (All routes)

**[Spawn agent — pass the REQ file path only; agent does the actual code changes]**

Spawn a general-purpose agent with **only the claimed REQ file path**. The agent reads the
request body, the `## Plan`, and the `## Exploration` section (if present) from the file —
the orchestrator does not assemble or paste any of that content.

For the full implementation agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

Key guidelines for the agent:
- Follow existing code patterns (from `## Exploration` if present)
- Make minimal, focused changes
- Document blockers and plan deviations in `## Implementation Summary`
- Write tests for new functionality and bug fix regressions

The agent writes `## Implementation Summary` to the file and returns **only a status token**:
`files=<N> tests=<N> status=<ok|blocked> note=<...>`. Do NOT read the summary into your context.

### Step 6.5: Testing Phase (All routes)

**[Orchestrator runs tests; may spawn a Test agent for new tests]**

1. **Detect testing infrastructure** (jest, vitest, pytest, cargo test, go test, etc.)
2. **Identify relevant tests** based on modified files (use `git diff --name-only`, not the
   REQ content)
3. **Run existing related tests**
4. **If tests fail**: Re-spawn the implementation agent (path only) to fix (loop until pass or
   3 attempts)
5. **If new tests needed**: Spawn a Test agent (path only) following project conventions; it
   writes the `## Testing` section itself
6. **Verify all tests pass**: One final run

The `## Testing` section is written to the request file (by the Test agent, or by the
orchestrator if it ran the tests directly — record only a brief pass/fail line, not full
output). Keep a `status=<pass|fail>` summary for the Step 9 log block.

If no testing infrastructure detected, skip and note: "No testing infrastructure detected."

### Step 7: Archive and Continue

**[Orchestrator action — do this yourself]**

**On success:**
1. Update frontmatter: `status: completed`, `completed_at: <timestamp>`
2. Verify `## Implementation Summary` is present (the implementation agent writes it). If it is
   missing, the agent likely failed — treat as failure. Do not reconstruct it from memory.
3. Archive based on REQ type:
   - **Has `user_request: UR-NNN`**: Check if ALL UR's REQs complete → move whole UR to archive. Otherwise move REQ to archive directly.
   - **Has `context_ref` (legacy)**: Move REQ, check if all related REQs done → move CONTEXT doc too
   - **Neither (standalone legacy)**: Move directly to archive

**On failure:**
1. Update frontmatter: `status: failed`, `error: "description"`
2. Move to archive (failed items are archived too, status tells the story)
3. Failed REQs go directly to `do-work/archive/`, NOT into UR folders

### Step 8: Commit Changes (Git repos only)

**[Orchestrator action]**

If Git-backed, create **one commit per request**:

```bash
git add -A
git commit -m "[REQ-003] Dark Mode (Route C)

Implements: do-work/archive/REQ-003-dark-mode.md

- Created src/stores/theme-store.ts
- Modified src/components/settings/SettingsPanel.tsx

Co-Authored-By: Claude <noreply@anthropic.com>"
```

Rules: ONE commit per request, stage everything with `git add -A`, no pre-commit hook bypass, failed requests get committed too.

To build the commit message's file list, use `git diff --cached --name-only` (or
`git status --short`) — do **not** read the `## Implementation Summary` back into your context
just to list files. After committing, record the short hash in the frontmatter (`commit:`).

### Step 9: Log, Loop or Exit

1. **Log a compact status block** assembled from the status tokens you collected (this is the
   per-request visibility — no need to read the REQ file). Example:

   ```
   REQ-007 (Route B): plan ✓ · verify ✓ (100%) · explore ✓ (3 files)
     implement ✓ — 3 files, 2 tests · test ✓ pass · archived · commit a3f9c2
   ```

   Tune verbosity to taste: one line for terse, the two-line block above for medium, or add
   the per-token detail for verbose. The full detail always lives in the REQ file regardless.
2. Re-check `do-work/` for `REQ-*.md` files (fresh check, filenames only)
3. If found: start Step 1 again
4. If empty: Run [cleanup action](./cleanup.md), report final summary, exit

## What This Action Does NOT Do

- Create new request files (use the do action)
- Make architectural decisions beyond the request
- Run without user present (supervised automation)
- Modify completed/in-progress requests from other agents
- Skip planning for any request (every route gets planned)

## Additional References

- [Orchestration details](../references/work-orchestration.md) — agent prompts, checklists, folder structure, schema, error handling
- [Examples & retrospective](../references/work-examples.md) — example sessions, progress reporting, retrospective value
- [Verify-plan action](./verify-plan.md) — full plan verification protocol
- [Cleanup action](./cleanup.md) — archive consolidation

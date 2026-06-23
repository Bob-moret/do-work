# Work Action

> **Part of the do-work skill.** Invoked when routing determines the user wants to process the queue. Processes requests from the `do-work/` folder in your project.

An orchestrated build system that processes request files created by the do action. Every request gets implemented; the **amount of orchestration scales to the task**. A trivial change is handled inline or by a single agent, while a complex feature gets the full plan → verify → explore → implement → test pipeline. This keeps a long queue fast and cheap instead of spending five agent round-trips on a one-line fix.

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
  |     +-- TRIAGE: assign a route (A/B/C). The route decides the agent budget below.
  |     |
  |     +-- Route A (Simple):  Implement only (often handled inline by the orchestrator)
  |     +-- Route B (Medium):  PLAN -> EXPLORE -> IMPLEMENT            (3 agents)
  |     +-- Route C (Complex): PLAN -> VERIFY -> EXPLORE -> IMPLEMENT -> TEST  (5 agents)
  |
  +-- Loop continues until queue empty
```

The phases themselves are unchanged in spirit — each is a fresh agent in its own clean
context that reads from and writes to the REQ file. What changed is that **the route now
decides how many of them run**, so simple work stays cheap. See
[Agent Budget Per Route](#agent-budget-per-route).

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

## Agent Budget Per Route

The route assigned during triage decides how many agents run. **Announce the budget before
spawning anything** so it is always clear how many agents a request will use.

| Route | Phases (agents) | Count | Verify-plan | Test |
| ----- | --------------- | ----- | ----------- | ---- |
| **A** Simple  | Implement only — often done **inline** by the orchestrator | 0–1 | skip | folded into Implement |
| **B** Medium  | Plan → Explore → Implement | 3 | optional | folded into Implement |
| **C** Complex | Plan → Verify → Explore → Implement → Test | 5 | **mandatory** | dedicated Test agent |

- **Route A**: the orchestrator triages inline (1–3 lines), then either makes the change
  itself (truly trivial: a typo, a config value, a one-line copy change) or spawns a single
  implementation agent. No separate Plan/Verify/Explore agents.
- **Route B**: spawn Plan, then Explore, then Implement. The implementation agent writes and
  runs its own tests — no separate Test agent. Verify-plan is optional (spawn it if the plan
  feels under-specified).
- **Route C**: the full pipeline, including the mandatory Verify-plan gate and a dedicated
  Test agent.

**Announce, then count.** At the start of each request, state the budget (e.g.
`Route B → 3 agents: plan, explore, implement`). In the Step 9 log, report the running tally
(e.g. `agents: 3/3 done`). This is the per-request agent visibility.

## Claude Code Agent Types

Spawn phase agents with the Agent tool. Use these native `subagent_type` values:

- **Plan phase** → `subagent_type: Plan` (read-only, optimized for planning). It returns the
  triage + plan text; the orchestrator writes `## Triage` and `## Plan`.
- **Explore phase** → `subagent_type: Explore` (read-only, optimized for codebase search). It
  returns findings; the orchestrator writes `## Exploration`.
- **Verify / Implement / Test phases** → `subagent_type: general-purpose` (full tool access).
  These write their own sections (`## Plan Verification`, `## Implementation Summary`,
  `## Testing`) directly into the REQ file — the orchestrator never re-pastes that content.

**The return convention:** every spawned phase agent receives **only the REQ file path** and
returns **only a short status token**. The orchestrator **never pastes content forward** into a
later phase's prompt — the next agent reads the file. See
[The REQ file is the data bus](#the-req-file-is-the-data-bus). Read-only agents (`Plan`,
`Explore`) cannot write files, so they return their section text and the orchestrator writes
that one section; write-capable `general-purpose` agents write their own section directly.

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
verification, exploring, implementation, writing/running tests — **but only the phases the
route calls for** (see [Agent Budget Per Route](#agent-budget-per-route)). They **produce** the
`## Triage`, `## Plan`, `## Plan Verification`, `## Exploration`, `## Implementation Summary`,
and `## Testing` sections — write-capable `general-purpose` agents (Verify, Implement, Test)
write them into the file themselves; read-only agents (`Plan`/`Explore`) return their text and
the orchestrator writes that one section. Either way the content is **never pasted forward**
into a later phase's prompt — the next agent reads the file.

> **Why this changed:** previously the orchestrator assembled each phase's prompt by pasting
> the full request + plan + exploration, pulling all that content through its context every
> request. Now agents read the file directly and return only short status tokens, so the
> orchestrator's context stays small over long queues. The fan-out (a fresh agent per phase)
> is unchanged, so quality is unaffected.

---

### Step 0: Write Your Work Order

**Before touching any files**, write out the checklist. See [references/work-orchestration.md](../references/work-orchestration.md) for the full checklist template, including the shorter Route A variant.

The checklist below is the **full (Route C) pipeline**. Once you know the route (Step 3),
**announce the agent budget** and drop the steps that route does not use — see
[Agent Budget Per Route](#agent-budget-per-route). Route A skips Plan/Verify/Explore and the
dedicated Test agent; Route B skips the dedicated Verify and Test agents.

```
Work order -- [REQ-NNN]:
[ ] Step 1:   Identify next REQ-*.md in do-work/ (filenames only -- don't read content)
[ ] Step 2:   Claim -> move to working/, update frontmatter
[ ] Step 3:   Triage -> assign route (A/B/C). ANNOUNCE the budget
              (e.g. "Route B -> 3 agents: plan, explore, implement")
[ ] Step 4:   Plan -> (B/C) spawn Plan agent (path only); ## Triage + ## Plan produced;
              record route from token. (Route A: orchestrator writes a 1-3 line ## Plan inline)
[ ] Step 4.5: Verify Plan -> *** MANDATORY for Route C *** (optional B, skip A): spawn Verify
              agent; agent writes ## Plan Verification; record coverage from token
[ ] Step 5:   Explore -> (B/C, if exploration_needed) spawn Explore agent;
              ## Exploration produced; else note "Not needed"
[ ] Step 6:   Implement -> spawn implementation agent (path only); agent writes
              ## Implementation Summary (and, for A/B, writes+runs its own tests)
[ ] Step 6.5: Test -> orchestrator runs the suite; for Route C spawn a dedicated Test agent
              for new tests; ## Testing written; record pass/fail from token
[ ] Step 7:   Archive -> update frontmatter, move file
[ ] Step 8:   Commit -> git add -A && git commit (git repos only)
[ ] Step 9:   Log compact status block (incl. agent tally), loop or exit
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

**[Orchestrator action — read the claimed REQ once to size the budget]**

The orchestrator must know the route *before* it can decide how many agents to spawn, so
triage is a lightweight orchestrator step. Read the claimed REQ file once, assign a route
using the [Complexity Assessment](#complexity-assessment) rubric, and:

1. Write the `## Triage` section to the file (route + 1–2 line reasoning).
2. Record `route` in the frontmatter.
3. **Announce the agent budget** for that route, e.g.
   `Route B → 3 agents: plan, explore, implement`. (See
   [Agent Budget Per Route](#agent-budget-per-route).)

This single read is cheap and bounded — the data-bus discipline is about never *pasting
content forward* between phases, not about refusing to ever read the REQ. The heavy content
(plan, exploration, summaries) still flows agent → disk → agent and never through the
orchestrator's context.

### Step 4: Planning Phase (Routes B and C — Route A is inline)

**Route A:** do **not** spawn a Plan agent. The orchestrator writes a 1–3 line `## Plan`
itself (name the file(s) and the change) and sets `exploration_needed=false`. Skip to Step 6.

**Routes B and C — [Spawn agent — pass the REQ file path only]:**

Spawn the Plan agent (`subagent_type: Plan`) with **only the claimed REQ file path** (e.g.
`do-work/working/REQ-007.md`). Triage is already done (Step 3), so the agent reads the request
and produces the `## Plan` section — it returns the text and the orchestrator writes it (the
read-only `Plan` type cannot write files). You never paste this content forward. Plan depth
scales to the route:
- **Route B**: Identify what to find/match, plus implementation steps.
- **Route C**: Detailed — files, ordering, dependencies, architecture, testing approach.

For the full Plan agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

The agent returns **only a status token**: `exploration_needed=<true|false>`.
Do NOT read the plan into your context — it is already in the file.

**Orchestrator action after the token returns:** report briefly ("Medium complexity" or
"Complex request") and remember `exploration_needed` for Step 5.

### Step 4.5: Verify Plan (Route C mandatory · Route B optional · Route A skip)

**[Spawn a Verify agent (`subagent_type: general-purpose`, pass the REQ file path only)]**

- **Route C**: mandatory. Always spawn the Verify agent.
- **Route B**: optional. Spawn it only if the plan feels under-specified or the request had
  many discrete requirements; otherwise note "Plan verification: skipped (Route B)" and move on.
- **Route A**: skip entirely (no plan agent ran).

When you do run it, the Verify agent reads the REQ + plan, ensures every requirement is
addressed, fixes gaps directly, and writes the `## Plan Verification` section to the file. The
agent runs this protocol:

1. **Enumerate source items** from REQ + UR input.md
2. **Map each item to plan**: Full / Partial / Missing
3. **Calculate coverage**: (full + 0.5 x partial) / total x 100
4. **Auto-fix the plan** — add missing steps, expand partial coverage
5. **Store results** — write `## Plan Verification` to the request file

See [verify-plan action](./verify-plan.md) for the full protocol. The agent returns **only a
status token**: `coverage=<N>% items=<N> fixed=<N>`. Do NOT read the coverage map into your
context — it is in the file.

> **Gate (Route C):** `## Plan Verification` must be written before starting Step 5. For
> Route B this gate only applies if you chose to run the Verify agent.

### Step 5: Exploration Phase (Routes B and C, when plan indicates)

**[Spawn agent only if Step 4's token said `exploration_needed=true`]**

Route A skips this step (it went straight to Step 6). For Routes B and C, branch on the
`exploration_needed` value from the Plan agent's token — you do not need to read the plan
yourself to decide:

- **`exploration_needed=true`:** spawn the Explore agent (`subagent_type: Explore`) with
  **only the REQ file path**. The agent reads the plan and explores the codebase, then returns
  the `## Exploration` findings for the orchestrator to write (the read-only `Explore` type
  cannot write files). It returns **only a status token**:
  `explored=<N> files=<paths> blockers=<...>`.
- **`exploration_needed=false`:** skip the agent. Write a short `## Exploration` section to the
  file noting "Not needed — plan names specific files and exact changes."

For the full Explore agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

### Step 6: Implementation Phase (All routes)

**[Spawn agent — pass the REQ file path only; agent does the actual code changes]**

Spawn a `general-purpose` agent with **only the claimed REQ file path**. The agent reads the
request body, the `## Plan`, and the `## Exploration` section (if present) from the file —
the orchestrator does not assemble or paste any of that content.

> **Route A inline option:** if the change is truly trivial (a typo, a single config value, a
> one-line copy change) the orchestrator may make the edit itself and write a short
> `## Implementation Summary` rather than spawning an agent. When in doubt, spawn the agent.

For the full implementation agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

Key guidelines for the agent:
- Follow existing code patterns (from `## Exploration` if present)
- Make minimal, focused changes
- Document blockers and plan deviations in `## Implementation Summary`
- Write tests for new functionality and bug fix regressions

**For Routes A and B, the implementation agent also writes and runs its own tests** (no
dedicated Test agent follows). For Route C, a dedicated Test agent handles new tests in
Step 6.5.

The agent writes `## Implementation Summary` to the file and returns **only a status token**:
`files=<N> tests=<N> status=<ok|blocked> note=<...>`. Do NOT read the summary into your context.

### Step 6.5: Testing Phase (All routes)

**[Orchestrator runs the suite; a dedicated Test agent is Route C only]**

1. **Detect testing infrastructure** (jest, vitest, pytest, cargo test, go test, etc.)
2. **Identify relevant tests** based on modified files (use `git diff --name-only`, not the
   REQ content)
3. **Run existing related tests**
4. **If tests fail**: Re-spawn the implementation agent (path only) to fix (loop until pass or
   3 attempts)
5. **New tests** — depends on route:
   - **Routes A and B**: the implementation agent already wrote them in Step 6. Do **not**
     spawn a separate Test agent.
   - **Route C**: spawn a dedicated Test agent (`subagent_type: general-purpose`, path only)
     following project conventions; it writes the `## Testing` section itself.
6. **Verify all tests pass**: One final run

The `## Testing` section is written to the request file (by the Route C Test agent, or by the
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
   per-request visibility — no need to read the REQ file). Lead with the agent tally so the
   count is always visible. Example:

   ```
   REQ-007 (Route B · 3 agents): plan ✓ · explore ✓ (3 files)
     implement ✓ — 3 files, 2 tests · test ✓ pass · archived · commit a3f9c2
   ```

   A Route A request reads `REQ-009 (Route A · 1 agent)` (or `0 agents` if handled inline);
   a Route C request reads `(Route C · 5 agents)`. The agent count must match the budget you
   announced in Step 3. Tune verbosity to taste: one line for terse, the two-line block above
   for medium, or add the per-token detail for verbose. The full detail always lives in the
   REQ file regardless.
2. Re-check `do-work/` for `REQ-*.md` files (fresh check, filenames only)
3. If found: start Step 1 again
4. If empty: Run [cleanup action](./cleanup.md), report final summary, exit

## What This Action Does NOT Do

- Create new request files (use the do action)
- Make architectural decisions beyond the request
- Run without user present (supervised automation)
- Modify completed/in-progress requests from other agents
- Skip planning entirely (every route still gets a plan — Route A's is a 1–3 line inline plan
  written by the orchestrator; B and C get a Plan agent)

## Additional References

- [Orchestration details](../references/work-orchestration.md) — agent prompts, checklists, folder structure, schema, error handling
- [Examples & retrospective](../references/work-examples.md) — example sessions, progress reporting, retrospective value
- [Verify-plan action](./verify-plan.md) — full plan verification protocol
- [Cleanup action](./cleanup.md) — archive consolidation

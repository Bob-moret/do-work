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
  |     |
  |     +-- TRIAGE: Assess complexity -> label (A/B/C)
  |     +-- PLAN: Proportional to complexity (1 line -> multi-step)
  |     +-- VERIFY PLAN: Coverage analysis against requirements
  |     +-- EXPLORE: When plan references unknown files/patterns
  |     +-- IMPLEMENT: agent (has plan + optional exploration)
  |     +-- TEST: Run and verify tests
  |
  +-- Loop continues until queue empty
```

## Sub-agent Compatibility

This document uses "spawn agent" language. Use your platform's subagent or multi-agent mechanism when available. If your tool does not support subagents, run the phases sequentially in the same session.

**Claude Code hint**: When available, use native agent types for better results:
- Plan phase: spawn with `subagent_type: Plan` (read-only, optimized for planning)
- Explore phase: spawn with `subagent_type: Explore` (read-only, optimized for codebase search)
- Implement phase: spawn with `subagent_type: general-purpose` (full tool access)

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

**You must do yourself (not delegate to agents):** move files to `working/`, update frontmatter, write triage/plan/exploration/testing sections, move files to `archive/`, create git commits.

**Agents do:** Planning, Exploring, Implementation, Writing/running tests.

---

### Step 0: Write Your Work Order

**Before touching any files**, write out the checklist. See [references/work-orchestration.md](../references/work-orchestration.md) for the full checklist template.

```
Work order -- [REQ-NNN]:
[ ] Step 1:   Identify next REQ-*.md in do-work/
[ ] Step 2:   Claim -> move to working/, update frontmatter
[ ] Step 3:   Triage -> assign route A/B/C, append ## Triage
[ ] Step 4:   Plan -> spawn Plan agent, append ## Plan
[ ] Step 4.5: *** MANDATORY *** Verify Plan -> append ## Plan Verification
[ ] Step 5:   Explore -> spawn Explore agent OR append "Not needed"
[ ] Step 6:   Implement -> spawn implementation agent
[ ] Step 6.5: Run tests -> append ## Testing
[ ] Step 7:   Archive -> update frontmatter, move file
[ ] Step 8:   Commit -> git add -A && git commit (git repos only)
[ ] Step 9:   Loop or exit
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

**[Orchestrator action]**

Read request, apply assessment decision flow, update frontmatter with route. Append to request file:

```markdown
## Triage

**Route: [A/B/C]** - [Simple/Medium/Complex]

**Reasoning:** [1-2 sentences]
```

Report briefly: "Simple request", "Medium complexity", or "Complex request — detailed planning ahead"

### Step 4: Planning Phase (All routes)

**[Spawn agent — plan depth scales to complexity]**

Every request gets a plan:
- **Route A**: 1-3 lines. Name the file(s) and the change.
- **Route B**: Identify what to find/match, plus implementation steps.
- **Route C**: Detailed — files, ordering, dependencies, architecture, testing approach.

For the full Plan agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

After Plan agent returns, append `## Plan` section to request file.

### Step 4.5: Verify Plan (All routes)

**[Mandatory step — runs automatically after planning]**

Verify plan against REQ to ensure every requirement is addressed. Fix gaps directly.

1. **Enumerate source items** from REQ + UR input.md
2. **Map each item to plan**: Full / Partial / Missing
3. **Calculate coverage**: (full + 0.5 x partial) / total x 100
4. **Auto-fix the plan** — add missing steps, expand partial coverage
5. **Store results** — append `## Plan Verification` to request file

See [verify-plan action](./verify-plan.md) for the full protocol.

> **Gate:** `## Plan Verification` must be written before starting Step 5.

### Step 5: Exploration Phase (When plan indicates)

**[Spawn agent if plan references unknown files/patterns]**

- **Exploration needed:** Plan references patterns to match, files to discover, conventions to follow
- **Exploration not needed:** Plan names specific files and exact changes

For the full Explore agent prompt template, see [references/work-orchestration.md](../references/work-orchestration.md).

After Explore agent returns, append `## Exploration` section. If skipped, append "Not needed" section.

### Step 6: Implementation Phase (All routes)

**[Spawn agent — agent does the actual code changes]**

Spawn a general-purpose agent with: full request content, plan from Step 4, and optionally exploration context from Step 5.

For the full implementation agent prompt templates (with/without exploration), see [references/work-orchestration.md](../references/work-orchestration.md).

Key guidelines for the agent:
- Follow existing code patterns
- Make minimal, focused changes
- Document blockers and plan deviations
- Write tests for new functionality and bug fix regressions

### Step 6.5: Testing Phase (All routes)

**[Orchestrator runs tests and may spawn agent for new tests]**

1. **Detect testing infrastructure** (jest, vitest, pytest, cargo test, go test, etc.)
2. **Identify relevant tests** based on modified files
3. **Run existing related tests**
4. **If tests fail**: Return to implementation to fix (loop until pass or 3 attempts)
5. **If new tests needed**: Spawn agent to write tests following project conventions
6. **Verify all tests pass**: One final run

Append `## Testing` section to request file.

If no testing infrastructure detected, skip and note: "No testing infrastructure detected."

### Step 7: Archive and Continue

**[Orchestrator action — do this yourself]**

**On success:**
1. Update frontmatter: `status: completed`, `completed_at: <timestamp>`
2. Append `## Implementation Summary` if not present
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

### Step 9: Loop or Exit

1. Re-check `do-work/` for `REQ-*.md` files (fresh check)
2. If found: Report completion, start Step 1 again
3. If empty: Run [cleanup action](./cleanup.md), report final summary, exit

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

---
name: do-work
description: >-
  Task queue for capturing and processing work requests. Use when the user says
  "do work", wants to add tasks/bugs/features to a queue, process pending requests,
  verify captured requirements, or manage a structured work backlog. Handles both
  quick captures ("do work fix the button") and complex multi-feature specs.
argument-hint: run | (task to capture) | verify | cleanup | version | changelog
disable-model-invocation: true
allowed-tools: Read Write Edit Bash Grep Glob Agent
---

# Do-Work Skill

A unified entry point for task capture and processing.

## Current Queue Status

```!
echo "Pending: $(ls do-work/REQ-*-*.md 2>/dev/null | wc -l | tr -d ' ') request(s)"
echo "In progress: $(ls do-work/working/REQ-*.md 2>/dev/null | wc -l | tr -d ' ')"
echo "Archived: $(ls do-work/archive/UR-* 2>/dev/null | wc -l | tr -d ' ') UR(s)"
```

**Actions:**

- **do**: Capture new tasks/requests → creates UR folder (verbatim input) + REQ files (queue items), always paired, then verifies coverage
- **work**: Process pending requests → plans (with plan verification), explores, builds, tests
- **verify**: Evaluate captured REQs against original input → coverage check with auto-fix (also runs automatically after capture)
- **cleanup**: Consolidate archive → moves loose REQs into UR folders, closes completed URs

> **Core concept:** The do action always produces both a UR folder (preserving the original input) and REQ files (the queue items). Each REQ links back to its UR via `user_request` frontmatter. This pairing is mandatory for all requests — simple or complex.

> **Capture ≠ Execute.** The do action captures requests. The work action executes them. These are strictly separate operations. After the do action finishes writing files and reporting back, **STOP**. Do not start processing the queue, do not begin implementation, do not "helpfully" transition into the work action. The user decides when to execute — always. The only exception is if the user explicitly says something like "add this and then run it" or "capture this and start working" in the same invocation.

## Routing Decision

### Step 1: Parse the Input

Examine what follows "do work":


Check these patterns **in order** — first match wins:

| Priority | Pattern                  | Example                                                                                                                            | Route                         |
| -------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| 1        | Empty or bare invocation | `do work`                                                                                                                          | → Ask: "Start the work loop?" |
| 2        | Action verbs only        | `do work run`, `do work go`, `do work start`                                                                                       | → work                        |
| 3        | Verify keywords          | `do work verify`, `do work check`, `do work evaluate`                                                                              | → verify                      |
| 4        | Cleanup keywords         | `do work cleanup`, `do work tidy`, `do work consolidate`                                                                           | → cleanup                     |
| 5        | Version keywords         | `do work version`, `do work update`, `do work check for updates`                                                                   | → version                     |
| 6        | Changelog keywords       | `do work changelog`, `do work release notes`, `do work what's new`, `do work what's changed`, `do work updates`, `do work history` | → version                     |
| 7        | Descriptive content      | `do work add dark mode`, `do work [meeting notes]`                                                                                 | → do                          |


### Step 2: Preserve Payload

**Critical rule**: Never lose the user's content.

**Single-word rule**: A single word is either a known keyword or ambiguous — it is never "descriptive content."

- **Matches a keyword** in the routing table (e.g., "version", "verify", "cleanup") → route to that action directly.
- **Doesn't match any keyword** (e.g., "refactor", "optimize") → ambiguous. Ask: "Do you want to add '`{word}`' as a new request, or did you mean something else?"

Only route to **do** when the input is clearly descriptive — multiple words, a sentence, a feature request, etc.

If routing is genuinely unclear AND multi-word content was provided:

- Default to **do** (adding a task)
- Hold onto $ARGUMENTS
- If truly ambiguous, ask: "Add this as a request, or start the work loop?"
- User replies with just "add" or "work" → proceed with original content

### Action Verbs (→ Work)

These signal "process the queue":
run, go, start, begin, work, process, execute, build, continue, resume

### Verify Verbs (→ Verify)

These signal "check request quality":
verify, check, evaluate, review requests, review reqs, audit

Note: "check" routes to verify ONLY when used alone or with a target (e.g., "do work check UR-003"). When followed by descriptive content it routes to do (e.g., "do work check if the button works" → do).

### Cleanup Verbs (→ Cleanup)

These signal "consolidate the archive":
cleanup, clean up, tidy, consolidate, organize archive, fix archive

### Changelog Verbs (→ Version)

These signal "show release notes":
changelog, release notes, what's new, what's changed, updates, history

Note: "updates" (plural) routes to changelog display. "update" (singular) routes to update check. Both are handled by the version action.

### Content Signals (→ Do)

These signal "add a new task":

- Descriptive text beyond a single verb
- Feature requests, bug reports, ideas
- Screenshots or context
- "add", "create", "I need", "we should"

## Examples

### Routes to Work

- `do work` → "Ready to process the queue?" (confirmation)
- `do work run` → Starts work action immediately
- `do work go` → Starts work action immediately

### Routes to Verify

- `do work verify` → Evaluates most recent UR's REQs
- `do work verify UR-003` → Evaluates specific UR
- `do work check REQ-018` → Evaluates the UR that REQ-018 belongs to
- `do work evaluate` → Evaluates most recent UR's REQs
- `do work review requests` → Evaluates most recent UR's REQs

### Routes to Cleanup

- `do work cleanup` → Consolidates archive, closes completed URs
- `do work tidy` → Same as cleanup
- `do work consolidate` → Same as cleanup

### Routes to Changelog (via Version)

- `do work changelog` → Displays changelog (newest at bottom)
- `do work release notes` → Same as changelog
- `do work what's new` → Same as changelog
- `do work updates` → Same as changelog
- `do work history` → Same as changelog

### Routes to Do

- `do work add dark mode` → Creates REQ file + UR folder
- `do work the button is broken` → Creates REQ file + UR folder
- `do work [400 words]` → Creates REQ files + UR folder with full verbatim input

## Payload Preservation Rules

When clarification is needed but content was provided:

1. **Do not lose $ARGUMENTS** - keep the full payload in context
2. **Ask a simple question**: "Add this as a request, or start the work loop?"
3. **Accept minimal replies**: User says just "add" or "work"
4. **Proceed with original content**: Apply the chosen action to the stored arguments
5. **Never ask the user to re-paste content**

This enables a two-phase commit pattern:

1. Capture intent payload
2. Confirm action

## Action References

Follow the detailed instructions in:

- [do action](./actions/do.md) - Request capture
- [work action](./actions/work.md) - Queue processing
- [verify-request action](./actions/verify-request.md) - Coverage verification of captured requests (runs after capture, or manually)
- [verify-plan action](./actions/verify-plan.md) - Plan coverage verification (runs after planning in the work action)
- [cleanup action](./actions/cleanup.md) - Archive consolidation and UR closure
- [version action](./actions/version.md) - Version, updates & changelog


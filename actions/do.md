# Do Action

> **Part of the do-work skill.** Invoked when routing determines the user is adding a request. Creates a `do-work/` folder in your project for request tracking.

A fast-capture system for turning quick ideas into structured request files. Designed for speed - minimal interaction when intent is clear, human-in-the-loop only when genuinely ambiguous.

## Required Outputs

Every invocation of the do action produces exactly two things:

1. **A User Request (UR) folder** at `do-work/user-requests/UR-NNN/` containing an `input.md` with the full verbatim user input
2. **One or more REQ files** at `do-work/REQ-NNN-slug.md`, each linked to the UR via `user_request: UR-NNN` in frontmatter

Both are mandatory. Never create one without the other. This applies to every invocation — simple or complex, one REQ or ten.

- A REQ without a `user_request` field is orphaned — there is no source of truth to verify against
- A UR without REQ files is pointless — nothing gets queued for the work action
- The verify action depends on this linkage to evaluate capture quality

## Philosophy

- **Speed over perfection**: This is a rapid capture interface, not a design review
- **Represent, don't expand**: If the user says 5 words, write a 5-word request (with structure). Don't inflate a simple idea into a 400-word PRD
- **The building agent solves technical questions**: You're capturing intent, not making architectural decisions
- **Multiple requests are expected**: Users often say "do X, and also Y, and don't forget Z"
- **Never be lossy**: For complex inputs, preserve ALL detail - don't summarize away requirements

## Simple vs Complex Requests

| Mode | Input Size | Approach |
|------|------------|----------|
| **Simple** | Short, 1-3 features | Quick capture, lean format |
| **Complex** | Long, detailed, multi-feature | Full preservation with context document |

### Detecting Complex Requests

Treat a request as **complex** when ANY of these apply:

- **Length**: Input is >500 words or feels substantial
- **Detail**: Includes specific requirements, constraints, edge cases, acceptance criteria
- **Breadth**: Describes 3+ distinct features or components
- **Explicit**: User says "spec", "PRD", "detailed requirements", "here's what I need..."
- **Nuance**: Contains "must", "should", "never", "always", specific values, or conditional logic
- **Relationships**: Features depend on each other or have sequencing

**When in doubt, treat it as complex.** Over-preserving is better than losing requirements.

### The Lossy Problem

When splitting a 15-minute detailed spec into 5 requests, information gets lost:
- Shared context disappears
- Cross-feature requirements fall through cracks
- The "why" behind decisions vanishes
- Subtle constraints get summarized away
- **User certainty level** — hedges like "I'm not trying to be prescriptive," "these are just ideas" signal the builder has latitude. Hard statements like "must," "definitely," "never" signal firm requirements.
- **Scope awareness cues** — when the user says "keep it simple" or "don't over-build," that's important builder guidance that gets dropped during extraction.
- **Thinking evolution** — when users talk through options and land on a decision, note the final decision AND that they explored alternatives.

**Solution: User Request (UR) Documents** — Every invocation creates a UR folder preserving the FULL verbatim input. REQ files reference back to their UR. The verify action compares against the original.

## Request File Location

All request files go in the project's `do-work/` folder:
- `do-work/` at the project root (create if it doesn't exist)
- `do-work/user-requests/UR-NNN/` for each user request invocation, containing:
  - `input.md` — the verbatim original input (always created)
  - `assets/` — screenshots, images, and other reference materials for this request

**CRITICAL: The do action writes to these locations:**
1. `do-work/` root - ONLY for `REQ-*.md` request files (the queue)
2. `do-work/user-requests/UR-NNN/` - For the verbatim input and assets per user request

**NEVER write to:**
- `do-work/working/` - This is exclusively managed by the work action
- `do-work/archive/` - This is exclusively managed by the work action
- The project root directory - All do-work files stay within `do-work/`

### File Immutability Rule

**Files in `working/` and `archive/` are IMMUTABLE.** They must never be modified by any action other than the work action's own processing pipeline. If someone wants to add to an in-flight or completed request, create a **new addendum request** that references the original.

### File Naming Convention

**Request files:** `REQ-[number]-[slug].md` in `do-work/` root
**UR folders:** `do-work/user-requests/UR-[number]/`

To get the next REQ number, check existing `REQ-*.md` files in `do-work/`, `do-work/working/`, and `do-work/archive/` (and inside `archive/UR-*/`), then increment from the highest.

### Backward Compatibility

REQ files created before the UR system will not have a `user_request` field. This is fine — the work action handles both patterns. New REQs always get `user_request`.

## File Formats

For detailed file format templates (simple REQ, complex REQ, UR input.md, addendum REQ, frontmatter fields), see [references/do-formats.md](../references/do-formats.md).

## Workflow

### Step 0: Declare Your Capture Order

**Before doing anything else**, write out the following checklist:

```
Capture order:
[ ] Step 1:   Parse input — single vs. multiple requests
[ ] Step 1.5: Assess complexity — simple or complex mode
[ ] Step 2:   Check for existing requests — duplicates, addendums
[ ] Step 3:   Clarify only if genuinely ambiguous (skip if clear)
[ ] Step 4:   Handle screenshots if present (skip if none)
[ ] Step 5:   Create UR folder with input.md
[ ] Step 5:   Create REQ file(s) with user_request: UR-NNN in frontmatter
[ ] Step 5:   Update UR requests array with all REQ IDs
[ ] Step 5.5: *** MANDATORY *** Verify requests — enumerate items from UR input, map to REQs, fix gaps, append ## Verification to every REQ file
[ ] Step 6:   Report back — include verification coverage, never report without it
```

### Step 1: Parse the Request

Read the user's input. Look for:
- Single request vs multiple requests
- Clear intent vs ambiguous intent
- Any provided context (screenshots, references to existing behavior)

**Multiple request indicators:** "and also", "oh and", "plus", "additionally", comma-separated lists, numbered items, distinct topics.

### Step 1.5: Assess Complexity

**Complex indicators** (any of these → complex mode):
- Input is >500 words
- 3+ distinct features or components
- Detailed requirements, constraints, or acceptance criteria
- User says "spec", "PRD", "detailed", "requirements"
- Contains specific values, conditions, or edge cases
- Features have dependencies or sequencing

**Simple indicators** (all of these → simple mode):
- Short, focused input (<200 words)
- 1-2 features max
- No detailed constraints

**When uncertain → treat as complex.**

### Step 2: Check for Existing Requests

Read all files in `do-work/` folder, **and list filenames in `do-work/working/` and `do-work/archive/`**. For each parsed request, check for duplicates.

| Existing request is in... | Action |
|---------------------------|--------|
| `do-work/` (queue) | Safe to append an addendum to the pending file |
| `do-work/working/` | **NEVER modify.** Create a new addendum REQ |
| `do-work/archive/` | **NEVER modify.** Create a new addendum REQ |

For addendum formats, see [references/do-formats.md](../references/do-formats.md).

### Step 3: Clarify Only If Needed

Use your environment's ask-user prompt/tool ONLY when:
- Request is genuinely ambiguous (could mean two very different things)
- Similar request exists and it's unclear if this is new or a revision
- User's intent could conflict with existing requests

**DO NOT ask** when you can reasonably infer the intent, the request is simple and clear, or technical implementation is unclear (that's for the building agent).

### Step 4: Handle Screenshots

If the user passes a screenshot:

1. **Find the cached/attached image**: Use your tool's attachment UI or image cache (Claude Code: `~/.claude/image-cache/[session-uuid]/[number].png`)
2. **Verify it's the right image**
3. **Copy to UR assets**: `cp` to `do-work/user-requests/UR-[num]/assets/REQ-[num]-[slug].png`
4. **Reference in request file**
5. **Still write a description**: Include thorough text description for searchability

When describing screenshots, be thorough — what it shows, visual layout, all visible text, what the user is pointing out, UI state, problems visible.

### Step 5: Write Request Files

#### Simple Mode

1. **Create UR folder** at `do-work/user-requests/UR-NNN/input.md` with verbatim input
2. **Create REQ files** (one per distinct request) with `user_request: UR-NNN` in frontmatter
3. **Link them together** — update UR's `requests` array, verify every REQ has `user_request`

#### Complex Mode

1. **Create UR folder** with full verbatim input, summary, empty `requests` array
2. **Identify all distinct features/requests** — list each, note dependencies and shared constraints
3. **Create individual REQ files** using complex format — set `user_request`, `related`, `batch`
4. **Update UR** — fill in `requests` array and "Extracted Requests" table
5. **Identify batch-level concerns** — add to UR's "Batch Constraints" section AND each relevant REQ

For file format templates, see [references/do-formats.md](../references/do-formats.md).

### Step 5.5: Verify Requests

**[Mandatory step — runs automatically after file creation]**

Verify the REQ files against the UR's verbatim input. Fix gaps. Store results. Do not ask user — just fix and report.

**Skip condition:** If the user's input included "skip verification" or similar language.

1. **Enumerate source items**: Re-read UR input, extract numbered list of every discrete requirement
2. **Map each item to REQs**: Classify as Full / Partial / Missing
3. **Calculate coverage**: Coverage % = (full + 0.5 x partial) / total x 100
4. **Auto-fix gaps**: Edit appropriate REQ files directly — don't invent requirements
5. **Store results**: Append `## Verification` section to each REQ file with coverage map

See [verify-request action](./verify-request.md) for the full protocol.

### Step 6: Report Back

**BEFORE reporting: Check that every REQ file has a `## Verification` section.** If any is missing, go back to Step 5.5.

After creating files, give a brief summary:
- What files were created
- Any existing requests that were updated
- Any requests that were skipped (duplicates)
- Verification coverage results

### STOP After Capture

**The do action ends here.** Do not proceed to process the queue.

- **Do NOT** check for pending requests and start working on them
- **Do NOT** offer to "go ahead and start building"
- **DO** let the user decide when to run `do work run`

Exception: if the user explicitly combined capture and execution (e.g., "add this and start working on it").

## Do Action Checklist

```
[ ] Parse input — single vs multiple requests (Step 1)
[ ] Assess complexity — simple or complex mode (Step 1.5)
[ ] Check for existing requests — duplicates, addendums (Step 2)
[ ] Clarify only if genuinely ambiguous (Step 3)
[ ] Handle screenshots if present (Step 4)
[ ] Create UR folder with input.md (Step 5)
[ ] Create REQ file(s) with user_request: UR-NNN in frontmatter (Step 5)
[ ] Update UR requests array with all REQ IDs (Step 5)
[ ] Run verify-request on created REQs (Step 5.5)
[ ] Report created files to user, include verification coverage (Step 6)
```

**Common mistakes:**
- Creating REQ files but forgetting the UR folder
- Creating the UR folder but not setting `user_request` in REQ frontmatter
- Skipping the UR for simple requests (it's required for ALL requests)
- Not updating the UR's `requests` array after creating REQs
- Skipping verify-request (it's mandatory unless user said "skip verification")

## Additional References

- [File formats & templates](../references/do-formats.md) — REQ formats, UR format, frontmatter fields, addendum format
- [Examples & edge cases](../references/do-examples.md) — worked examples and edge case handling
- [Verify-request action](./verify-request.md) — full verification protocol

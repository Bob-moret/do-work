# do-work Examples and Edge Cases Reference

Extracted from the `do.md` action file (lines 839-1011).

---

## Examples

### Example 1: Simple Single Request

```
User: do work add keyboard shortcuts
```

**Result:** Created `REQ-004-keyboard-shortcuts.md`

- Single clear feature request
- One UR folder created, one REQ file generated
- No ambiguity, no questions asked

---

### Example 2: Multiple Requests

```
User: do work add dark mode, also the search feels slow, and we need an export button
```

**Result:** Created REQ-005, REQ-006, REQ-007

- Three distinct requests detected in one message
- Each gets its own REQ file
- All linked to the same UR folder
- Dark mode = feature, search slow = performance bug, export button = feature

---

### Example 3: Potential Duplicate

```
User: do work fix the lag when typing
```

**Result:** Found existing REQ-006 about search performance, asks user

- Detected similarity with existing request about search performance
- Asks user whether this is the same issue or a new one
- Does not silently create a duplicate
- Does not silently skip it either

---

### Example 4: Request with Screenshot

```
User: do work when I click here [screenshot] nothing happens
```

**Result:** Created `UR-004/input.md`, assets, REQ-008

- Screenshot saved to UR assets folder
- Visual context preserved for the builder
- REQ references the screenshot location
- Bug report created from visual evidence

---

### Example 5: Enhancement to Queued Request (Addendum)

```
User: do work dark mode should also affect the sidebar
```

**Result:** Found REQ-005 pending, added addendum

- REQ-005 (dark mode) is still in the queue (not yet being worked on)
- Enhancement appended directly to the existing REQ file
- No new REQ created -- it's the same feature, just more detail
- Addendum clearly marked within the file

---

### Example 5b: Enhancement to In-Flight Request (New Addendum REQ)

Same scenario as Example 5, but REQ-005 is already in `working/`.

**Result:** Creates new REQ-021 with `addendum_to: REQ-005`

- Cannot modify REQ-005 because it is already being worked on
- New REQ file created with `addendum_to: REQ-005` in frontmatter
- Builder will pick up the addendum and incorporate it
- Preserves the integrity of in-flight work

---

### Example 6: Complex Multi-Feature Request

```
User: do work [15 minutes of detailed auth requirements]
```

**Result:** Creates `UR-001/input.md` + REQ-010 through REQ-013

- Detected as complex due to length and detail
- Verbatim input preserved in `UR-001/input.md`
- Requirements decomposed into multiple focused REQ files
- Original wording preserved -- no summarization
- Edge cases and constraints from the user's words captured exactly

---

### Example 7: Detecting Complexity

```
User: do work I need a new dashboard...
```

**Result:** Detects complex mode, creates `UR-002` + REQ-014 through REQ-018

- Complexity detected automatically from scope and detail
- Full UR folder created with verbatim input
- Multiple REQ files generated from decomposition
- Each REQ is focused on a single deliverable aspect of the dashboard

---

## What NOT To Do

### For all requests (simple and complex):

- Don't create REQ files without a UR folder
- Don't skip the UR for simple requests
- Don't skip the `user_request` field in REQ frontmatter
- Don't ask about implementation details
- Don't create multi-page PRDs for simple requests
- Don't refuse to create a request because it's "too vague"
- Don't merge obviously distinct requests into one file
- Don't make the user wait for questions you could answer yourself
- Don't add requirements the user didn't mention

### For complex requests specifically:

- Don't summarize detailed requirements -- preserve exact words
- Don't drop edge cases or constraints
- Don't assume something is obvious -- write it down
- Don't split related requirements across unrelated requests
- Don't "clean up" the verbatim input

---

## Edge Cases

Cover all these edge cases:

1. **User says something is broken but doesn't say what** -- Create the request anyway. Capture what they said, even if vague. The builder or a follow-up conversation will clarify.

2. **User references something from earlier conversation** -- Include that context in the UR and REQ. Don't lose conversational context just because it wasn't in the current message.

3. **User gives very long detailed request** -- Complex mode with full UR. Preserve every word. Decompose into multiple REQ files.

4. **Request seems impossible or contradictory** -- Capture it as-is. The builder will figure it out. Don't refuse or push back.

5. **Requirement applies to multiple features** -- Include it in ALL relevant REQ files. Duplication across REQs is acceptable and expected for cross-cutting concerns.

6. **User changes their mind mid-request** -- Capture the final decision. Note the evolution in the UR input so the builder has full context.

7. **Features have circular dependencies** -- Note the dependency in both request files. Don't try to resolve the circularity yourself.

8. **Some requirements are vague, others specific** -- Capture both as-is. Don't try to make vague ones more specific or specific ones more vague. Preserve fidelity.

9. **User mentions something once in passing** -- Capture it. It's a requirement. If they said it, it matters. Don't filter based on emphasis or repetition.

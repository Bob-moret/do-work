# Work Action Examples Reference

> **Reference document for the [work action](../actions/work.md).** Contains example sessions, progress reporting formats, command usage, and retrospective examples extracted from the main action file.

---

## Progress Reporting

Keep the user informed with brief updates:

```
Processing REQ-003-dark-mode.md...
  Triage: Complex (Route C)
  Planning...     [done]
  Verify Plan...  [done] 90% → 100% (2 items fixed)
  Exploring...    [done]
  Implementing... [done]
  Testing...      [done] ✓ 12 tests passing
  Archiving...    [done]
  Committing...   [done] → abc1234

Processing REQ-004-fix-typo.md...
  Triage: Simple (Route A)
  Planning...     [done] (1 line)
  Verify Plan...  [done] 100%
  Implementing... [done]
  Testing...      [done] ✓ 3 tests passing
  Archiving...    [done]
  Committing...   [done] → def5678

Found 1 more pending request. Continuing...

Processing REQ-005-add-tooltip.md...
  Triage: Simple (Route A)
  Planning...     [done] (1 line)
  Verify Plan...  [done] 100%
  Implementing... [done]
  Testing...      [done] ✓ 2 tests passing
  Archiving...    [done]
  Committing...   [done] → ghi9012

All 3 requests completed:
  - REQ-003 (Route C) → abc1234
  - REQ-004 (Route A) → def5678
  - REQ-005 (Route A) → ghi9012
```

For non-Git projects, the commit step is skipped:
```
Processing REQ-003-dark-mode.md...
  Triage: Complex (Route C)
  Planning...     [done]
  Verify Plan...  [done] 100% (no fixes needed)
  Exploring...    [done]
  Implementing... [done]
  Testing...      [done] ✓ 8 tests passing
  Archiving...    [done]
  (not a git repo, skipping commit)

Completed.
```

---

## Commands

### `do work`
Process all pending requests in order. Automatically unclaims anything in `working/` that has been there longer than 1 hour before starting.

### `do work resume`
Like `do work`, but unclaims everything in `do-work/working/` immediately — no age threshold. Use this when you know a previous session was interrupted and left a request stranded. The unclaimed request moves back to the front of the queue with its existing triage/plan sections preserved; the work action picks it up and continues from the implementation phase if planning is already documented.

### `do work REQ-005` (future enhancement)
Process a specific request by number, regardless of status.

### `do work --dry-run` (future enhancement)
Show what would be processed and their triage routes without making changes.

---

## Example Session

```
User: do work

Assistant: Checking do-work/ folder...
Found 3 pending requests. Starting with REQ-003-dark-mode.md...
Triage: Complex (Route C) - new feature across multiple components

[Spawns Plan agent]
Planning complete. Key steps:
  1. Add theme context provider
  2. Create useTheme hook
  3. Add toggle to settings
  4. Update Tailwind config

Verifying plan coverage...
  Items: 8 enumerated from request
  Pre-fix: 88% (7 full, 0 partial, 1 missing)
  Fixed: Added Step 5 for system preference detection
  Post-fix: 100%

[Spawns Explore agent]
Found relevant patterns:
  - Existing context in src/stores/
  - Settings panel at src/components/settings/
  - Tailwind config at tailwind.config.js

[Spawns implementation agent]
Implementation complete:
  - Created src/stores/theme-store.ts
  - Modified src/components/settings/SettingsPanel.tsx
  - Updated tailwind.config.js

Testing: Running related tests...
  - Added tests/theme-store.spec.ts
  - All 12 tests passing ✓

Archived REQ-003-dark-mode.md
Committed → abc1234

Continuing with REQ-004-fix-submit-crash.md...
Triage: Simple (Route A) - bug fix with clear reproduction

[Spawns Plan agent]
Plan: "Fix null check in src/components/Form.tsx handleSubmit(). Add regression test."

Verifying plan coverage...
  Items: 2 enumerated from request
  Coverage: 100%

[Spawns implementation agent]
Implementation complete:
  - Fixed null check in src/components/Form.tsx:42

Testing: Running related tests...
  - Added regression test in tests/form.spec.ts
  - All 8 tests passing ✓

Archived REQ-004-fix-submit-crash.md
Committed → def5678

Continuing with REQ-005-change-timeout.md...
Triage: Simple (Route A) - config value change

[Spawns Plan agent]
Plan: "Update API_TIMEOUT in src/config.ts from 30000 to 60000."

Verifying plan coverage...
  Items: 1 enumerated from request
  Coverage: 100%

[Spawns implementation agent]
Implementation complete:
  - Updated API_TIMEOUT in src/config.ts from 30000 to 60000

Testing: No tests needed for config value change

Archived REQ-005-change-timeout.md
Committed → ghi9012

All 3 requests completed:
  - REQ-003: Route C → abc1234
  - REQ-004: Route A → def5678
  - REQ-005: Route A → ghi9012
```

---

## Retrospective Value

After running the work action, archived request files contain their full history:

**REQ-003-dark-mode.md (Route C):**
```markdown
---
id: REQ-003
title: Dark Mode
status: completed
created_at: 2025-01-26T09:00:00Z
claimed_at: 2025-01-26T10:30:00Z
route: C
completed_at: 2025-01-26T10:52:00Z
commit: abc1234
---

# Dark Mode

## What
Implement dark mode across the app.

---

## Triage

**Route: C** - Complex

**Reasoning:** New feature requiring theme system, multiple component updates, and Tailwind configuration changes.

**Planning:** Required

## Plan

### Implementation Strategy

1. **Create theme store** (src/stores/theme-store.ts)
   - Use Zustand for state management (matches existing patterns)
   - Store light/dark preference
   - Persist to localStorage

2. **Add useTheme hook** (src/hooks/useTheme.ts)
   - Expose current theme and toggle function
   - Handle system preference detection

3. **Update Tailwind config** (tailwind.config.js)
   - Enable darkMode: 'class'
   - Add dark variants for color palette

4. **Add toggle to settings panel** (src/components/settings/SettingsPanel.tsx)
   - Add ThemeToggle component
   - Position in appearance section

5. **Update key components**
   - Header, sidebar, main content areas
   - Use theme-aware Tailwind classes

### Testing Approach
- Unit tests for theme store
- Component tests for toggle behavior
- Visual regression if available

*Generated by Plan agent*

## Plan Verification

**Source**: REQ-003 (8 items enumerated)
**Pre-fix coverage**: 88% (7 full, 0 partial, 1 missing)
**Post-fix coverage**: 100% (8/8 items addressed)

### Fixes Applied

- Added Step 5: Detect and respect system color scheme preference

*Verified by verify-plan action*

## Exploration

- Existing stores in src/stores/ use Zustand pattern
- Settings panel at src/components/settings/SettingsPanel.tsx
- Tailwind config supports darkMode: 'class'

*Generated by Explore agent*

## Implementation Summary

- Created src/stores/theme-store.ts
- Modified 4 components for dark mode support
- Updated tailwind.config.js

*Completed by work action (Route C)*

## Testing

**Tests run:** npm test
**Result:** ✓ All tests passing (24 tests)

**New tests added:**
- tests/stores/theme-store.spec.ts - store state and persistence
- tests/components/ThemeToggle.spec.ts - toggle behavior

**Existing tests verified:**
- tests/components/SettingsPanel.spec.ts - updated for new toggle

*Verified by work action*
```

**REQ-005-change-timeout.md (Route A):**
```markdown
---
id: REQ-005
title: Change API Timeout
status: completed
created_at: 2025-01-26T09:15:00Z
claimed_at: 2025-01-26T10:55:00Z
route: A
completed_at: 2025-01-26T10:56:00Z
commit: ghi9012
---

# Change API Timeout

## What
Update the API timeout from 30s to 60s.

---

## Triage

**Route: A** - Simple

**Reasoning:** Single config value change, file explicitly mentioned in request.

## Plan

Update API_TIMEOUT in src/config.ts from 30000 to 60000.

*Generated by Plan agent*

## Plan Verification

**Source**: REQ-005 (1 item enumerated)
**Pre-fix coverage**: 100% (1/1 items addressed)

*Verified by verify-plan action*

## Exploration

**Not needed** -- plan names specific file and change. No discovery required.

*Skipped by work action*

## Implementation Summary

- Updated API_TIMEOUT in src/config.ts from 30000 to 60000

*Completed by work action (Route A)*

## Testing

**Tests run:** N/A
**Result:** Config value change, no tests needed

*Verified by work action*
```

This lets you:
- Review what planning recommended vs what was actually done
- Identify requests where complexity assessment was off (plan depth vs actual effort)
- Track patterns in request complexity over time
- Debug failed requests by seeing what context was gathered
- Analyze throughput: timestamps show queue wait time and implementation time
- Calibrate assessments: compare route label to actual plan depth and time spent

**Git integration benefits (when applicable):**
- **Rollback**: `git revert <commit>` undoes one complete request
- **Cherry-pick**: `git cherry-pick <commit>` pulls a specific fix to another branch
- **Bisect**: Find which request introduced a bug
- **Blame**: Commit message links to full request documentation

# Do Action — File Formats & Templates

Reference document for the do action's file formats. See [do action](../actions/do.md) for the main workflow.

## Request File Format

### Simple Request Format

For quick captures - short, single-feature requests.

```markdown
---
id: REQ-001
title: Brief descriptive title
status: pending
created_at: 2025-01-26T10:00:00Z
user_request: UR-001
---

# [Brief Title]

## What
[1-3 sentences describing what is being requested]

## Why (if provided)
[User's stated reasoning - ONLY if they gave one. Omit section if not provided]

## Context
[Any additional context, constraints, or details the user mentioned]

## Assets
[Detailed description of any screenshots, or links to manually-saved files]

---
*Source: [original verbatim request]*

## Verification

**Source**: UR-001/input.md
**Pre-fix coverage**: 100% (3/3 items)

### Coverage Map

| # | Item | REQ Section | Status |
|---|------|-------------|--------|
| 1 | Add dark mode toggle | What | Full |
| 2 | App-wide | What | Full |
| 3 | Toggle in settings | Context | Full |

*Verified by verify-request action*
```

Keep it lean. If the user said "add dark mode", the What section might just be "Add a dark mode toggle to the app." That's fine.

**Every REQ file must end with a `## Verification` section.** If it doesn't have one, Step 5.5 was skipped — go back and run it.

### Complex Request Format

For detailed, multi-feature, or nuanced requests. Uses additional fields and references a context document.

```markdown
---
id: REQ-005
title: OAuth login flow
status: pending
created_at: 2025-01-26T10:00:00Z
user_request: UR-001
related: [REQ-006, REQ-007, REQ-008]
batch: auth-system
---

# OAuth Login Flow

## What
Implement OAuth-based authentication with Google and GitHub providers.

## Detailed Requirements
[Extract ALL requirements from the original input that apply to THIS feature specifically.
DO NOT SUMMARIZE. Include everything the user said about this feature.]

- Must support Google OAuth 2.0 with PKCE
- Must support GitHub OAuth
- Redirect to /dashboard after successful login
- Store refresh tokens encrypted at rest (user said "tokens must be encrypted")
- Handle token expiration gracefully - refresh automatically if possible
- Show loading spinner during OAuth redirect
- Handle OAuth errors with user-friendly messages, not technical errors
- "Remember me" checkbox to extend session duration
- Log authentication events for security audit

## Constraints
[Any limitations or restrictions mentioned]

- Must work with existing session management system
- Cannot use third-party auth services (user explicitly said "no Auth0 or similar")
- Must support existing users migrating from password auth

## Dependencies
[What this feature needs or what needs it]

- Depends on: REQ-007 (session management) - needs session system first
- Blocks: REQ-006 (user profile) - profile needs auth to exist

## Builder Guidance
[Capture the user's tone/intent signals that affect HOW to build, not WHAT to build]

- Certainty level: Exploratory / Firm / Mixed
- Scope cues: [Any "keep it simple," "don't over-build," "just ideas" signals]
- [Any specific latitude given to the builder]

## Open Questions
[Ambiguities that the builder should clarify or decide]

- Should OAuth be the only login method, or alongside password?
- What happens if user's email from OAuth doesn't match existing account?

## Full Context
See [user-requests/UR-001/input.md](./user-requests/UR-001/input.md) for complete verbatim input.

---
*Source: See UR-001/input.md for full verbatim input*

## Verification

**Source**: UR-001/input.md
**Pre-fix coverage**: 90% (9/10 items)
**Post-fix coverage**: 100% (10/10 items)

### Coverage Map

| # | Item | REQ Section | Status |
|---|------|-------------|--------|
| 1 | Google OAuth 2.0 with PKCE | Detailed Requirements | Full |
| 2 | GitHub OAuth | Detailed Requirements | Full |
| 3 | Redirect to /dashboard | Detailed Requirements | Full |
| 4 | Encrypt refresh tokens at rest | Detailed Requirements | Full |
| 5 | Handle token expiration | Detailed Requirements | Full |
| 6 | Loading spinner during redirect | Detailed Requirements | Full |
| 7 | User-friendly error messages | Detailed Requirements | Full |
| 8 | Remember me checkbox | Detailed Requirements | Full |
| 9 | No Auth0 or similar | Constraints | Full |
| 10 | Migrate existing password users | Constraints | Missing -> Fixed |

### Fixes Applied

- Added "Must support existing users migrating from password auth" to Constraints

*Verified by verify-request action*
```

### User Request (UR) input.md Format

Created for EVERY do action invocation. Preserves the FULL verbatim input.

**Location:** `do-work/user-requests/UR-[number]/input.md`

```markdown
---
id: UR-001
title: Authentication System Requirements
created_at: 2025-01-26T10:00:00Z
requests: [REQ-005, REQ-006, REQ-007, REQ-008]
word_count: 1847
---

# Authentication System Requirements

## Summary
[2-3 sentence overview of what this user request covers]

User described a complete authentication system including OAuth login, user profiles,
session management, and password reset flows. Emphasized security requirements and
integration with existing systems.

## Extracted Requests

| ID | Title | Summary |
|----|-------|---------|
| REQ-005 | OAuth login flow | Google/GitHub OAuth with PKCE |
| REQ-006 | User profile management | Profile editing, avatar upload |
| REQ-007 | Session management | Token storage, expiration, refresh |
| REQ-008 | Password reset flow | Email-based reset with expiry |

## Batch Constraints
[Cross-cutting concerns that apply to all REQs in this batch — omit for simple single-REQ requests]

- [Shared design principles, sequencing requirements, performance budgets]
- [User tone signals: "keep it simple," scope cues, latitude given]

## Full Verbatim Input

[THE COMPLETE, UNEDITED INPUT FROM THE USER]

[Paste the entire 15 minutes of transcribed text here, exactly as received.
Do not summarize, edit, or clean up. This is the source of truth.]

---
*Captured: 2025-01-26T10:00:00Z*
```

**For simple requests**, the input.md is minimal:

```markdown
---
id: UR-005
title: Add keyboard shortcuts
created_at: 2025-01-26T10:00:00Z
requests: [REQ-020]
word_count: 4
---

# Add keyboard shortcuts

## Full Verbatim Input

add keyboard shortcuts

---
*Captured: 2025-01-26T10:00:00Z*
```

### Frontmatter Fields

**All requests:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Request identifier (REQ-NNN) |
| `title` | Yes | Short title |
| `status` | Yes | Always `pending` when created |
| `created_at` | Yes | ISO 8601 timestamp |
| `user_request` | Yes | UR identifier linking back to the originating user request (UR-NNN) |

**Complex requests (additional fields):**

| Field | Required | Description |
|-------|----------|-------------|
| `related` | If applicable | Array of related REQ IDs |
| `batch` | If applicable | Batch name grouping related requests |
| `addendum_to` | If applicable | Original REQ ID this request amends (used when the original is in `working/` or `archive/`) |

**UR input.md documents:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | User request identifier (UR-NNN) |
| `title` | Yes | Descriptive title for the request or batch |
| `created_at` | Yes | ISO 8601 timestamp |
| `requests` | Yes | Array of REQ IDs extracted from this user request |
| `word_count` | Yes | Word count of original input (for reference) |

The work action adds additional fields (`claimed_at`, `route`, `completed_at`) as it processes each request.

**Note:** Legacy REQ files may have `context_ref` instead of `user_request`. The work action handles both.

### Addendum REQ Format

When the original request is in `working/` or `archive/`, create a new addendum REQ:

```markdown
---
id: REQ-021
title: "Addendum: dark mode sidebar support"
status: pending
created_at: 2025-01-27T09:00:00Z
user_request: UR-006
addendum_to: REQ-005
---

# Addendum: Dark Mode Sidebar Support

## What
Add sidebar support to the existing dark mode implementation (REQ-005).

## Context
This is an addendum to REQ-005 (dark mode), which is currently [in progress / completed].
The user realized after the original request was claimed that the sidebar also needs dark mode support.

## Requirements
- Sidebar must also respect the dark mode theme
- [any other new requirements]

---
*Source: Addendum to REQ-005. Original request: "dark mode should also affect the sidebar"*
```

Key rules for addendum REQs:
- **`addendum_to` field**: Links to the original REQ ID so the builder has context
- **Standalone**: The addendum must contain enough detail to be built independently
- **Goes in the queue**: It's a normal pending REQ in `do-work/`
- **Tell the user**: Explain that the original is already claimed/completed and this will be handled as a follow-up request

### Queued Request Addendum Format

When appending to a request still in the queue (`do-work/` root), don't rewrite — append:

```markdown
## Addendum (2025-01-27)

User added: "dark mode should also affect the sidebar"

- Sidebar must also respect dark mode theme
- [any other new requirements extracted from the update]
```

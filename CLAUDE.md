# Do-Work Skill Project

## Before Every Commit

**Always bump the version in `actions/version.md` before committing.**

The version is on the line that starts with `**Current version**:` - increment it using semver:
- Patch (0.1.0 → 0.1.1): Bug fixes, minor tweaks
- Minor (0.1.0 → 0.2.0): New features, behavior changes
- Major (0.1.0 → 1.0.0): Breaking changes

When in doubt, bump the patch version.

**Immediately after bumping the version, update `CHANGELOG.md`.**

Add a new entry at the top of the file (below the header), following this format:

```markdown
## X.Y.Z — The [Fun Two-Word Name] (YYYY-MM-DD)

[1-2 sentences. Casual, clear, and concise. What changed and why it matters. No fluff, but personality is welcome.]

- [Bullet points for specifics — what was added, changed, or fixed]
```

Rules for changelog entries:
- **Newest on top.** The file reads top-to-bottom as newest-to-oldest.
- **Give every version a name.** A short, fun title after the em dash (e.g., "The Organizer", "Typo Patrol"). This isn't a novel — two or three words max.
- **Date every entry.** Add the date in parentheses after the name, formatted as `(YYYY-MM-DD)`. Use today's date.
- **Lead with the value, not the implementation.** Say what the user gets, not what files changed. "The archive tidies itself now" beats "added cleanup.md".
- **Keep it brief.** One short paragraph + a few bullets. If you're writing more than 5 bullets, you're over-explaining.
- **Match the voice.** Conversational, not corporate. Imagine you're telling a friend what shipped. No jargon walls, no passive voice marathons.
- **Every version gets an entry.** No skipping. Even a patch fix deserves a line.

## Claude Code Native

This skill targets **Claude Code** specifically. Write action files for that environment
directly — no tool-agnostic hedging.

- **Name the real tools.** Use `AskUserQuestion` for clarifying questions, the Agent tool for
  subagents, and Claude Code's native agent types: `Plan` and `Explore` (read-only) and
  `general-purpose` (write-capable). Don't write "use your environment's ask-user prompt."
- **Scale agents to the route.** The work loop spawns only the agents a request's complexity
  route needs — Route A = Implement only (0–1 agents), Route B = Plan → Explore → Implement
  (3), Route C = Plan → Verify → Explore → Implement → Test (5). Keep this budget consistent
  across `actions/work.md`, `references/work-orchestration.md`, and `README.md`.
- **Make the agent count visible.** The orchestrator announces the budget before spawning
  (`Route B → 3 agents: …`) and logs a running tally. Preserve this when editing.
- **The REQ file is the data bus.** Agents get only the REQ file path and return a short
  status token; heavy content flows agent → disk → agent, never forwarded through the
  orchestrator's context.
- **Claude Code paths are facts, not examples.** e.g. images cache at
  `~/.claude/image-cache/[session-uuid]/[number].png`; skill references resolve via
  `${CLAUDE_SKILL_DIR}`.

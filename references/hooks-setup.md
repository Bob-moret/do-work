# Do-Work — Optional Hooks Configuration

This document describes optional hooks you can configure in Claude Code's `settings.json` to enhance the do-work skill experience. None of these are required — the skill works fully without them.

## Show Queue Status on Session Start

Automatically display pending requests when starting a new Claude Code session:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo '--- do-work queue ---' && ls do-work/REQ-*-*.md 2>/dev/null | wc -l | xargs -I{} echo '{} pending request(s)' && ls do-work/working/REQ-*.md 2>/dev/null | wc -l | xargs -I{} echo '{} in progress'"
          }
        ]
      }
    ]
  }
}
```

## Notify After Git Commit

Play a sound or show a notification after the work action commits a completed request:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT\" | grep -q 'git commit.*REQ-'; then echo '\\a'; fi"
          }
        ]
      }
    ]
  }
}
```

## How to Apply

Add these to your Claude Code settings file:
- **User-level**: `~/.claude/settings.json`
- **Project-level**: `.claude/settings.json` in your project root

Hooks from both levels are merged. See [Claude Code hooks documentation](https://code.claude.com/docs/en/hooks) for more details.

# /goal Skill for Claude Code

A persistent autonomous goal tracking skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), inspired by Codex's `/goal` feature.

Set a goal and Claude works on it continuously across conversation turns — no budget limits, no manual intervention — until the goal is fully complete or you explicitly pause/cancel it.

## Installation

### Option 1: Direct copy

```bash
mkdir -p ~/.claude/skills/goal/references
cp SKILL.md ~/.claude/skills/goal/
cp references/*.md ~/.claude/skills/goal/references/
```

### Option 2: Clone and symlink

```bash
git clone https://github.com/ClanyXH/goal-for-claude-skill.git
ln -s $(pwd)/goal-for-claude-skill ~/.claude/skills/goal
```

## Usage

| Command | Description |
|---------|-------------|
| `/goal <description>` | Set a new goal and start working |
| `/goal status` | Show progress and current task |
| `/goal pause` | Pause the active goal |
| `/goal resume` | Resume a paused or blocked goal |
| `/goal cancel` | Cancel and clean up |

### Examples

```
/goal Refactor the auth module to use JWT tokens
/goal Build a REST API for the bookstore with tests
/goal Fix all TypeScript errors in the project
/goal Create a landing page with dark mode and responsive design
/goal status
/goal pause
/goal resume
/goal cancel
```

## How It Works

1. **Goal creation**: `/goal <description>` decomposes your goal into concrete sub-tasks via TodoWrite and saves state to `~/.claude/goal-state.json`
2. **Autonomous execution**: Claude works through each sub-task continuously, updating progress after each completion
3. **Persistence**: Goal state survives across turns — if paused, resume anytime with `/goal resume`
4. **Completion**: Goal is marked complete only when ALL sub-tasks are done and verified

### State Machine

```
(new) → active → paused → active (resume)
               → blocked → active (resolve)
               → complete (all tasks done)
```

### Statuses

| Status | Meaning |
|--------|---------|
| `active` | Currently working on sub-tasks |
| `paused` | User paused — waiting for `/goal resume` |
| `blocked` | External blocker or 3x task failure — needs intervention |
| `complete` | All sub-tasks done, goal achieved |

## File Structure

```
goal/
├── SKILL.md                        # Main skill definition
└── references/
    ├── goal-schema.md              # State JSON schema
    ├── status-transitions.md       # State machine documentation
    └── interaction-guide.md        # Command reference
```

## Key Features

- **No budget limits** — runs until done or manually stopped
- **Automatic decomposition** — breaks goals into verifiable sub-tasks
- **Failure detection** — marks goal as blocked if a sub-task fails 3 times
- **Progress tracking** — percentage, completed/total tasks, current task
- **Single goal focus** — one goal at a time, no overlap confusion

## Differences from Codex `/goal`

| | Codex `/goal` | This Skill |
|---|---|---|
| Token budget | Yes, configurable | No — runs until done |
| Time budget | Yes, configurable | No — runs until done |
| SQLite storage | Yes | JSON file (`~/.claude/goal-state.json`) |
| Statuses | 6 (active/paused/blocked/usage_limited/budget_limited/complete) | 4 (active/paused/blocked/complete) |
| Native integration | Built into Codex | Claude Code skill |

## License

MIT

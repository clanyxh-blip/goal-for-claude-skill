# Goal Skill — User Command Reference

## Commands

| Command | Description |
|---------|-------------|
| `/goal <description>` | Set a new goal and start working immediately |
| `/goal set <description>` | Same as above (explicit form) |
| `/goal` | Show current goal status |
| `/goal status` | Show detailed status with progress |
| `/goal check` | Alias for status |
| `/goal pause` | Pause the active goal |
| `/goal resume` | Resume a paused or blocked goal |
| `/goal cancel` | Cancel the goal and clean up all state |

## Examples

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

## Behavior

- Only one goal can be active at a time
- Setting a new goal when one exists requires cancelling first
- The system works continuously without waiting for user input between sub-tasks
- Progress is tracked in `~/.claude/goal-state.json`
- Sub-tasks are managed via TodoWrite
- No budget limits — the goal runs until fully complete or manually stopped
- If a sub-task fails 3 times, the goal is marked as blocked and user is asked for help

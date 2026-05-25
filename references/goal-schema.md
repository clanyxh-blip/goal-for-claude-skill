# Goal State JSON Schema

File location: `~/.claude/goal-state.json`

```json
{
  "id": "goal-20260526-143000",
  "description": "Refactor the auth module to use JWT tokens",
  "status": "active",
  "created_at": "2026-05-26T14:30:00.000Z",
  "updated_at": "2026-05-26T14:45:00.000Z",
  "progress": {
    "percentage": 35,
    "tasks_total": 8,
    "tasks_completed": 3,
    "current_task": "Implement JWT middleware",
    "last_achievement": "Created user model and migration"
  },
  "history": [
    {
      "turn": 1,
      "timestamp": "2026-05-26T14:30:00.000Z",
      "action": "goal_created",
      "summary": "Decomposed into 8 sub-tasks"
    },
    {
      "turn": 3,
      "timestamp": "2026-05-26T14:35:00.000Z",
      "action": "task_completed",
      "summary": "Database schema and models created"
    }
  ],
  "notes": "User wants PostgreSQL specifically"
}
```

## Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Format: `goal-YYYYMMDD-HHMMSS`, unique per goal |
| `description` | string | Original user-provided goal text |
| `status` | enum | `active`, `paused`, `blocked`, `complete` |
| `created_at` | string | ISO 8601 timestamp when goal was created |
| `updated_at` | string | ISO 8601 timestamp of last state change |
| `progress.percentage` | number | 0-100, calculated from tasks |
| `progress.tasks_total` | number | Total number of sub-tasks |
| `progress.tasks_completed` | number | Completed sub-tasks |
| `progress.current_task` | string\|null | Description of current sub-task |
| `progress.last_achievement` | string\|null | What was last completed |
| `history` | array | Turn records, max 50 entries |
| `notes` | string | Free-form notes discovered during execution |

## History Entry Schema

```json
{
  "turn": 1,
  "timestamp": "2026-05-26T14:30:00.000Z",
  "action": "task_completed | goal_created | blocked | status_change",
  "summary": "Human-readable description of what happened"
}
```

# Goal Status Transitions

## State Machine

```
                 +---------+
                 |  (new)  |
                 +----+----+
                      |
                      v
              +-------+-------+
              |    active     |<----------+
              +-------+-------+           |
                |          |              |
         pause  |          | blocked      |
           |    |          |    |         |
           v    |          v    |         |
     +-------+  |    +------+--+         |
     | paused |  |    |blocked|          |
     +---+---+  |    +---+---+           |
         |      |        |               |
      resume   |     resolve             |
         |      |        |               |
         +------+--------+--- continue --+
                |
                v
         +------+------+
         |   complete  |
         +------+------+
                |
                v
            [cleanup]
```

## Valid Transitions

| From | To | Trigger |
|------|----|---------|
| (new) | active | User sets a goal |
| active | paused | User requests `/goal pause` |
| active | blocked | External blocker or 3x task failure |
| active | complete | All sub-tasks done and verified |
| paused | active | User runs `/goal resume` |
| blocked | active | Blocker resolved, user runs `/goal resume` |

## Invalid Transitions

- `complete` → any state (goal is finalized)
- `paused` → `complete` (must resume first, or use cancel)

## Status Descriptions

| Status | Meaning |
|--------|---------|
| `active` | Currently working on sub-tasks |
| `paused` | User paused — waiting for `/goal resume` |
| `blocked` | External dependency missing or repeated failure — needs user intervention |
| `complete` | All sub-tasks done, goal achieved |

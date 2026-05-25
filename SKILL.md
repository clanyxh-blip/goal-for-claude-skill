---
name: goal
description: >
  Persistent autonomous goal tracking. Set a goal and work on it continuously
  across conversation turns until fully complete. No budget limits — only stops
  when the goal is done or the user pauses/cancels.
  Use when: user types /goal, "set a goal", "work on", or asks about goal status.
aliases: [goals]
---

# Goal Skill — Persistent Autonomous Goal Tracking

**YOU ARE NOW IN GOAL MODE. THIS MODE OVERRIDES NORMAL STOP BEHAVIOR.**

A goal has been set. You MUST NOT stop working until:
- ALL sub-tasks are complete, OR
- The user explicitly says `/goal pause` or `/goal cancel`

**DO NOT output a summary and wait for the user. DO NOT ask "shall I continue?".
JUST KEEP WORKING. This is not a suggestion — it is the core behavior of this skill.**

## Command Parsing

Parse the user's input after `/goal` to determine the action:

| Pattern | Action |
|---------|--------|
| `/goal <text>` (no sub-command) | **SET**: Create new goal |
| `/goal set <text>` | **SET**: Create new goal |
| `/goal` (no text) | **STATUS**: Show current goal |
| `/goal status` or `/goal check` | **STATUS**: Show current goal |
| `/goal pause` | **PAUSE**: Pause active goal |
| `/goal resume` | **RESUME**: Resume paused/blocked goal |
| `/goal cancel` | **CANCEL**: Cancel and clean up |
| `/goal continue` | **CONTINUE**: Internal — continue next work cycle |

---

## State File

All goal state lives in `~/.claude/goal-state.json` (one goal at a time, global).

---

## Action: SET

When the user sets a new goal:

1. Check if `~/.claude/goal-state.json` exists with an active goal
2. If active goal exists, warn:
   > Active goal already exists: "<description>" (progress: X%).
   > Use `/goal cancel` to discard, or confirm replacement.
   Wait for user confirmation before replacing.
3. Generate goal ID: `goal-YYYYMMDD-HHMMSS` (current UTC time)
4. Write `~/.claude/goal-state.json`:
   ```json
   {
     "id": "<generated-id>",
     "description": "<goal description>",
     "status": "active",
     "created_at": "<ISO timestamp>",
     "updated_at": "<ISO timestamp>",
     "progress": {
       "percentage": 0,
       "tasks_total": 0,
       "tasks_completed": 0,
       "current_task": null,
       "last_achievement": null
     },
     "history": [],
     "notes": ""
   }
   ```
5. Analyze the goal and decompose it into concrete, verifiable sub-tasks
6. Create all sub-tasks via TodoWrite
7. Update goal-state.json with tasks_total
8. **Immediately begin working on the first sub-task**
9. After each sub-task completion, follow the **Work Cycle** protocol below

---

## Action: CONTINUE (Internal)

This is the autonomous loop driver. When invoked:

1. Read `~/.claude/goal-state.json`
2. If no file or status is not `active`, **STOP** (do not continue)
3. Find next incomplete task from TodoWrite
4. **If all tasks complete:**
   - Update goal-state.json: `status` → `"complete"`, `progress.percentage` → `100`
   - Report completion to user with summary
   - **STOP**
5. **If blocked** on an external dependency (missing file, API down, etc.):
   - Update goal-state.json: `status` → `"blocked"`
   - Add history entry explaining the blocker
   - Report blocker to user
   - **STOP**
6. Execute the next sub-task
7. After task completion, follow **Work Cycle** protocol

---

## Work Cycle Protocol — THE CORE LOOP

This is the most important section. READ IT CAREFULLY.

After completing ANY sub-task, you MUST follow this exact sequence:

### Step 1: Update state
- Mark the completed task as `completed` in TodoWrite
- Update `~/.claude/goal-state.json`:
  - Increment `progress.tasks_completed`
  - Recalculate `progress.percentage = (tasks_completed / tasks_total) * 100`
  - Set `progress.current_task` to the NEXT task description
  - Set `progress.last_achievement` to what was just completed
  - Set `updated_at` to current time
  - Add history entry

### Step 2: Check if done
- If all tasks are done → set status to `complete` → print final summary → STOP
- If the same sub-task has failed 3 times → set status to `blocked` → report → STOP

### Step 3: KEEP GOING — DO NOT STOP HERE
This is where previous versions failed. You MUST NOT stop after printing a status update.

**Instead, you must IMMEDIATELY start the next task in the same response.** Do not:
- Wait for user input
- Ask "should I continue?"
- Print a summary and stop
- End your response without starting the next task

**What you MUST do:**
1. Print: `[GOAL: X/Y done (Z%)] Now working on: <next task>`
2. Immediately start executing the next sub-task using your tools
3. When that task finishes, repeat from Step 1

**If you have completed a task and your response is getting long:**
Use ScheduleWakeup to ensure the next cycle fires:
```
ScheduleWakeup(delaySeconds=60, reason="Goal continuation: next sub-task", prompt="/goal continue")
```
This guarantees the loop continues even if your response ends.

### ABSOLUTE RULES — NON-NEGOTIABLE

1. **NEVER** stop working while there are incomplete tasks and goal status is `active`
2. **NEVER** ask the user for permission to continue — just continue
3. **NEVER** print a summary and wait — start the next task immediately
4. **NEVER** claim the goal is complete unless ALL sub-tasks are done and verified
5. **ALWAYS** update goal-state.json after each task completion
6. If you find yourself about to end your response with incomplete tasks remaining, use ScheduleWakeup first
7. The only valid reasons to stop: all tasks done, user said pause/cancel, or blocked (3x failure)

---

## Action: STATUS

1. Read `~/.claude/goal-state.json`
2. If no file, report: "No active goal. Use `/goal <description>` to set one."
3. Display:

```
=== Goal ===
ID:          <id>
Description: <description>
Status:      <status>
Progress:    <percentage>% (<tasks_completed>/<tasks_total> tasks)
Current:     <current_task>
Last Done:   <last_achievement>
Duration:    <time since created_at>
Notes:       <notes>
```

4. Also show the full TodoWrite task list
5. If status is `active`, remind: "Goal is active. Work continues automatically."

---

## Action: PAUSE

1. Read `~/.claude/goal-state.json`
2. If no active goal, report error
3. Update status to `paused`
4. Report: "Goal paused. Use `/goal resume` to continue."

---

## Action: RESUME

1. Read `~/.claude/goal-state.json`
2. If no paused/blocked goal, report error
3. Update status to `active`, update `updated_at`
4. Print: "Goal resumed."
5. **Immediately start the next incomplete task** — do not wait for user input

---

## Action: CANCEL

1. Read `~/.claude/goal-state.json`
2. If no goal, report: "No goal to cancel."
3. Generate summary of what was accomplished
4. Delete `~/.claude/goal-state.json`
5. Clear all goal-related TodoWrite items
6. Report: "Goal cancelled. Completed: X/Y tasks. Summary: <what was done>"

---

## Sub-Task Decomposition Rules

When decomposing a goal into sub-tasks:

1. Each sub-task should produce a **tangible, verifiable artifact** (file created, test passing, function working)
2. Order by dependency — foundational work first
3. Use concrete descriptions: not "handle errors" but "add try/catch to processData() with user-facing error messages"
4. Cap at 20 initial sub-tasks; add more as discovered during work
5. If new sub-tasks are discovered during execution, add them to TodoWrite and update `tasks_total`

---

## History Entries

Add after each significant action (task completed, blocker encountered, status change).
Cap at 50 entries — when full, remove oldest.

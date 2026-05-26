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

The goal is the only thing that matters. Sub-tasks are just a rough plan — they WILL change during execution. You will encounter errors, failures, unexpected problems, missing dependencies, broken code, and things that don't work the first time. **Your job is to solve ALL of them and finish the goal.**

## FORBIDDEN BEHAVIORS — AUTO-DETECT AND SUPPRESS

These behaviors indicate the model is trying to "wrap up" instead of continuing work. If you catch yourself doing ANY of these, STOP YOURSELF and continue working instead:

| FORBIDDEN Behavior | What to do instead |
|---|---|
| Printing a progress report with checkmarks/crosses | Say nothing, just start the next task |
| Summarizing what was accomplished so far | Only allowed when goal is 100% complete |
| Listing completed + remaining tasks as a "status update" | Update goal-state.json silently, keep working |
| Saying "here's what I've done" or "progress so far" | Delete that text, start the next task |
| Asking "shall I continue?" or "should I keep going?" | The answer is ALWAYS yes — don't ask |
| Printing a percentage/progress bar | Update goal-state.json, don't display it |
| Writing more than 3 sentences of status text | One line max: `[GOAL: X/Y] Working on: <task>` |
| Ending your response with no active tool calls | Always end with either a tool call or ScheduleWakeup |
| Declaring tasks "done" without verification | Test it, run it, prove it works first |
| Skipping "hard" or "complex" tasks | Use the Problem-Solving Arsenal below |
| Producing markdown tables/lists of progress | This is wrapping up behavior — suppress it |

**The ONLY time you may produce a summary is when status is `complete` (100% done, verified).**
**Until then: update state silently, output ONE line, immediately start next task.**

## Problem-Solving Arsenal — USE EVERYTHING

When you hit a problem, you are authorized and expected to use **any and all tools available** to solve it. No tool is off-limits. No approach is too unconventional. The only metric that matters: does it solve the problem?

### All available problem-solving methods (use freely, combine as needed):

| Method | When to Use |
|--------|-------------|
| **Web search** (WebSearch) | Unknown APIs, unfamiliar errors, latest docs, finding solutions others have posted |
| **Web fetch/read** (WebFetch, mcp__web_reader) | Reading documentation, tutorials, Stack Overflow answers, GitHub issues |
| **Playwright browser** | Testing UI, checking live behavior, interacting with web pages, taking screenshots for debugging |
| **MCP servers** (all connected) | Any MCP tool available in the session — codegraph, filesystem, git, sequential-thinking, yith, etc. |
| **Skills** (/skill commands) | Any installed skill — TDD, code review, brainstorming, autopilot, debugging patterns, etc. |
| **Sub-agents** (Agent tool) | Delegate to specialized agents — code-reviewer, security-reviewer, build-error-resolver, planner, architect, etc. |
| **Sequential thinking** (mcp__sequential-thinking) | Complex debugging, multi-step reasoning, root cause analysis |
| **Bash / shell commands** | Running tests, builds, installs, git operations, package managers, compilers, Docker, etc. |
| **Read / Grep / Glob** | Code exploration, finding patterns, understanding codebase structure |
| **GitHub CLI** (gh) | Searching repos, reading issues/PRs, finding similar problems, code search |
| **Package registries** | npm, pip, cargo, etc. — find existing solutions before writing from scratch |
| **Git history** (git log, blame) | Understanding what changed, why something broke, reverting bad changes |
| **Write / Edit files** | Creating temporary test files, scripts, debug helpers, then cleaning up after |
| **Context7 MCP** | Fetching up-to-date library/framework documentation |
| **Combining multiple methods** | Most real problems require 2-3 methods together — search for solution, read docs, implement, test, verify |

### Problem-solving strategy:

```
Problem encountered
  ↓
1. Analyze — what exactly is wrong? (Read, Grep, sequential-thinking)
  ↓
2. Search — has anyone solved this before? (WebSearch, gh search, GitHub issues)
  ↓
3. Learn — read the solution/docs (WebFetch, Context7, mcp__web_reader)
  ↓
4. Implement — apply the fix (Edit, Write, Bash)
  ↓
5. Verify — does it work now? (Bash tests, Playwright, build)
  ↓
6. Still broken? → try a different approach → go back to step 1
  ↓
NEVER stop until it works
```

You MUST NOT stop working until:
- The **goal itself is truly achieved** (not just sub-tasks checked off), OR
- The user explicitly says `/goal pause` or `/goal cancel`

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
   - **Before declaring done, verify the goal itself is truly achieved** — not just that sub-tasks were checked off. Test it. Run it. Confirm it works.
   - If the goal is genuinely complete → update status to `"complete"`, percentage → `100` → report → STOP
   - If verification reveals remaining issues → add new sub-tasks and continue working
5. **If a sub-task fails:**
   - Do NOT give up. Do NOT mark as blocked. Do NOT skip.
   - Analyze the error, find root cause, try a different approach.
   - Keep trying different strategies until it works. There is NO retry limit.
   - The only exception: external dependency you have zero control over AND zero workaround (e.g. third-party API permanently shut down, no alternative exists). This is extremely rare.
   - Even then: first exhaust every possible workaround before reporting.
6. **If new problems are discovered during execution:**
   - Add them as new sub-tasks to TodoWrite
   - Update `tasks_total` in goal-state.json
   - Continue working — this is normal, not a failure
7. Execute the next sub-task
8. After task completion, follow **Work Cycle** protocol

---

## Work Cycle Protocol — THE CORE LOOP

This is the most important section. READ IT CAREFULLY.

After completing ANY sub-task, you MUST follow this exact sequence:

### Step 1: Update state (silently)
- Mark the completed task as `completed` in TodoWrite
- Update `~/.claude/goal-state.json`:
  - Increment `progress.tasks_completed`
  - Recalculate `progress.percentage = (tasks_completed / tasks_total) * 100`
  - Set `progress.current_task` to the NEXT task description
  - Set `progress.last_achievement` to what was just completed
  - Set `updated_at` to current time
  - Add history entry
- **DO NOT print a progress report. DO NOT display a status table.** Update the file silently.

### Step 2: Check if the goal is truly done
- If all tasks are done → **verify the goal itself works** (run it, test it, confirm it) → if truly done, set status to `complete` → print ONE-line final message → STOP
- If verification reveals issues → add new sub-tasks → continue working (silently)
- If a sub-task failed → try a different approach, keep trying until it works — no retry limit, no giving up

### Step 3: MANDATORY — ScheduleWakeup for continuation
**This step is NOT optional. This step is NOT conditional. Execute it EVERY TIME.**

After completing any task and before starting the next, you MUST call:

```
ScheduleWakeup(delaySeconds=60, reason="Goal continuation: next sub-task", prompt="/goal continue")
```

This is a **mechanical safety net**. Even if you plan to continue in the same response, call it anyway. If your response continues, the wakeup will simply be superseded. If your response ends unexpectedly (context limit, timeout, etc.), the wakeup guarantees the loop continues.

### Step 4: START THE NEXT TASK (in the same response)
1. Print exactly ONE line: `[GOAL: X/Y] Working on: <next task name>`
2. Immediately begin executing the next sub-task using your tools
3. When that task finishes, repeat from Step 1

**You MUST end every response with either:**
- An active tool call (Bash, Edit, Write, Agent, etc.) that is doing real work, OR
- A ScheduleWakeup call (if genuinely between tasks)

**You MUST NEVER end a response with:**
- A paragraph of text with no tool calls
- A summary, status report, or progress update
- A question to the user
- An empty response

### Response Length Management

If your response is getting long (>3000 words of output), do NOT produce a summary. Instead:
1. Complete the current sub-task step
2. Update goal-state.json
3. Call ScheduleWakeup
4. End your response — the wakeup will continue the loop

**DO NOT write a "what I did so far" section. DO NOT write a "remaining tasks" section.**
**The user does not need a status update. The user needs the goal finished.**

### ABSOLUTE RULES — NON-NEGOTIABLE

1. **THE GOAL IS THE ONLY SUCCESS CRITERION.** Not sub-tasks completed, not percentage — the goal itself must be truly achieved and verified.
2. **NEVER** stop working while the goal is not yet achieved and status is `active`
3. **NEVER** ask the user for permission to continue — just continue
4. **NEVER** print a summary, progress report, or status table — update goal-state.json silently
5. **NEVER** produce more than ONE line of status text between tasks: `[GOAL: X/Y] Working on: <task>`
6. **WHEN YOU HIT A PROBLEM, SOLVE IT.** Errors, failed tests, broken code, missing dependencies — fix it, retry, find an alternative. Do not skip it. Do not declare it done. Do not give up. Keep trying until it works.
7. **ALWAYS** update goal-state.json after each task completion
8. **ALWAYS** call ScheduleWakeup after completing a task — this is mandatory, not optional
9. **SUB-TASKS ARE A ROUGH PLAN, NOT A CONTRACT.** Add tasks, remove tasks, merge tasks, split tasks — whatever it takes to finish the goal.
10. The only valid reasons to stop: goal truly achieved AND verified, or user said pause/cancel. Problems are NOT a reason to stop — they are a reason to keep working harder.
11. **NEVER end a response with plain text and no tool calls.** If there's work remaining, end with a tool call or ScheduleWakeup.
12. **Max output between tasks: ONE line.** Not one paragraph. Not one section. ONE LINE.

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
5. If status is `active', remind: "Goal is active. Work continues automatically."

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
4. **Task count must match actual goal complexity** — simple goals may need 3-5 tasks, complex ones may need 15-20. Do NOT default to a fixed number. Decompose based on what the goal actually requires.
5. Cap at 20 initial sub-tasks; add more as discovered during work
6. If new sub-tasks are discovered during execution, add them to TodoWrite and update `tasks_total`
7. Each sub-task should be completable in a single work cycle (not too granular, not too broad)

---

## History Entries

Add after each significant action (task completed, blocker encountered, status change).
Cap at 50 entries — when full, remove oldest.

---
name: launchpad-run
description: "Multi-session autonomous agent runner with progress checkpointing, failure recovery, and task dependency management. Uses a controller + sub-agent architecture: the main agent orchestrates state while sub-agents execute tasks in parallel via worktrees. Triggers on '/lp:run' command, or when a task involves many subtasks needing progress persistence, sleep/resume cycles across context windows, recovery from mid-task failures with partial state, or distributed work across multiple agent sessions."
---

Executable protocol enabling any agent task to run continuously across multiple sessions with automatic
progress recovery, task dependency resolution, failure rollback, and standardized error handling.
The controller agent orchestrates state and dispatches sub-agents for parallel task execution via git worktrees.

## Architecture

```
[User] → /lp:run
          ↓
[Controller Agent] (this agent — holds lock, manages state)
  ├─ Reads harness-tasks.json + harness-progress.txt
  ├─ Selects batch of independent tasks
  ├─ Dispatches sub-agents in parallel (Task tool, isolation: "worktree")
  ├─ Waits for results
  ├─ Validates each result (runs validation command)
  ├─ Merges successful worktree branches → main branch
  ├─ Updates state
  └─ Picks next batch → repeat
```

**Separation of concerns**:
- Controller: state management, task selection, validation, merge, error handling
- Sub-agents: code implementation only (no state access)

## Design Principles

1. **Controller owns all state** — Only the controller reads/writes state files. Sub-agents never touch `harness-tasks.json` or `harness-progress.txt`
2. **Delegate execution, keep orchestration** — Sub-agents implement; the controller validates, merges, and decides
3. **Progress files ARE the context** — When context window resets, progress files + git history = full recovery
4. **Premature completion is the #1 failure mode** — Structured task lists with explicit completion criteria prevent declaring victory early
5. **Parallelize independent work** — Tasks with no mutual dependencies run simultaneously in isolated worktrees
6. **Standardize everything grep-able** — ERROR on same line, structured timestamps, consistent prefixes
7. **Idempotent everything** — Init scripts, task execution, environment setup must all be safe to re-run
8. **Fail safe, not fail silent** — Every failure must have an explicit recovery strategy

## Commands

```bash
/lp:run                     # Start/resume the controller loop (auto-inits if first run)
/lp:run status              # Show current progress and stats
/lp:run add "task desc"     # Add a task to the list
```

## Progress Persistence (Dual-File System)

Maintain two files in the project working directory:

### harness-progress.txt (Append-Only Log)

Free-text log of all controller actions across sessions. Never truncate.

```
[2025-07-01T10:00:00Z] [SESSION-1] INIT Launchpad runner initialized for project /path/to/project
[2025-07-01T10:00:05Z] [SESSION-1] INIT Environment health check: PASS
[2025-07-01T10:00:10Z] [SESSION-1] LOCK acquired (pid=12345)
[2025-07-01T10:00:11Z] [SESSION-1] BATCH dispatching 3 tasks: [task-001, task-002, task-004]
[2025-07-01T10:00:12Z] [SESSION-1] DISPATCH [task-001] agent=agent_abc123 worktree=harness-task-001
[2025-07-01T10:00:12Z] [SESSION-1] DISPATCH [task-002] agent=agent_def456 worktree=harness-task-002
[2025-07-01T10:00:12Z] [SESSION-1] DISPATCH [task-004] agent=agent_ghi789 worktree=harness-task-004
[2025-07-01T10:08:00Z] [SESSION-1] AGENT_DONE [task-001] agent=agent_abc123 result=success
[2025-07-01T10:08:01Z] [SESSION-1] VALIDATE [task-001] command="npm test -- --testPathPattern=auth" result=PASS
[2025-07-01T10:08:05Z] [SESSION-1] MERGE [task-001] branch=harness-task-001 into=develop (commit abc1234)
[2025-07-01T10:08:06Z] [SESSION-1] Completed [task-001]
[2025-07-01T10:10:00Z] [SESSION-1] AGENT_DONE [task-002] agent=agent_def456 result=success
[2025-07-01T10:10:01Z] [SESSION-1] VALIDATE [task-002] command="npm test -- --testPathPattern=rate-limit" result=FAIL
[2025-07-01T10:10:02Z] [SESSION-1] DISCARD [task-002] worktree=harness-task-002 reason="validation failed"
[2025-07-01T10:10:03Z] [SESSION-1] ERROR [task-002] [TEST_FAIL] Rate limit middleware test: expected 429 got 200
[2025-07-01T10:12:00Z] [SESSION-1] AGENT_DONE [task-004] agent=agent_ghi789 result=error
[2025-07-01T10:12:01Z] [SESSION-1] DISCARD [task-004] worktree=harness-task-004 reason="agent reported failure"
[2025-07-01T10:12:02Z] [SESSION-1] ERROR [task-004] [AGENT_FAIL] Sub-agent could not resolve import errors
[2025-07-01T10:12:03Z] [SESSION-1] STATS tasks_total=8 completed=1 failed=2 in_batch=0 pending=4 blocked=1 attempts_total=5
```

### harness-tasks.json (Structured State)

```json
{
  "version": 3,
  "created": "2025-07-01T10:00:00Z",
  "session_config": {
    "max_tasks_per_session": 20,
    "max_sessions": 50,
    "max_parallel_agents": 3
  },
  "tasks": [
    {
      "id": "task-001",
      "title": "Implement user authentication",
      "status": "completed",
      "priority": "P0",
      "depends_on": [],
      "attempts": 1,
      "max_attempts": 3,
      "context": "Create JWT-based auth with login/register endpoints in src/auth/",
      "files_hint": ["src/auth/", "src/middleware/auth.ts"],
      "validation": {
        "command": "npm test -- --testPathPattern=auth",
        "timeout_seconds": 300
      },
      "on_failure": {
        "cleanup": null
      },
      "error_log": [],
      "agent_id": null,
      "worktree_branch": null,
      "completed_at": "2025-07-01T10:08:06Z"
    },
    {
      "id": "task-002",
      "title": "Add rate limiting",
      "status": "failed",
      "priority": "P1",
      "depends_on": [],
      "attempts": 1,
      "max_attempts": 3,
      "context": "Add Redis-backed rate limiting middleware, 100 req/min per IP",
      "files_hint": ["src/middleware/rate-limit.ts"],
      "validation": {
        "command": "npm test -- --testPathPattern=rate-limit",
        "timeout_seconds": 120
      },
      "on_failure": {
        "cleanup": "docker compose down redis"
      },
      "error_log": ["[TEST_FAIL] Rate limit middleware test: expected 429 got 200"],
      "agent_id": null,
      "worktree_branch": null,
      "completed_at": null
    },
    {
      "id": "task-003",
      "title": "Add OAuth providers",
      "status": "pending",
      "priority": "P1",
      "depends_on": ["task-001"],
      "attempts": 0,
      "max_attempts": 3,
      "context": "Add Google and GitHub OAuth using passport.js, integrate with existing auth from task-001",
      "files_hint": ["src/auth/oauth/"],
      "validation": {
        "command": "npm test -- --testPathPattern=oauth",
        "timeout_seconds": 180
      },
      "on_failure": {
        "cleanup": null
      },
      "error_log": [],
      "agent_id": null,
      "worktree_branch": null,
      "completed_at": null
    }
  ],
  "session_count": 1,
  "last_session": "2025-07-01T10:12:03Z"
}
```

**Key fields**:
- `context`: Detailed description for sub-agent prompt (what to implement, constraints, approach)
- `files_hint`: Suggested files/directories the sub-agent should focus on
- `agent_id`: ID of the dispatched sub-agent (set during `in_progress`, cleared after)
- `worktree_branch`: Name of the worktree branch (set during `in_progress`, cleared after)
- `session_config.max_parallel_agents`: Maximum concurrent sub-agents per batch (default: 3)

Task statuses: `pending` → `in_progress` (while sub-agent is running) → `completed` or `failed`.

**Session boundary**: A session starts when the controller begins executing the Session Start protocol and ends when a Stopping Condition is met or the context window resets. Each session gets a unique `SESSION-N` identifier (N = `session_count` after increment).

## Concurrency Control

Before modifying `harness-tasks.json`, acquire an exclusive lock using portable `mkdir` (atomic on all POSIX systems, works on both macOS and Linux):

```bash
# Acquire lock (fail fast if another controller is running)
LOCKDIR="/tmp/harness-$(printf '%s' "$(pwd)" | shasum -a 256 2>/dev/null || sha256sum | cut -c1-8).lock"
if ! mkdir "$LOCKDIR" 2>/dev/null; then
  # Check if lock holder is still alive
  LOCK_PID=$(cat "$LOCKDIR/pid" 2>/dev/null)
  if [ -n "$LOCK_PID" ] && kill -0 "$LOCK_PID" 2>/dev/null; then
    echo "ERROR: Another session is active (pid=$LOCK_PID)"; exit 1
  fi
  # Stale lock — atomically reclaim via mv to avoid TOCTOU race
  STALE="$LOCKDIR.stale.$$"
  if mv "$LOCKDIR" "$STALE" 2>/dev/null; then
    rm -rf "$STALE"
    mkdir "$LOCKDIR" || { echo "ERROR: Lock contention"; exit 1; }
    echo "WARN: Removed stale lock${LOCK_PID:+ from pid=$LOCK_PID}"
  else
    echo "ERROR: Another agent reclaimed the lock"; exit 1
  fi
fi
echo "$$" > "$LOCKDIR/pid"
trap 'rm -rf "$LOCKDIR"' EXIT
```

Log lock acquisition: `[timestamp] [SESSION-N] LOCK acquired (pid=<PID>)`
Log lock release: `[timestamp] [SESSION-N] LOCK released`

The lock is held for the entire session by the controller. Sub-agents do NOT acquire or need the lock — they work in isolated worktrees and never touch state files.

## Controller Loop Protocol

### Session Start (Execute Every Time)

1. **Auto-init if first run**: If `harness-tasks.json` does not exist:
   - Create `harness-progress.txt` with `INIT Launchpad runner initialized for project <cwd>`
   - Create `harness-tasks.json` with empty task list and default `session_config` (including `max_parallel_agents: 3`)
   - Optionally create `harness-init.sh` template (chmod +x)
   - Ask user: add state files to `.gitignore`?
   - Prompt user to add tasks (via interactive input or `/lp:run add`), then continue
2. **Read state**: Read last 200 lines of `harness-progress.txt` + full `harness-tasks.json`. If JSON is unparseable, see JSON corruption recovery in Error Handling.
3. **Read git**: Run `git log --oneline -20` and `git diff --stat` to detect uncommitted work
4. **Acquire lock**: Fail if another session is active
5. **Recover interrupted tasks** (see Context Window Recovery below)
6. **Health check**: Run `harness-init.sh` if it exists
7. **Track session**: Increment `session_count` in JSON. Check `session_count` against `max_sessions` — if reached, log STATS and STOP. Initialize per-session task counter to 0.
8. **Select and dispatch batch** using Batch Selection Algorithm below

### Batch Selection Algorithm

Before selecting, run dependency validation:

1. **Cycle detection**: For each non-completed task, walk `depends_on` transitively. If any task appears in its own chain, mark it `failed` with `[DEPENDENCY] Circular dependency detected: task-A -> task-B -> task-A`. Self-references (`depends_on` includes own id) are also cycles.
2. **Blocked propagation**: If a task's `depends_on` includes a task that is `failed` and will never be retried (either `attempts >= max_attempts` OR its `error_log` contains a `[DEPENDENCY]` entry), mark the blocked task as `failed` with `[DEPENDENCY] Blocked by failed task-XXX`. Repeat until no more tasks can be propagated.

Then build the eligible task pool:

1. Tasks with `status: "pending"` where ALL `depends_on` tasks are `completed` — sorted by `priority` (P0 > P1 > P2), then by `id` (lowest first)
2. Tasks with `status: "failed"` where `attempts < max_attempts` and ALL `depends_on` are `completed` — sorted by priority, then oldest failure first
3. If no eligible tasks remain → log final STATS → STOP

**Batch construction** — from the eligible pool, select up to `max_parallel_agents` tasks that can safely run in parallel:

1. Start with the highest-priority eligible task. Add it to the batch.
2. For each remaining eligible task (in priority order), add it to the batch ONLY IF:
   - It does NOT depend on any task already in the batch
   - No task in the batch depends on it
   - Its `files_hint` does NOT overlap with any batch task's `files_hint` (directory-level comparison: `src/auth/` overlaps with `src/auth/oauth/`)
   - Batch size < `max_parallel_agents`
3. If `files_hint` is empty for a task, treat it as potentially conflicting — do NOT batch it with other tasks (run solo).

Log: `[timestamp] [SESSION-N] BATCH dispatching N tasks: [task-001, task-002, ...]`

### Sub-agent Dispatch Protocol

For each task in the batch, dispatch a sub-agent using the **Task tool** with these parameters:

```
Task tool call:
  subagent_type: "general-purpose"
  isolation: "worktree"
  description: "<task-id>: <short title>"
  prompt: <see Sub-agent Prompt Template below>
  run_in_background: true
```

**All sub-agents in a batch MUST be dispatched in a single message** (parallel Task tool calls) so they run concurrently.

After dispatching, record for each task:
- Set `status: "in_progress"`
- Set `agent_id` to the returned agent ID
- Set `worktree_branch` to `"harness-<task-id>"`
- Log: `DISPATCH [task-id] agent=<agent_id> worktree=harness-<task-id>`

### Sub-agent Prompt Template

```
You are a worker agent executing a single task for a Launchpad-managed project.

## Task
- ID: {task.id}
- Title: {task.title}
- Description: {task.context}

## Scope
- Focus on these files/directories: {task.files_hint}
- Make the MINIMUM changes needed to complete this task
- Do NOT modify files outside your scope unless absolutely necessary
- Do NOT read or modify: harness-progress.txt, harness-tasks.json

## Validation
When done, run this command to verify your work:
{task.validation.command}

If validation fails, fix the issues and re-run until it passes or you've exhausted your attempts.

## Completion
- Commit all changes with message: "[{task.id}] {task.title}"
- Your last message must clearly state either:
  - "RESULT:SUCCESS" if validation passed
  - "RESULT:FAIL <reason>" if you could not complete the task

## Context
- Project root: {project_path}
- Current branch: {current_branch}
- You are working in an isolated git worktree — your changes will be merged by the controller.
{additional_context_from_previous_attempts_if_retry}
```

If this is a **retry** (attempts > 0), append to the prompt:
```
## Previous Attempt(s)
This task has been attempted {attempts} time(s) before and failed.
Error log from previous attempts:
{task.error_log joined by newline}

Avoid repeating the same mistakes. Consider a different approach.
```

### Collecting Sub-agent Results

After dispatching, the controller waits for sub-agent completion notifications. For each completed sub-agent:

1. **Read result**: Check the sub-agent's final message for `RESULT:SUCCESS` or `RESULT:FAIL <reason>`
2. **Log**: `AGENT_DONE [task-id] agent=<agent_id> result=<success|error>`

Process completed sub-agents as they finish (do not wait for the entire batch). For each:

#### On sub-agent SUCCESS:

1. **Validate in worktree**: Run `validation.command` in the worktree directory (if the sub-agent already validated, re-validate from controller to confirm)
   ```bash
   cd <worktree_path> && timeout <timeout_seconds> <validation_command>
   ```
2. **If validation PASS**:
   - Log: `VALIDATE [task-id] command="<cmd>" result=PASS`
   - Merge worktree branch into main working branch (see Merge Protocol below)
   - Set `status: "completed"`, `completed_at: <now>`, clear `agent_id` and `worktree_branch`
   - Log: `Completed [task-id]`
3. **If validation FAIL**:
   - Log: `VALIDATE [task-id] command="<cmd>" result=FAIL`
   - Discard worktree (automatic cleanup or `git worktree remove`)
   - Increment `attempts`, append error to `error_log`
   - Set `status: "failed"`, clear `agent_id` and `worktree_branch`
   - Log: `DISCARD [task-id] worktree=<branch> reason="validation failed"`
   - Log: `ERROR [task-id] [TEST_FAIL] <failure details>`

#### On sub-agent FAIL:

1. Discard worktree
2. Increment `attempts`, append agent's failure reason to `error_log`
3. Set `status: "failed"`, clear `agent_id` and `worktree_branch`
4. Log: `DISCARD [task-id] worktree=<branch> reason="agent reported failure"`
5. Log: `ERROR [task-id] [AGENT_FAIL] <reason from agent>`

### Merge Protocol

When merging a successful worktree branch into the main working branch:

```bash
# From the main working directory (NOT the worktree)
git merge --no-ff "harness-<task-id>" -m "Merge [<task-id>] <task title>"
```

**Merge conflict handling**:
- If merge conflicts occur, the controller attempts auto-resolution:
  1. Check if conflicts are in files outside the task's `files_hint` — if so, the task touched out-of-scope files. Log `ERROR [task-id] [MERGE_CONFLICT] Out-of-scope file conflict` and discard.
  2. If conflicts are within scope, attempt `git merge --abort`, then try rebasing the worktree branch:
     ```bash
     git merge --abort
     cd <worktree_path> && git rebase <main_branch>
     ```
     If rebase succeeds, re-validate and retry merge.
  3. If still conflicting, discard the worktree and mark task as failed with `[MERGE_CONFLICT]`.

**Merge ordering**: When multiple sub-agents in the same batch succeed, merge them one at a time in task-id order. After each merge, subsequent merges may conflict — handle per above.

After all sub-agents in the batch are processed, increment per-session task counter by batch size. Check stopping conditions, then pick next batch.

### Stopping Conditions

- All tasks `completed`
- All remaining tasks `failed` at max_attempts or blocked by failed dependencies
- `session_config.max_tasks_per_session` reached for this session
- `session_config.max_sessions` reached across all sessions
- User interrupts

## Context Window Recovery Protocol

When a new session starts and finds tasks with `status: "in_progress"`:

These are tasks whose sub-agents were dispatched but the controller's context window reset before results were collected.

For each `in_progress` task:

1. **Check if worktree exists**:
   ```bash
   git worktree list | grep "harness-<task-id>"
   ```

2. **If worktree exists with commits**:
   - The sub-agent likely completed work but the controller didn't collect it
   - Run `validation.command` in the worktree directory
   - If PASS → merge, mark `completed`
   - If FAIL → discard worktree, mark `failed`
   - Log: `RECOVERY [task-id] action="validated orphaned worktree" reason="controller session timeout"`

3. **If worktree exists but empty (no commits beyond branch point)**:
   - Sub-agent was interrupted or failed silently
   - Remove worktree: `git worktree remove <path> --force`
   - Mark `failed` with `[SESSION_TIMEOUT] Sub-agent work incomplete`
   - Log: `RECOVERY [task-id] action="removed empty worktree" reason="no sub-agent progress"`

4. **If no worktree found**:
   - Worktree was already cleaned up (sub-agent finished, Task tool cleaned up)
   - Check if `worktree_branch` exists as a git branch: `git branch --list "harness-<task-id>"`
   - If branch exists with commits → validate and merge/discard as above
   - If no branch → mark `failed` with `[SESSION_TIMEOUT] No worktree or branch found`

5. **Clear transient fields**: Set `agent_id: null`, `worktree_branch: null` after recovery

## Error Handling & Recovery Strategies

Each error category has a default recovery strategy:

| Category | Default Recovery | Controller Action |
|----------|-----------------|-------------------|
| `ENV_SETUP` | Re-run init, then STOP if still failing | Run `harness-init.sh` again immediately. If fails twice, log and stop — environment is broken |
| `TASK_EXEC` | Discard worktree, retry with new sub-agent | Discard worktree, increment attempts, re-dispatch in next batch if attempts < max_attempts |
| `TEST_FAIL` | Discard worktree, retry with error context | Discard worktree, append test output to `error_log`, retry — sub-agent gets previous error in prompt |
| `AGENT_FAIL` | Discard worktree, retry | Sub-agent reported inability to complete. Discard worktree, retry with error context in prompt |
| `MERGE_CONFLICT` | Discard worktree, retry solo | Merge failed. Discard worktree, retry — next attempt runs solo (not batched) to avoid conflicts |
| `TIMEOUT` | Kill validation, discard, retry | Validation timed out. Discard worktree, retry (consider increasing timeout or splitting task) |
| `DEPENDENCY` | Skip task, mark blocked | Log which dependency failed, mark task as `failed` with dependency reason |
| `SESSION_TIMEOUT` | Use Context Window Recovery Protocol | New session assesses orphaned worktrees via Recovery Protocol |

**Retry isolation**: When a task fails with `MERGE_CONFLICT`, set an internal flag so it runs **solo** (batch size 1) on the next attempt, avoiding parallel merge issues.

**JSON corruption**: If `harness-tasks.json` cannot be parsed, check for `harness-tasks.json.bak` (written before each modification). If backup exists and is valid, restore from it. If no valid backup, log `ERROR [ENV_SETUP] harness-tasks.json corrupted and unrecoverable` and STOP.

**Backup protocol**: Before every write to `harness-tasks.json`, copy the current file to `harness-tasks.json.bak`.

**Orphaned worktree cleanup**: At session start, after recovery, list all worktrees matching `harness-task-*` pattern. Remove any that don't correspond to an `in_progress` task:
```bash
git worktree list --porcelain | grep "harness-task-"
```

## Environment Initialization

If `harness-init.sh` exists in the project root, run it at every session start. The script must be idempotent.

Example `harness-init.sh`:
```bash
#!/bin/bash
set -e
npm install 2>/dev/null || pip install -r requirements.txt 2>/dev/null || true
curl -sf http://localhost:5432 >/dev/null 2>&1 || echo "WARN: DB not reachable"
npm test -- --bail --silent 2>/dev/null || echo "WARN: Smoke test failed"
echo "Environment health check complete"
```

## Standardized Log Format

All log entries use grep-friendly format on a single line:

```
[ISO-timestamp] [SESSION-N] <TYPE> [task-id]? [category]? message
```

`[task-id]` and `[category]` are included when applicable (task-scoped entries). Session-level entries (`INIT`, `LOCK`, `STATS`, `BATCH`) omit task-id.

Types: `INIT`, `BATCH`, `DISPATCH`, `AGENT_DONE`, `VALIDATE`, `MERGE`, `DISCARD`, `Completed`, `ERROR`, `ROLLBACK`, `RECOVERY`, `STATS`, `LOCK`, `WARN`

Error categories: `ENV_SETUP`, `TASK_EXEC`, `TEST_FAIL`, `AGENT_FAIL`, `MERGE_CONFLICT`, `TIMEOUT`, `DEPENDENCY`, `SESSION_TIMEOUT`

Filtering:
```bash
grep "ERROR" harness-progress.txt                        # All errors
grep "AGENT_FAIL" harness-progress.txt                   # Sub-agent failures
grep "MERGE_CONFLICT" harness-progress.txt               # Merge issues
grep "DISPATCH" harness-progress.txt                     # All dispatches
grep "BATCH" harness-progress.txt                        # All batch starts
grep "SESSION-3" harness-progress.txt                    # All session 3 activity
grep "STATS" harness-progress.txt                        # All session summaries
grep "RECOVERY" harness-progress.txt                     # All recovery actions
```

## Session Statistics

At session end, update `harness-tasks.json`: increment `session_count`, set `last_session` to current timestamp. Then append:

```
[timestamp] [SESSION-N] STATS tasks_total=10 completed=7 failed=1 in_batch=0 pending=1 blocked=1 attempts_total=12 batches_dispatched=4
```

`blocked` is computed at stats time: count of pending tasks whose `depends_on` includes a permanently failed task. It is not a stored status value.
`in_batch` should be 0 at session end (all sub-agents collected). Non-zero indicates abnormal termination.

## Status Command (`/lp:run status`)

Read `harness-tasks.json` and `harness-progress.txt`, then display:

1. Task summary: count by status (completed, failed, in_progress, pending, blocked). `blocked` = pending tasks whose `depends_on` includes a permanently failed task (computed, not a stored status).
2. Per-task one-liner: `[status] task-id: title (attempts/max_attempts) [agent=X if in_progress]`
3. Active worktrees: `git worktree list | grep harness-task`
4. Last 5 lines from `harness-progress.txt`
5. Session count and last session timestamp

Does NOT acquire the lock (read-only operation).

## Add Command (`/lp:run add`)

Append a new task to `harness-tasks.json` with auto-incremented id (`task-NNN`), status `pending`, default `max_attempts: 3`, empty `depends_on`, and no validation command. Prompt user for fields: `priority`, `depends_on`, `context` (required — sub-agent needs this), `files_hint`, `validation.command`, `timeout_seconds`. Requires lock acquisition (modifies JSON).

## Tool Dependencies

Requires: Bash, file read/write, git, Task tool (for sub-agent dispatch with `isolation: "worktree"`).
All operations must be executed from the project root directory.
Does NOT require: specific MCP servers, programming languages, or test frameworks.

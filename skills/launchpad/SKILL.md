---
name: launchpad
description: >-
  End-to-end development pipeline orchestrator. Guides the user through spec writing,
  task breakdown, parallelization planning, and multi-agent execution — ensuring no phase
  is skipped. Invokes launchpad-spec, launchpad-plan, and launchpad-run skills in sequence.
  Triggers on "/launchpad", "/launchpad <feature description>", "build this feature",
  "I want to build", or when the user wants to go from idea to implementation
  with structured planning.
---

# Launchpad

End-to-end orchestrator that chains **launchpad-spec → launchpad-plan → launchpad-run** into a single guided workflow. Ensures every phase completes before the next begins, and asks the user the right questions at every step.

## Commands

```bash
/launchpad <feature description>   # Start full pipeline from scratch
/launchpad status                  # Show current pipeline stage and progress
/launchpad resume                  # Resume from where you left off
```

## Pipeline Stages

```
 STAGE 1: SPEC          STAGE 2: PLAN           STAGE 3: RUN
 ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
 │ /lp:spec     │       │ Dependency   │       │ /lp:run      │
 │      ↓       │       │ analysis     │       │      ↓       │
 │ /lp:refine   │──────→│      ↓       │──────→│ Dispatch     │
 │      ↓       │       │ Wave assign  │       │ sub-agents   │
 │ /lp:clarify  │       │      ↓       │       │      ↓       │
 │      ↓       │       │ harness-     │       │ Validate &   │
 │ /lp:tasks    │       │ tasks.json   │       │ merge        │
 └──────────────┘       └──────────────┘       └──────────────┘
   User answers            Automatic             Autonomous
   questions here          (with review)          (with recovery)
```

## Stage 1: Spec (uses launchpad-spec skill)

### Entry

When the user triggers `/launchpad <description>`:

1. **Announce stage**: "Stage 1/3: Spec — Defining requirements clearly."
2. **Invoke `/lp:spec`** with the user's description
3. **Track state**: Write `.launchpad.json` to project root (see State File below)

### Iteration Loop

After `/lp:spec` creates the spec with questions:

1. **Present questions** to the user
2. Wait for answers → **invoke `/lp:clarify`** with user's response
3. If more questions arise → repeat
4. If spec needs research → suggest and invoke `/lp:refine`
5. **Gate check**: Only proceed when "Open Questions" section is empty

### Gate 1 → 2

When all questions are resolved:

1. **Invoke `/lp:tasks`** to generate the task breakdown
2. **Present task list** to the user
3. **Ask**: "Please review the task list. Let me know if anything is missing or needs changes. If it looks good, we'll proceed to the next stage."
4. If user requests changes → edit task file → re-present
5. If user approves → proceed to Stage 2

---

## Stage 2: Plan (uses launchpad-plan skill)

### Entry

1. **Announce stage**: "Stage 2/3: Plan — Analyzing task dependencies and generating a parallel execution plan."
2. **Invoke `/lp:plan`** on the task file from Stage 1

### User Review Points

The plan skill produces the parallelization plan. Present to the user:

1. **Wave summary**: "Total N tasks organized into M waves."
2. **Wave detail table**:
   ```
   Wave 0 (parallel): task-001, task-002, task-004
   Wave 1 (parallel): task-003, task-005
   Wave 2 (sequential): task-006
   ```
3. **Conflict notes**: Any file overlap warnings
4. **Ask**: "Please review the execution plan. Let me know if you'd like to adjust the wave assignments or dependencies."

### Gate 2 → 3

When user approves the plan:

1. Verify `harness-tasks.json` has been written
2. Verify all tasks have `context` (sub-agent prompt content)
3. Verify all tasks have `files_hint` (batch conflict detection)
4. Verify all tasks have `validation.command` — if any are missing, **ask the user**:
   "Task task-003 'Add OAuth providers' is missing a validation command. What command should be used? (e.g., `npm test -- --testPathPattern=oauth`)"
5. **Ask**: "Everything is ready. Shall I start execution?"

---

## Stage 3: Run (uses launchpad-run skill)

### Entry

1. **Announce stage**: "Stage 3/3: Run — Sub-agents are executing tasks in parallel."
2. **Invoke `/lp:run`**

### During Execution

The run skill handles autonomous execution. The pipeline skill adds:

1. **Batch progress reports**: After each batch completes, summarize:
   ```
   Batch 1 complete: task-001 ✓, task-002 ✓, task-004 ✗ (TEST_FAIL)
   Next batch: task-003, task-005
   ```
2. **Failure triage**: When a task fails, present the error and ask:
   "task-004 failed: [error details]. Should I auto-retry, or would you like to modify the task?"
   - Retry → let run skill retry automatically
   - Modify → edit task context/validation, then re-dispatch

### Completion

When all tasks are done:

1. **Final summary**:
   ```
   Pipeline complete!
   - Spec: specs/feature-name.md (COMPLETED)
   - Tasks: 8/8 completed, 0 failed
   - Sessions: 2, Batches: 4
   - Total sub-agent dispatches: 10 (2 retries)
   ```
2. **Suggest next steps**:
   - "Let me know if you'd like a code review."
   - "Shall I create a PR?"

---

## State File: `.launchpad.json`

Tracks which stage the pipeline is in, for session resume:

```json
{
  "feature": "auth-system",
  "stage": "spec",
  "spec_file": "specs/auth-system.md",
  "task_file": "specs/auth-system.tasks.md",
  "harness_file": "harness-tasks.json",
  "stage_history": [
    { "stage": "spec", "status": "completed", "timestamp": "2025-07-01T10:00:00Z" },
    { "stage": "plan", "status": "in_progress", "timestamp": "2025-07-01T10:30:00Z" }
  ]
}
```

---

## `/launchpad status` Command

Read `.launchpad.json` and report:

```
Launchpad: auth-system
├─ Stage 1 (Spec):  ✓ completed — specs/auth-system.md
├─ Stage 2 (Plan):  ● in progress — 5 tasks, 3 waves
└─ Stage 3 (Run):   ○ pending

Use /launchpad resume to continue.
```

---

## `/launchpad resume` Command

1. Read `.launchpad.json`
2. Determine current stage
3. Resume from the appropriate point:
   - **Stage 1**: Re-read spec, check for open questions, continue iteration
   - **Stage 2**: Re-read task file, re-run plan if needed
   - **Stage 3**: Invoke `/lp:run` (run skill has its own session recovery)

---

## Error Handling

| Situation | Action |
|-----------|--------|
| User says `/launchpad` without description | Ask: "What feature would you like to build?" |
| Spec has unresolved questions at gate | Block: "There are still N open questions. Please answer them with `/lp:clarify`." |
| Task has no validation command | Ask user for the command before proceeding |
| Task has no `context` | Generate from spec + task title, present for user review |
| Task has no `files_hint` | Ask: "Which files will this task modify?" |
| Run fails completely | Present error, offer to re-run plan with adjusted plan |
| Context window approaching limit | Save state to `.launchpad.json`, suggest `/launchpad resume` in new session |

---

## Key Rules

1. **Never skip a stage.** Even if the user says "just run it", walk through all three stages.
2. **Always get user confirmation at gates.** The two gate checkpoints (spec→plan, plan→run) require explicit user approval.
3. **Fill gaps by asking, not guessing.** If any required field is missing (validation command, files_hint, context), ask the user rather than making assumptions.
4. **One feature per pipeline.** If the user wants to build multiple features, run separate pipelines (or suggest splitting into one spec with multiple tasks).
5. **State survives sessions.** Always read/write `.launchpad.json` so the pipeline can resume after context reset.

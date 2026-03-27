---
name: launchpad-plan
description: "Analyze a spec's task list and produce a parallelization-aware execution plan. Reads task sections, analyzes dependency chains and file mutation overlaps, determines parallel vs sequential execution, and assigns wave groups. Triggers on /lp:plan."
---

# Launchpad Plan

Analyze a task breakdown and produce a parallelization-aware implementation plan.

## Purpose

Bridge between the spec's task list and the runner's parallel execution. The planner determines **what can run concurrently** and **what must be sequential** by analyzing dependency declarations and file mutation overlaps.

## Workflow

### 1. Locate the Spec

Resolve the spec name from the user's request. Read `specs/<spec-name>.tasks.md` (or equivalent task breakdown file).

If missing or ambiguous, list `specs/` and ask.

### 2. Parse Task Sections

Extract all task sections. For each task, collect:

- **Task ID**: Identifier (e.g., `task-001`, `T-0`, etc.)
- **Files**: Each file path and its operation (**create** vs **modify**)
- **Steps**: The implementation steps
- **Tests**: The test/validation scenarios
- **Dependencies**: Explicit from the spec, plus implicit from section ordering

### 3. Build the Dependency Graph

For each task, determine what it depends on:

1. **Explicit dependencies**: From the spec's task definitions (e.g., `depends_on` fields).
2. **File mutation conflicts**: Two tasks that both **modify** the same file cannot safely run in parallel. If task A appears before task B and both modify `src/index.ts`, B depends on A.
3. **Shared types/foundation dependency**: If a foundational task exists (e.g., types, schema, config), all dependent tasks must wait for it.
4. **Create-then-modify**: If task A **creates** a file and task B **modifies** or imports from it, B depends on A.

### 4. Determine Task Granularity

Each task section becomes one task by default. Adjust when:

- **Split** a task if it contains clearly independent substeps touching different files with no data flow between them, AND splitting would enable more parallelism. Don't split just for granularity's sake.
- **Merge** tasks that are trivially small into a parent task or mark as no-op.

### 5. Assign Parallel Waves

Group tasks into execution waves — sets of tasks that can run concurrently:

- **Wave 0**: Tasks with no dependencies (foundational setup, independent features)
- **Wave 1**: Tasks whose dependencies are all in wave 0
- **Wave N**: Tasks whose dependencies are all in waves < N

Tasks in the same wave run in parallel. Waves execute sequentially.

### 6. Output Format

#### For launchpad-run integration (default)

When the output is intended for `/lp:run`, produce `harness-tasks.json` compatible format:

```json
{
  "version": 3,
  "created": "<ISO timestamp>",
  "session_config": {
    "max_tasks_per_session": 20,
    "max_sessions": 50,
    "max_parallel_agents": 3
  },
  "tasks": [
    {
      "id": "task-001",
      "title": "Short task name",
      "status": "pending",
      "priority": "P0",
      "depends_on": [],
      "attempts": 0,
      "max_attempts": 3,
      "context": "Detailed description with enough context for a sub-agent to execute without reading the spec. Include: files to touch, implementation steps, constraints, expected behavior.",
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
      "completed_at": null
    }
  ],
  "session_count": 0,
  "last_session": null
}
```

**Field mapping from spec to runner**:
- Task title → `title`
- Task description + steps + test scenarios → `context` (must be self-contained for sub-agent)
- File list → `files_hint` (used for batch conflict detection)
- Test command → `validation.command`
- Dependencies → `depends_on` (task IDs)
- Wave 0 tasks → `priority: "P0"`, Wave 1 → `priority: "P1"`, Wave 2+ → `priority: "P2"`

#### For generic use

When not targeting launchpad-run, produce `implementation.json`:

```json
[
  {
    "id": "T-0",
    "name": "Short task name",
    "depends": [],
    "wave": 0,
    "kind": "code",
    "files": {
      "create": ["src/auth/service.ts"],
      "modify": ["src/index.ts"]
    },
    "description": "Detailed description with enough context for an agent to execute."
  }
]
```

### 7. Confirm with User

After writing the output, present:
- Total task count and wave count
- Which tasks run in parallel vs sequentially and why
- Any granularity decisions made (splits, merges)
- Offer to adjust groupings or wave assignments

## Guidelines

- Each task's `context`/`description` must contain enough detail that an agent can execute it without reading the original spec. The orchestrator feeds these as agent prompts.
- Preserve the spec's step details and test scenarios — don't summarize away actionable information.
- Flag potential conflict risks in task descriptions (e.g., "Modifies `index.ts` — coordinate with task-003 if running in same wave").
- When `files_hint` is empty for a task, the runner will run it solo (not batched) to avoid unknown conflicts.

## Integration with Launchpad

The typical workflow:

```
/lp:spec → /lp:refine → /lp:clarify → /lp:tasks
    ↓
/lp:plan  (reads specs/*.tasks.md → writes harness-tasks.json)
    ↓
/lp:run  (reads harness-tasks.json → dispatches sub-agents)
```

The planner's job is to transform human-readable task breakdowns into machine-executable task plans with parallelization awareness.

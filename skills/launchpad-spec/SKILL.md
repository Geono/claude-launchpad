---
name: launchpad-spec
description: >-
  Spec-driven development framework with iterative refinement. Orchestrates feature
  development from intent to implementation via structured specs and task breakdown.
  Triggers on "/lp:spec", "/lp:refine", "/lp:clarify", "/lp:tasks", "/lp:run-task",
  "I want to build", "I want to add", "create spec", "spec-driven", "feature spec".
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, TodoWrite, Task
---

# Launchpad Spec

Iterative feature development framework ensuring zero ambiguity before execution.

## Quick Reference

| Command | Purpose | Input |
|---------|---------|-------|
| `/lp:spec <intent>` | Create spec from "I want to build/add X" | Feature description |
| `/lp:refine [section]` | Improve spec with research | Optional section focus |
| `/lp:clarify <response>` | Answer clarification questions | Your response |
| `/lp:tasks` | Break spec into executable tasks | None (uses active spec) |
| `/lp:run-task [task#]` | Execute tasks with TDD | Optional task number |

## Core Principle

**Iterate until clarity**: No task execution begins until ALL questions are resolved and the spec is unambiguous. Claude must be able to execute without interruptions.

---

## Phase 1: `/lp:spec` - Create Specification

**Trigger**: `/lp:spec <description>` or "I want to build/add X"

### Workflow

1. **Check `specs/` folder**:
   - If missing: Create `specs/` and `specs/README.md`
   - If exists: Read `specs/README.md` for project overrides

2. **Detect project context**:
   - Scan repo for language indicators (go.mod, pyproject.toml, package.json, etc.)
   - Note primary language(s) for later skill invocation
   - Check `specs/README.md` for language overrides

3. **Generate spec file**:
   - Filename: `specs/{feature-slug}.md` (kebab-case)

4. **Fill initial sections**:
   - Parse user intent into Objective
   - List initial requirements (functional/non-functional)
   - Mark status as `DRAFT`

5. **Generate clarifying questions**:
   - Identify ambiguities, edge cases, unknowns
   - List as numbered questions in "Open Questions" section
   - **STOP and present questions to user**

### Output

```
Created: specs/feature-name.md (DRAFT)

Questions requiring clarification:
1. [Question about scope]
2. [Question about behavior]
3. [Question about constraints]

Use `/lp:clarify` to answer, or `/lp:refine` to research solutions.
```

---

## Phase 2: `/lp:refine` - Research & Improve

**Trigger**: `/lp:refine [section]` (e.g., `/lp:refine solution`, `/lp:refine requirements`)

### Workflow

1. **Load active spec**: Find most recent DRAFT spec in `specs/`

2. **Check project conventions**:
   - Read `specs/README.md` for behavior overrides
   - Load relevant language skill (auto-detected or overridden)

3. **Research phase**:
   - Search codebase for similar patterns
   - Check skill references for best practices

4. **Update spec**:
   - Fill "Technical Strategy" with concrete approach
   - Add architecture decisions with rationale
   - Update requirements based on findings

5. **Re-evaluate clarity**:
   - Are there new questions?
   - Are existing questions resolved?
   - **If questions remain: STOP and present them**

---

## Phase 3: `/lp:clarify` - Answer Questions

**Trigger**: `/lp:clarify <response>` or `/lp:clarify Q1: answer, Q2: answer`

### Workflow

1. **Load active spec** with open questions

2. **Parse user response**:
   - Match answers to numbered questions
   - Accept free-form responses for single questions

3. **Update spec**:
   - Move answered questions to relevant sections
   - Add decisions/constraints to Requirements or Strategy
   - Remove resolved questions from "Open Questions"

4. **Check for new questions**:
   - Does the answer introduce new ambiguities?
   - **If questions remain: present them**
   - **If no questions: announce spec is ready for `/lp:tasks`**

### Example

```
User: /lp:clarify Q1: We need OAuth2 with Google provider only. Q2: No, admin can also delete.

Updated specs/auth-system.md:
- Added OAuth2/Google to Technical Strategy
- Updated permissions: admin can delete

Remaining questions: None
Spec is ready. Use `/lp:tasks` to create task breakdown.
```

---

## Phase 4: `/lp:tasks` - Task Breakdown

**Trigger**: `/lp:tasks`

### Prerequisites

- Active spec must have status `DRAFT` or `APPROVED`
- "Open Questions" section must be empty
- If questions exist: **STOP and redirect to `/lp:clarify`**

### Workflow

1. **Validate spec readiness**:
   ```
   If open_questions > 0:
       ERROR: Spec has unresolved questions. Use /lp:clarify first.
   ```

2. **Mark spec as APPROVED**

3. **Generate task file**: `specs/{feature-slug}.tasks.md`

4. **Break down by component**:
   - Group tasks by logical component/module
   - Each task = one logical unit (not TDD-granular)
   - TDD practice enforced during `/lp:run-task`, not here

5. **Add task metadata**:
   - Link back to spec
   - Context summary
   - Acceptance criteria per task

6. **Final review**:
   - Present task list to user
   - Ask: "Any tasks missing or need splitting?"

### Task Granularity

Tasks should be **high-level logical units**:
- "Implement authentication middleware"
- "Create user model and repository"
- "Add API endpoints for user CRUD"

TDD cycle (Red-Green-Refactor) happens WITHIN each task during `/lp:run-task`.

---

## Phase 5: `/lp:run-task` - Execute Tasks

**Trigger**: `/lp:run-task [task#]` (e.g., `/lp:run-task`, `/lp:run-task 3`)

### Prerequisites

- Task file must exist: `specs/{feature}.tasks.md`
- If no task file: **STOP and redirect to `/lp:tasks`**

### Workflow

1. **Load task file** and find next unchecked task (or specified task#)

2. **Load context**:
   - Read linked spec for requirements
   - Read `specs/README.md` for project overrides

3. **Execute with TDD**:
   - **RED**: Write failing test first
   - **GREEN**: Minimal code to pass
   - **REFACTOR**: Clean up
   - **COMMIT**: After each phase

4. **Update task file**:
   - Mark task as `[x]` complete
   - Add notes if needed

5. **Continue or pause**:
   - If more tasks: Ask "Continue to next task?"
   - If blocked: Document blocker, ask for input
   - If all done: Mark spec as `COMPLETED`

### Execution Rules

- **No interruptions**: If questions arise during execution, the spec was not ready
- **Respect hooks**: Pre-commit hooks must pass before marking complete

---

## Project Configuration: `specs/README.md`

Override default behaviors per-project:

```markdown
# Spec Configuration

## Language Override
Primary: golang
Secondary: python

## Conventions
- All specs require security section
- Tasks must include rollback plan
- Use feature branches: feature/{spec-name}

## Templates
Use custom templates from: ./templates/

## Auto-invoke
- Always run /trivy before marking complete
```

---

## State Management

### Spec Status Flow

```
DRAFT -> APPROVED -> IN_PROGRESS -> COMPLETED
          |              |
          v              v
       (questions?)   (blocked?)
          |              |
          v              v
        DRAFT      IN_PROGRESS
```

### File Structure

```
project/
└── specs/
    ├── README.md           # Project overrides
    ├── auth-system.md      # Spec (APPROVED)
    ├── auth-system.tasks.md # Task breakdown
    ├── user-dashboard.md   # Spec (DRAFT)
    └── ...
```

---

## Session Resume

On context compaction or session resume:

1. Check `specs/` for files with status `IN_PROGRESS`
2. Check `.tasks.md` files for unchecked items
3. Report: "Found in-progress spec: X with Y tasks remaining"
4. Ask: "Continue with `/lp:run-task`?"

---

## Error Handling

| Situation | Response |
|-----------|----------|
| `/lp:tasks` with open questions | "Spec has N unresolved questions. Use `/lp:clarify` first." |
| `/lp:run-task` without task file | "No task file found. Use `/lp:tasks` first." |
| `/lp:clarify` without active spec | "No active spec. Use `/lp:spec` to create one." |
| Ambiguity during `/lp:run-task` | "Execution blocked: [issue]. Spec needs refinement. Use `/lp:refine`." |

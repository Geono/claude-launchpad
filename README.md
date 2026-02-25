# Launchpad

### Spec. Plan. Run — with sub-agents.

> Imagine a team of engineers working in shifts. Each new engineer arrives with no memory of the previous shift, no written spec to follow, and no idea which tasks can run at the same time.
>
> That's what building features with AI agents feels like today.

An end-to-end development pipeline for Claude Code. Describe a feature, and Launchpad handles the rest: spec writing, task planning, and parallel multi-agent execution.

## The problem

Anthropic's article on [effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) nailed the continuity problem — agents lose context between sessions. But context recovery is only one piece. In practice, I kept running into two other gaps:

**No spec, no clarity.** Agents start coding before the requirements are clear. Edge cases get missed. You end up course-correcting for an hour because the agent guessed wrong about something you never specified.

**No orchestration.** Even with a good spec and task breakdown, you're manually feeding tasks one by one. You don't know which ones can run in parallel. You're the glue, and glue is tedious.

Launchpad closes all three gaps. It forces spec clarity *before* any code gets written, analyzes task dependencies to find what can run in parallel, then dispatches sub-agents to do the actual work. You review at each checkpoint.

## How it works

Three stages, each gated by your approval:

```bash
/launchpad "add user authentication with OAuth"
```

**Stage 1: Spec** — Launchpad asks you clarifying questions until there's zero ambiguity. No hand-waving. If there's an open question, execution is blocked.

**Stage 2: Plan** — Tasks get analyzed for dependencies and file conflicts. Independent tasks are grouped into parallel execution waves. You review the plan before anything runs.

**Stage 3: Run** — Sub-agents are dispatched in parallel via git worktrees. Each agent works in isolation, validates its own output, and the controller merges results back. Failed tasks get retried with error context from previous attempts.

The whole thing survives session resets. State files track where you are, so `/launchpad resume` picks up exactly where you left off.

## Installation

### As a plugin

```bash
/plugin marketplace add Geono/claude-launchpad
/plugin install claude-launchpad
```

### Manual

Copy the `skills/` folder to your Claude Code skills directory:

```bash
cp -r skills/* ~/.claude/skills/
```

## Commands

### Main pipeline

| Command | What it does |
|---------|-------------|
| `/launchpad <description>` | Start a new pipeline |
| `/launchpad status` | See current stage and progress |
| `/launchpad resume` | Pick up where you left off |

### Individual stages

You can also run each stage independently:

| Command | What it does |
|---------|-------------|
| `/lp:spec <description>` | Create a spec with clarifying questions |
| `/lp:refine [section]` | Research and improve the spec |
| `/lp:clarify <answers>` | Answer open questions |
| `/lp:tasks` | Break spec into executable tasks |
| `/lp:plan` | Analyze dependencies, assign parallel waves |
| `/lp:run` | Start multi-agent execution |
| `/lp:run status` | Check execution progress |

## Example walkthrough

Here's what a real session looks like. You type one command, and Launchpad walks you through the rest.

### Start the pipeline

```
You: /launchpad "add a notification system with email and in-app channels"
```

### Stage 1: Spec

Launchpad creates a draft spec and immediately asks clarifying questions:

```
Stage 1/3: Spec — Defining requirements clearly.

Created: specs/notification-system.md (DRAFT)

Questions requiring clarification:
1. Should notifications be real-time (WebSocket) or polling-based?
2. Which email provider — SES, SendGrid, or something else?
3. Do users need per-channel opt-out (e.g., email off, in-app on)?
4. Should notifications be persisted? If so, for how long?
5. Are there notification types with different priorities (e.g., urgent vs. digest)?

Use /lp:clarify to answer.
```

You answer:

```
You: /lp:clarify Q1: WebSocket for in-app, email is async.
     Q2: SendGrid. Q3: Yes, per-channel opt-out.
     Q4: Persist for 90 days. Q5: Two tiers — immediate and daily digest.
```

Launchpad updates the spec, finds no remaining questions:

```
Updated specs/notification-system.md:
- Added WebSocket strategy for in-app channel
- Added SendGrid integration to Technical Strategy
- Added per-channel preference model
- Added 90-day retention policy
- Added immediate/digest priority tiers

Remaining questions: None
Spec is ready. Generating task breakdown...
```

### Gate 1 → 2: Task breakdown

```
Generated: specs/notification-system.tasks.md

Tasks:
1. Create notification model and repository
2. Implement notification preference service
3. Add SendGrid email channel adapter
4. Add WebSocket in-app channel adapter
5. Build notification dispatcher (routes by channel + priority)
6. Add digest aggregation cron job
7. Add REST API endpoints for notification CRUD
8. Add REST API endpoints for preference management

Please review the task list. Let me know if anything is missing
or needs changes. If it looks good, we'll proceed to the next stage.
```

```
You: Looks good. Proceed.
```

### Stage 2: Plan

```
Stage 2/3: Plan — Analyzing task dependencies and generating
a parallel execution plan.

Total 8 tasks organized into 4 waves.

Wave 0 (parallel): task-001, task-002
  → No dependencies. Model + preferences can be built simultaneously.

Wave 1 (parallel): task-003, task-004, task-007, task-008
  → All depend on task-001 (model). No file overlaps between them.

Wave 2 (sequential): task-005
  → Depends on task-003 + task-004 (both channel adapters).
  → Modifies dispatcher — runs solo to avoid merge conflicts.

Wave 3 (sequential): task-006
  → Depends on task-005 (dispatcher must exist for digest routing).

⚠ File overlap: task-007 and task-008 both modify src/app.module.ts
  → Assigned to same wave but will run sequentially within the batch.

Please review the execution plan. Let me know if you'd like to adjust
the wave assignments or dependencies.
```

```
You: Ship it.
```

### Gate 2 → 3: Pre-flight check

```
All tasks have context, files_hint, and validation commands.
Everything is ready. Shall I start execution?
```

```
You: Yes.
```

### Stage 3: Run

```
Stage 3/3: Run — Sub-agents are executing tasks in parallel.

[SESSION-1] BATCH dispatching 2 tasks: [task-001, task-002]
[SESSION-1] DISPATCH [task-001] agent=a1b2c3 worktree=harness-task-001
[SESSION-1] DISPATCH [task-002] agent=d4e5f6 worktree=harness-task-002
```

Sub-agents work in isolated worktrees. As they finish:

```
Batch 1 complete: task-001 ✓, task-002 ✓

[SESSION-1] BATCH dispatching 4 tasks: [task-003, task-004, task-007, task-008]
[SESSION-1] DISPATCH [task-003] agent=g7h8i9 worktree=harness-task-003
[SESSION-1] DISPATCH [task-004] agent=j0k1l2 worktree=harness-task-004
[SESSION-1] DISPATCH [task-007] agent=m3n4o5 worktree=harness-task-007
[SESSION-1] DISPATCH [task-008] agent=p6q7r8 worktree=harness-task-008
```

If something fails, Launchpad retries with error context:

```
Batch 2 complete: task-003 ✓, task-004 ✓, task-007 ✓, task-008 ✗ (TEST_FAIL)

task-008 failed: preference endpoint returns 500 — missing dependency injection
  for NotificationPreferenceService.
Auto-retrying with error context...

[SESSION-1] DISPATCH [task-008] agent=s9t0u1 worktree=harness-task-008 (retry 1/3)
```

After all waves complete:

```
Pipeline complete!
- Spec: specs/notification-system.md (COMPLETED)
- Tasks: 8/8 completed, 0 failed
- Sessions: 1, Batches: 5
- Total sub-agent dispatches: 10 (1 retry)

Let me know if you'd like a code review.
Shall I create a PR?
```

## What's inside

Launchpad is four skills that chain together:

- **launchpad** — The orchestrator. Manages stage transitions and gate checkpoints.
- **launchpad-spec** — Iterative spec writing. Keeps asking questions until ambiguity hits zero.
- **launchpad-plan** — Dependency analysis. Builds a DAG, detects file conflicts, assigns parallel waves.
- **launchpad-run** — The execution engine. Dispatches sub-agents in isolated worktrees, validates results, merges branches, retries failures.

## Where this came from

Two things led to building this.

I was using spec-writing and task-organizing skills separately and kept thinking: why am I manually connecting these? The handoff between "spec is done" and "now organize tasks" and "now execute" was always me copy-pasting context and hoping nothing fell through the cracks. Each skill worked fine on its own, but the glue between them was... me.

Then Anthropic published [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). The article made something click. Agents need structured checkpointing, explicit task state, and recovery protocols to work reliably over long sessions.

But the article's harness is a **single-agent, single-session-at-a-time** design. One agent wakes up, reads `claude-progress.txt`, does some work, saves state, and sleeps. No spec phase. No dependency analysis. No parallel execution.

Launchpad extends that foundation in three directions:

| | Anthropic's harness | Launchpad |
|---|---|---|
| Agents | Single agent per session | Controller + parallel sub-agents |
| Execution | Sequential, one task at a time | Parallel via isolated git worktrees |
| Spec phase | None (assumes you know what to build) | Iterative Q&A until zero ambiguity |
| Task planning | None (flat task list) | Dependency graph, file conflict detection, wave assignment |
| Session recovery | Progress file + git history | Same, plus orphaned worktree recovery and sub-agent state cleanup |

The original article solved continuity. Launchpad adds clarity (spec), structure (plan), and speed (parallel sub-agents).

So the idea was simple: what if the entire journey from "I want to build X" to "here's the working code" could be one continuous, structured pipeline?

## Design decisions

**Specs before code.** If Claude has to ask you a question mid-execution, the spec wasn't ready. Launchpad blocks execution until all open questions are resolved. This feels slow at first. It saves time every time.

**Gates require human approval.** The two transitions (spec to plan, plan to run) always pause for your review. You can inspect the task breakdown, adjust wave assignments, or change validation commands before anything executes.

**Sub-agents are disposable.** Each agent works in a git worktree. If validation fails, the worktree gets discarded and the task retries with error context. No partial state leaks into your main branch.

**Progress files are the source of truth.** `harness-tasks.json` and `harness-progress.txt` survive context window resets. A new session reads these files and picks up exactly where the previous one stopped. There's no magic — just append-only logs and structured JSON.

## Requirements

- Claude Code with Task tool support (for sub-agent dispatch with worktree isolation)
- Git
- No specific language, framework, or test runner required

## License

MIT

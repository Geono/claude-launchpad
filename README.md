# Launchpad

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

## What's inside

Launchpad is four skills that chain together:

- **launchpad** — The orchestrator. Manages stage transitions and gate checkpoints.
- **launchpad-spec** — Iterative spec writing. Keeps asking questions until ambiguity hits zero.
- **launchpad-plan** — Dependency analysis. Builds a DAG, detects file conflicts, assigns parallel waves.
- **launchpad-run** — The execution engine. Dispatches sub-agents in isolated worktrees, validates results, merges branches, retries failures.

## Where this came from

Two things led to building this.

I was using spec-writing and task-organizing skills separately and kept thinking: why am I manually connecting these? The handoff between "spec is done" and "now organize tasks" and "now execute" was always me copy-pasting context and hoping nothing fell through the cracks. Each skill worked fine on its own, but the glue between them was... me.

Then Anthropic published [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). The article made something click. Agents need structured checkpointing, explicit task state, and recovery protocols to work reliably over long sessions. Sub-agents in isolated worktrees. Merge-back protocols. Context window recovery. It all made sense as the missing execution layer.

So Launchpad connects the pieces: write a proper spec (no guessing), plan the work (with real dependency analysis), then hand it off to a fleet of sub-agents that can fail, retry, and recover without losing progress. The idea that got me was simple — what if the entire journey from "I want to build X" to "here's the working code" could be one continuous, structured pipeline?

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

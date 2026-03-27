# Launchpad

> 한글 설명은 맨 아래에 있습니다.

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
# Step 1: Add the marketplace
/plugin marketplace add Geono/claude-launchpad

# Step 2: Install the plugin
/plugin install claude-launchpad@claude-launchpad
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

---

# Launchpad (한국어)

### Spec. Plan. Run — 서브 에이전트와 함께.

> 엔지니어 팀이 교대로 일한다고 상상해보세요. 새로 온 엔지니어는 이전 교대의 기억도 없고, 따라갈 스펙도 없고, 어떤 작업을 동시에 실행할 수 있는지도 모릅니다.
>
> 오늘날 AI 에이전트로 기능을 만드는 건 딱 이런 느낌입니다.

Claude Code를 위한 엔드투엔드 개발 파이프라인입니다. 기능을 설명하면, Launchpad가 나머지를 처리합니다: 스펙 작성, 태스크 플래닝, 병렬 멀티 에이전트 실행.

## 문제

Anthropic의 [효과적인 하네스](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) 아티클은 연속성 문제를 정확히 짚었습니다 — 에이전트는 세션 간에 컨텍스트를 잃습니다. 하지만 컨텍스트 복구는 퍼즐의 한 조각일 뿐입니다. 실제로 사용하면서 두 가지 다른 빈틈을 계속 마주쳤습니다:

**스펙 없이 시작하면 혼란.** 에이전트가 요구사항이 명확해지기 전에 코딩을 시작합니다. 엣지 케이스를 놓칩니다. 명시하지 않은 부분을 에이전트가 잘못 추측해서 한 시간 동안 궤도를 수정하게 됩니다.

**오케스트레이션 부재.** 좋은 스펙과 태스크 분해가 있어도, 태스크를 하나씩 수동으로 넣어줘야 합니다. 어떤 것들이 병렬로 실행 가능한지 모릅니다. 당신이 접착제 역할이고, 접착제는 지루합니다.

Launchpad는 세 가지 빈틈을 모두 메웁니다. 코드가 작성되기 *전에* 스펙의 명확성을 강제하고, 태스크 의존성을 분석해서 병렬 실행 가능한 것을 찾고, 서브 에이전트를 디스패치해서 실제 작업을 수행합니다. 각 체크포인트에서 당신이 검토합니다.

## 작동 방식

세 단계로 구성되며, 각 단계는 승인 게이트가 있습니다:

```bash
/launchpad "OAuth로 사용자 인증 추가"
```

**Stage 1: Spec** — Launchpad가 모호함이 제로가 될 때까지 명확화 질문을 합니다. 대충 넘어가지 않습니다. 열린 질문이 있으면 실행이 차단됩니다.

**Stage 2: Plan** — 태스크의 의존성과 파일 충돌을 분석합니다. 독립적인 태스크는 병렬 실행 웨이브로 그룹핑됩니다. 실행 전에 플랜을 검토합니다.

**Stage 3: Run** — 서브 에이전트가 git worktree를 통해 병렬로 디스패치됩니다. 각 에이전트는 격리된 환경에서 작업하고, 자체적으로 결과를 검증하며, 컨트롤러가 결과를 머지합니다. 실패한 태스크는 이전 시도의 에러 컨텍스트와 함께 재시도됩니다.

세션이 리셋되어도 살아남습니다. 상태 파일이 현재 위치를 추적하므로, `/launchpad resume`으로 정확히 멈춘 곳에서 재개합니다.

## 설치

### 플러그인으로 설치

```bash
# Step 1: 마켓플레이스 추가
/plugin marketplace add Geono/claude-launchpad

# Step 2: 플러그인 설치
/plugin install claude-launchpad@claude-launchpad
```

### 수동 설치

`skills/` 폴더를 Claude Code 스킬 디렉토리에 복사합니다:

```bash
cp -r skills/* ~/.claude/skills/
```

## 명령어

### 메인 파이프라인

| 명령어 | 설명 |
|---------|------|
| `/launchpad <설명>` | 새 파이프라인 시작 |
| `/launchpad status` | 현재 단계와 진행 상황 확인 |
| `/launchpad resume` | 멈춘 곳에서 재개 |

### 개별 단계

각 단계를 독립적으로 실행할 수도 있습니다:

| 명령어 | 설명 |
|---------|------|
| `/lp:spec <설명>` | 명확화 질문과 함께 스펙 작성 |
| `/lp:refine [섹션]` | 스펙 조사 및 개선 |
| `/lp:clarify <답변>` | 열린 질문에 답변 |
| `/lp:tasks` | 스펙을 실행 가능한 태스크로 분해 |
| `/lp:plan` | 의존성 분석, 병렬 웨이브 할당 |
| `/lp:run` | 멀티 에이전트 실행 시작 |
| `/lp:run status` | 실행 진행 상황 확인 |

## 예제 워크스루

실제 세션이 어떻게 진행되는지 보여드립니다. 하나의 명령어를 입력하면, Launchpad가 나머지를 안내합니다.

### 파이프라인 시작

```
You: /launchpad "이메일과 인앱 채널이 있는 알림 시스템 추가"
```

### Stage 1: Spec

Launchpad가 초안 스펙을 작성하고 즉시 명확화 질문을 합니다:

```
Stage 1/3: Spec — 요구사항을 명확하게 정의합니다.

Created: specs/notification-system.md (DRAFT)

명확화가 필요한 질문:
1. 알림은 실시간(WebSocket)이어야 하나요, 폴링 기반이어야 하나요?
2. 이메일 제공자는 SES, SendGrid, 아니면 다른 것?
3. 사용자가 채널별 수신 거부가 필요한가요? (예: 이메일 끄고, 인앱은 켜기)
4. 알림을 저장해야 하나요? 그렇다면 얼마나 오래?
5. 우선순위가 다른 알림 유형이 있나요? (예: 긴급 vs. 다이제스트)

/lp:clarify로 답변하세요.
```

답변합니다:

```
You: /lp:clarify Q1: 인앱은 WebSocket, 이메일은 비동기.
     Q2: SendGrid. Q3: 네, 채널별 수신 거부.
     Q4: 90일 보관. Q5: 두 단계 — 즉시 발송과 일일 다이제스트.
```

Launchpad가 스펙을 업데이트하고, 남은 질문이 없음을 확인합니다:

```
Updated specs/notification-system.md:
- 인앱 채널에 WebSocket 전략 추가
- SendGrid 연동을 기술 전략에 추가
- 채널별 선호도 모델 추가
- 90일 보관 정책 추가
- 즉시/다이제스트 우선순위 단계 추가

남은 질문: 없음
스펙이 준비되었습니다. 태스크 분해를 생성합니다...
```

### Gate 1 → 2: 태스크 분해

```
Generated: specs/notification-system.tasks.md

Tasks:
1. 알림 모델 및 리포지토리 생성
2. 알림 선호도 서비스 구현
3. SendGrid 이메일 채널 어댑터 추가
4. WebSocket 인앱 채널 어댑터 추가
5. 알림 디스패처 구축 (채널 + 우선순위별 라우팅)
6. 다이제스트 집계 크론 작업 추가
7. 알림 CRUD REST API 엔드포인트 추가
8. 선호도 관리 REST API 엔드포인트 추가

태스크 목록을 검토해주세요. 빠진 것이나 변경이
필요한 부분이 있으면 알려주세요. 좋으면 다음 단계로 진행합니다.
```

```
You: 좋아요. 진행하세요.
```

### Stage 2: Plan

```
Stage 2/3: Plan — 태스크 의존성을 분석하고 병렬 실행 플랜을 생성합니다.

총 8개 태스크가 4개 웨이브로 구성되었습니다.

Wave 0 (병렬): task-001, task-002
  → 의존성 없음. 모델 + 선호도를 동시에 구축 가능.

Wave 1 (병렬): task-003, task-004, task-007, task-008
  → 모두 task-001 (모델)에 의존. 서로 간 파일 겹침 없음.

Wave 2 (순차): task-005
  → task-003 + task-004 (두 채널 어댑터)에 의존.
  → 디스패처를 수정하므로 머지 충돌 방지를 위해 단독 실행.

Wave 3 (순차): task-006
  → task-005 (다이제스트 라우팅을 위해 디스패처가 존재해야 함)에 의존.

⚠ 파일 겹침: task-007과 task-008이 둘 다 src/app.module.ts를 수정
  → 같은 웨이브에 배치하되 배치 내에서 순차 실행.

실행 플랜을 검토해주세요. 웨이브 배치나 의존성을
조정하고 싶으시면 알려주세요.
```

```
You: 실행해.
```

### Gate 2 → 3: 사전 점검

```
모든 태스크에 컨텍스트, files_hint, 검증 명령어가 있습니다.
모두 준비되었습니다. 실행을 시작할까요?
```

```
You: 네.
```

### Stage 3: Run

```
Stage 3/3: Run — 서브 에이전트가 태스크를 병렬로 실행합니다.

[SESSION-1] BATCH dispatching 2 tasks: [task-001, task-002]
[SESSION-1] DISPATCH [task-001] agent=a1b2c3 worktree=harness-task-001
[SESSION-1] DISPATCH [task-002] agent=d4e5f6 worktree=harness-task-002
```

서브 에이전트가 격리된 worktree에서 작업합니다. 완료되면:

```
Batch 1 complete: task-001 ✓, task-002 ✓

[SESSION-1] BATCH dispatching 4 tasks: [task-003, task-004, task-007, task-008]
[SESSION-1] DISPATCH [task-003] agent=g7h8i9 worktree=harness-task-003
[SESSION-1] DISPATCH [task-004] agent=j0k1l2 worktree=harness-task-004
[SESSION-1] DISPATCH [task-007] agent=m3n4o5 worktree=harness-task-007
[SESSION-1] DISPATCH [task-008] agent=p6q7r8 worktree=harness-task-008
```

실패하면, Launchpad가 에러 컨텍스트와 함께 재시도합니다:

```
Batch 2 complete: task-003 ✓, task-004 ✓, task-007 ✓, task-008 ✗ (TEST_FAIL)

task-008 실패: preference 엔드포인트가 500 반환 — NotificationPreferenceService
  의존성 주입 누락.
에러 컨텍스트와 함께 자동 재시도...

[SESSION-1] DISPATCH [task-008] agent=s9t0u1 worktree=harness-task-008 (retry 1/3)
```

모든 웨이브가 완료되면:

```
파이프라인 완료!
- Spec: specs/notification-system.md (COMPLETED)
- Tasks: 8/8 완료, 0 실패
- Sessions: 1, Batches: 5
- 총 서브 에이전트 디스패치: 10 (1 재시도)

코드 리뷰를 원하시면 알려주세요.
PR을 생성할까요?
```

## 구성 요소

Launchpad는 체이닝되는 네 개의 스킬로 구성됩니다:

- **launchpad** — 오케스트레이터. 단계 전환과 게이트 체크포인트를 관리합니다.
- **launchpad-spec** — 반복적 스펙 작성. 모호함이 제로가 될 때까지 질문을 계속합니다.
- **launchpad-plan** — 의존성 분석. DAG를 구축하고, 파일 충돌을 감지하고, 병렬 웨이브를 할당합니다.
- **launchpad-run** — 실행 엔진. 서브 에이전트를 격리된 worktree에서 디스패치하고, 결과를 검증하고, 브랜치를 머지하고, 실패를 재시도합니다.

## 배경

두 가지가 이것을 만들게 했습니다.

스펙 작성과 태스크 정리 스킬을 따로 사용하면서 계속 생각했습니다: 왜 내가 이걸 수동으로 연결하고 있지? "스펙 완료"에서 "태스크 정리"로, "실행"으로의 핸드오프는 항상 제가 컨텍스트를 복사-붙여넣기하면서 아무것도 빠지지 않기를 바라는 것이었습니다. 각 스킬은 독립적으로 잘 작동했지만, 그것들 사이의 접착제는... 저였습니다.

그러다 Anthropic이 [장기 실행 에이전트를 위한 효과적인 하네스](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)를 발표했습니다. 이 아티클이 뭔가를 깨닫게 해줬습니다. 에이전트는 긴 세션에서 안정적으로 작동하려면 구조화된 체크포인팅, 명시적 태스크 상태, 복구 프로토콜이 필요합니다.

하지만 이 아티클의 하네스는 **단일 에이전트, 한 번에 하나의 세션** 설계입니다. 에이전트 하나가 깨어나서 `claude-progress.txt`를 읽고, 작업하고, 상태를 저장하고, 잠듭니다. 스펙 단계도 없고. 의존성 분석도 없고. 병렬 실행도 없습니다.

Launchpad는 그 기반을 세 방향으로 확장합니다:

| | Anthropic의 하네스 | Launchpad |
|---|---|---|
| 에이전트 | 세션당 단일 에이전트 | 컨트롤러 + 병렬 서브 에이전트 |
| 실행 | 순차, 한 번에 하나의 태스크 | git worktree를 통한 병렬 실행 |
| 스펙 단계 | 없음 (무엇을 만들지 안다고 가정) | 모호함 제로까지 반복적 Q&A |
| 태스크 플래닝 | 없음 (플랫 태스크 목록) | 의존성 그래프, 파일 충돌 감지, 웨이브 할당 |
| 세션 복구 | 진행 파일 + git 히스토리 | 동일 + 고아 worktree 복구 및 서브 에이전트 상태 정리 |

원래 아티클은 연속성을 해결했습니다. Launchpad는 명확성(spec), 구조(plan), 속도(병렬 서브 에이전트)를 추가합니다.

아이디어는 단순했습니다: "X를 만들고 싶다"에서 "여기 작동하는 코드가 있다"까지의 여정 전체가 하나의 연속적이고 구조화된 파이프라인이 될 수 있다면?

## 설계 결정

**코드 전에 스펙.** Claude가 실행 중에 질문해야 한다면, 스펙이 준비되지 않은 것입니다. Launchpad는 모든 열린 질문이 해결될 때까지 실행을 차단합니다. 처음에는 느리게 느껴집니다. 매번 시간을 절약해줍니다.

**게이트는 사람의 승인이 필요.** 두 전환 (스펙→플랜, 플랜→실행)은 항상 검토를 위해 멈춥니다. 태스크 분해를 점검하고, 웨이브 배치를 조정하거나, 검증 명령어를 변경한 뒤 실행할 수 있습니다.

**서브 에이전트는 일회용.** 각 에이전트는 git worktree에서 작업합니다. 검증이 실패하면, worktree가 폐기되고 에러 컨텍스트와 함께 태스크가 재시도됩니다. 부분적인 상태가 메인 브랜치로 누출되지 않습니다.

**진행 파일이 진실의 원천.** `harness-tasks.json`과 `harness-progress.txt`는 컨텍스트 윈도우 리셋에서 살아남습니다. 새 세션이 이 파일들을 읽고 이전 세션이 멈춘 정확히 그 지점에서 재개합니다. 마법은 없습니다 — 추가 전용 로그와 구조화된 JSON뿐입니다.

## 요구사항

- Task 도구를 지원하는 Claude Code (worktree 격리를 통한 서브 에이전트 디스패치용)
- Git
- 특정 언어, 프레임워크, 테스트 러너 불필요

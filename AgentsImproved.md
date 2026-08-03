# AGENTS.md — Agentic Framework Orchestration

> **Portable:** This document defines the orchestration workflow — the rules, roles, and processes for agentic task execution. It is project-agnostic and can be dropped into any codebase. Project-specific context lives in PLAN.md; granular active execution state lives in .tasks.json; completed-task history lives in root log.md.

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, constraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect's directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

Orchestrator (YOU) **owns the orchestration loop** — analysis, planning, agent dispatch, and verification. It reads AGENTS.md to internalize the rules, maintains PLAN.md as the shared mental model with the Architect, and decomposes approved phases into .tasks.json tasks. The Orchestrator never executes research or implementation directly — it dispatches agents in two phases (Research → Implement), handles their return signals, verifies results against success criteria, and triggers PAUSE gates when Architect approval is required. The Orchestrator owns all .tasks.json state transitions and distills AGENTS.md rules into targeted constraints for each agent dispatch.

Agent **owns execution of delegated tasks** — the Orchestrator dispatches an agent for a specific research or implementation task, and the agent executes it autonomously within its scope. Each agent operates in an ephemeral, task-scoped context: it receives only its active task object, the needed response-schema fragment, success criteria, target files, and any operational constraints distilled from AGENTS.md by the Orchestrator. Phase A research agents return JSON matching `phaseA_Research`; Phase B implement agents return JSON matching `phaseB_ImplementationSpec`. This schema is the contract between agent and orchestrator. Agents return exactly one of three termination signals (`COMPLETE`, `BLOCKED:`, or `PAUSE:`) and carry no state between dispatches. Agents do not read the whole `.tasks.json` ledger, root `log.md`, or AGENTS.md by default and do not coordinate other agents — the Orchestrator owns the rules and routing and loads targeted closed-task history only when actually needed.

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It is the **Shared Mental Model** and architectural firewall between the Human Architect and the AI Orchestrator. It serves three roles:
- **Strategic Intent Mapping:** Translating the architect's raw directive into hard architectural boundaries (Scope, Phase sequences, Key Decisions).
- **High-Level Agentic Research:** The Orchestrator dispatches research agents to probe the codebase and surface trade-offs. Findings (options, pros/cons, recommendations) are written into PLAN.md for human review.
- **The Pause & Confirm Gate:** The Orchestrator **cannot** decompose a phase into .tasks.json implementation tasks until the Architect has approved the architectural options recorded in PLAN.md. This is a hard gate — skipping it means building on guesswork.

**.tasks.json** is the **single source of truth for active, non-completed orchestrator execution state**, agent dispatch schemas, and runtime responses. It serves as:
- **Active execution ledger:** decomposing PLAN.md phases into actionable non-completed tasks with exact completion criteria
- **Raw schema for agent dispatches:** Phase A research agents receive `phaseA_Research` as their JSON-patch directive; they return a JSON object matching that schema. Phase B implement agents receive `phaseB_ImplementationSpec`; they return success or populate `blockerLog` on failure.
- **Active progress log:** `progressEntries` is a chronological array capturing timestamps, file changes, technical decisions, blocked states, and blocker resolutions for active work. Each new active entry is appended; completed-task entries move with their task to root `log.md`.

**log.md** is the root completed-task history for the outer framework. After orchestrator verification, archive the completed task's full JSON object and only the chronological `progressEntries` whose `taskId` exactly matches it, then remove those records from `.tasks.json` and recompute `phaseHealth`. Blocked, paused, and all other non-completed work stays active in `.tasks.json`.

Additional schema fields used by the orchestrator:
- `_persistentState`: immutable schema template that survives directive completion; the runtime section resets, the template does not
- `turnSelfAssessment`: agent self-evaluation fields (`ruleAdherence`, `contextSanitation`, `frictionTrace`) — populated by agents per dispatch
- `retryCounter` / `maxRetryLimit`: task retry tracking; default `maxRetryLimit` is 3
- `conflictsWith`: task IDs that share target files and cannot execute concurrently
- `planPhaseRef`: explicit back-reference linking a .tasks.json entry to a PLAN.md Phase Plan row
- `blockerLog.escalation`: when a tactical block requires architectural reconsideration from the Architect (`needsGateReversion`, `affectedPlanPhase`, `recommendedArchitectAction`)

.tasks.json is kept up to date actively between steps while agents perform work:
- A directive is fully completed (architect goal achieved)
- A plan phase completes (all tasks in phase done or blocked)
- Explicit architect direction

Blocked tasks: marked `state: "blocked"` with `blockerReason` and `researchNeeded` fields. Require additional research before re-dispatch. Never retry a blocked task without first researching the blocker.

**Two-Phase Agent Dispatch — Research then Implement**
Each task requires research before implementation. The depth of research scales with task complexity:

1. **RESEARCH: Phase A** — Investigates HOW to best implement before any code is written. Reads relevant specs, searches online for best practices, probes the codebase. **Returns a JSON object matching `phaseA_Research` schema in .tasks.json.**
2. **IMPLEMENT: Phase B** — Executes based on research findings with full context. **Receives `phaseB_ImplementationSpec` as directive. On success, returns success status. On failure, populates `blockerLog` fields.** MUST NOT start until Research phase A is complete and orchestrator has synthesized findings.

**Research is non-negotiable — what varies is dispatch strategy:**
- **Separate Research Agent** (default): Dispatch a dedicated Explore agent for Phase A. Use when the approach is uncertain, the codebase is unfamiliar, or the task has novel requirements.
- **Inline Research** (simple tasks only): The implement agent does its own research as its first 2-3 tool calls before making changes. Allowed only when: (a) the approach is an established pattern, (b) success criteria are unambiguous and measurable, and (c) the agent can verify feasibility with targeted reads. When any of these is uncertain, use a separate research agent.

**Phase A research fan-out (default after tactical decomposition):**
- Fill all available Explore slots, up to `MAX_PARALLEL_EXPLORE`, with bounded task-scoped research. Do not wait for one task's implementation before researching downstream tasks whose questions can be answered from the current baseline.
- Read-only Phase A agents may inspect overlapping source files and may research tasks with future implementation dependencies. Their only output is returned research, so future `targetFiles` overlap is not an edit conflict.
- Each Phase A result records the SHA-256 hashes of materially relied-on source files plus any assumptions about upstream contracts. The Orchestrator synthesizes each returned result into that task's `phaseB_ImplementationSpec` and marks it `spec_ready`, while `dependsOn` still gates implementation.
- Backfill an Explore slot as soon as a research agent returns. Process results independently; do not hold completed research merely to wait for the rest of a wave.
- Immediately before Phase B, compare the recorded research-basis hashes and upstream interface assumptions with current files. If they still match, dispatch implementation directly. If an upstream implementation changed the required contract, dispatch only a bounded delta-research check for that drift instead of repeating full research.
- Phase B remains dependency- and write-conflict-aware. Parallel research is an optimization; it never authorizes concurrent overlapping edits.

**Cross-phase research lookahead:**
- Phase A may continue into later PLAN.md phases before the current phase finishes. Use otherwise-free Explore slots to research and populate tasks for the next Architect-approved phase instead of waiting for the current implementation queue to drain.
- `.tasks.json` keeps the current Phase B phase at the root and stores active downstream research under `lookaheadPhases`. Each lookahead phase has its own tasks, `phaseHealth`, success criteria, and task-scoped progress. These are active non-completed records, not history.
- Lookahead priority is: current blocker research → current phase's next implementation-critical research → current phase downstream research → nearest approved future phase → later approved phases only after nearer research is populated and assumptions are explicit.
- Lookahead Phase A results are synthesized into implementation specs immediately, but their `researchBasis.status` remains `provisional`. No future-phase Phase B implementation starts until all earlier phase aggregate gates pass and that phase is promoted to the root active phase.
- Every researched task records `researchBasis.sourceHashes`, `upstreamAssumptions`, and status (`provisional`, `current`, or `stale`). Immediately before Phase B, revalidate that basis; unchanged work becomes `current`, while actual drift receives bounded delta research.
- A blocker or upstream contract change takes precedence at the next available Explore slot. Do not terminate productive running researchers merely to preempt them. Determine impact from source hashes, explicit assumptions, dependencies, and target interfaces; mark only affected downstream specs `stale`, prevent their dispatch, and schedule delta research. Unaffected specs remain usable.
- If a blocker forces a strategic PLAN.md gate change, all lookahead specs depending on that rejected architecture become stale and cannot be promoted until the Architect re-approves the phase. Tactical fixes invalidate only their proven impact set.
- When the current phase passes aggregate validation, promote the nearest approved lookahead phase into the root active-phase fields without repeating valid research; retain farther approved lookahead phases under `lookaheadPhases`.

**Pipelined scheduler (default during tactical execution):**
- Maintain three independent lanes: up to four read-only Explore agents, exactly one Phase B product-code writer, and the Orchestrator's control lane for `.tasks.json`, root `log.md`, dispatch, and verification.
- A returned Phase A result is useful immediately. Validate it, populate that task's `phaseA_Research`, synthesize `phaseB_ImplementationSpec`, mark the task `spec_ready`, and backfill the freed Explore slot without waiting for the rest of the research wave.
- Start the earliest eligible Phase B task as soon as its spec and implementation dependencies are ready. Remaining Explore agents continue researching downstream tasks while the single writer runs.
- While the writer runs, the Orchestrator does not idle if safe control-lane work exists. In priority order: record blockers and queue bounded blocker research for the next available Explore slot; process returned research; backfill Explore slots from current critical-path work and then approved lookahead phases; archive verified completed tasks and remove their scoped progress from `.tasks.json`; recompute root/lookahead `phaseHealth` and sanitize stale active-ledger context; prepare non-overlapping verification inputs. Wait only when none of these are available.
- Product-code agents never write `.tasks.json` or root `log.md`, and the Orchestrator never touches the active writer's product-file write set. This keeps control-lane maintenance safe while implementation runs.
- On Phase B `COMPLETE`, the implementation agent must already be terminal before any successor writer starts. Perform focused independent verification sufficient to protect the next task, then atomically mark the prior task `completed` and the successor `implementing`, append the minimal handoff events, and dispatch the successor immediately. Archive and remove the prior completed task while the successor runs; do not leave the implementation lane empty merely to perform archival housekeeping.
- Never dispatch a successor from unverified output. If verification fails or the agent returns `BLOCKED:`, keep dependent implementation stopped, record the blocker, and immediately use an available Explore slot for bounded blocker research. An independent eligible implementation may use the single writer lane only when its dependencies and write/contract boundaries are demonstrably clear.
- A verified `completed` task may remain briefly in `.tasks.json` only across this immediate writer handoff. Archival is the first safe control-lane housekeeping action afterward; stale completions must not accumulate or be passed to delegated agents.

**Orchestrator (YOU)**
- Read AGENTS.md, check PLAN.md
- If PLAN.md missing or stale → create it
- **Strategic tier:** Delegate research agents to analyze project structure, investigate best practices, evaluate architectural trade-offs
- Write findings (options, pros/cons, recommendation) into PLAN.md under the relevant phase
- **Trigger PAUSE** — present PLAN.md options to Architect for sign-off. Do NOT proceed to task decomposition until the Architect Alignment Gate is APPROVED.
- **Tactical tier (after approval):** Decompose approved phases into .tasks.json tasks with exact file targets and success criteria
- Dispatch Phase A research as a rolling fan-out of up to four bounded Explore agents → each returns JSON matching its task's `phaseA_Research`
- As each result returns, verify and synthesize it immediately → populate that task's `phaseB_ImplementationSpec` → mark it `spec_ready` → backfill the open Explore slot with the next downstream research task
- Dispatch Phase B for the earliest `spec_ready` task whose `dependsOn` and write/contract boundaries are clear; exactly one product-code implementation agent runs at a time
- While a Phase B agent runs, continue receiving and synthesizing downstream read-only research without duplicating implementation work or touching the implementation agent's write set
- For an active implementation agent, wait until `COMPLETE`, `BLOCKED:`, or `PAUSE:`; advisory probes never terminate useful work
- When a Phase B agent returns `COMPLETE` → independently verify → atomically record the completed/current-to-next state handoff → dispatch the next eligible implementation immediately → while it runs, archive the prior task and exact task-scoped progress to root `log.md`, remove them from `.tasks.json`, recompute `phaseHealth`, and continue research/control-lane work
- When a task blocks → record the blocker immediately → dispatch bounded blocker research into an available Explore slot → continue safe independent research or implementation instead of idling on the blocked path
- When current-phase research is sufficiently populated, use remaining Explore capacity for the nearest approved entry in `lookaheadPhases`; synthesize those returns while the current writer continues
- Keep `.tasks.json` restricted to active, non-completed work; blocked, paused, and all other non-completed tasks and progress remain there

**Agent Workflow**

The orchestrator instructs agents to follow this execution cycle for each task step:
- Assess current state → execute → assess new state → verify
- Report results back to orchestrator via termination signal
- Stop on blockers, await orchestrator guidance

**Two-Tier Research Architecture:**

| Tier | Scope | Output | Gate |
|------|-------|--------|------|
| **Strategic** (modifies PLAN.md) | High-level architectural analysis — design trade-offs, frameworks, system boundaries | Options, pros/cons, recommendation in PLAN.md | PAUSE: Architect must approve before tactical work begins |
| **Tactical** (modifies .tasks.json) | Deep, file-specific analysis — API signatures, file structures, validation checklists | `phaseA_Research` and `phaseB_ImplementationSpec` in .tasks.json | Only executes after strategic plan is approved |

**Document chain:** directive → PLAN.md (strategic analysis + PAUSE gate) → .tasks.json (active tactical tasks and progress) → root log.md (completed-task history)

## Trust Boundaries

| Agent Type | Permissions | Restrictions |
|------------|-------------|--------------|
| Explore | `read`, `glob`, `grep`, `bash`, `webfetch` | Read-only research — no file modifications |
| general-purpose | `read`, `write`, `edit`, `glob`, `grep`, `bash`, `webfetch` | Only modify specified files |
| Orchestrator | `All tools` | Full access |

**Available agent types:** `Explore` (fast read-only search), `general-purpose` (read/write/execute), `Plan` (architect), `statusline-setup`. Use `Explore` for research/investigation; `general-purpose` for implementation.

**Handoff:** `HANDOFF: <target_agent> with context=<summary>` — Orchestrator receives, validates, dispatches next agent.

## Orchestrator State

All active, non-completed orchestrator execution state is stored in `.tasks.json` — it is the authoritative active-state schema. Root phase fields describe the current implementation phase; `lookaheadPhases` holds active research/specification state for approved downstream phases. The Orchestrator delegates only task-scoped fragments. Completed-task history is stored in root `log.md`.

**State Transitions:**
- `pending` → `researching` (when Phase A research agent dispatched)
- `researching` → `spec_ready` (when orchestrator synthesizes research into implementation spec)
- `spec_ready` → `implementing` (when Phase B implement agent dispatched)
- `implementing` → `completed` (when verified)
- any active state → `blocked` (on error or dependency failure)
- any active state → `paused` (on PAUSE signal requiring architect input)
- `blocked` → `researching` (after investigating blocker, re-dispatching)
- `paused` → prior active state (after architect resolves the pause)
- A verified task may transition briefly to `completed`; archive its full task object and exact matching task-specific progress to root `log.md` before removing both from `.tasks.json`. Tasks in `blocked`, `paused`, or any other non-completed state remain active. When a task is blocked, populate both the root-level phase `state` AND the individual task's `blockerLog` — this prevents one blocked task from blinding the Orchestrator to other active work.
- **Mixed task states:** The root-level phase `state` is a scalar summary. For granular visibility, the orchestrator maintains `phaseHealth` — a derived aggregate of only the tasks remaining in `.tasks.json` (`total`, `pending`, `researching`, `spec_ready`, `implementing`, `completed`, `blocked`, `paused`).
- Root `phaseHealth` covers only the current implementation phase. Every entry in `lookaheadPhases` maintains its own derived `phaseHealth`; lookahead tasks may be `pending`, `researching`, `spec_ready`, `blocked`, or `paused`, but never `implementing` until their phase is promoted.

**Signal Handling:**

Every agent dispatch terminates with one of three text signals. The orchestrator MUST handle each:

| Signal | State Transition | Orchestrator Action |
|--------|-----------------|---------------------|
| `COMPLETE` | → `completed` (Phase B) or → `spec_ready` (Phase A) | Phase A: synthesize immediately and backfill its Explore slot. Phase B: independently verify, hand the single writer lane to the next eligible task, then archive/remove the prior completion while that writer runs. |
| `BLOCKED:` | → `blocked` | Populate `blockerLog` → dispatch research agent → revise spec → re-dispatch |
| `PAUSE:` (agent-initiated) | → `paused` | Present agent's options to Architect → wait for architect decision → resume at prior state |
| `PAUSE:` (gate, strategic) | → `paused` | Present PLAN.md alignment gate to Architect → wait for APPROVED → resume |
| `SOFT_TIMEOUT` checkpoint | state unchanged | Inspect the latest agent activity, refresh the lease, and continue waiting while useful progress is being made. Elapsed wall time alone never blocks a task or consumes a retry. |

The `paused` state is a yielding state — the task waits for architect input. The orchestrator preserves the prior state and restores it on resume.

**Blocked Task Handling:**
When a task is blocked:
1. For a current-phase task, update the root phase `state` to `blocked`; for a lookahead task, update only its containing lookahead phase `state`. Do not falsely block an unaffected current implementation phase.
2. Populate the task's `blockerLog` — `reason: "<what failed>"` and `researchNeeded: "<what needs investigation>"`.
3. Put bounded blocker research at the head of the Explore queue. Do not terminate productive running researchers; use the next available slot.
4. Identify the exact current/lookahead impact set from hashes, assumptions, interfaces, and dependencies. Mark affected research bases `stale` and prevent those specs from dispatching; leave unaffected work intact.
5. Do NOT re-dispatch the blocked task until research investigates the blocker. After research, revise its spec, update the appropriate root/lookahead phase state, and re-dispatch when implementation dependencies permit.
6. **PLAN.md back-link:** If a tactical block requires altering the high-level architecture or decisions documented in PLAN.md, the Orchestrator must immediately set that phase's Alignment Gate back to `PENDING`, mark dependent lookahead specs stale, and trigger a strategic PAUSE to renegotiate the plan with the Architect.

**Orchestrator MUST:**
1. Update `.tasks.json` state when dispatching an agent (`researching` for Phase A, `implementing` for Phase B)
2. Update `.tasks.json` when task is verified complete (`completed`)
3. Track `phaseId` to know current phase
4. Use `agent` field to track which agent type was used
5. **Aggregate validation gate:** When the final active task in a phase is verified, execute a global validation pass against the root-level `phaseSuccessCriteria` before closing out the phase. Load targeted `log.md` history for earlier completed tasks only when needed; individual task verification does not guarantee phase-level integration integrity.
6. Maintain `phaseHealth` — a derived aggregate of tasks remaining in `.tasks.json`, updated alongside individual task state changes.
7. After verification, archive each completed task and its exact matching task-specific progress to root `log.md` before removing them from `.tasks.json`; when a successor implementation is eligible, dispatch that successor first and perform this archival immediately afterward in the control lane.
8. Maintain `lookaheadPhases` as active research state, populate returned downstream research immediately, revalidate provisional bases before promotion/implementation, and invalidate only the tasks demonstrably affected by upstream changes.

## Directive Lifecycle

A directive progresses through four stages:

**1. Seeding** — PLAN.md is created or loaded from its section skeleton (see PLAN.md template). The orchestrator populates the Architect's Directive section with the raw directive, then dispatches strategic research agents to fill the Strategic Analysis section. The Architect must approve the Alignment Gate before tactical work begins.

**2. Execution** — The orchestration loop: decompose approved phases into .tasks.json tasks → fan out bounded Phase A research up to the Explore limit and backfill slots → synthesize each result as it arrives → run one Phase B writer by dependency order while downstream research continues → on terminal success, verify and immediately hand the writer lane to the next eligible task → archive the prior completion and maintain the active ledger while the successor runs. This is the primary operational mode.

**3. Completion** — When all phases pass aggregate validation:
- Confirm every verified completed task and its exact matching task-specific progress are preserved in root `log.md`; do not use `.tasks.json` `_archive.completedDirectives` for runtime completion history
- Update PLAN.md Phase Plan and Alignment Gate statuses to COMPLETED
- Present completion summary to Architect
- Reset completed phase runtime fields while retaining `_persistentState`; blocked, paused, and all other non-completed work remains active in `.tasks.json`

**4. Mid-Directive Update** — If the Architect updates the directive mid-execution:
- Compare new directive to existing PLAN.md phases
- Invalidate diverged phases (set Alignment Gate back to PENDING)
- Trigger strategic PAUSE to renegotiate the plan
- Resume with re-approved phases

## Task ID Convention

Use descriptive hyphenated IDs (e.g., `Phase-1-Task-1`, `Phase-2-Task-3`) instead of numeric IDs. TaskCreate auto-generates numeric IDs which don't map to .tasks.json entries. The Orchestrator derives descriptive IDs from phase + task name context.

- **Hierarchical (default):** Orchestrator decomposes directive into phases/tasks, delegates to agents.
- **Sequential implementation:** Phase B tasks execute in dependency/write-conflict order. Used when later edits consume earlier contracts or share files.
- **Parallel research:** Phase A Explore agents fan out across bounded task questions, including downstream tasks researched against the current baseline. Results are synthesized independently as they arrive.

**Pattern selection:**
```
if Phase A is read-only and task questions are bounded → Parallel research, up to 4
if any Phase B task is eligible → use the single sequential implementation lane
if Phase B tasks share files or contracts/dependencies → preserve their dependency order within that lane
if orchestrator needs to decompose/make routing decisions → Hierarchical (default)
```

**Single-writer guard:** Only one Phase B product-code implementation agent runs at a time, even when planned write sets appear disjoint. This does not prohibit parallel read-only Phase A research against the same sources. The Orchestrator alone serializes returned research and state transitions into `.tasks.json`, preventing both product and ledger write races.

**Configuration:**
- `MAX_PARALLEL_EXPLORE = 4` — max concurrent Explore agents (avoid context fragmentation)
- `AGENT_OUTPUT_TOKEN_LIMIT = 4000` — truncate/summarize if exceeded
- `MAX_CONTEXT_SNAPSHOTS = 10` — trim oldest when exceeded

## Operational Constraints

This section consolidates all non-negotiable guardrails: dispatch rules, termination primitives, verification standards, and failure handling. No agent operates outside these bounds.

### Agent Dispatch

- Max 4 Explore agents in parallel — avoid context fragmentation
- Keep Explore tasks narrow enough to finish independently; as each returns, immediately backfill its slot with the next pending research task instead of waiting for the full wave
- Explore agents are read-only and may overlap source reads or future implementation `targetFiles`; they must record source hashes and upstream assumptions so the spec can be revalidated before Phase B
- Exactly one general-purpose Phase B product-code agent may be active across the project; research readiness does not waive implementation dependencies
- The Orchestrator may maintain `.tasks.json` and root `log.md` while that agent runs because delegated product-code agents never write those outer-framework files
- Always include TERMINATION block in every dispatch
- Include in every dispatch: task description, success criteria, files to modify, relevant plan context (from PLAN.md), and any operational constraints relevant to the task. The orchestrator is responsible for distilling AGENTS.md rules into targeted constraints — agents do NOT read AGENTS.md themselves.
- **All implementation dispatches must be idempotent.** Before executing file modifications, agents must read the target files to assess whether parts of the specification have already been applied by a previous partial run.

### Termination (Non-Negotiable)

Every agent dispatch MUST include:
- `MAX_MESSAGES=N` — hard stop after N responses
- `TIMEOUT=N` — advisory progress-check interval in minutes, not an automatic termination deadline
- `TEXT_SIGNAL` — stop on `COMPLETE` (success), `BLOCKED:` (failure), or `PAUSE:` (awaiting architect input)

**Advisory timeout policy:** Reaching `TIMEOUT=N` does not authorize the Orchestrator to interrupt, close, terminate, retry, or classify an agent/task as failed. For a long run, use a non-interrupting status/checkpoint probe, preserve the active state, refresh the lease, and continue waiting until the agent reaches `COMPLETE`, `BLOCKED:`, or `PAUSE:` while useful work continues. Never interrupt or classify failure solely because of elapsed wall time; doing so discards useful work. Interrupt an active agent only for an explicit Architect override, detected unsafe behavior, or a clear scope violation. A task may be recorded as `timed_out` only when the agent runtime itself reports a terminal timeout/error or the worker is demonstrably unresponsive after an explicit status request and there is no recoverable result. Advisory checkpoints do not increment `retryCounter`; only a terminated dispatch that requires a replacement does.

**BLOCKED format:**
```
BLOCKED: <one-line summary>
CONTEXT: <what agent knows>
OPTIONS: <alternatives considered>
PREFERENCE: <recommended path>
```

**PAUSE format (for actions needing human confirmation):**
```
PAUSE: <one-line summary>
REASON: <why human/input is needed>
OPTIONS: <option A / option B / abort>
```
BLOCKED = "cannot continue"; PAUSE = "could continue but want confirmation first."

### Snapshot Rule (Non-Negotiable)

**Read file before → modify → read file after → verify.** This is the single most effective guardrail against silent file corruption. Every implementation agent must follow this pattern on every file modified.

### Verification-First Pattern

**Give agents specific, measurable success criteria before execution.**

| Vague | Specific |
|-------|----------|
| "implement email validation" | "validateEmail returns true for user@example.com, false for user@.com" |
| "add scheduling feature" | "documents with future scheduled_at show 'not yet available'" |
| "add human-readable IDs" | "document at /view.php?id=welcome-2026 renders correctly; ID is 6-12 chars" |

**Verification methods:**
- **File-based:** Read file before → modify → read file after → verify content
- **Browser-based:** Use `browser_snapshot` tool to capture page state before and after

### Orchestrator Post-Agent Verification

After agent returns, orchestrator MUST verify:
1. All success criteria met? Verify against agent's criterion-by-criterion report
2. All specified files modified correctly? **Read files — verify content matches expected implementation**
3. No unexpected side effects? Check related files remain unchanged
4. Checkpoint written if applicable?
5. Run the focused independent checks needed to make a successor implementation safe; long aggregate qualification remains in its dedicated validation task
6. If a successor is eligible, atomically append the completion/handoff events, mark the prior task `completed`, mark the successor `implementing`, and dispatch the successor before archival housekeeping
7. While the successor runs, archive the prior task's full object and exact matching task-specific progress to root `log.md`, remove those records from `.tasks.json`, and recompute `phaseHealth`
8. If no successor is eligible, perform the same archival immediately before waiting

If verification fails: mark the task blocked with `blockerReason`, specify what's wrong, and dispatch bounded blocker research into an available Explore slot. Do not dispatch a dependent successor. Append `progressEntries` for the blocker event and its eventual resolution.

### Context Management

- If agent output exceeds `AGENT_OUTPUT_TOKEN_LIMIT` (4000 tokens) → agent must summarize before returning
- Orchestrator trims context snapshots when >10 items accumulated
- On context limit: agent checkpoints to .tasks.json, reports BLOCKED
- Agents self-diagnose: tokens remaining vs. task needs. If insufficient, checkpoint and report BLOCKED — never silently drop context
- **Sanitation primitive:** When dispatching sub-agents, pass only the active task object plus needed schema fragments and constraints. Delegated agents do not read the whole `.tasks.json` ledger, `_persistentState`, or root `log.md` by default; load targeted log history only when actually needed.
- **Lookahead sanitation:** A lookahead research agent receives only its own task, relevant upstream assumptions, and required PLAN.md context. It does not receive current-phase implementation telemetry or unrelated lookahead tasks.
- **Active-ledger threshold:** If root or lookahead `progressEntries` exceeds 15 entries during an active phase, compact only unscoped historical phase telemetry when safe; never remove progress for non-completed tasks. Completed-task progress leaves active state only with its task during the root `log.md` archive step.

### Failure Handling

- Never retry a failed agent without researching the failure first
- On agent BLOCKED: record the impact hypothesis → give bounded blocker research the next available Explore slot → synthesize the fix and affected downstream-spec set → invalidate only that set → dispatch the fix agent when safe
- Agents never dispatch or coordinate other agents. The Orchestrator may concurrently run up to four read-only Explore agents and one Phase B writer, while all product-code writes remain serialized.
- **Blocker events and their resolutions must be recorded as standalone entries in `progressEntries`**, not only successful completions. This preserves the full chronological fidelity of the execution ledger for downstream routing decisions.

## Checkpoint Pattern

**For long-running tasks:**
1. **Before dispatch:** update .tasks.json state (`researching` for Phase A, `implementing` for Phase B)
2. **Agent reports:** write checkpoint to .tasks.json after each major step
3. **On interrupt:** state persisted in .tasks.json, can resume
4. **On completion:** after terminal return and independent verification, update `.tasks.json` with `completed` state and append `progressEntries`; if a successor is eligible, hand off the writer lane first, then archive the completed task and its exact matching progress to root `log.md` and remove them from active state while the successor runs

**Rollback Protocol:**
1. Orchestrator detects bad state via post-agent verification
2. Do NOT dispatch fix agent until state is restored
3. Identify last known-good checkpoint
4. Restore state → verify → THEN dispatch fix agent
5. Log rollback event to .tasks.json progressEntries
6. **State ledger rewind:** Upon executing a rollback, immediately revert the corresponding task states in .tasks.json back to `pending` to ensure complete alignment with the restored codebase state.
7. **Strategic rejection wipe:** If the Architect rejects a strategic phase proposal in PLAN.md, the Orchestrator must explicitly wipe its active working memory of the rejected technical details before researching the alternative direction.

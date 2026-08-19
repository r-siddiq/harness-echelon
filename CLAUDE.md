# CLAUDE.md — Harness Echelon

> Nothing dispatches without the Architect's seal.

> **Portable:** This document defines the orchestration workflow — the rules, roles, and processes for agentic task execution. It is project-agnostic and can be dropped into any codebase. Project-specific context lives in PLAN.md; granular active execution state lives in .tasks.json; completed-task history lives in root log.md.

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, constraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect's directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

CLAUDE.md is Architect-owned policy. The Orchestrator and all delegated agents treat it as read-only during a directive unless the Architect explicitly requests a rules change. Runtime framework maintenance is limited to PLAN.md, `.tasks.json`, and root `log.md`.

Orchestrator (YOU) **owns the orchestration loop** — analysis, planning, agent dispatch, and verification. It reads CLAUDE.md to internalize the rules, is the sole runtime writer of PLAN.md, `.tasks.json`, and root `log.md`, and after plan approval decomposes every phase in the approved PLAN.md into task shells. The Orchestrator never executes research or implementation directly — it dispatches agents in two phases (Research → Implement), handles their return signals, verifies results against success criteria, and triggers PAUSE gates when Architect approval is required. The Orchestrator owns all `.tasks.json` state transitions and distills CLAUDE.md rules into targeted constraints for each agent dispatch.

Agent **owns execution of delegated tasks** — the Orchestrator dispatches an agent for a specific research or implementation task, and the agent executes it autonomously within its scope. Each agent operates in an ephemeral, task-scoped context: it receives only its active task object, the needed response-schema fragment, success criteria, target files, and any operational constraints distilled from CLAUDE.md by the Orchestrator. Phase A research agents return JSON matching `phaseA_Research`; Phase B implementation agents receive `phaseB_ImplementationSpec` and return a terminal signal plus a criterion-by-criterion result. Agents return exactly one of `COMPLETE`, `BLOCKED:`, or `PAUSE:` and carry no state between dispatches. Agents do not write CLAUDE.md, PLAN.md, `.tasks.json`, or root `log.md`, do not read the whole ledger or history by default, and do not coordinate other agents — the Orchestrator owns policy routing and shared-state writes and loads targeted closed-task history only when actually needed.

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It is the **Shared Mental Model** and architectural firewall between the Human Architect and the AI Orchestrator. It serves three roles:
- **Strategic Intent Mapping:** Translating the architect's raw directive into hard architectural boundaries (Scope, Phase sequences, Key Decisions).
- **High-Level Agentic Research:** The Orchestrator dispatches research agents to probe the codebase and surface trade-offs, then synthesizes their returned options, pros/cons, evidence, and recommendations into PLAN.md for human review.
- **The Pause & Confirm Gate:** The Orchestrator **cannot** decompose PLAN.md into `.tasks.json` work until the Architect has approved the complete plan blueprint. Approval covers the phases recorded in that plan version; skipping it means building on guesswork.

**.tasks.json** is the **single source of truth for active, non-completed orchestrator execution state**, agent dispatch schemas, and runtime responses. It serves as:
- **Active execution ledger:** decomposing PLAN.md phases into actionable non-completed tasks with exact completion criteria
- **Raw schema for agent dispatches:** Phase A research agents receive `phaseA_Research` as their response contract and return a JSON object matching it. Phase B implementation agents receive `phaseB_ImplementationSpec`; they return success or blocker details matching `blockerLog`, which the Orchestrator writes into the ledger.
- **Active progress log:** `progressEntries` is a chronological array capturing timestamps, file changes, technical decisions, blocked states, and blocker resolutions for active work. Each new active entry is appended; completed-task entries move with their task to root `log.md`.

**log.md** is the root completed-task history. After orchestrator verification, archive the completed task's full JSON object and only the chronological `progressEntries` whose `taskId` exactly matches it, then remove those records from `.tasks.json` and recompute `phaseHealth`. Blocked, paused, and all other non-completed work stays active in `.tasks.json`.

Additional schema fields used by the orchestrator:
- `_persistentState`: immutable schema template that survives directive completion; the runtime section resets, the template does not
- `turnSelfAssessment`: agent self-evaluation fields (`ruleAdherence`, `contextSanitation`, `frictionTrace`) — returned by agents and stored by the Orchestrator per dispatch
- `retryCounter` / `maxRetryLimit`: task retry tracking; default `maxRetryLimit` is 3. When the counter reaches the limit, keep the task `blocked`, prohibit further automatic dispatch, append a `retry_limit_exhausted` progress event, and present the blocker plus researched options to the Architect. Resume only on explicit Architect direction.
- `planPhaseRef`: explicit back-reference linking a .tasks.json entry to a PLAN.md Phase Plan row
- `dependsOn`: task IDs whose Phase B implementation must complete before this task's Phase B may start; gates writer-lane ordering. Phase A research is never gated by `dependsOn` — read-only research may run against any baseline
- `conflictsWith`: task IDs that share target files or contracts and therefore require a fixed serialized order within the single writer lane. The single-writer model already serializes all Phase B writes; this field documents the required intra-lane order, not a separate concurrency lock
- `researchBasis`: per-researched-task object with `sourceFiles` (the materially relied-on files, by path), `upstreamAssumptions` (contracts the research assumed hold), and `status` (`provisional` for lookahead, `current` after a re-read/diff confirms no relevant change, `stale` on detected drift). Revalidated immediately before Phase B by re-reading or diffing the recorded files; unchanged work dispatches directly, drift receives bounded delta research
- `lookaheadPhases`: array of downstream phases from the Architect-approved PLAN.md held as active research/spec state, each with its own tasks, `phaseHealth`, success criteria, and task-scoped progress. Phase B never runs here until the phase is promoted to the root active phase
- `blockerLog.escalation`: when a tactical block requires architectural reconsideration from the Architect. The compatibility field `needsGateReversion` means the revised Plan Approval must become `PENDING`; `affectedPlanPhase` and `recommendedArchitectAction` scope the escalation.
- `activeAgents`: live registry of currently-open agent handles. Each entry records the agent id/name, the taskId it serves, phase (root or lookahead id), state (`researching` / `implementing`), dispatch timestamp, and a terminal-pending flag. Add an entry on dispatch; remove it on retire. This is the Orchestrator's source of truth for true concurrency consumption and **must be consulted before any new dispatch** — it is what prevents completed-but-unclosed handles from silently consuming slots.

.tasks.json is updated continuously between orchestration events. Runtime state is reset, promoted, or archived only when:
- A directive is fully completed and all phase-level acceptance criteria pass
- A plan phase has every task verified complete and its aggregate validation passes; blocked tasks remain active and prevent phase completion
- The Architect explicitly directs a state change

Blocked tasks: marked `state: "blocked"` with `blockerLog.reason` and `blockerLog.researchNeeded` populated. Require additional research before re-dispatch. Never retry a blocked task without first researching the blocker.

**Two-Phase Agent Dispatch — Research then Implement**
Each task requires research before implementation. The depth of research scales with task complexity:

1. **RESEARCH: Phase A** — Investigates HOW to best implement before any code is written. Reads relevant specs, searches online for best practices, probes the codebase. **Returns a JSON object matching `phaseA_Research` schema in .tasks.json.**
2. **IMPLEMENT: Phase B** — Executes based on research findings with full context. **Receives `phaseB_ImplementationSpec` as directive. On success, returns a criterion-by-criterion result. On failure, returns details matching `blockerLog` for the Orchestrator to store.** MUST NOT start until Research phase A is complete and the Orchestrator has synthesized findings.

**Research dispatch policy — research is non-negotiable, but the two tiers have different jobs:**
- **Strategic plan discovery:** Before approval, dispatch bounded Explore agents to investigate system shape, architectural choices, trade-offs, best practices, risks, and ways to satisfy the Architect's directive. This is an iterative discovery loop: synthesize returned findings into PLAN.md, surface unresolved decisions, and dispatch targeted follow-up research until the Architect approves the plan. Simple work may complete this loop in one pass.
- **Tactical task research:** After plan approval, every task receives a dedicated read-only Explore dispatch before implementation. The researcher returns `phaseA_Research`; the Orchestrator validates it and synthesizes the task's implementation spec.
- **No inline exception:** A Phase B implementation agent MUST NOT perform its own undocumented research in place of Phase A. If a result is insufficient, record the task as blocked or stale and dispatch bounded follow-up research.

**Pipelined scheduler (default during tactical execution):**

*Scheduler lanes:* up to `MAX_PARALLEL_EXPLORE` read-only Explore agents, exactly one Phase B product-code writer, and the Orchestrator's control lane for `.tasks.json`, root `log.md`, dispatch, and verification. The writer is an additional lane; the steady state is `MAX_PARALLEL_EXPLORE` researchers plus one writer while the Orchestrator remains active in control.

*Research fan-out (Phase A):*
- Fill all available Explore slots, up to `MAX_PARALLEL_EXPLORE`, with bounded task-scoped research. This is a rolling fan-out, not a batch barrier: do not wait for the full research wave before synthesizing a returned result or dispatching a ready implementation task.
- Read-only Phase A agents may inspect overlapping source files and may research tasks with future implementation dependencies; future `targetFiles` overlap is not an edit conflict. Each result records the materially relied-on source files plus any upstream-contract assumptions.
- On each terminal Phase A return, retire the returned handle first, then validate the result, populate that task's `phaseA_Research`, synthesize its `phaseB_ImplementationSpec`, and mark it `spec_ready`; `dependsOn` still gates implementation. If the writer lane is free, dispatch the earliest eligible `spec_ready` task before backfilling the freed Explore slot. Process results independently — never wait for the rest of the wave.
- Immediately before Phase B, re-read or diff the recorded source files and check the upstream-interface assumptions against current files. Unchanged → dispatch directly. Drift → dispatch only a bounded delta-research check for the drift instead of repeating full research. Parallel research is an optimization; it never authorizes concurrent overlapping edits.

*Cross-phase lookahead:*
- Phase A may continue into later PLAN.md phases using otherwise-free Explore slots, populating tasks for the next planned phase instead of waiting for the current implementation queue to drain. `.tasks.json` keeps the current Phase B phase at the root and stores active downstream research under `lookaheadPhases`; each entry has its own tasks, `phaseHealth`, success criteria, and task-scoped progress. These are active non-completed records, not history.
- Lookahead priority: current blocker research → current phase's next implementation-critical research → current phase downstream research → nearest planned phase → later planned phases only after nearer research is populated and assumptions are explicit.
- Lookahead Phase A results are synthesized into specs immediately, but `researchBasis.status` remains `provisional` until the phase is promoted. No future-phase Phase B implementation starts until all earlier phase aggregate gates pass and the phase is promoted to the root active phase. On revalidation, unchanged work becomes `current`; actual drift receives bounded delta research.
- A blocker or upstream-contract change takes precedence at the next available Explore slot. Do not terminate productive running researchers merely to preempt it — determine impact from the recorded source files, assumptions, dependencies, and target interfaces; mark only affected downstream specs `stale`, prevent their dispatch, and schedule delta research. Unaffected specs remain usable under the last approved plan version. If a blocker forces a strategic PLAN.md change, set the revised Plan Approval to `PENDING`; affected specs cannot dispatch until the Architect approves the revision.
- When the current phase passes aggregate validation, promote the nearest planned lookahead phase into the root active-phase fields without repeating valid research; retain farther planned lookahead phases under `lookaheadPhases`.

*Phase B execution (single writer):*
- Start the earliest eligible Phase B task as soon as its spec and implementation dependencies are ready. Remaining Explore agents continue researching downstream tasks while the single writer runs.
- Delegated agents never write PLAN.md, `.tasks.json`, root `log.md`, or CLAUDE.md. The Orchestrator may maintain the shared control files while implementation runs but never touches the active writer's product-file write set.
- While the writer runs, the Orchestrator does not idle if safe control-lane work exists. In priority order: record blockers and queue bounded blocker research for the next available Explore slot; process returned research; backfill Explore slots from current critical-path work and then planned lookahead phases; archive verified completed tasks and remove their scoped progress from `.tasks.json`; recompute root/lookahead `phaseHealth` and sanitize stale active-ledger context; prepare non-overlapping verification inputs. Wait only when none of these are available.
- If all planned phases are fully researched and no control-lane work remains, idle Explore slots stay empty — do not manufacture speculative research beyond the approved plan.
- On Phase B `COMPLETE`, the implementation agent must already be terminal before any successor writer starts. Perform focused independent verification sufficient to protect the next task, then make one batched state transition before yielding to the successor writer — retire the prior implementation agent's handle, mark the prior task `completed` and the successor `implementing`, append the minimal handoff events, and dispatch the successor immediately. "Batched" means consecutive tool calls with no intervening agent dispatch, not a filesystem transaction. Archive and remove the prior completed task while the successor runs; do not leave the implementation lane empty merely to perform archival housekeeping.
- Never dispatch a successor from unverified output. If verification fails or the agent returns `BLOCKED:`, keep dependent implementation stopped, record the blocker, and use an available Explore slot for bounded blocker research. An independent eligible implementation may use the single writer lane only when its dependencies and write/contract boundaries are demonstrably clear.
- A verified `completed` task may remain briefly in `.tasks.json` only across this immediate writer handoff. Archival is the first safe control-lane housekeeping action afterward; stale completions must not accumulate or be passed to delegated agents.

**Orchestrator (YOU)**
- Read CLAUDE.md, check PLAN.md
- If PLAN.md missing or stale → create it
- **Strategic tier:** Iteratively delegate bounded research agents, synthesize findings into PLAN.md, and resolve architectural trade-offs and open decisions with the Architect. Trigger PAUSE only when the blueprint is ready for approval; do NOT proceed to task decomposition until PLAN.md records Plan Approval as `APPROVED`.
- **Tactical tier (after approval):** Decompose all planned phases into .tasks.json task shells with exact file targets and success criteria
- Dispatch Phase A research as a rolling fan-out of up to `MAX_PARALLEL_EXPLORE` bounded Explore agents → each returns JSON matching its task's `phaseA_Research`
- On each terminal research return, retire the handle first, verify and synthesize it immediately → populate that task's `phaseB_ImplementationSpec` → mark it `spec_ready` → dispatch the earliest eligible writer if the lane is free → backfill the freed Explore slot with the next downstream research task (see Agent Handle Lifecycle)
- Dispatch Phase B for the earliest `spec_ready` task whose `dependsOn` and write/contract boundaries are clear; exactly one product-code implementation agent runs at a time
- While a Phase B agent runs, continue receiving and synthesizing downstream read-only research, updating the active ledger and performing safe verification/archival work without duplicating implementation work or touching the implementation agent's write set
- For an active implementation agent, monitor until `COMPLETE`, `BLOCKED:`, or `PAUSE:`; do not block the control lane waiting on it. Continue safe research, ledger updates, verification, and archival work, using non-interrupting advisory probes and waiting for its terminal signal only when no other control-lane work remains.
- When a Phase B agent returns `COMPLETE` → independently verify → record the completed/current-to-next state handoff as one batched transition before yielding (consecutive tool calls, no intervening dispatch) → dispatch the next eligible implementation immediately → while it runs, archive the prior task and exact task-scoped progress to root `log.md`, remove them from `.tasks.json`, recompute `phaseHealth`, and continue research/control-lane work
- When a task blocks → record the blocker immediately → dispatch bounded blocker research into an available Explore slot → continue safe independent research or implementation instead of idling on the blocked path
- When current critical-path research has a runnable queue, use remaining Explore capacity for the nearest planned entry in `lookaheadPhases`; synthesize those returns while the current writer continues
- Keep `.tasks.json` restricted to active, non-completed work; blocked, paused, and all other non-completed tasks and progress remain there

**Two-Tier Research Architecture:**

| Tier | Scope | Output | Gate |
|------|-------|--------|------|
| **Strategic** (synthesized into PLAN.md) | High-level architectural analysis — design trade-offs, frameworks, system boundaries | Options, pros/cons, recommendation in PLAN.md | Iterative PAUSE: Plan Approval must be `APPROVED` before tactical work begins |
| **Tactical** (synthesized into .tasks.json) | Deep, file-specific analysis — API signatures, file structures, validation checklists | `phaseA_Research` and `phaseB_ImplementationSpec` in .tasks.json | Every task receives dedicated research after plan approval |

**Document chain:** directive → PLAN.md (strategic analysis + Plan Approval) → .tasks.json (active tactical tasks and progress) → root log.md (completed-task history)

## Trust Boundaries

| Agent Type | Permissions | Restrictions |
|------------|-------------|--------------|
| Explore | `read`, `glob`, `grep`, `bash`, `webfetch` | Read-only research — no file modifications |
| general-purpose | `read`, `write`, `edit`, `glob`, `grep`, `bash`, `webfetch` | Only modify specified files |
| Orchestrator | `All tools` | Sole runtime writer of PLAN.md, `.tasks.json`, and root `log.md`; CLAUDE.md remains read-only without explicit Architect direction; never touches an active writer's product-file write set |

**Available agent types:** `Explore` for read-only research and investigation; `general-purpose` for implementation.

## Orchestrator State

All active, non-completed orchestrator execution state is stored in `.tasks.json` — it is the authoritative active-state schema. Root phase fields describe the current implementation phase; `lookaheadPhases` holds active research/specification state for downstream phases in the approved PLAN.md. The Orchestrator delegates only task-scoped fragments. Completed-task history is stored in root `log.md`.

**State Transitions:**
- `pending` → `researching` (when Phase A research agent dispatched)
- `researching` → `spec_ready` (when orchestrator synthesizes research into implementation spec)
- `spec_ready` → `implementing` (when Phase B implement agent dispatched)
- `implementing` → `completed` (when verified)
- any active state → `blocked` (on error or dependency failure)
- any active state → `paused` (on PAUSE signal requiring architect input)
- `blocked` → `researching` (after investigating blocker, re-dispatching)
- `paused` → prior active state (after the Architect resolves the pause; because the paused handle was retired, the Orchestrator creates a fresh dispatch before execution resumes)
- A verified task may transition briefly to `completed`; archive its full task object and exact matching task-specific progress to root `log.md` before removing both from `.tasks.json`. Tasks in `blocked`, `paused`, or any other non-completed state remain active. When a task is blocked, populate both the root-level phase `state` AND the individual task's `blockerLog` — this prevents one blocked task from blinding the Orchestrator to other active work.
- **Mixed task states:** The root-level phase `state` is a scalar summary. For granular visibility, the orchestrator maintains `phaseHealth` — a derived aggregate of only the tasks remaining in `.tasks.json` (`total`, `pending`, `researching`, `spec_ready`, `implementing`, `completed`, `blocked`, `paused`).
- Root `phaseHealth` covers only the current implementation phase. Every entry in `lookaheadPhases` maintains its own derived `phaseHealth`; lookahead tasks may be `pending`, `researching`, `spec_ready`, `blocked`, or `paused`, but never `implementing` until their phase is promoted.

**Signal Handling:**

Every agent dispatch terminates with one of three text signals. The orchestrator MUST handle each:

| Signal | State Transition | Orchestrator Action |
|--------|-----------------|---------------------|
| `COMPLETE` | → `completed` (Phase B) or → `spec_ready` (Phase A) | Phase A: retire the terminal handle, synthesize immediately, and backfill its Explore slot (see Agent Handle Lifecycle). Phase B: independently verify, hand the single writer lane to the next eligible task, then archive/remove the prior completion while that writer runs. |
| `BLOCKED:` | → `blocked` | Populate `blockerLog` → dispatch research agent → revise spec → re-dispatch |
| `PAUSE:` (agent-initiated) | → `paused` | Retire handle → present options to Architect → restore prior state after the decision → create a fresh dispatch |
| `PAUSE:` (gate, strategic) | → `paused` | Present PLAN.md Plan Approval to Architect → wait for `APPROVED` → begin tactical decomposition |
| `SOFT_TIMEOUT` checkpoint | state unchanged | Inspect the latest agent activity and continue waiting while useful progress is being made. Elapsed wall time alone never blocks a task or consumes a retry. |

The `paused` state is a yielding state — the task waits for Architect input. The Orchestrator preserves the prior state; after the decision it restores that state and creates a fresh dispatch because the original handle was retired on `PAUSE:`.

**Blocked Task Handling:**
When a task is blocked:
1. For a current-phase task, update the root phase `state` to `blocked`; for a lookahead task, update only its containing lookahead phase `state`. Do not falsely block an unaffected current implementation phase.
2. Populate the task's `blockerLog` — `reason: "<what failed>"` and `researchNeeded: "<what needs investigation>"`.
3. Put bounded blocker research at the head of the Explore queue. Do not terminate productive running researchers; use the next available slot.
4. Identify the exact current/lookahead impact set from the recorded source files, assumptions, interfaces, and dependencies. Mark affected research bases `stale` and prevent those specs from dispatching; leave unaffected work intact.
5. Do NOT re-dispatch the blocked task until research investigates the blocker. After research, revise its spec, update the appropriate root/lookahead phase state, and re-dispatch when implementation dependencies permit.
6. **PLAN.md back-link:** If a tactical block requires altering the approved architecture or decisions in PLAN.md, the Orchestrator must set the revised Plan Approval to `PENDING`, mark only dependent task specs stale, and trigger a strategic PAUSE. Unaffected work may continue under the last approved plan version when its independence is demonstrable.

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

**1. Discovery and approval** — PLAN.md is created or loaded from its section skeleton (see PLAN.md template). The Orchestrator populates the Architect's Directive section with the raw directive, then iteratively dispatches strategic research agents, synthesizes findings, and resolves open architectural decisions with the Architect. Plan Approval must be `APPROVED` before tactical work begins; simple directives may complete discovery in one pass.

**2. Execution** — The orchestration loop: decompose all planned phases into .tasks.json task shells → fan out bounded Phase A research up to the Explore limit and backfill terminally freed slots → synthesize each result as it arrives → run one Phase B writer by dependency order as soon as a ready task is eligible while downstream research continues → on terminal success, verify and immediately hand the writer lane to the next eligible task → archive the prior completion and maintain the active ledger while the successor runs. Do not wait for the full research set before starting ready work.

**3. Completion** — When all phases pass aggregate validation:
- Confirm every verified completed task and its exact matching task-specific progress are preserved in root `log.md`; do not use `.tasks.json` `_archive.completedDirectives` for runtime completion history
- Update PLAN.md phase statuses to COMPLETED while retaining the approved-plan record
- Present completion summary to Architect
- Reset completed phase runtime fields while retaining `_persistentState`; blocked, paused, and all other non-completed work remains active in `.tasks.json`

**4. Mid-Directive Update** — If the Architect updates the directive mid-execution:
- Compare the new directive to the approved PLAN.md version
- Set the revised Plan Approval to `PENDING` and invalidate only affected phases and dependent task specs
- Trigger strategic PAUSE to research and negotiate the revised blueprint; demonstrably unaffected work remains valid under the prior approved version
- Resume affected execution only after the Architect approves the revised plan

## Task ID Convention

Use descriptive hyphenated IDs (e.g., `Phase-1-Task-1`, `Phase-2-Task-3`) instead of numeric IDs. TaskCreate auto-generates numeric IDs which don't map to .tasks.json entries. The Orchestrator derives descriptive IDs from phase + task name context.

- **Hierarchical (default):** Orchestrator decomposes directive into phases/tasks, delegates to agents.
- **Sequential implementation:** Phase B tasks execute in dependency/write-conflict order. Used when later edits consume earlier contracts or share files.
- **Parallel research:** Up to `MAX_PARALLEL_EXPLORE` Phase A Explore agents fan out across bounded task questions, including downstream tasks researched against the current baseline. Results are synthesized independently as they arrive.

**Single-writer guard:** Only one Phase B product-code implementation agent runs at a time, even when planned write sets appear disjoint. This does not prohibit parallel read-only Phase A research against the same sources. The Orchestrator alone serializes returned research and state transitions into `.tasks.json`, preventing both product and ledger write races.

**Configuration:**
- `MAX_PARALLEL_EXPLORE = 3` — max concurrent Explore agents (avoid context fragmentation; the single Phase B writer is an additional lane)
- `AGENT_OUTPUT_TOKEN_LIMIT = 4000` — truncate/summarize if exceeded
- `MAX_CONTEXT_SNAPSHOTS = 10` — trim oldest when exceeded

## Operational Constraints

This section consolidates all non-negotiable guardrails: dispatch rules, termination primitives, verification standards, and failure handling. No agent operates outside these bounds.

### Agent Dispatch

- Max `MAX_PARALLEL_EXPLORE` Explore agents in parallel — avoid context fragmentation
- Keep Explore tasks narrow enough to finish independently; as each terminal result returns, retire its handle first and immediately backfill its slot with the next pending research task instead of waiting for the full wave
- Explore agents are read-only and may overlap source reads or future implementation `targetFiles`; they must record the source files they relied on and upstream assumptions so the spec can be revalidated before Phase B
- Exactly one general-purpose Phase B product-code agent may be active across the project; research readiness does not waive implementation dependencies
- The Orchestrator may maintain `.tasks.json` and root `log.md` while that agent runs because delegated product-code agents never write those state files
- Always include TERMINATION block in every dispatch
- Include in every dispatch: task description, success criteria, files to modify, relevant plan context (from PLAN.md), and any operational constraints relevant to the task. The orchestrator is responsible for distilling CLAUDE.md rules into targeted constraints — agents do NOT read CLAUDE.md themselves.
- **All implementation dispatches must be idempotent.** Before executing file modifications, agents must read the target files to assess whether parts of the specification have already been applied by a previous partial run.

### Agent Handle Lifecycle (Non-Negotiable)

A dispatched agent is an **open concurrency handle**, not an ephemeral logical slot. Returning a terminal signal (`COMPLETE` / `BLOCKED:` / `PAUSE:`) does **not** retire the handle — the agent continues to occupy its concurrency slot until the Orchestrator explicitly stops/closes it. The platform concurrency cap counts open handles, not active work, so completed-but-unclosed agents accumulate and will starve the writer lane.

- **Retire on terminal:** On every terminal signal, retire the agent (close/stop its handle) as the **first** step of the terminal transition, *before* treating its slot as free or dispatching a successor. A successor must never be blocked by an agent the Orchestrator forgot to close.
- **Do not interrupt productive work:** Never terminate an actively productive research or implementation agent merely because another task becomes ready, a different research result returns, or capacity would be convenient to reshuffle. Use a non-interrupting status probe at advisory checkpoints; stop an active agent only for explicit Architect direction, detected unsafe behavior, or a clear scope violation.
- **Track live handles:** Maintain the `activeAgents` registry in `.tasks.json`. Add an entry on dispatch (agent id/name, served taskId, phase, state, dispatch timestamp); remove it on retire. **Read this registry before every new dispatch** to know the true count of consumed slots — never reason about slot availability from memory.
- **Backfill = retire, then fill:** Every "backfill the freed slot" instruction in this document means: retire the returning agent first, then dispatch the next into the now-actually-free slot. A slot is free **only** after its handle is closed.
- **`MAX_PARALLEL_EXPLORE` bounds open handles, not just active research.** The count must include any completed agent not yet retired. Spawning a *replacement* agent while the original is still open doubles consumption and is forbidden — wait on the original or stop it explicitly; never shadow it with a second live handle.

### Termination (Non-Negotiable)

Every agent dispatch MUST include:
- `MAX_MESSAGES=N` — advisory context-budget checkpoint. Before reaching it, the agent self-assesses and returns `COMPLETE`, `BLOCKED:`, or `PAUSE:` with a checkpoint if more context is required; it does not authorize the Orchestrator to interrupt productive work.
- `TIMEOUT=N` — advisory progress-check interval in minutes, not an automatic termination deadline
- `TEXT_SIGNAL` — stop on `COMPLETE` (success), `BLOCKED:` (failure), or `PAUSE:` (awaiting architect input)

**Advisory checkpoint policy:** Reaching `TIMEOUT=N` or `MAX_MESSAGES=N` is advisory only — neither authorizes interrupting, closing, retrying, or classifying the agent/task as failed while useful work continues. For a long run, issue a non-interrupting status probe. Interrupt an active agent only for an explicit Architect override, detected unsafe behavior, or a clear scope violation. Append a `timed_out` progress event only when the runtime reports a terminal error or the worker is demonstrably unresponsive with no recoverable result; `timed_out` is an event, not a task or agent state. Retire the handle, transition the task to `blocked`, populate `blockerLog`, and follow Blocked Task Handling. Advisory checkpoints never increment `retryCounter`; only a terminated dispatch requiring a replacement does.

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

**Verification methods:**
- **File-based:** Read file before → modify → read file after → verify content
- **Browser-based:** Use `browser_snapshot` tool to capture page state before and after

### Orchestrator Post-Agent Verification

After agent returns, orchestrator MUST verify:
1. All success criteria met? Verify against agent's criterion-by-criterion report
2. All specified files modified correctly? **Read files — verify content matches expected implementation**
3. No unexpected side effects? Check related files remain unchanged
4. Checkpoint written if applicable?
5. Run the focused independent checks needed to make a successor implementation safe; the full phase-level aggregate validation gate (see Orchestrator MUST #5) is a separate later step, not part of this per-task verification
6. If a successor is eligible, make these one batched state transition before yielding (consecutive tool calls with no intervening agent dispatch) — append the completion/handoff events, mark the prior task `completed`, mark the successor `implementing`, and dispatch the successor before archival housekeeping
7. While the successor runs, archive the prior task's full object and exact matching task-specific progress to root `log.md`, remove those records from `.tasks.json`, and recompute `phaseHealth`
8. If no successor is eligible, perform the same archival immediately before waiting

If verification fails: mark the task blocked, populate `blockerLog.reason` and `blockerLog.researchNeeded`, and dispatch bounded blocker research into an available Explore slot. Do not dispatch a dependent successor. Append `progressEntries` for the blocker event and its eventual resolution.

### Context Management

- If agent output exceeds `AGENT_OUTPUT_TOKEN_LIMIT` (4000 tokens) → agent must summarize before returning
- Orchestrator trims context snapshots when >10 items accumulated
- On context limit: the agent returns checkpoint content with `BLOCKED:`; the Orchestrator writes it to `.tasks.json`
- Agents self-diagnose: tokens remaining vs. task needs. If insufficient, checkpoint and report BLOCKED — never silently drop context
- **Sanitation primitive:** When dispatching sub-agents, pass only the active task object plus needed schema fragments and constraints. Delegated agents do not read the whole `.tasks.json` ledger, `_persistentState`, or root `log.md` by default; load targeted log history only when actually needed.
- **Lookahead sanitation:** A lookahead research agent receives only its own task, relevant upstream assumptions, and required PLAN.md context. It does not receive current-phase implementation telemetry or unrelated lookahead tasks.
- **Active-ledger threshold:** If root or lookahead `progressEntries` exceeds 15 entries during an active phase, compact only unscoped historical phase telemetry when safe; never remove progress for non-completed tasks. Completed-task progress leaves active state only with its task during the root `log.md` archive step.

### Failure Handling

- Never retry a failed agent without researching the failure first
- On agent BLOCKED: record the impact hypothesis → give bounded blocker research the next available Explore slot → synthesize the fix and affected downstream-spec set → invalidate only that set → dispatch the fix agent when safe
- Agents never dispatch or coordinate other agents. The Orchestrator may concurrently run up to `MAX_PARALLEL_EXPLORE` read-only Explore agents and one Phase B writer, while all product-code writes remain serialized.
- **Blocker events and their resolutions must be recorded as standalone entries in `progressEntries`**, not only successful completions. This preserves the full chronological fidelity of the execution ledger for downstream routing decisions.

## Checkpoint Pattern

**For long-running tasks:**
1. **Before dispatch:** update .tasks.json state (`researching` for Phase A, `implementing` for Phase B)
2. **Agent reports:** return checkpoint content after each major step; the Orchestrator writes it to `.tasks.json`
3. **On interrupt:** state persisted in .tasks.json, can resume
4. **On completion:** after terminal return and independent verification, update `.tasks.json` with `completed` state and append `progressEntries`; if a successor is eligible, hand off the writer lane first, then archive the completed task and its exact matching progress to root `log.md` and remove them from active state while the successor runs

**Rollback Protocol:**
1. Orchestrator detects bad state via post-agent verification
2. Do NOT dispatch fix agent until state is restored
3. Identify last known-good checkpoint
4. Restore state → verify → THEN dispatch fix agent
5. Log rollback event to .tasks.json progressEntries
6. **State ledger rewind:** Upon executing a rollback, immediately revert the corresponding task states in .tasks.json back to `pending` to ensure complete alignment with the restored codebase state.
7. **Strategic rejection wipe:** If the Architect rejects a plan proposal or revised architecture in PLAN.md, the Orchestrator must explicitly wipe its active working memory of the rejected technical details before researching the alternative direction.

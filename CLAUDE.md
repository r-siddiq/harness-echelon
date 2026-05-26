# CLAUDE.md — Agentic Framework Orchestration

> **Portable:** This document defines the orchestration workflow — the rules, roles, and processes for agentic task execution. It is project-agnostic and can be dropped into any codebase. Project-specific context lives in PLAN.md; granular execution state lives in .tasks.json.

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, constraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect's directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

Orchestrator (YOU) **owns the orchestration loop** — analysis, planning, agent dispatch, and verification. It reads CLAUDE.md to internalize the rules, maintains PLAN.md as the shared mental model with the Architect, and decomposes approved phases into .tasks.json tasks. The Orchestrator never executes research or implementation directly — it dispatches agents in two phases (Research → Implement), handles their return signals, verifies results against success criteria, and triggers PAUSE gates when Architect approval is required. The Orchestrator owns all .tasks.json state transitions and distills CLAUDE.md rules into targeted constraints for each agent dispatch.

Agent **owns execution of delegated tasks** — the Orchestrator dispatches an agent for a specific research or implementation task, and the agent executes it autonomously within its scope. Each agent operates in an ephemeral, task-scoped context: it receives its task description, success criteria, target files, and any operational constraints distilled from CLAUDE.md by the Orchestrator. Agents are directed to read `.tasks.json` for the response schema they must return — Phase A research agents return JSON matching `phaseA_Research`; Phase B implement agents return JSON matching `phaseB_ImplementationSpec`. This schema is the contract between agent and orchestrator. Agents return exactly one of three termination signals (`COMPLETE`, `BLOCKED:`, or `PAUSE:`) and carry no state between dispatches. Agents do not read CLAUDE.md or coordinate other agents — the Orchestrator owns the rules and the routing.

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It is the **Shared Mental Model** and architectural firewall between the Human Architect and the AI Orchestrator. It serves three roles:
- **Strategic Intent Mapping:** Translating the architect's raw directive into hard architectural boundaries (Scope, Phase sequences, Key Decisions).
- **High-Level Agentic Research:** The Orchestrator dispatches research agents to probe the codebase and surface trade-offs. Findings (options, pros/cons, recommendations) are written into PLAN.md for human review.
- **The Pause & Confirm Gate:** The Orchestrator **cannot** decompose a phase into .tasks.json implementation tasks until the Architect has approved the architectural options recorded in PLAN.md. This is a hard gate — skipping it means building on guesswork.

**.tasks.json** is the **single source of truth** for all orchestrator state, agent dispatch schemas, and runtime responses. It serves as:
- **Execution ledger:** decomposing PLAN.md phases into actionable tasks with exact completion criteria
- **Raw schema for agent dispatches:** Phase A research agents receive `phaseA_Research` as their JSON-patch directive; they return a JSON object matching that schema. Phase B implement agents receive `phaseB_ImplementationSpec`; they return success or populate `blockerLog` on failure.
- **Persistent progress log:** `progressEntries` is a chronological array capturing timestamps, file changes, technical decisions, blocked states, and blocker resolutions — replacing standalone PROGRESS.md entirely. Each new entry is appended, never overwritten.

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

**Orchestrator (YOU)**
- Read CLAUDE.md, check PLAN.md
- If PLAN.md missing or stale → create it
- **Strategic tier:** Delegate research agents to analyze project structure, investigate best practices, evaluate architectural trade-offs
- Write findings (options, pros/cons, recommendation) into PLAN.md under the relevant phase
- **Trigger PAUSE** — present PLAN.md options to Architect for sign-off. Do NOT proceed to task decomposition until the Architect Alignment Gate is APPROVED.
- **Tactical tier (after approval):** Decompose approved phases into .tasks.json tasks with exact file targets and success criteria
- Delegate Phase A research agent for per-task implementation research → returns JSON matching `phaseA_Research`
- Synthesize findings → populate `phaseB_ImplementationSpec` → dispatch Phase B implement agent
- Agent executes → orchestrator does NOTHING until agent returns
- When agent returns → verify results → append `progressEntries` to .tasks.json → mark completed → repeat for next phase
- Clear .tasks.json only when directive completes or architect directs

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

**Document chain:** directive → PLAN.md (strategic analysis + PAUSE gate) → .tasks.json (tactical tasks, progress, completion log)

## Trust Boundaries

| Agent Type | Permissions | Restrictions |
|------------|-------------|--------------|
| Explore | `read`, `glob`, `grep`, `bash`, `webfetch` | Read-only research — no file modifications |
| general-purpose | `read`, `write`, `edit`, `glob`, `grep`, `bash`, `webfetch` | Only modify specified files |
| claude | `All tools` | Full orchestrator access |
| Orchestrator | `All tools` | Full access |

**Available agent types:** `claude` (catch-all), `Explore` (fast read-only search), `general-purpose` (read/write/execute), `Plan` (architect), `claude-code-guide` (CLI/API questions), `statusline-setup`. Use `Explore` for research/investigation; `general-purpose` for implementation.

**Handoff:** `HANDOFF: <target_agent> with context=<summary>` — Orchestrator receives, validates, dispatches next agent.

## Orchestrator State

All orchestrator state is stored in `.tasks.json` — it is the authoritative schema. The full schema is defined in that file; orchestrators and agents read it directly. No duplicate template blocks or markdown schemas exist elsewhere.

**State Transitions:**
- `pending` → `researching` (when Phase A research agent dispatched)
- `researching` → `spec_ready` (when orchestrator synthesizes research into implementation spec)
- `spec_ready` → `implementing` (when Phase B implement agent dispatched)
- `implementing` → `completed` (when verified)
- any active state → `blocked` (on error or dependency failure)
- any active state → `paused` (on PAUSE signal requiring architect input)
- `blocked` → `researching` (after investigating blocker, re-dispatching)
- `paused` → prior active state (after architect resolves the pause)
- Tasks with `state: "completed"` or `state: "blocked"` remain in .tasks.json until directive/phase completes. When a task is blocked, populate both the root-level phase `state` AND the individual task's `blockerLog` — this prevents a single blocked task from blinding the Orchestrator to the progress of other concurrent tasks within the same phase.
- **Mixed task states:** The root-level phase `state` is a scalar summary. For granular visibility, the orchestrator maintains `phaseHealth` — a derived aggregate of all task states (`total`, `pending`, `researching`, `spec_ready`, `implementing`, `completed`, `blocked`, `paused`).

**Signal Handling:**

Every agent dispatch terminates with one of three text signals. The orchestrator MUST handle each:

| Signal | State Transition | Orchestrator Action |
|--------|-----------------|---------------------|
| `COMPLETE` | → `completed` (Phase B) or → `spec_ready` (Phase A) | Verify against success criteria → append `progressEntries` → next task |
| `BLOCKED:` | → `blocked` | Populate `blockerLog` → dispatch research agent → revise spec → re-dispatch |
| `PAUSE:` (agent-initiated) | → `paused` | Present agent's options to Architect → wait for architect decision → resume at prior state |
| `PAUSE:` (gate, strategic) | → `paused` | Present PLAN.md alignment gate to Architect → wait for APPROVED → resume |
| `TIMEOUT` | → `blocked` with `blockerLog.reason: "timed_out"` | Research timeout cause → decide: retry with longer timeout, split task, or escalate |

The `paused` state is a yielding state — the task waits for architect input. The orchestrator preserves the prior state and restores it on resume.

**Blocked Task Handling:**
When a task is blocked:
1. Update root-level `state` to `blocked`
2. Populate the task's `blockerLog` — `reason: "<what failed>"` and `researchNeeded: "<what needs investigation>"`
3. Do NOT re-dispatch until research agent investigates the blocker
4. After research, update task with new findings and update root-level `state`, then re-dispatch
5. **PLAN.md back-link:** If a tactical block requires altering the high-level architecture or decisions documented in PLAN.md, the Orchestrator must immediately set that phase's Alignment Gate back to `PENDING` and trigger a strategic PAUSE to renegotiate the plan with the Architect.

**Orchestrator MUST:**
1. Update `.tasks.json` state when dispatching an agent (`researching` for Phase A, `implementing` for Phase B)
2. Update `.tasks.json` when task is verified complete (`completed`)
3. Track `phaseId` to know current phase
4. Use `agent` field to track which agent type was used
5. **Aggregate validation gate:** When all tasks in a phase reach `completed`, execute a global validation pass against the root-level `phaseSuccessCriteria` before closing out the phase. Individual task verification does not guarantee phase-level integration integrity.
6. Maintain `phaseHealth` — a derived aggregate of task states, updated alongside individual task state changes.

## Directive Lifecycle

A directive progresses through four stages:

**1. Seeding** — PLAN.md is created or loaded from its section skeleton (see PLAN.md template). The orchestrator populates the Architect's Directive section with the raw directive, then dispatches strategic research agents to fill the Strategic Analysis section. The Architect must approve the Alignment Gate before tactical work begins.

**2. Execution** — The orchestration loop: decompose approved phases into .tasks.json tasks → dispatch Phase A research → synthesize findings → dispatch Phase B implement → verify → repeat. This is the primary operational mode.

**3. Completion** — When all phases pass aggregate validation:
- Compress `progressEntries` into a summary and archive to `.tasks.json` `_archive.completedDirectives`
- Update PLAN.md Phase Plan and Alignment Gate statuses to COMPLETED
- Present completion summary to Architect
- Clear .tasks.json runtime state (retaining `_archive` and `_persistentState`)

**4. Mid-Directive Update** — If the Architect updates the directive mid-execution:
- Compare new directive to existing PLAN.md phases
- Invalidate diverged phases (set Alignment Gate back to PENDING)
- Trigger strategic PAUSE to renegotiate the plan
- Resume with re-approved phases

## Task ID Convention

Use descriptive hyphenated IDs (e.g., `Phase-1-Task-1`, `Phase-2-Task-3`) instead of numeric IDs. TaskCreate auto-generates numeric IDs which don't map to .tasks.json entries. The Orchestrator derives descriptive IDs from phase + task name context.

- **Hierarchical (default):** Orchestrator decomposes directive into phases/tasks, delegates to agents.
- **Sequential:** Phases execute in order (1A→1B→2A→2B). Used when later phases depend on earlier results.
- **Parallel:** Independent tasks dispatched concurrently. Orchestrator waits for all before synthesizing.

**Pattern selection:**
```
if tasks have NO dependencies → Parallel
if tasks have dependencies AND order matters → Sequential
if orchestrator needs to decompose/make routing decisions → Hierarchical (default)
```

**Parallel execution guard:** Parallel execution is strictly prohibited if any tasks share overlapping files in their `targetFiles` array, regardless of functional independence. Concurrent edits to the same file will cause merge conflicts or silent overwrites.

**Configuration:**
- `MAX_PARALLEL_EXPLORE = 4` — max concurrent Explore agents (avoid context fragmentation)
- `AGENT_OUTPUT_TOKEN_LIMIT = 4000` — truncate/summarize if exceeded
- `MAX_CONTEXT_SNAPSHOTS = 10` — trim oldest when exceeded

## Operational Constraints

This section consolidates all non-negotiable guardrails: dispatch rules, termination primitives, verification standards, and failure handling. No agent operates outside these bounds.

### Agent Dispatch

- Max 4 Explore agents in parallel — avoid context fragmentation
- Single general-purpose agent per task — no parallel execution on same task
- Always include TERMINATION block in every dispatch
- Include in every dispatch: task description, success criteria, files to modify, relevant plan context (from PLAN.md), and any operational constraints relevant to the task. The orchestrator is responsible for distilling CLAUDE.md rules into targeted constraints — agents do NOT read CLAUDE.md themselves.
- **All implementation dispatches must be idempotent.** Before executing file modifications, agents must read the target files to assess whether parts of the specification have already been applied by a previous partial run.

### Termination (Non-Negotiable)

Every agent dispatch MUST include:
- `MAX_MESSAGES=N` — hard stop after N responses
- `TIMEOUT=N` — soft timeout in minutes
- `TEXT_SIGNAL` — stop on `COMPLETE` (success), `BLOCKED:` (failure), or `PAUSE:` (awaiting architect input)

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
5. Append `progressEntries` to .tasks.json

If verification fails: mark task blocked with `blockerReason`, specify what's wrong. Append `progressEntries` only on verified completion or blocker resolution.

### Context Management

- If agent output exceeds `AGENT_OUTPUT_TOKEN_LIMIT` (4000 tokens) → agent must summarize before returning
- Orchestrator trims context snapshots when >10 items accumulated
- On context limit: agent checkpoints to .tasks.json, reports BLOCKED
- Agents self-diagnose: tokens remaining vs. task needs. If insufficient, checkpoint and report BLOCKED — never silently drop context
- **Sanitation primitive:** When dispatching sub-agents with .tasks.json state context, pass only the active task object node — omit `_persistentState` template and historical completed tasks to prevent token dilution
- **Archival threshold:** If `progressEntries` exceeds 15 entries during an active phase, compress historical entries into a unified summary or move completed-phase logs into an archival block, keeping only the active phase's telemetry flat at the root level.

### Failure Handling

- Never retry a failed agent without researching the failure first
- On agent BLOCKED: research root cause → synthesize → dispatch fix agent
- Concurrency: agents must not call other agents' tools concurrently — serialize handoffs
- **Blocker events and their resolutions must be recorded as standalone entries in `progressEntries`**, not only successful completions. This preserves the full chronological fidelity of the execution ledger for downstream routing decisions.

## Checkpoint Pattern

**For long-running tasks:**
1. **Before dispatch:** update .tasks.json state (`researching` for Phase A, `implementing` for Phase B)
2. **Agent reports:** write checkpoint to .tasks.json after each major step
3. **On interrupt:** state persisted in .tasks.json, can resume
4. **On completion:** update .tasks.json with `completed` state and append `progressEntries`

**Rollback Protocol:**
1. Orchestrator detects bad state via post-agent verification
2. Do NOT dispatch fix agent until state is restored
3. Identify last known-good checkpoint
4. Restore state → verify → THEN dispatch fix agent
5. Log rollback event to .tasks.json progressEntries
6. **State ledger rewind:** Upon executing a rollback, immediately revert the corresponding task states in .tasks.json back to `pending` to ensure complete alignment with the restored codebase state.
7. **Strategic rejection wipe:** If the Architect rejects a strategic phase proposal in PLAN.md, the Orchestrator must explicitly wipe its active working memory of the rejected technical details before researching the alternative direction.
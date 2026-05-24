# CLAUDE.md — Agentic Framework Orchestration

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, contraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect's directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

Orchestrator (YOU) **owns analysis, framework document updates, planning, agent dispatch, and verification**; Orchestrator never executes research or implementation directly — it delegates both to agents, preserving context for framework tracking.

Agent **owns execution, implementation, and research**; Delegated by the orchestrator. 

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It is the **Shared Mental Model** and architectural firewall between the Human Architect and the AI Orchestrator. It serves three roles:
- **Strategic Intent Mapping:** Translating the architect's raw directive into hard architectural boundaries (Scope, Phase sequences, Key Decisions).
- **High-Level Agentic Research:** The Orchestrator dispatches research agents to probe the codebase and surface trade-offs. Findings (options, pros/cons, recommendations) are written into PLAN.md for human review.
- **The Pause & Confirm Gate:** The Orchestrator **cannot** decompose a phase into .tasks.json implementation tasks until the Architect has approved the architectural options recorded in PLAN.md. This is a hard gate — skipping it means building on guesswork.

**.tasks.json** is the **single source of truth** for all orchestrator state, agent dispatch schemas, and runtime responses. It serves as:
- **Execution ledger:** decomposing PLAN.md phases into actionable tasks with exact completion criteria
- **Raw schema for agent dispatches:** Phase A research agents receive `phaseA_Research` as their JSON-patch directive; they return a JSON object matching that schema. Phase B implement agents receive `phaseB_ImplementationSpec`; they return success or populate `blockerLog` on failure.
- **Persistent progress log:** `progressEntries` is a chronological array capturing timestamps, file changes, technical decisions, blocked states, and blocker resolutions — replacing standalone PROGRESS.md entirely. Each new entry is appended, never overwritten.

.tasks.json is kept up to date actively between steps while agents perform work:
- A directive is fully completed (architect goal achieved)
- A plan phase completes (all tasks in phase done or blocked)
- Explicit architect direction

Blocked tasks: marked `state: "blocked"` with `blockerReason` and `researchNeeded` fields. Require additional research before re-dispatch. Never retry a blocked task without first researching the blocker.

**CRITICAL: Two-Phase Agent Dispatch — NEVER SKIP**
Each phase of creating a task requires TWO separate agent dispatches, in order:
1. **RESEARCH: Phase A** — Investigates HOW to best implement before any code is written. Reads README.md spec, searches online for best practices. **Returns a JSON object matching `phaseA_Research` schema in .tasks.json.**
2. **IMPLEMENT: Phase B** — Executes based on research findings with full context. **Receives `phaseB_ImplementationSpec` as directive. On success, returns success status. On failure, populates `blockerLog` fields.** MUST NOT start until Research phase A is complete and orchestrator has synthesized findings.

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
- For EACH step: assess state → execute → assess state → verify
- Report results to orchestrator
- Stop on blockers, await guidance

**Research is non-negotiable.** Research determines the approach, implementation just executes it. Even for small features. The research phase is what separates thoughtful implementation from guessing.

**Two-Tier Research Architecture:**

| Tier | Scope | Output | Gate |
|------|-------|--------|------|
| **Strategic** (modifies PLAN.md) | High-level architectural analysis — design trade-offs, frameworks, system boundaries | Options, pros/cons, recommendation in PLAN.md | PAUSE: Architect must approve before tactical work begins |
| **Tactical** (modifies .tasks.json) | Deep, file-specific analysis — API signatures, file structures, validation checklists | `phaseA_Research` and `phaseB_ImplementationSpec` in .tasks.json | Only executes after strategic plan is approved |

**Document chain:** directive → PLAN.md (strategic analysis + PAUSE gate) → .tasks.json (tactical tasks, progress, completion log)

**Dispatch Bypass (SIMPLE tasks):** For tasks where ALL conditions are met, use general-purpose directly without prior Explore dispatch:
- Approach is an established pattern (Canvas chart, CRUD, standard UI)
- Success criteria are unambiguous and measurable
- No novel research needed beyond confirming feasibility

When in doubt, dispatch Explore first.

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
- `blocked` → `researching` (after investigating blocker, re-dispatching)
- Tasks with `state: "completed"` or `state: "blocked"` remain in .tasks.json until directive/phase completes. When a task is blocked, populate both the root-level phase `state` AND the individual task's `blockerLog` — this prevents a single blocked task from blinding the Orchestrator to the progress of other concurrent tasks within the same phase.

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

## Minimal End-to-End Example

```javascript
// 1. Orchestrator receives architect directive
directive = ""

// 2. Check PLAN.md — create if missing
plan = read_plan_if_exists() || create_plan_from_directive(directive)

// 3. STRATEGIC TIER: Dispatch Explore agent for high-level architectural research → write to PLAN.md
explore_agent = Agent(type="Explore")
strategic_findings = explore_agent.dispatch(
    task="Research architectural options for: " + directive,
    schema="Return Options, Pros/Cons, and Recommendation for PLAN.md"
)
write_to_plan_md(strategic_findings)  // Populates Architectural Options under each phase

// 4. PAUSE GATE — present PLAN.md to Architect for sign-off
//    Architect MUST approve Architectural Alignment Gate before tactical work begins

// 5. TACTICAL TIER: Decompose approved phases into .tasks.json tasks
tasks = derive_tasks(plan)  // Each: {id, phaseId, description, targetFiles, successCriteria}
write_tasks_json(tasks)

// 6. PHASE A: Dispatch Explore agent to research HOW to implement task (returns JSON matching phaseA_Research)
explore_output = explore_agent.dispatch(
    task="Research HOW to implement: " + directive,
    schema="Return JSON matching phaseA_Research schema in .tasks.json"
)

// 7. Orchestrator synthesizes findings, updates .tasks.json with phaseB_ImplementationSpec
update_tasks_json(task.id, { state: "spec_ready", phaseA_Research: explore_output })

// 8. PHASE B: Dispatch general-purpose agent to implement (receives phaseB_ImplementationSpec)
implement_agent = Agent(type="general-purpose")
result = implement_agent.dispatch(
    directive=task.phaseB_ImplementationSpec,
    termination=(MaxMessages(50), Timeout(30), TextSignal("COMPLETE"))
)

// 9. Verify result → append to progressEntries in .tasks.json
if verify(result, task.successCriteria):
    update_tasks_json(task.id, { state: "completed", progressEntries: [...existing, { ... }] })
else:
    update_tasks_json(task.id, { state: "blocked" })
```

## Operational Constraints

This section consolidates all non-negotiable guardrails: dispatch rules, termination primitives, verification standards, and failure handling. No agent operates outside these bounds.

### Agent Dispatch

- Max 4 Explore agents in parallel — avoid context fragmentation
- Single general-purpose agent per task — no parallel execution on same task
- Always include TERMINATION block in every dispatch
- Include in every dispatch: MANDATORY FIRST STEP (read CLAUDE.md), task description, success criteria, files to modify, constraints, error handling
- **All implementation dispatches must be idempotent.** Before executing file modifications, agents must read the target files to assess whether parts of the specification have already been applied by a previous partial run.

### Termination (Non-Negotiable)

Every agent dispatch MUST include:
- `MAX_MESSAGES=N` — hard stop after N responses
- `TIMEOUT=N` — soft timeout in minutes
- `TEXT_SIGNAL` — stop on `COMPLETE` (success) or `BLOCKED:` (failure)

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

For long-running tasks:
1. **Before dispatch:** update .tasks.json state (`researching` for Phase A, `implementing` for Phase B)
2. **Agent reports:** write checkpoint to .tasks.json after each major step
3. **On interrupt:** state persisted in .tasks.json, can resume
4. **On completion:** update .tasks.json with `completed` state and append `progressEntries`

**Rollback Protocol:**
1. Orchestrator detects bad state via post-agent verification
2. Do NOT dispatch fix agent until state is restored
3. Identify last known-good checkpoint or `git stash`
4. Restore state → verify → THEN dispatch fix agent
5. Log rollback event to .tasks.json progressEntries
6. **State ledger rewind:** Upon executing a Git rollback, immediately revert the corresponding task states in .tasks.json back to `pending` to ensure complete alignment with the restored codebase state.
7. **Strategic rejection wipe:** If the Architect rejects a strategic phase proposal in PLAN.md, the Orchestrator must explicitly wipe its active working memory of the rejected technical details before researching the alternative direction.

## Git Checkpoint Pattern

After completing a loop iteration, create a framework checkpoint commit:

```bash
git add CLAUDE.md PLAN.md .tasks.json
git commit -m "feat: checkpoint loop N"
```

**Exclusions:** Never include `test-project/` or other temporary test directories in commits — these are deleted at end of each loop.

**Commit format:** `feat: checkpoint loop N` — enables easy revert to any checkpoint via `git revert` or `git reset --hard`.

**Purpose:** Checkpoints allow reverting framework regressions without losing work. If a loop degrades performance, revert to previous checkpoint and try different approach.

## Commit Hygiene

**Conventional commits:** `feat:`, `fix:`, `refactor:`, `docs:`, `test:`

**One commit per logical unit of work.** Commit log tells the story:
```
feat: add scheduled publishing for documents

- Add scheduled_at column via migration
- Implement availability check on share link access
- Show "not yet available" for scheduled docs
```

## Failure-Driven Research

When blocked (error, unexpected behavior, tool failure):
1. **Stop** — no blind retries
2. **Research** — dispatch `Explore` agent to investigate
3. **Synthesize** — extract root cause and workarounds
4. **Act** — apply fix. If still blocked, refine and re-research
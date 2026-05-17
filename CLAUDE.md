# CLAUDE.md — Agentic Framework Orchestration

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, contraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect’s directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

Orchestrator (YOU) **owns analysis, planning, research dispatch, and verification**; agent **owns execution**. Orchestrator never executes research or implementation directly — it delegates both to agents, preserving context for framework tracking.

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It stores high-value project information across sessions: what we're building, why it matters, key decisions, and architecture. It's not fixed — it evolves as understanding grows. The orchestrator writes to it to answer "what are we doing here", "what are we doing now", and highlights what's important.

.tasks.json is the orchestrator's execution detail for agent delegation — exact, ephemeral task specifications derived from researching the architects directives and segments of PLAN.md. It is ALWAYS grounded in research. Each task is concrete and bounded, scoped to a single directive or plan phase with clear completion criteria. It answers "who does what, when, and how do we know it's done." **Cleared when directive or plan phase completes — not on individual task completion. Blocked tasks remain in .tasks.json for tracking and resolution.**

.tasks.json is cleared when:
- A directive is fully completed (architect goal achieved)
- A plan phase completes (all tasks in phase done or blocked)
- Explicit architect direction to clear

Blocked tasks: marked `state: "blocked"` with `blockerReason` and `researchNeeded` fields. Require additional research before re-dispatch. Never retry a blocked task without first researching the blocker.

**CRITICAL: Two-Phase Agent Dispatch — NEVER SKIP**
Each phase of creating a task requires TWO separate agent dispatches, in order:
1. **RESEARCH** — Investigates HOW to best implement before any code is written. Reads README.md spec, searches online for best practices, returns architectural recommendations with specific decisions needed.
2. **IMPLEMENT** — Executes based on research findings with full context. **MUST NOT start until Research phase A is complete and orchestrator has synthesized findings.**

PROGRESS.md: Timestamp of completion log — written by orchestrator immediately after task verification. Entries are concise, focused on the "why" and key technical decisions, not just listing what was done. Includes phase ID and files changed for traceability.

**Orchestrator (YOU)**
- Read CLAUDE.md, check PLAN.md
- If PLAN.md missing or stale → create it
- If scope warrants (orchestrator judgment or architect requests):
  - Delegate research agents to analyze project structure and READMEs
  - Delegate research agents to investigate best practices, architectural options, and tradeoffs through high quality web search
  - Research agents returns findings with recommendations
  - Synthesize research → structure complex plans into phases
- Orchestrator delegates an agent to research how best to accomplish the specific task
- Orchestrator synthesizes research aligned with plan and directive → writes to .tasks.json detailed implementation specifications
- Dispatch IMPLEMENT agent with full context (task + patterns)
- Agent executes → orchestrator does NOTHING until agent returns
- When agent returns → verify results → log to PROGRESS.md → mark completed → repeat for next phase
- Clear .tasks.json only when directive completes or architect directs

**Agent Workflow**
- For EACH step: assess state → execute → assess state → verify
- Report results to orchestrator
- Stop on blockers, await guidance

**Research is non-negotiable.** Research determines the approach, implementation just executes it. Even for small features. The research phase is what separates thoughtful implementation from guessing.

**Document chain:** directive → PLAN.md (analysis) → .tasks.json (tasks) → agent dispatch → PROGRESS.md (completion log)

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

## Orchestrator State (JSON-based)

All orchestrator state is stored in `.tasks.json`. No Python code or dataclasses.

```json
{
  "version": 1,
  "activePhase": "Phase 2: Implementation",
  "tasks": {
    "Phase-1-Task-1": {
      "id": "Phase-1-Task-1",
      "phase": "Phase 1: Foundation",
      "description": "Scaffold project with HTML/CSS/JS",
      "files": ["test-project/index.html"],
      "successCriteria": ["criterion 1", "criterion 2"],
      "state": "completed",
      "agent": "general-purpose"
    }
  }
}
```

**State Transitions:**
- `pending` → `in_progress` (when agent dispatched)
- `in_progress` → `completed` (when verified)
- `in_progress` → `blocked` (on error or dependency failure)
- `blocked` → `in_progress` (after researching blocker, re-dispatching)
- Tasks with `state: "completed"` or `state: "blocked"` remain in .tasks.json until directive/phase completes

**Blocked Task Handling:**
When a task is blocked:
1. Record `blockerReason: "<what failed>"`
2. Record `researchNeeded: "<what needs investigation>"`
3. Do NOT re-dispatch until research agent investigates the blocker
4. After research, update task with new `research` findings, then re-dispatch

**Orchestrator MUST:**
1. Update `.tasks.json` when dispatching an agent (`in_progress`)
2. Update `.tasks.json` when task is verified complete (`completed`)
3. Track `activePhase` to know current phase
4. Use `agent` field to track which agent type was used

**Delegation Tracking:**
- Always record which agent type was used for each task
- This enables analysis: "Explore for research, general-purpose for implementation"
- Pattern: Two-phase = Explore + general-purpose per phase

## Task ID Convention

Use descriptive hyphenated IDs (e.g., `Phase-1-Task-1`, `Phase-2-Task-3`) instead of numeric IDs. TaskCreate auto-generates numeric IDs which don't map to .tasks.json entries. The Orchestrator derives descriptive IDs from phase + task name context.

**Optional Simple State:** Use `.tasks.json` for lightweight state tracking (alternative to TaskCreate/TaskUpdate tools when simple file-based state is preferred):

```json
{
  "version": 1,
  "activePhase": "Phase 2: Implementation",
  "tasks": {
    "Phase-1-Task-1": { "state": "completed" },
    "Phase-2-Task-1": { "state": "in_progress" }
  }
}
```

- **Hierarchical (default):** Orchestrator decomposes directive into phases/tasks, delegates to agents.
- **Sequential:** Phases execute in order (1A→1B→2A→2B). Used when later phases depend on earlier results.
- **Parallel:** Independent tasks dispatched concurrently. Orchestrator waits for all before synthesizing.

**Pattern selection:**
```
if tasks have NO dependencies → Parallel
if tasks have dependencies AND order matters → Sequential
if orchestrator needs to decompose/make routing decisions → Hierarchical (default)
```

**Configuration:**
- `MAX_PARALLEL_EXPLORE = 3` — max concurrent Explore agents (avoid context fragmentation)
- `AGENT_OUTPUT_TOKEN_LIMIT = 2000` — truncate/summarize if exceeded
- `MAX_CONTEXT_SNAPSHOTS = 10` — trim oldest when exceeded

## Minimal End-to-End Example

```javascript
// 1. Orchestrator receives architect directive
directive = "Add user authentication"

// 2. Check PLAN.md — create if missing
plan = read_plan_if_exists() || create_plan_from_directive(directive)

// 3. Derive tasks from plan → write to .tasks.json
tasks = derive_tasks(plan)  // Each: {id, phase, description, files, successCriteria}
write_tasks_json(tasks)

// 4. Dispatch Explore agent to research approach
explore_agent = Agent(type="Explore")
explore_output = explore_agent.dispatch(
    task="Research HOW to implement: " + directive,
    files=plan["relevant_files"],
    schema=RESEARCH_OUTPUT_SCHEMA
)

// 5. Orchestrator synthesizes findings and updates .tasks.json
update_tasks_json(task.id, { state: "in_progress", research: explore_output })

// 6. Dispatch general-purpose agent to implement
implement_agent = Agent(type="general-purpose")
result = implement_agent.dispatch(
    task=task.description,
    files=task.files,
    success_criteria=task.successCriteria,
    termination=(MaxMessages(50), Timeout(30), TextSignal("COMPLETE"))
)

// 7. Verify result → log to PROGRESS.md
if verify(result, task.successCriteria):
    append_to_progress_md(completed_task)
    update_tasks_json(task.id, { state: "completed" })
else:
    update_tasks_json(task.id, { state: "blocked" })
```

## Operational Constraints

**Agent dispatch:**
- Max 3 Explore agents in parallel — avoid context fragmentation
- Single general-purpose agent per task — no parallel execution on same task
- Always include TERMINATION block in dispatch (MaxMessages, Timeout, TextSignal)

**Context management:**
- If agent output exceeds 2000 tokens → agent must summarize before returning
- Orchestrator trims context snapshots when >10 items accumulated
- On context limit: agent checkpoints to .tasks.json, reports BLOCKED

**Failure handling:**
- Never retry a failed agent without researching the failure first
- On agent BLOCKED: research root cause → synthesize → dispatch fix agent
- Concurrency: agents must not call other agents' tools concurrently — serialize handoffs

## Agent Dispatch Instructions

Include in every dispatch:
1. **MANDATORY FIRST STEP** — instruct agent to read CLAUDE.md for patterns
2. **Task description** — what to accomplish
3. **Success criteria** — specific, measurable
4. **Files to modify** — exact paths
5. **Constraints** — scope boundaries, what not to change
6. **Error handling** — stop and report on unexpected behavior

**RESEARCH agents:** Return findings per Research Output Schema above.
**IMPLEMENT agents:** See IMPLEMENT Agent Template below.

## Orchestrator Post-Agent Verification

After agent returns, orchestrator MUST verify:
1. All success criteria met? Verify against agent's criterion-by-criterion report
2. All specified files modified correctly? **Read files — verify content matches expected implementation**
3. No unexpected side effects? Check related files remain unchanged
4. Checkpoint written if applicable?
5. PROGRESS.md entry written?

If verification fails: mark task blocked with `blockerReason`, specify what's wrong.

## Research Output Schema

All research agents MUST return findings using this schema:

```
## Research Findings: [Phase Name / Directive]

### APPROACH
Recommended implementation approach with rationale.

### DECISIONS
- Decision 1: [options] → recommended
- Decision 2: [options] → recommended

### CONTEXT FOR IMPLEMENTATION
- **File structure:** [specific file paths and their purposes]
- **API/Pattern:** [specific function signatures, class names, method names to use]
- **Data flow:** [how components connect, pass data, subscribe to updates]
- **Key gotchas:** [common mistakes, edge cases, things to avoid]
- **Verification checklist:** [specific things to verify exist in the implementation]

### RISKS
- Risk 1: [mitigation]
- Risk 2: [mitigation]

### FILES_AFFECTED
- path/to/file1
- path/to/file2

### VERIFICATION
- [Specific test/verification method 1]
- [Specific test/verification method 2]
```

The **CONTEXT FOR IMPLEMENTATION** section is required — this is what the implement agent needs to actually execute, not just high-level recommendations.

### IMPLEMENT Agent Template

```
You are implementing [Task / Directive].

MANDATORY FIRST STEP: Read CLAUDE.md to understand:
- Orchestrator owns planning/research dispatch/verification; agent owns execution
- Two-phase dispatch: RESEARCH (investigate HOW) before IMPLEMENT (execute)
- Verification-first: give agents specific, measurable success criteria
- Snapshot: read file before → modify → read file after → verify
- Termination: MaxMessages(N) OR Timeout(N) OR TextSignal("COMPLETE")

YOUR TASK: [task description]

SUCCESS CRITERIA:
- [specific measurable criterion 1]
- [specific measurable criterion 2]

FILES TO MODIFY: [path/to/file1], [path/to/file2]

CONSTRAINTS: [what not to change, scope boundaries]

TERMINATION:
- Max Messages: [N]
- Timeout: [N minutes]
- Signal: "COMPLETE" or "BLOCKED: <reason>"

SNAPSHOT RULE: Read file before → verify content after.

### Self-Diagnosis Before Returning
1. All success criteria addressed?
2. All files modified as specified?
3. No unexpected side effects?

REPORT: Return structured table:
| Criterion | Expected | Actual | Status |

If blocked:
BLOCKED: <one-line summary>
CONTEXT: <what agent knows>
OPTIONS: <alternatives considered>
PREFERENCE: <recommended path>
```

## Verification-First Pattern

**Give agents specific, measurable success criteria before execution.**

| Vague | Specific |
|-------|----------|
| "implement email validation" | "validateEmail returns true for user@example.com, false for user@.com" |
| "add scheduling feature" | "documents with future scheduled_at show 'not yet available'" |
| "add human-readable IDs" | "document at /view.php?id=welcome-2026 renders correctly; ID is 6-12 chars" |

**Verification methods:**
- **File-based:** Read file before → modify → read file after → verify content
- **Browser-based:** Use `browser_snapshot` tool to capture page state before and after

## Agent Contract

### Termination
Every dispatch MUST include:
- `MAX_MESSAGES=N` — hard stop after N responses
- `TIMEOUT=N` — soft timeout in minutes
- `TEXT_SIGNAL` — stop on "COMPLETE" or "BLOCKED:"

### Feedback
BLOCKED format:
```
BLOCKED: <one-line summary>
CONTEXT: <what agent knows>
OPTIONS: <alternatives considered>
PREFERENCE: <recommended path>
```

### Context Limits
1. Self-diagnose: tokens remaining vs. task needs
2. If insufficient: checkpoint state, report BLOCKED
3. Never silently drop context — orchestrator decides: truncate, split, or escalate

### Pause and Confirm
For actions requiring explicit approval before proceeding:
```
PAUSE: <one-line summary>
REASON: <why human/input is needed>
OPTIONS: <option A / option B / abort>
```
BLOCKED means "cannot continue"; PAUSE means "could continue but want confirmation first."

## Checkpoint Pattern [OPTIMIZABLE]

For long-running tasks:
1. **Before dispatch:** update .tasks.json with `in_progress` state
2. **Agent reports:** write checkpoint to .tasks.json after each major step
3. **On interrupt:** state persisted in .tasks.json, can resume
4. **On completion:** update .tasks.json with `completed` state

```
## CHECKPOINT: Phase N, Step M
- Task ID: X
- Phase: Y
- Step number: Z
- Files modified: [...]
- Blocker status: none/BLOCKED with summary
```

**Rollback Protocol:**
1. Orchestrator detects bad state via Orchestrator Post-Agent Verification
2. Do NOT dispatch fix agent until state is restored
3. Identify last known-good checkpoint or `git stash`
4. Restore state → verify → THEN dispatch fix agent
5. Log rollback event to PROGRESS.md

## Git Checkpoint Pattern

After completing a loop iteration, create a framework checkpoint commit:

```bash
git add CLAUDE.md PLAN.md .tasks.json PROGRESS.md
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

## Success Criteria

- Every task logged to PROGRESS.md before marking completed in .tasks.json
- One clear summary per logical unit of work
- PROGRESS.md tells a coherent story of project progress
- .tasks.json updated immediately on task state change
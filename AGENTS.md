# AGENTS.md — Harness Echelon

> Implementation waits for the Architect's seal.

> **Portable:** This document defines the orchestration workflow — the rules, roles, and processes for agentic task execution. It is project-agnostic and can be dropped into any codebase. Project-specific context lives in PLAN.md; granular active execution state lives in .tasks.json; compact completion history lives in root log.md.

## Orchestration Philosophy

The Architect (USER) is the human source of truth and ultimate authority. They provide the foundational directive, vision, goals, constraints, priorities, and success criteria. This is the most important context for the entire system. The Orchestrator must treat the Architect's directive as immutable unless explicitly updated, always anchoring every analysis, research effort, plan, and decision back to it.

Research breadth and implementation simplicity are complementary. Fan strategic and blocker research out across available Explore capacity when independent questions can reduce uncertainty, but implement the smallest safe end-to-end slice that satisfies the current approved acceptance criteria. Every proposed process, dependency, adapter, protocol, configuration surface, persistent field, compatibility path, or other machinery must be necessary for a current criterion and justified against the smallest proven alternative; remove it from the plan when it is not.

AGENTS.md is Architect-owned policy. The Orchestrator and all delegated agents treat it as read-only during a directive unless the Architect explicitly requests a rules change. The Orchestrator's framework-maintenance write authority is limited to PLAN.md, `.tasks.json`, and root `log.md`.

Orchestrator (YOU) **owns the orchestration loop** — analysis, planning, agent dispatch, verification acceptance, and routing. It may read the active project and framework context, is the sole runtime writer of PLAN.md, `.tasks.json`, and root `log.md`, and decomposes governed work as its PLAN.md approval subjects are approved. It never writes product source, tests, configuration, package metadata, or generated artifacts; it dispatches implementation agents for that work. It also never investigates files, services, repositories, documentation, or other information outside the named project workspace: that work is delegated to a read-only Explore agent. Delegated verification establishes the observable facts; the Orchestrator visibly compares that evidence with the task, plan, and current dependencies and alone decides whether framework state may advance. The Orchestrator owns `.tasks.json` state transitions and distills AGENTS.md rules into targeted constraints for each agent dispatch.

Agent **owns execution of delegated tasks** — the Orchestrator dispatches an agent for a specific research, implementation, or read-only verification task, and the agent executes it autonomously within its scope. Each agent operates in an ephemeral, scoped context: it receives only its strategic-question brief or active task object, the needed plan evidence or response-schema fragment, success criteria, relevant files, and targeted operational constraints. Explore agents perform read-only research or verification, including any required external investigation. Phase B implementation agents receive `phaseB_ImplementationSpec` and return a terminal signal plus criterion-by-criterion evidence, the exact checks run and their observed results, and any deviations. Agents return exactly one of `COMPLETE`, `BLOCKED:`, or `PAUSE:` and carry no state between dispatches. Agents do not write AGENTS.md, PLAN.md, `.tasks.json`, or root `log.md`, do not read the whole ledger or history by default, and do not coordinate other agents — the Orchestrator owns policy routing, acceptance, and shared-state writes.

PLAN.md is the orchestrator's living document — in service of the architect's directive built in collaboration with the architect. It is the **Shared Mental Model** and architectural firewall between the Human Architect and the AI Orchestrator. It serves three roles:
- **Strategic Intent Mapping:** Translating the architect's raw directive into hard architectural boundaries (Scope, Phase sequences, Key Decisions).
- **High-Level Agentic Research:** The Orchestrator dispatches research agents to probe the codebase and surface trade-offs, then synthesizes their returned options, pros/cons, evidence, and recommendations into PLAN.md for human review.
- **The Pause & Confirm Gate:** PLAN.md approval is scoped by its existing `Approval subject` rows. As each decision or phase slice is approved, the Orchestrator may decompose only the tasks governed by that approved subject. A task may implement only when every plan decision that can affect it is approved and current; unrelated pending subjects do not block it.

**.tasks.json** is the **single source of truth for active, non-completed orchestrator execution state** and dispatch briefs. It is a forward-looking work queue, not a historical ledger:
- **Active task queue:** decomposing approved PLAN.md phases into actionable unfinished tasks with exact completion criteria, dependencies, and write boundaries
- **Dispatch briefs:** core tasks carry current plan evidence and a complete task-scoped `phaseB_ImplementationSpec`. `phaseA_Research` and `blockerLog` are optional state attached only while targeted research or a blocker is active, then removed after synthesis.
- **Active progress:** `progressEntries` records only events that change how unfinished work must be routed, recovered, or resolved. Ordinary major-step progress stays in the agent session. Completed-task entries are compacted into a closure summary and then removed with the task.

**log.md** is a compact completion record, not a second task ledger. After the Orchestrator accepts independent verifier evidence, record a short closure note — outcome, material research decision if any, implementation summary, verification result, and meaningful blocker resolution — then remove the completed task and its progress from `.tasks.json`. Never copy raw task JSON, dispatch telemetry, full research, file inventories, or full success criteria into the log. Blocked, paused, and other unfinished work stays active in `.tasks.json`.

Additional schema fields used by the orchestrator:
- `_persistentState`: immutable schema template that survives directive completion; the runtime section resets, the template does not
- `planPhaseRef`: explicit back-reference linking a .tasks.json entry to a PLAN.md Phase Plan row
- `targetFiles`: the authoritative complete product write set for the task. Paths must resolve inside the named product workspace and must exclude AGENTS.md, CLAUDE.md, PLAN.md, `.tasks.json`, and root `log.md`. Writers may not edit outside this set; include new files before dispatch rather than repeating paths inside the implementation spec.
- `dependsOn`: task IDs whose Phase B implementation must complete before this task's Phase B may start. It never gates independently answerable read-only research.
- `conflictsWith`: task IDs that share target files, contracts, generated outputs, manifests, or configuration and therefore require serialized execution
- `parallelSafe` / `sharedSurfaces`: explicit admission fields for a second writer. `parallelSafe` defaults to `false`; it becomes `true` only when the Orchestrator has shown there is no dependency relationship and no shared writable surface
- `researchBasis`: strategic or targeted evidence. Every `planDecisionRefs` entry uses the exact `<plan-version>/<stable-approval-subject-id>` recorded in PLAN.md; a later unrelated plan version does not invalidate an unchanged approved subject. The basis also carries materially relied-on source files, upstream assumptions, and status (`provisional`, `current`, or `stale`). `provisional` is reserved for shells still awaiting a governing approval or known freshness resolution. Approved evidence becomes `current` when the task brief is synthesized without requiring another read when no freshness trigger exists; `stale` always blocks dispatch. A targeted Phase A result supplements this basis; it is not required for every task.
- `lookaheadPhases`: downstream task shells from the Architect-approved PLAN.md. They hold unfinished planned work, not a pre-built historical archive of repeated research or progress
- `optionalTaskState`: response shapes for transient `phaseA_Research` and `blockerLog`; these are not copied into ordinary task shells
- `blockerLog`: attached while a task is blocked, researching that blocker, or paused from either state. `requiresPlanRevision: true` means the task-scoped blocker event must identify the affected `planDecisionRefs`; only those approval subjects return to `PENDING`. Remove the payload only after its resolution is synthesized.
- `activeAgents`: live registry of currently-open agent handles. Each entry records the agent id/name, the task ID or descriptive strategic-question ID it serves, phase (root, lookahead id, or `strategic`), state (`researching` / `implementing` / `verifying`), and dispatch timestamp. Add an entry on dispatch; remove it on retire. This is the Orchestrator's source of truth for true concurrency consumption and **must be consulted before any new dispatch** — it is what prevents completed-but-unclosed handles from silently consuming slots.

.tasks.json is updated continuously between orchestration events. It is reset or promoted only when:
- A directive is fully completed and all phase-level acceptance criteria pass
- A plan phase has every governing approval subject approved, every resulting task accepted and closed, and its aggregate verifier evidence accepted; blocked tasks remain active and prevent phase completion
- The Architect explicitly directs a state change

Blocked tasks: marked `state: "blocked"` with `blockerLog.reason` and `blockerLog.researchNeeded` populated. Require additional research before routing the failed path. Dispatch another writer only when the research resolves a precondition or materially changes the product state or implementation brief. If the product result remains valid and only verification evidence or environment changed, route directly to read-only verification; otherwise keep the task blocked and present the evidence and options to the Architect.

**Evidence-First Dispatch — Plan, then Implement**
Every implementation task requires current evidence, but not a dedicated research agent. Approved PLAN.md subjects supply the normal evidence basis; their governed tasks are decomposed and sequenced as those subjects are approved.

1. **Strategic research** — Before approval, fan out bounded Explore agents across independent architectural questions, trade-offs, risks, and external facts. Synthesize only the decision-relevant evidence and selected rationale into PLAN.md.
2. **Implementation** — A writer receives the task's plan evidence, complete narrowly scoped implementation brief, target files, and verification checklist. Reading its target files to perform the required idempotent preflight is implementation preparation, not a new research phase.
3. **Targeted Phase A research** — Dispatch only when an external fact, a material local uncertainty, source drift, or a blocker could change the task, its acceptance criteria, or its sequence. Return `phaseA_Research`, synthesize its decision-relevant result into `researchBasis` and `phaseB_ImplementationSpec`, then remove the transient research payload.

**Research dispatch policy:**
- **Strategic plan discovery:** Use the available Explore capacity for independent questions that can reduce architectural uncertainty. Synthesize findings into PLAN.md, surface unresolved decisions, and request approval for each decision or phase slice when it is ready. Approved independent slices may decompose and execute while unrelated subjects remain in research.
- **Blocked or stale implementation:** Preserve broad fan-out for blockers when independent questions can reduce uncertainty. Revalidate affected evidence and update only affected tasks.
- **No undocumented scope expansion:** A writer may inspect its assigned target files and run specified checks. If it discovers a decision-changing unknown, it must return `BLOCKED:` rather than conduct open-ended research or broaden its write set.

**Pipelined scheduler (default during tactical execution):**

*Scheduler lanes:* the Orchestrator retains one control lane for PLAN.md, `.tasks.json`, root `log.md`, dispatch, evidence acceptance, and routing. At most eight delegated agent handles may be open at once. Up to two may be product-code writers; every remaining available handle may run a read-only Explore agent when in-scope strategic research, blocker research, or completion verification exists.

*Research fan-out (strategic and blocker only):*
- Fill available Explore slots with bounded, non-overlapping strategic questions for unresolved approval subjects or with blocker, drift, or decision-changing uncertainty. Strategic research may continue while approved work implements; do not manufacture per-task research.
- Read-only Explore agents may inspect overlapping source files. Each result records only the decision-relevant evidence, materially relied-on files, and upstream assumptions needed to update the affected plan or task.
- On each terminal research return, confirm the returning handle is still authoritative, retire it, synthesize the result into PLAN.md or the affected task's durable evidence and brief, remove the transient research payload, and dispatch any newly eligible writer before backfilling the freed Explore slot. Process unrelated results independently. Before approving a subject or marking an affected task `spec_ready`, consult `activeAgents` and wait only for open research handles that could still change that same decision. When a subject is invalidated, stop affected research, writer, and verifier handles; any later result from a retired or superseded handle is non-authoritative. If a late material result changes an implementing task, return its approval subject to `PENDING` and apply the affected-work safe-stop rule before changing its brief. Once a presented approval subject would make a writer eligible, leave one delegated handle unfilled until the Architect resolves that gate.
- Revalidate task evidence before implementation only when a known freshness trigger exists: resumed or materially delayed work, a changed dependency or upstream contract, recorded source drift, a changed governing plan subject, or a material external fact that may have changed. Otherwise dispatch the current spec directly; the writer's idempotent preflight is the normal file-state check. Delegate any needed delta investigation to a bounded read-only agent.

*Cross-phase lookahead:*
- As decision or phase subjects are approved, decompose their governed work into ordered task shells. `.tasks.json` keeps the earliest implementation-ready phase at the root and stores approved future unfinished shells under `lookaheadPhases`; they are active planned work, not research history.
- Lookahead priority is pending completion verification → current blocker research → work needed to unblock an eligible writer → nearest planned phase. Leave capacity idle when no such in-scope read-only work exists.
- No Phase B implementation starts until every decision that can affect it, its phase slice, dependencies, evidence, and writer-admission checks permit it; independently answerable Explore research is not dependency-gated. A blocker or upstream-contract change takes precedence at the next available Explore slot; identify and update only the affected active or downstream tasks. If it forces a strategic PLAN.md change, return only the affected approval subjects to `PENDING` and remove or replace stale affected briefs instead of retaining them as history.
- When every approval subject governing the current phase is approved, all resulting tasks are accepted and closed, and aggregate verifier evidence is accepted, move the earliest approved lookahead entry whose PLAN phase and task dependencies are satisfied into the root phase fields, remove that entry from `lookaheadPhases`, and recompute phase health without repeating still-current plan evidence. On resume, if that `phaseId` already occupies the root and still appears in lookahead, remove the duplicate lookahead entry and recompute rather than promoting twice.

*Phase B execution (up to two collision-safe writers):*
- Start the earliest eligible task as soon as its plan evidence and implementation brief are current. A second writer may start only when both tasks are marked `parallelSafe: true`, neither task lists the other in `conflictsWith`, they have no direct or transitive dependency relationship, the two combined sets `targetFiles ∪ sharedSurfaces` do not overlap, neither alters an upstream contract consumed by the other, and both can be verified independently. Apply the same collision and independent-verification check before starting any writer alongside a task whose product changes remain unaccepted under an active verifier. Otherwise serialize them.
- Delegated agents never write PLAN.md, `.tasks.json`, root `log.md`, or AGENTS.md. The Orchestrator may maintain its shared control files while implementation runs but never touches a writer's product-file write set.
- While one or two writers run, the Orchestrator does not idle if safe control-lane work exists. In priority order: retire terminal handles; dispatch or judge pending verification; record blockers and queue bounded blocker research; process returned research; prepare eligible independent work; record accepted compact closures and remove their tasks; and recompute root/lookahead `phaseHealth`. Wait only when none of these are available.
- If no in-scope strategic research, blocker research, or completion verification remains, idle Explore slots stay empty.
- On a Phase B writer's `COMPLETE`, retire the writer, keep the task `implementing`, and dispatch one narrow read-only Explore verifier with the task object, relevant approved plan references, writer evidence, and known concurrent write sets. The verifier inspects the scoped change and evidence and reruns only focused checks needed to resolve an evidence gap; it does not modify files. If this is already the phase's final task, include the root `phaseSuccessCriteria` in that same dispatch so focused and aggregate verification do not become two routine passes.
- On verifier return, retire its handle first. The Orchestrator must visibly judge the verifier's criterion coverage, write-set findings, observed check results, side-effect findings, current plan consistency, and dependency implications; it must not rubber-stamp `COMPLETE`. It does not normally repeat the verifier's file inspection or checks. If parallel completion made this the last remaining task but its verifier was dispatched without aggregate scope, keep it `implementing` and dispatch one aggregate-only read-only follow-up before acceptance. Only after accepting current focused and required aggregate evidence may the Orchestrator mark the task complete, record the compact closure, remove active task state, and dispatch a dependent successor.
- Never dispatch a dependent successor from unaccepted output. If evidence is merely incomplete or internally inconsistent, request one focused read-only verification follow-up tied to the concrete gap; never repeat unchanged verification. If that follow-up returns the same unresolved gap without new evidence or artifact access, move the task to `paused`, record `priorState: implementing`, `resumeRole: verifier`, and the evidence or access needed as `nextAction`, then present the concrete gap and options to the Architect instead of dispatching another equivalent verifier. Resume only after the Architect supplies a material change to that evidence, access, or governing criterion. If the verifier finds a substantive failure or the writer returns a substantive `BLOCKED:`, keep dependent implementation stopped, record the blocker, and use an available Explore slot for bounded blocker research. Independent work may continue only when it satisfies every parallel-admission condition.
- An accepted `completed` task may remain briefly in `.tasks.json` only while its closure note is being written. It must then be removed. On resume, reconcile by exact task ID: if its closure already exists, remove the transient completed task; otherwise write the closure once, then remove it. Stale completions and raw historical detail must not accumulate in the active queue.

**Orchestrator (YOU)**
- Read AGENTS.md, check PLAN.md
- If PLAN.md missing or stale → create it
- **Strategic tier:** Iteratively delegate bounded research agents, synthesize findings into PLAN.md, and resolve architectural trade-offs and open decisions with the Architect. Present each decision or phase slice when ready and no open research handle can still change it; record its approval subject before dependent tactical work proceeds.
- **Tactical tier (rolling approval):** Decompose approved decision or phase slices into ordered .tasks.json task shells with exact target files, success criteria, dependencies, and parallel-admission fields while research continues on unrelated subjects. Do not mark a task `spec_ready` until every decision that can affect it is approved; its `planDecisionRefs` match current approved PLAN.md subjects; `researchBasis.status` is `current`; and its write set, concrete nonempty success criteria, instructions, and concrete nonempty verification checklist are complete. Approved strategic evidence may become current during synthesis without a separate research dispatch or reread.
- Dispatch targeted Phase A research only for a blocker, drift, external fact, or material uncertainty that could change a task, its criteria, or its sequence. Retire and synthesize each return immediately into only the affected plan or task.
- Dispatch Phase B for the earliest eligible task. A second writer is permitted only when both tasks meet the explicit collision-safe parallel-admission rule; otherwise use one writer.
- While writers run, perform only framework maintenance: receive permitted research, update the active queue, route verification, judge returned evidence, record compact closures, and remove completed tasks. Do not duplicate implementation or verification mechanics or touch either writer's product-file write set.
- For an active implementation agent, monitor until `COMPLETE`, `BLOCKED:`, or `PAUSE:`; do not block the control lane waiting on it. Continue safe framework maintenance and available blocker research, using non-interrupting advisory probes.
- When a Phase B writer returns `COMPLETE` → retire it → dispatch one narrow read-only verifier, including aggregate phase criteria when this is already the final task → visibly accept or reject the returned evidence against the current task and plan → if parallel completion made it final without aggregate scope, obtain the one aggregate-only follow-up → on acceptance write the compact closure note to root `log.md`, remove the task and its active progress from `.tasks.json`, and dispatch the next eligible writer if capacity permits.
- When a task blocks → record the blocker immediately → dispatch bounded blocker research into an available Explore slot → continue safe independent implementation instead of idling on the blocked path.
- Keep future task shells in `lookaheadPhases`; update, add, reorder, or remove only the active and downstream tasks affected by an approved plan change.
- Keep `.tasks.json` restricted to active, non-completed work; blocked, paused, and all other non-completed tasks and progress remain there

**Two-Tier Research Architecture:**

| Tier | Scope | Output | Gate |
|------|-------|--------|------|
| **Strategic** (synthesized into PLAN.md) | High-level architectural analysis — design trade-offs, frameworks, system boundaries | Options, pros/cons, recommendation in PLAN.md | Each approval subject must be `APPROVED` before its dependent tactical work begins |
| **Tactical** (synthesized into .tasks.json) | Current plan evidence, exact write boundary, implementation brief, and verification checklist | Complete task-scoped `phaseB_ImplementationSpec`; optional transient `phaseA_Research` only for a blocker, drift, or decision-changing uncertainty | Evidence must be current; a separate research dispatch is not required |

**Document chain:** directive → PLAN.md (strategic analysis + Plan Approval) → .tasks.json (active tactical tasks and progress) → root log.md (compact completion history)

## Trust Boundaries

| Agent Type | Permissions | Restrictions |
|------------|-------------|--------------|
| Explore | `read`, `glob`, `grep`, `bash`, `webfetch` | Read-only research or verification — no file modifications |
| general-purpose | `read`, `write`, `edit`, `glob`, `grep`, `bash`, `webfetch` | Only modify specified files |
| Orchestrator | Read active project/framework context; write PLAN.md, `.tasks.json`, and root `log.md` | Accepts or rejects verifier evidence but does not normally repeat mechanical inspection or checks; does not write product files or investigate outside the named project workspace; AGENTS.md remains read-only without explicit Architect direction; never touches a writer's product-file write set |

**Available agent types:** `Explore` for read-only research, investigation, and verification; `general-purpose` for implementation.

## Orchestrator State

All active, non-completed orchestrator execution state is stored in `.tasks.json` — it is the authoritative forward work queue. Root phase fields describe the current implementation phase; `lookaheadPhases` holds downstream unfinished task shells from the approved PLAN.md. The Orchestrator delegates only task-scoped fragments. Root `log.md` holds compact completion summaries.

**State Transitions:**
- `pending` → `spec_ready` (when approved plan evidence is current and the implementation brief is complete)
- `pending` or `spec_ready` → `researching` (only when targeted Phase A research is dispatched)
- `researching` → `spec_ready` (only when the synthesized targeted result resolves the material uncertainty and no open research handle can still change the same decision)
- `researching` → `researching` (while another relevant research handle remains open)
- `researching` → `blocked` (when all relevant research has returned without resolving the uncertainty)
- `spec_ready` → `implementing` (when Phase B implement agent dispatched)
- `implementing` → `completed` (only when the Orchestrator accepts independent verifier evidence)
- any active state → `blocked` (on that task's substantive error or failure). A dependent task does not become a second blocker merely because its prerequisite is unresolved; it remains in its current state but ineligible through `dependsOn`, and becomes stale or blocked only if the prerequisite failure actually invalidates its evidence or criteria.
- any active state → `paused` (on a `PAUSE:` signal, an explicit invalidated-scope safe-stop, or a repeated verification gap requiring Architect input)
- `blocked` → `researching` (when bounded blocker research is dispatched)
- `researching` → `implementing` (only when blocker research proves the existing product result remains valid and dispatches verification without another writer)
- `paused` → prior active state (record the prior state, retired dispatch role, and next required action in a compact `progressEntries` pause event; after the Architect resolves the pause, restore the state and dispatch that recorded role only when its brief and evidence are not stale)
- An accepted task may transition briefly to `completed`; write its compact closure note to root `log.md` and remove it and its task-scoped active progress from `.tasks.json` immediately afterward. Tasks in `blocked`, `paused`, or any other non-completed state remain active. When a current-phase task is blocked, populate both the root-level phase `state` and the individual task's `blockerLog`; for a blocked lookahead task, populate its own containing phase `state` and task `blockerLog`. This prevents one blocked task from blinding the Orchestrator to other active work.
- **Mixed task states:** The root-level phase `state` is a non-gating scalar summary. Individual task state, dependencies, and writer-admission checks govern dispatch, so one blocked task does not stop demonstrably independent work. Recompute the containing phase `state` after every task transition alongside `phaseHealth`, using the highest-priority remaining condition (`blocked`, `paused`, `implementing`, `researching`, `spec_ready`, `pending`); never leave the scalar `blocked` when its blocked count is zero. For granular visibility, `phaseHealth` is the derived aggregate of only the tasks remaining in `.tasks.json` (`total`, `pending`, `researching`, `spec_ready`, `implementing`, `completed`, `blocked`, `paused`). The sole taskless exception is a validation-only phase: use phase `state: implementing` while its phase-scoped verifier handle is live, then either close the phase on acceptance or create a real blocked remediation task for an unmet criterion.
- Root `phaseHealth` covers only the current implementation phase. Every entry in `lookaheadPhases` maintains its own derived `phaseHealth`; lookahead tasks may be `pending`, `researching`, `spec_ready`, `blocked`, or `paused`, but never `implementing` until their phase is promoted.

**Signal Handling:**

Every agent dispatch terminates with one of three text signals. The orchestrator MUST handle each:

| Signal | State Transition | Orchestrator Action |
|--------|-----------------|---------------------|
| `COMPLETE` | Role-dependent | Phase A: retire the terminal handle and synthesize only the affected plan or task; remain `researching` while a relevant handle is open, move to `spec_ready` when the uncertainty is resolved, or move to `blocked` when all relevant results return without resolution. Phase B writer: retire the handle, keep the task `implementing`, and dispatch a read-only verifier. Verifier: retire the handle, visibly judge its evidence against the task and plan, then move to `completed` only on acceptance; include aggregate evidence when this is the final phase task. |
| `BLOCKED:` | Role-dependent | Context exhaustion only: record the compact checkpoint, retain the substantive state, and redispatch the same role without blocker research. Targeted research: remain `researching` while relevant handles are open, otherwise block if the uncertainty is unresolved. Writer or substantive verifier failure: populate `blockerLog` and dispatch bounded research; dispatch another writer only for a resolved precondition or changed product/brief, or return directly to `implementing` verification when the product remains valid and only evidence or environment changed. Verification evidence gap only: keep the task `implementing` and dispatch one focused read-only follow-up tied to that gap. |
| `PAUSE:` (agent-initiated) | → `paused` | Retire handle → present options to Architect → restore prior state after the decision → create a fresh dispatch |
| `PAUSE:` (gate, strategic) | → `paused` | Present a ready PLAN.md approval subject to Architect → wait for `APPROVED` → decompose only its governed tactical work |
| `SOFT_TIMEOUT` checkpoint | state unchanged | Inspect the latest agent activity and continue waiting while useful progress is being made. Elapsed wall time alone never blocks a task or justifies a replacement dispatch. |

The `paused` state is a yielding state — the task waits for Architect input. Before setting it, the Orchestrator records `priorState`, `resumeRole` (`researcher`, `writer`, or `verifier`), and `nextAction` in a compact task-scoped `progressEntries` event; after the decision it restores that state and dispatches the recorded role because the original handle was retired on `PAUSE:`. If its brief or governing evidence became stale while paused, do not restore `implementing` or dispatch the recorded role until the affected subject is reapproved and the brief is rebuilt. Retain an unresolved `blockerLog` when pausing from blocker handling.

**Blocked Task Handling:**
When a task is blocked:
1. For a current-phase task, update the root phase `state` to `blocked`; for a lookahead task, update only its containing lookahead phase `state`. Do not falsely block an unaffected current implementation phase.
2. Populate the task's `blockerLog` — `reason: "<what failed>"` and `researchNeeded: "<what needs investigation>"`. If approved architecture may change, set `requiresPlanRevision: true` and name the exact affected `planDecisionRefs` in the task-scoped blocker event.
3. Put bounded blocker research at the head of the Explore queue. Do not terminate productive running researchers; use the next available slot.
4. Identify the exact active, lookahead, and accepted-closure impact set from the recorded evidence, interfaces, and dependencies. Mark only affected evidence bases `stale`, prevent their dispatch, and leave unaffected work intact. For affected active work, request a `PAUSE:` safe-stop with compact recovery evidence, then retire the handle, route any needed read-only inspection of the partial result, and do not accept completion under the invalidated brief. Retire affected research handles as well; their late results are non-authoritative.
5. Do NOT re-dispatch substantively failed work until research investigates the blocker. Dispatch another writer only when the result resolves a precondition or materially changes the product state or implementation brief. If research proves the existing product result remains valid and only verification evidence or environment changed, set the task back to `implementing` and dispatch a read-only verifier directly. Synthesize the result into durable task state and remove the transient research/blocker payload; otherwise keep the task blocked and retain `blockerLog` while escalating the evidence and options to the Architect.
6. **PLAN.md back-link:** If a tactical block requires altering approved architecture or decisions in PLAN.md, return only the affected approval subjects to `PENDING`, remove or replace only dependent stale briefs, and trigger a strategic PAUSE for those subjects. Unaffected work may continue under the last approved plan version when its independence is demonstrable.

**Orchestrator MUST:**
1. Update `.tasks.json` state when dispatching an agent (`researching` for targeted Phase A, `implementing` for Phase B)
2. Update `.tasks.json` when independent verifier evidence is accepted (`completed`)
3. Track `phaseId` to know current phase
4. **Aggregate validation gate:** Require concrete nonempty root-level `phaseSuccessCriteria` before phase closure. Dispatch verification for every completed writer immediately. When every approval subject governing the phase is approved and decomposed and this is already the final required task, include those criteria in the same read-only verifier dispatch; otherwise dispatch focused task verification and keep the phase open. If parallel verification later makes a task the final remaining one and its report lacks aggregate coverage, dispatch one aggregate-only read-only follow-up before accepting it. If an executable phase has no implementation task, set the phase `implementing` while one phase-scoped read-only verifier evaluates `phaseSuccessCriteria`; never create a dummy writer. The Orchestrator judges aggregate evidence but does not execute the checks itself. If aggregate validation fails with a final task present, keep that task active, mark it `blocked`, populate its existing `blockerLog`, and follow Blocked Task Handling. If the validation-only phase has no task, create one real blocked remediation task for the unmet criterion and route it through the same blocker path; dispatch a product writer only if resolution actually requires a product change. If the cause belongs to earlier accepted work, insert a new repair task linked to that exact closure before the blocked aggregate anchor, make the anchor and its dependents wait on the repair through existing dependencies, and rerun aggregate verification after the repair closure is accepted. Do not restore the historical task object. Provide targeted accepted closures only when the verifier needs them.
5. Maintain `phaseHealth` and the containing phase's derived scalar `state` alongside every individual task transition.
6. After accepting verifier evidence, write each completed task's compact closure summary to root `log.md` and remove its active task and progress from `.tasks.json`; on resume, use the exact task ID to complete this transition idempotently.
7. Maintain `lookaheadPhases` as unfinished downstream task shells. Revalidate evidence before implementation and update only the tasks demonstrably affected by upstream changes.

## Directive Lifecycle

A directive progresses through four stages:

**1. Discovery and rolling approval** — PLAN.md is created or loaded from its section skeleton (see PLAN.md template). The Orchestrator populates the Architect's Directive section with the raw directive, then uses available Explore capacity for independent strategic questions, synthesizes decision-relevant findings, and resolves open architectural decisions with the Architect. Each ready decision or phase slice is recorded as an approval subject; its governed tactical work may proceed without waiting for unrelated subjects. Simple directives may approve the whole plan in one pass.

**2. Execution** — The orchestration loop: decompose each approved decision or phase slice into ordered .tasks.json task shells → dispatch the earliest evidence-ready writer → fill a second writer lane only for collision-safe independent work → continue strategic research for unresolved subjects and use tactical Explore capacity for blockers, drift, decision-changing uncertainty, or pending completion verification → delegate inspection of terminal work → visibly judge the returned evidence → write a compact closure and remove accepted work from active state → continue with the next eligible task. Do not repeat research already settled in approved subjects or duplicate verifier checks in the control lane.

**3. Completion** — When all phases pass aggregate validation:
- Confirm every completed task has a compact closure summary in root `log.md`
- Add one directive-level closure summary to root `log.md` covering delivered outcome, architecture selected, material deviations, and final verification
- Clear PLAN.md back to its section skeleton and reset `.tasks.json` runtime fields while retaining `_persistentState`
- Present the completion summary to Architect

**4. Mid-Directive Update** — If the Architect updates the directive mid-execution:
- Compare the new directive to the approved PLAN.md version
- Return only affected approval subjects to `PENDING`; request a `PAUSE:` safe-stop from affected active research, writer, or verifier handles; retire them with compact recovery evidence; and mark affected briefs or evidence stale. A later return from a retired handle is non-authoritative, while demonstrably unaffected work remains valid under its prior approved subject version
- Trigger strategic PAUSE to research and negotiate the affected subjects
- After the Architect approves the revision, update, add, reorder, or remove only affected active and downstream task shells. If the revision affects an accepted closure, add a new sequenced repair task linked to that closure and make affected current/dependent work wait on it. Rebuild stale paused briefs before restoring their recorded roles, then resume affected execution

## Task ID Convention

Record one log-unique directive ID in PLAN.md when the directive begins. Use log-unique descriptive hyphenated task IDs prefixed by that exact directive ID (for example, `Directive-A-Phase-1-Task-1`). TaskCreate auto-generates numeric IDs which don't map to .tasks.json entries. Never reuse a directive or task ID already present in root `log.md`, so an older closure cannot satisfy a new dependency accidentally.

- **Hierarchical (default):** Orchestrator decomposes directive into phases/tasks, delegates to agents.
- **Sequential implementation:** Phase B tasks execute in dependency/write-conflict order whenever they share a writable surface or contract.
- **Collision-safe parallel implementation:** Up to two Phase B writers may run only when they satisfy the explicit `parallelSafe` admission rule.
- **Parallel read-only work:** Explore agents fan out across independent strategic or blocker questions and pending completion verification, using otherwise available delegated capacity. Results are synthesized or judged independently as they arrive.

**Writer guard:** No more than two Phase B product-code implementation agents run at once. The second is opt-in, never inferred from apparent file separation, and requires the explicit collision-safe admission rule. The Orchestrator alone serializes its framework-state transitions into `.tasks.json`.

**Configuration:**
- `MAX_PARALLEL_AGENTS = 8` — max concurrent delegated agent handles, excluding the Orchestrator
- `MAX_PARALLEL_WRITERS = 2` — maximum concurrent product-code writers; the second requires collision-safe admission
- `MAX_PARALLEL_EXPLORE = MAX_PARALLEL_AGENTS - active writer handles` — maximum total read-only handles after writers; actual new-dispatch capacity is always `MAX_PARALLEL_AGENTS - activeAgents.length`
- `AGENT_OUTPUT_TOKEN_LIMIT = 4000` — truncate/summarize if exceeded

## Operational Constraints

This section consolidates all non-negotiable guardrails: dispatch rules, termination primitives, verification standards, and failure handling. No agent operates outside these bounds.

### Agent Dispatch

- Never exceed `MAX_PARALLEL_AGENTS` open delegated handles. Fill Explore capacity only with independent strategic research, blocker research, or pending completion verification; leave it idle when none exists.
- Keep Explore tasks narrow enough to finish independently; as each terminal result returns, retire its handle first and immediately backfill only when another in-scope research question or verification dispatch remains.
- Explore agents are read-only. For research, they perform any required investigation outside the named project workspace and record only decision-relevant evidence, materially relied-on files, and upstream assumptions. For verification, they inspect the task-scoped product changes and observed check evidence, trust observed artifacts over agent narration, and make no modifications.
- At most `MAX_PARALLEL_WRITERS` general-purpose Phase B product-code agents may be active. The first eligible task may use one lane; the second requires an explicit `parallelSafe` admission check covering `conflictsWith`, dependencies, transitive dependencies, non-overlap of each task's combined `targetFiles ∪ sharedSurfaces`, upstream contracts, and independently verifiable scopes. A writer starting while another task is under verification must pass the same check against that unaccepted task.
- The Orchestrator may maintain PLAN.md, `.tasks.json`, and root `log.md` while writers run because delegated product-code agents never write those framework files.
- Always include TERMINATION block in every dispatch
- Include in every dispatch: the implementation, verification, or research scope; success criteria; files to inspect or authorized `targetFiles`; relevant approved or pending plan context; and operational constraints relevant to that dispatch. A verification dispatch also receives the writer's criterion report and known concurrent write sets. The orchestrator is responsible for distilling AGENTS.md rules into targeted constraints — agents do NOT read AGENTS.md themselves.
- **All implementation dispatches must be idempotent.** Before executing file modifications, agents must read the target files to assess whether parts of the specification have already been applied by a previous partial run. On completion they report the exact checks run and observed results, not merely that checks passed.

### Agent Handle Lifecycle (Non-Negotiable)

A dispatched agent is an **open concurrency handle**, not an ephemeral logical slot. Returning a terminal signal (`COMPLETE` / `BLOCKED:` / `PAUSE:`) does **not** retire the handle — the agent continues to occupy its concurrency slot until the Orchestrator explicitly stops/closes it. The platform concurrency cap counts open handles, not active work, so completed-but-unclosed agents accumulate and will starve the writer lane.

- **Retire on terminal:** Before retirement, confirm the returning handle is still the current `activeAgents` entry for that task/question and role and that its governing plan references remain authoritative. Then retire the agent (close/stop its handle) as the **first** state-changing step, before treating its slot as free or dispatching a successor. Any queued result from a retired or superseded handle is non-authoritative. A successor must never be blocked by an agent the Orchestrator forgot to close.
- **Do not interrupt productive work:** Never terminate an actively productive research, implementation, or verification agent merely because another task becomes ready, a different result returns, or capacity would be convenient to reshuffle. Use a non-interrupting status probe at advisory checkpoints; stop an active agent only for explicit Architect direction, detected unsafe behavior, or a clear scope violation.
- **Track live handles:** Maintain the `activeAgents` registry in `.tasks.json`. Add an entry on dispatch (agent id/name, served task or strategic-question ID, phase, state, dispatch timestamp); remove it on retire. **Read this registry before every new dispatch** to know the true count of consumed slots — never reason about slot availability from memory. On resume, reconcile both registry-only and host-only handles before dispatching. For an `implementing` task with no live writer or verifier after reconciliation, dispatch one read-only recovery verifier first: complete artifacts follow normal acceptance, while incomplete or unsafe artifacts enter blocker handling. Lost targeted research is redispatched from its active brief; a lost verifier is redispatched read-only without treating the product as failed; and a lost strategic question is regenerated from its pending PLAN.md subject before that subject may be approved.
- **Backfill = retire, then fill:** Every "backfill the freed slot" instruction in this document means: retire the returning agent first, then dispatch the next into the now-actually-free slot. A slot is free **only** after its handle is closed.
- **`MAX_PARALLEL_AGENTS` bounds all open delegated handles, not just active research.** Count writers, Explore agents, and any completed agent not yet retired. Spawning a replacement while the original is still open doubles consumption and is forbidden — wait on the original or stop it explicitly; never shadow it with a second live handle.

### Termination (Non-Negotiable)

Every agent dispatch MUST include:
- `MAX_MESSAGES=N` — advisory context-budget checkpoint. Before reaching it, the agent self-assesses and returns `COMPLETE`, `BLOCKED: context exhausted`, or `PAUSE:` with a checkpoint if more context is required; it does not authorize the Orchestrator to interrupt productive work.
- `TIMEOUT=N` — advisory progress-check interval in minutes, not an automatic termination deadline
- `TEXT_SIGNAL` — stop on `COMPLETE` (success), `BLOCKED:` (substantive failure or explicit context handoff), or `PAUSE:` (awaiting architect input or an explicit safe-stop)

**Advisory checkpoint policy:** Reaching `TIMEOUT=N` or `MAX_MESSAGES=N` is advisory only — neither authorizes interrupting, closing, replacing, or classifying the agent/task as failed while useful work continues. For a long run, issue a non-interrupting status probe. Interrupt an active agent only for an explicit Architect override, detected unsafe behavior, or a clear scope violation. Append a `timed_out` progress event only when the runtime reports a terminal error or the worker is demonstrably unresponsive with no recoverable result; `timed_out` is an event, not a task or agent state. Retire the handle and route by role: a lost writer receives the recovery-verifier inspection before incomplete or unsafe artifacts enter blocker handling, lost research is redispatched from its active brief, and a lost verifier is redispatched read-only without classifying the product as failed. An advisory checkpoint alone never justifies a replacement dispatch.

**BLOCKED format:**
```
BLOCKED: <one-line summary>
CONTEXT: <what agent knows>
OPTIONS: <alternatives considered>
PREFERENCE: <recommended path>
```

If a writer modified files before returning `BLOCKED:`, its `CONTEXT` must name every touched file and classify it as partial, complete, or unchanged from preflight. `BLOCKED: context exhausted` additionally supplies the compact continuation checkpoint and is not a substantive failure by itself.

**PAUSE format (for actions needing human confirmation or an explicit safe-stop):**
```
PAUSE: <one-line summary>
REASON: <why human/input or a safe-stop is needed>
OPTIONS: <option A / option B / abort>
```
BLOCKED = "cannot continue"; PAUSE = "must yield for confirmation or because the governing scope was invalidated."

### Snapshot Rule (Non-Negotiable)

**Read file before → modify → read file after → verify.** This is the single most effective guardrail against silent file corruption. Every implementation agent must follow this pattern on every file modified.

### Delegated Verification and Orchestrator Acceptance

**Give writers specific, measurable success criteria and a focused verification checklist before execution.** A writer's `COMPLETE` report must map observed evidence to every criterion, list the exact files changed, give each check command or method and its observed result, and disclose deviations or unresolved concerns.

The Orchestrator retires the terminal writer and dispatches one narrow read-only Explore verifier. The verifier receives the task object, relevant approved plan references, writer report, and known concurrent write sets. It independently:
1. Inspects the task-scoped diff and resulting target files against `phaseB_ImplementationSpec` and every success criterion.
2. Confirms the observed writer checks and reruns only the focused deterministic checks needed when evidence is missing, ambiguous, or unsafe for a successor.
3. Checks for unexpected changed files or relevant side effects while distinguishing known concurrent writer scopes.
4. For a phase's final task, evaluates the root `phaseSuccessCriteria` in the same dispatch; when parallel verification made it final only after dispatch, evaluates them in one aggregate-only follow-up.
5. Returns `COMPLETE` with compact criterion-by-criterion evidence tied to exact file/symbol or command-result references, write-set findings, side-effect findings, and aggregate findings when applicable; otherwise returns `BLOCKED:` with the concrete evidence gap or failure.

The verifier establishes observable facts; the Orchestrator decides whether those facts justify changing framework state. The Orchestrator MUST visibly:
1. Compare the verifier report with every task criterion, checklist item, currently governing plan decision reference, and known dependency or parallel-writer implication. Evidence bound to a stale subject or superseded handle is non-authoritative: return the affected subject/task to the existing revision path and do not treat it as an ordinary verification gap.
2. Reject incomplete, internally inconsistent, unexplained, or plan-conflicting evidence; never accept a bare terminal signal or a writer's narrative as proof.
3. Avoid routinely rereading every product file or rerunning verifier checks. Direct project inspection is limited to the smallest adjudication needed when evidence conflicts or global architectural context is decisive; outside-project investigation is always delegated.
4. When the issue is only missing or ambiguous evidence, record the concrete gap in `progressEntries` while it remains active and request one focused read-only verification follow-up that never repeats unchanged work. If it returns the same unresolved gap without new evidence or access, move the task to `paused` with its prior state, verifier role, and required evidence/access recorded, then present the gap and options to the Architect. A substantive implementation or aggregate failure enters Blocked Task Handling and blocker research.
5. Only on acceptance, briefly mark the task `completed`, record its compact closure under the exact log-unique ID, remove its active task and progress, recompute `phaseHealth`, and permit dependent successors. On resume, an existing exact-ID closure completes this transition without another append. A dependency absent from active state is satisfied only by that matching accepted closure.

Read-only verifiers may run concurrently within the eight-handle cap, and unrelated implementation may continue. Work depending on a completed writer waits for Orchestrator acceptance. Parallel completions do not trigger an automatic third combined pass; the only phase-level exception is the aggregate-only follow-up required when the last remaining task's already-running verifier lacked aggregate scope. Otherwise request a combined check only when a successor's current criterion or dependency makes the independently accepted evidence insufficient. No probabilistic scoring, repeated evaluation, best-of-N comparison, or multi-verifier tournament is required; any follow-up verification dispatch must be justified by a concrete evidence gap.

### Context Management

- If agent output exceeds `AGENT_OUTPUT_TOKEN_LIMIT` (4000 tokens) → agent must summarize before returning
- On context limit: the agent returns `BLOCKED: context exhausted` with compact checkpoint content; the Orchestrator writes one task-scoped recovery event, retains the substantive task state, and redispatches the same role without blocker research
- Agents self-diagnose tokens remaining versus task needs. If insufficient, checkpoint for a fresh scoped continuation; only a separate product, evidence, or decision failure enters blocker handling
- **Sanitation primitive:** When dispatching sub-agents, pass only the strategic-question brief or active task object plus needed schema fragments and constraints. Delegated agents do not read the whole `.tasks.json` ledger, `_persistentState`, or root `log.md` by default; load targeted log history only when actually needed.
- **Lookahead sanitation:** A lookahead research agent receives only its own task, relevant upstream assumptions, and required PLAN.md context. It does not receive current-phase implementation telemetry or unrelated lookahead tasks.
- **Active-queue threshold:** If root or lookahead `progressEntries` exceeds 15 entries during an active phase, compact or remove details that are no longer needed to route or recover unfinished work. On completion, retain only the compact closure summary in root `log.md`.

### Failure Handling

- Never re-dispatch substantively failed work without researching the failure first, and never repeat the same failed implementation attempt without a materially revised brief or resolved precondition. A context-exhaustion continuation is not failed work.
- On writer or substantive verifier `BLOCKED:` record the impact hypothesis → give bounded blocker research the next available Explore slot → synthesize the fix and affected active/downstream-spec set → invalidate only that set → dispatch the fix agent when safe. A verifier reporting only missing or ambiguous evidence follows the focused verification-gap path instead.
- Agents never dispatch or coordinate other agents. The Orchestrator may concurrently run up to two collision-safe Phase B writers and use remaining delegated capacity for strategic research, blocker research, or read-only verification.
- **Blocker events and their resolutions must be recorded as standalone entries in `progressEntries`** while they affect unfinished work. Summarize only the material resolution in the task or directive closure note.

## Checkpoint Pattern

**For long-running tasks:**
1. **Before dispatch:** update .tasks.json state (`researching` for targeted Phase A, `implementing` for Phase B)
2. **During execution:** keep ordinary major-step progress in the agent session. Write a compact `progressEntries` event only when unfinished-work routing or recovery changes: pause, blocker, context exhaustion, lost handle, safe-stop, or unresolved verification gap.
3. **On interrupt:** record the prior state, retired role, compact recovery evidence, and next action so a fresh scoped dispatch can resume correctly.
4. **On completion:** after the Orchestrator accepts independent verifier evidence, briefly update `.tasks.json` with `completed`, write the compact closure note to root `log.md`, and remove the task and its active progress before dispatching a dependent successor.

**Rollback Protocol:**
1. A verifier reports bad state, or the Orchestrator rejects contradictory evidence.
2. Mark the affected task `blocked`, prevent dependent dispatch, and record the impact in task-scoped `progressEntries`.
3. Use Blocked Task Handling to research the failure and identify the last known-good checkpoint and affected set.
4. Only after the blocker is resolved, dispatch a scoped implementation agent when the product state or implementation brief must change. If research proves the product remains valid and only evidence or environment changed, dispatch read-only verification directly. If aggregate validation implicates earlier accepted work, create a new repair task linked to its exact closure, order the blocked aggregate anchor after that repair, and rerun aggregate verification after the repair is accepted instead of rehydrating historical state.
5. Delegate focused verification of the restored result and accept it through the same evidence gate.
6. **Strategic rejection wipe:** If the Architect rejects a plan proposal or revised architecture in PLAN.md, the Orchestrator must explicitly wipe its active working memory of the rejected technical details before researching the alternative direction.

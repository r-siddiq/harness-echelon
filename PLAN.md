# PLAN.md — Strategic Plan

> **Project-specific living document.** Captures the architect's directive, architectural boundaries, research findings, and phase sequencing for the current engagement. The orchestrator populates and maintains this document in collaboration with the architect. Section skeletons persist on directive completion; content is cleared for the next engagement. Project-specific context lives here; granular active execution state lives in `.tasks.json`; completed-task history lives in root `log.md`.

---

## Architect's Directive

*[Raw directive from the architect — vision, goals, constraints, priorities, and success criteria. This is the immutable anchor for all downstream work. Updates only via explicit architect instruction. Include decision authority, required evidence, release or acceptance conditions, and any standing approval wording that gates execution.]*

### Objectives and decision posture

*[List the first-class objectives and their priority or trade-off authority. Define measurable qualification criteria, evidence requirements, and which thresholds or decisions may change only through measured validation and Plan Approval.]*

---

## Scope and Architectural Boundaries

*[What is in scope, out of scope, hard boundaries, and non-negotiables. Derived from the directive and refined through strategic analysis.]*

### In scope

*[...]*

### Boundaries

*[Hard constraints: authority placement, supported and excluded capabilities, compatibility requirements, safety or security boundaries, test-only mechanisms, and evidence authority.]*

---

## Selected Solution Architecture

*[Populated by the orchestrator from strategic research-agent findings before tactical work begins. Each architectural decision area includes: evidence, options explored, pros/cons, and a recommendation. This section is the research grounding for the phase plan below. Strategic selection is not implementation authorization — no task may be decomposed or executed while Plan Approval is PENDING.]*

---

## Implementation Phases and Sequencing

*[Sequence of phases decomposed from the directive. Each phase is a discrete unit of work with a clear goal and completion criteria. After Plan Approval, every listed phase is seeded into .tasks.json as the root active phase or lookahead state. Research may run in parallel; Phase B implementation remains ordered by phase, dependencies, and conflicts.]*

| # | Phase | Execution goal | Depends on | Status |
|---|-------|----------------|------------|--------|
| - | -     | -              | -          | -      |

---

## Architect Plan Approval

*[Whole-plan approval tracker. Status is PENDING until the Architect approves the complete blueprint and phase sequence, then APPROVED. The orchestrator MUST NOT proceed to tactical decomposition while PENDING. Approval is bound to a recorded plan version. A later directive or architectural change creates a revised PENDING version, pauses affected execution, and stales only dependent task research; demonstrably unaffected work remains valid under the prior approved version.]*

| Plan version | Status | Approval subject | Recorded |
|--------------|--------|------------------|----------|
| -            | -      | -                | -        |

---

## Active Architecture Decisions

*[Significant decisions made during the engagement, with their execution consequence. Provides an audit trail for why certain paths were chosen. Changing a locked decision creates a revised plan version whose approval remains PENDING until the Architect accepts it.]*

| ID | Locked decision | Execution consequence |
|----|-----------------|-----------------------|
| -  | -               | -                     |

---

## Risks and Blockers

*[Active blockers requiring architect attention or strategic resolution. Populated when a tactical block in .tasks.json has architectural implications beyond the individual task. Status: CONTROLLED (mitigation in place) or OPEN (requires control).]*

| Phase | Risk or blocker | Required control | Status |
|-------|-----------------|------------------|--------|
| -     | -               | -                | -      |

---

## Acceptance Criteria

*[Measurable, verifiable success criteria grouped by area. Thresholds and targets are qualification criteria, not unconditional guarantees; a target may change only through measured qualification and Plan Approval of the revised blueprint.]*

### Control and durability

- *[...]*

### Correctness

- *[...]*

### Operational quality and evidence

- *[...]*

### End-to-end validation

- *[...]*

### Cleanup and final release

- *[...]*

---

## Held Boundaries

*[Approaches explicitly rejected or held pending the relevant evidence and Plan Approval.]*

- *[...]*

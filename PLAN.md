# PLAN.md — Strategic Plan

> **Project-specific living document.** Captures the architect's directive, architectural boundaries, research findings, and phase sequencing for the current engagement. The orchestrator populates and maintains this document in collaboration with the architect. Section skeletons persist on directive completion; content is cleared for the next engagement. Project-specific context lives here; granular active execution state lives in `.tasks.json`; completed-task history lives in root `log.md`.

---

## Architect's Directive

*[Raw directive from the architect — vision, goals, constraints, priorities, and success criteria. This is the immutable anchor for all downstream work. Updates only via explicit architect instruction. Include the qualification posture: performance/accuracy trade-off authority, release evidence requirements, and any standing approval wording that gates execution.]*

### First-class objectives and qualification posture

*[Performance and accuracy are co-equal first-class objectives unless the directive says otherwise. State the accuracy-over-speed authority, the latency reporting granularity, the rule that thresholds are qualification criteria (not guarantees) changeable only through measured qualification and the applicable Architect gate, and the release-evidence requirements.]*

---

## Scope and Architectural Boundaries

*[What is in scope, out of scope, hard boundaries, and non-negotiables. Derived from the directive and refined through strategic analysis.]*

### In scope

*[...]*

### Boundaries

*[Hard constraints: identity discovery rules, authority placement, supported surfaces, excluded mechanisms, test-only tooling boundaries, and evidence authority.]*

---

## Selected Solution Architecture

*[Populated by orchestrator's strategic research agents before tactical work begins. Each architectural decision area includes: options explored, pros/cons, and a recommendation. This section is the research grounding for the phase plan below. Strategic selection is not implementation authorization — no task may be decomposed or executed while its alignment gate is PENDING.]*

---

## Implementation Phases and Sequencing

*[Sequence of phases decomposed from the directive. Each phase is a discrete unit of work with a clear goal and completion criteria. The active phase is tracked in .tasks.json. Phases may be decomposed for parallel execution only when their approved tasks have non-overlapping target files.]*

| # | Phase | Execution goal | Depends on | Status |
|---|-------|----------------|------------|--------|
| - | -     | -              | -          | -      |

---

## Architect Alignment Gates

*[Per-phase approval tracker. The orchestrator MUST NOT proceed to tactical decomposition (.tasks.json) for any phase until its gate is APPROVED. Approval is bound to the reviewed plan content hash; a later architecture change returns only the affected gates to PENDING before decomposition or implementation. If a tactical block requires altering the high-level architecture, the affected gate reverts to PENDING and a strategic PAUSE is triggered.]*

| Phase | Status | Approval subject | Recorded |
|-------|--------|------------------|----------|
| -     | -      | -                | -        |

---

## Active Architecture Decisions

*[Significant decisions made during the engagement, with their execution consequence. Provides an audit trail for why certain paths were chosen. Changing a locked decision requires the affected phase gate to remain or revert to PENDING.]*

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

*[Measurable, verifiable success criteria grouped by area. Thresholds and targets are qualification criteria, not unconditional guarantees; a target may change only through measured qualification and the applicable Architect gate.]*

### Control and durability

- *[...]*

### Correctness

- *[...]*

### Telemetry and performance qualification

- *[...]*

### End-to-end validation

- *[...]*

### Cleanup and final release

- *[...]*

---

## Held Boundaries

*[Approaches explicitly rejected or held pending the relevant evidence and the applicable Architect gate.]*

- *[...]*

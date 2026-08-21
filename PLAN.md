# PLAN.md — Strategic Plan

> **Project-specific living document.** Captures the architect's directive, architectural boundaries, decision-relevant research, and phase sequencing for the current engagement. The orchestrator populates and maintains this document in collaboration with the architect. It remains current and forward-looking: on directive completion, summarize the delivered outcome here into root `log.md`, then clear this document back to its section skeleton for the next engagement. Project-specific context lives here; granular active execution state lives in `.tasks.json`; compact completion history lives in root `log.md`.

---

## Architect's Directive

**Directive ID:** *[Assign one log-unique ID when this directive begins; never reuse an ID already present in root `log.md`.]*

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

*[Populated progressively by the orchestrator from strategic research-agent findings. Record only the decision-relevant evidence, options, trade-offs, selected rationale, important implementation consequences, and verification implications needed to govern the current engagement; do not retain raw research transcripts or task-level reconnaissance. Select the smallest safe end-to-end solution that satisfies the approved criteria and remove unjustified machinery. Each task may be decomposed and executed only after every approval subject that can affect it is APPROVED; unrelated pending subjects do not block it.]*

---

## Implementation Phases and Sequencing

*[Sequence of phases decomposed from the directive. Each phase is a discrete unit of work with a clear goal and completion criteria. As its governing decision or phase subjects are approved, seed its work into .tasks.json as the root active phase or approved lookahead task shells with dependencies, target write sets, shared surfaces, and explicit parallel safety. Preserve each PLAN phase dependency in task sequencing and promotion. Strategic research may continue for unrelated subjects; targeted follow-up research is reserved for blockers, drift, or decision-changing uncertainty. Phase B implementation is serialized by default and may use a second writer only when the collision-safe admission rule is met. A phase with no implementation task closes only after one phase-scoped read-only verifier evaluates its aggregate criteria; never create a dummy writer.]*

| # | Phase | Execution goal | Depends on | Status |
|---|-------|----------------|------------|--------|
| - | -     | -              | -          | -      |

---

## Architect Plan Approval

*[Rolling approval tracker. Give every decision or phase slice a stable subject ID. A subject remains PENDING until the Architect approves it, then APPROVED for the recorded plan version. Tasks reference it as `<plan-version>/<subject-id>` and may be decomposed and executed when all of their other decision references are also approved; pending unrelated subjects do not block them. A later directive or architectural change returns only affected subjects to PENDING, stops affected work, and leaves demonstrably unaffected work valid under its prior approved version. Update affected task shells only after the revised subject is approved.]*

| Plan version | Status | Subject ID | Approval scope | Recorded |
|--------------|--------|------------|----------------|----------|
| -            | -      | -          | -              | -        |

---

## Active Architecture Decisions

*[Significant decisions currently governing the engagement, with their execution consequence. Changing a locked decision creates a revised plan version whose approval remains PENDING until the Architect accepts it. At directive closure, preserve only the selected architecture and material deviation in the compact directive summary in root `log.md`, then clear this table.]*

| ID | Locked decision | Execution consequence |
|----|-----------------|-----------------------|
| -  | -               | -                     |

---

## Risks and Blockers

*[Active blockers requiring architect attention or strategic resolution. Populated when a tactical block in .tasks.json has architectural implications beyond the individual task. Status: CONTROLLED (mitigation in place) or OPEN (requires control). Clear resolved entries; preserve only a material resolution in the relevant compact closure note.]*

| Phase | Risk or blocker | Required control | Status |
|-------|-----------------|------------------|--------|
| -     | -               | -                | -      |

---

## Acceptance Criteria

*[Nonempty measurable, verifiable success criteria grouped by area. Thresholds and targets are qualification criteria, not unconditional guarantees; a target may change only through measured qualification and Plan Approval of the revised blueprint. Every implementation phase must have at least one aggregate criterion before it may close.]*

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

---

## Directive Closure and Reset

*[When every phase has passed aggregate validation, write one compact directive summary under the exact Directive ID in root `log.md`: delivered outcome, selected architecture, material deviations or blocker resolutions, and final verification. On resume, do not append the same Directive ID twice; if its closure already exists, complete the reset only. Then clear all engagement-specific content in this document back to the template skeleton. Do not preserve raw research, completed task objects, or detailed implementation history in PLAN.md.]*

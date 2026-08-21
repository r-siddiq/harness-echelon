# Closed-Task Summaries

This root log is a compact record of completed work. It is not a second task ledger.

After accepting independent verifier evidence, the Orchestrator records one short closure note, then removes the completed task and its active progress from `.tasks.json`. On resume, reconcile by exact task ID: an existing closure is not appended again and means the transient completed task may be removed. Do not copy raw task JSON, Phase A research, Phase B briefs, full success criteria, file inventories, dispatch events, or agent telemetry here. Keep detailed implementation history in the repository and keep active routing detail only in `.tasks.json`.

## `DIRECTIVE-ID-Phase-X-Task-Y`

### Closure note

- **Outcome:** *[What was completed and its user- or system-visible result.]*
- **Decision or evidence:** *[Only the material research decision, if one affected the result; otherwise “Used approved plan evidence.”]*
- **Implementation:** *[Concise summary of the change; no raw file list or task object.]*
- **Verification:** *[Compact verifier evidence and Orchestrator acceptance outcome.]*
- **Blocker resolution:** *[Only if material; otherwise “None.”]*

---

## Directive closure — `DIRECTIVE-ID`

- **Delivered outcome:** *[What the completed directive delivered.]*
- **Selected architecture:** *[The material architecture decision retained for future context.]*
- **Material deviations or blocker resolutions:** *[Only meaningful departures from the approved plan.]*
- **Final verification:** *[Accepted aggregate verifier evidence and result.]*

After this summary is written, reset PLAN.md to its skeleton and reset `.tasks.json` runtime state. On resume, an existing exact Directive ID means complete the reset without appending another directive closure. Do not reproduce the completed plan or task history here.

# TASKS.md — Ephemeral Task Specifications

## Purpose

TASKS.md is the orchestrator's execution brief — exact, ephemeral task specifications derived from PLAN.md and architect directives. Each task is scoped to a single deliverable with clear completion criteria. Tasks are cleared when complete — no residual state.

## Entry Format

```markdown
## Task: [Task Name]

**State:** pending | in_progress | completed | blocked
**Purpose:** One sentence on what this task accomplishes

**Success Criteria:**
- [Specific measurable criterion 1]
- [Specific measurable criterion 2]

**Files to modify:**
- path/to/file1
- path/to/file2
```

## Task State Transitions

```
pending → in_progress → completed
                    ↘ blocked (dependency or error)
```

| State | Who Sets | Meaning |
|-------|----------|---------|
| `pending` | Orchestrator | Task queued, not yet dispatched |
| `in_progress` | Orchestrator | Agent dispatched, working on task |
| `completed` | Agent → Orchestrator | Task verified, logged to PROGRESS.md, cleared from TASKS.md |
| `blocked` | Agent → Orchestrator | Dependency not met or error; await guidance |

## Workflow

```
Orchestrator reads PLAN.md → identifies next phase
→ Derives exact task specifications → writes to TASKS.md
→ Dispatches IMPLEMENT agent with full context
→ Agent executes → reports → orchestrator verifies
→ On success: log to PROGRESS.md → commit → clear from TASKS.md
→ On failure: mark blocked → await guidance
```
# PLAN.md — Architect Directive Mapping

## Purpose

PLAN.md is the orchestrator's living document — in service of the architect's directive. It maps directives to file structure, stores high-value analysis and research, and captures project architecture. It answers "what are we doing here", "what are we doing now", and highlights what's important. Evolves as understanding grows.

## Entry Format

```markdown
## Directive: [Directive Name]

**Architect Intent:** One sentence on why this directive matters

**Scope:** Brief description of what's in scope and what's explicitly out of scope

**File Structure Mapping:**
- path/to/file1 — purpose
- path/to/file2 — purpose

**Key Decisions:**
- Decision: [what was chosen] → [alternative considered]

**Risks:**
- Risk: [description] — [mitigation]

**Phase Sequence (if applicable):**
1. Phase name — brief description
2. Phase name — brief description
```

## Sections

### Active Directives
List of current architect directives being worked on.

### Architecture
High-level project structure, patterns, and technical decisions.

### Key Decisions
Significant decisions made, the options considered, and why one was chosen.

### Risks & Mitigations
Known risks and how they've been mitigated.

### Research Notes
High-value findings from research agents that inform architectural decisions.

### File Structure
Canonical project layout with purpose for each directory/file.
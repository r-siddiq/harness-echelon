# PROGRESS.md — Timestamped Completion Log

## Purpose

This is a short, timestamped log of completed tasks. Each entry is concise — like a well-written git commit message but with slightly more detail. Entries are written to this log immediately after a task is verified complete, BEFORE the commit.

---

## Entry Format

```markdown
## YYYY-MM-DDTHH:MM:SSZ — Phase X: Phase Title

Brief summary of what was accomplished. Focus on the "why" and the key
technical decision, not just listing what was done. One paragraph max.

Files: list/of/changed/files.php, another/file.php
```

**Fields:**
- **Timestamp:** ISO 8601 UTC (`Z` suffix)
- **Phase ID:** Matches TASKS.md phase (e.g., `Phase 1`, `Phase 2`)
- **Summary:** What was done, why, key technical decisions — one paragraph
- **Files Changed:** List of modified files (comma-separated or one per line)

---

## Guidelines

1. **Log BEFORE commit** — entry should exist before running git commit
2. **Concise but informative** — one paragraph, specific about what was done and why
3. **Include phase ID** — so TASKS.md can be referenced for details
4. **Timestamps are UTC** — ISO 8601 format with `Z` suffix
5. **Focus on the "why"** — not just what was done, but why that approach was taken

---
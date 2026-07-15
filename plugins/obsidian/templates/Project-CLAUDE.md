---
type: project-memory
project: {{project-name}}
updated: {{date:YYYY-MM-DD}}
---

# {{project-name}}

## Status
Current phase, active branch, what's in progress.

## Architecture & Decisions
| Decision | Rationale | Date |
|-|-|-|

## Key Paths & Structure
- `src/...` — description

## Conventions & Patterns
- Convention 1

## Blockers & Warnings
- None yet

## Active Tasks
Backlog items and decisions deferred while working on a task live as individual markdown files with `type: task` frontmatter (see `Templates/Task.md`) in `Ready-to-Dev/`.

```dataview
TABLE priority, severity, created
FROM "Projects/{{project-name}}/Ready-to-Dev"
WHERE type = "task"
SORT priority DESC, created ASC
```

## Next Steps
- [ ] First action

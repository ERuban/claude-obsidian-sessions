---
type: home
---

# Claude Work Vault

## Quick Access
- [[Templates/]] — Templates (Session, Task, Project-CLAUDE)

## Projects
```dataview
TABLE status, updated
FROM "Projects"
WHERE type = "project-memory"
SORT updated DESC
```

## Ready to Dev (backlog & deferred decisions)
Tasks live as individual `.md` files in `Projects/{name}/Ready-to-Dev/`. Use `Templates/Task.md`.

```dataview
TABLE project, priority, severity, ticket
FROM "Projects"
WHERE type = "task" AND status = "ready-to-dev"
SORT priority DESC, created ASC
```

## Recent Sessions
```dataview
TABLE project, topic, outcome
FROM "Projects"
WHERE type = "session"
SORT date DESC
LIMIT 10
```

## Pending Checkboxes (from session logs)
```dataview
TASK
FROM "Projects"
WHERE !completed
SORT date DESC
LIMIT 20
```

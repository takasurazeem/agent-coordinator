---
number: 1
title: "/coordinator skill: orchestrate sub-sessions"
status: done
rank: 1
priority: High
size: null
labels:
  - enhancement
assignees:
  - claude
agent:
  state: done
  model: claude-opus-5-5
  updatedAt: 2026-11-06T05:05:42.855Z
  by: claude
createdAt: 2026-11-05T04:01:01.952Z
updatedAt: 2026-11-06T05:05:42.884Z
---

## Idea

Claude acts as orchestrator, delegating tasks to sub-sessions, each with its own **model** and **effort level**.

```mermaid
flowchart TD
    O[Coordinator] --> A[Session A: model opus, high effort]
    O --> B[Session B: model sonnet, low effort]
    A --> R[Merge results]
    B --> R
```

- [ ] Define skill frontmatter + invocation
- [ ] Model/effort routing rules
- [ ] Merge + report format


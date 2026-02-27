# Task Design Checklist

## Input Checklist

- Task IDs are explicit (`Txx`).
- Target files are confirmed (`requirements.md`, `task_design.md`, `design.md`, `decision.md`).
- Task dependencies in `task_design.md` are acknowledged.

## Context Checklist

- Locate task table row and detail section for each task ID.
- Extract affected chapter ranges from `章节范围（design.md）`.
- Extract DoD bullets and subtask constraints.
- Extract linked requirement IDs from `requirements.md`.

## Design Update Checklist

- Required chapter sections in `docs/design.md` are updated.
- Cross-section references are kept consistent.
- API/auth/error/idempotency/concurrency semantics are aligned.
- Diagrams use `mermaid` only.

## Decision Checklist

- Non-obvious design choices are recorded in ADR format.
- ADR number is sequential and date is current.
- Superseded ADRs are explicitly linked when applicable.

## Completion Checklist

- DoD items are mapped to concrete updated sections.
- Requirement baseline has no new contradiction.
- Remaining gaps are explicitly listed if any.

## ADR Template

```markdown
### ADR-XXX: <决策标题>

- 决策：
- 原因：
- 影响：
- 替代关系：<如无可写“无”>
```

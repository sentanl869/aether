# No-Gap Checklist

Use this checklist before claiming supplemental task split is complete.

## 1. Gap Completeness

- Every analysis gap is mapped to one and only one task/subtask owner.
- No task/subtask exists without at least one concrete gap owner.
- Cross-chapter gaps are split or explicitly linked with dependency notes.

## 2. Requirement Traceability

- Every supplemental task references requirement IDs (or requirement groups).
- No requirement in scope is left unmapped when analysis says it is missing.
- Requirement wording does not conflict with `docs/requirements.md`.

## 3. Design-Scope Coverage

- Every task lists impacted `docs/design.md` sections.
- Scope includes all affected sections, not only primary chapter.
- Linked sections in dependency tasks are added when contract alignment is required.

## 4. DoD Testability

- DoD items are objective and reviewable, not vague.
- Each DoD can be verified by text evidence in design/task docs.
- If a gap needs sequence/model/API/error alignment, DoD states it explicitly.

## 5. Dependency Soundness

- New task dependencies are valid against current task graph.
- No circular dependency introduced.
- Parallelizable tasks freeze shared baseline first (if relevant).

## 6. Status and Planning Integrity

- New supplemental tasks are created with `TODO` status.
- Existing `DONE` tasks are not silently rewritten as completed-by-default.
- If only planning is requested, no execution progress is claimed.

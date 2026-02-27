---
name: design-updater
description: Update docs/design.md for specific task IDs (for example T21) defined in docs/task/task_design.md, using docs/requirements.md as the requirement baseline, and record design decisions in docs/decision.md. Use when users ask to complete task-based design writing or revisions by task number and require decision logging.
---

# Design Updater

## Overview

Complete one or more design tasks by task ID (for example `T21`). Read requirement and task definitions, write implementable design content in `docs/design.md`, and append decision records in `docs/decision.md`.

## Inputs

Required input:

- Task IDs (`T21`, `T19,T20`, etc.)

Optional input:

- File path overrides

Default paths:

- `docs/requirements.md`
- `docs/task/task_design.md`
- `docs/design.md`
- `docs/decision.md`

If task IDs are missing, ask for explicit task IDs before editing.

## Workflow

### 1. Build Task Context

- Read `docs/task/task_design.md` for each task ID:
  - Task table row (`章节范围`、`依赖`、`状态`)
  - Task detail section (`目标`、`输出物`、`DoD`、子任务细则)
- Read `docs/requirements.md` and extract requirement IDs relevant to the task scope.
- Read related sections in `docs/design.md` and identify missing content, conflicts, and sections requiring synchronized updates.
- Read recent related ADRs in `docs/decision.md` to avoid conflicting decisions.

### 2. Freeze Decisions Before Writing

- Identify non-obvious design choices needed to satisfy DoD (for example role boundaries, auth behavior, state transitions, idempotency/concurrency rules, error mapping).
- If requirements are ambiguous, choose one implementable option and record rationale in `docs/decision.md`.
- Keep terms and contract style consistent with existing canonical wording in `docs/design.md`.

### 3. Update `docs/design.md`

- Edit only sections in the task scope plus minimum linked sections needed for consistency.
- Ensure the updated design can directly guide coding. At minimum, cover:
  - Data structures and domain fields
  - State transitions and lifecycle rules
  - API contracts (OpenAPI 3.0 + RESTful)
  - Error codes and failure branches
  - Concurrency and idempotency constraints
  - Critical sequence flows (`mermaid` only)
  - Boundary conditions and forbidden operations
- Keep requirement traceability explicit in the design text.
- Do not claim task completion unless task DoD is materially satisfied.

### 4. Update `docs/decision.md`

- Append ADR records for decisions made during this task.
- Use the next ADR number in sequence and current date.
- For each ADR, include: `决策`、`原因`、`影响`、`替代关系（如有）`.
- If a decision supersedes old ADRs, explicitly reference replaced ADR IDs.

### 5. Final Consistency Pass

- Cross-check consistency among `requirements.md`、`task_design.md`、`design.md`、`decision.md`.
- Verify terminology and role naming are globally consistent.
- Verify each task DoD item can be traced to updated design sections.
- If coverage is partial, explicitly report remaining gaps instead of claiming full completion.

## Edit Rules

- Write concise design documentation, not implementation code.
- Preserve existing section hierarchy and numbering style in destination docs.
- Use `mermaid` for all diagrams; never use images or PlantUML.
- Avoid editing unrelated tasks or historical ADR content unless a supersession link is required.

## Response Rules

When finishing a task execution, report:

- Changed files and key updated sections
- Newly added ADR IDs
- DoD coverage summary and unresolved items

## Reference

Load [references/task-design-checklist.md](references/task-design-checklist.md) before handling multi-section or multi-task updates.

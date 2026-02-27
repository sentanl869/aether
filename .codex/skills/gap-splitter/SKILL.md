---
name: gap-splitter
description: Split analysis findings into supplemental design tasks in docs/task/task_design.md, with strict no-gap checks against docs/requirements.md and current docs/design.md coverage. Use when users ask to decompose missing design content into executable task/subtask entries, ensure no omission, or request task_design.md补充任务拆分.
---

# Gap Splitter

Decompose analysis results into implementable supplemental tasks in
`docs/task/task_design.md`. Keep structure consistent with existing task entries
(`任务拆分与依赖` table, task detail section, optional subtask table, DoD细则).

## Inputs

Required input:

- Analysis result that identifies missing or conflicting design coverage

Optional input:

- Target chapter scope in `docs/design.md`
- Preferred task granularity (new Task vs Subtask-only)

Default files:

- `docs/requirements.md`
- `docs/design.md`
- `docs/task/task_design.md`

## Workflow

### 1. Build the Gap Inventory

- Parse analysis findings into atomic gaps.
- For each gap, record:
  - requirement IDs
  - affected design sections
  - missing artifact type (model/API/state/error/idempotency/sequence/boundary)
  - evidence of current mismatch or missing text
- Merge duplicates and split mixed gaps so each gap is independently verifiable.

### 2. Choose Task Shape

- Use a new `Txx` task when gaps cross multiple chapters, involve baseline rules, or need dependency management.
- Use `Txx-yy` subtasks under an existing task when changes are localized and dependency chain is unchanged.
- Keep one task focused on one coherent closure objective. Do not group unrelated gaps into one task.

### 3. Write Supplemental Tasks into `task_design.md`

- Update the `任务拆分与依赖` table:
  - add new task rows with sequential IDs (`T22`, `T23`, ...)
  - set dependency and initial status (`TODO`)
  - set chapter scope aligned to real impacted sections
- Add task detail blocks following existing style:
  - `目标`
  - `对应章节`
  - `输出物`
  - `DoD`
  - `进度`
- If needed, add `子任务拆分` table and `DoD 细则` (`Txx-D1`, `Txx-D2`, ...).
- Keep wording implementation-oriented and directly testable.

### 4. Run No-Omission Verification

- Load [references/no-gap-checklist.md](references/no-gap-checklist.md).
- Verify every gap has exactly one owning task/subtask.
- Verify every owning task has explicit DoD acceptance points.
- Verify chapter scope, requirement IDs, and DoD evidence chain are consistent.
- If any gap is unresolved, keep status as pending and explicitly mark remaining hole.

### 5. Final Consistency Pass

- Ensure inserted tasks do not break existing task dependency order.
- Ensure terminology is consistent with `requirements.md` and `design.md`.
- Ensure no contradictory requirement status is introduced by new supplemental tasks.

## Edit Rules

- Edit only `docs/task/task_design.md` unless user explicitly asks cross-file updates.
- Preserve existing heading levels, table schema, and task writing style.
- Do not mark tasks `DONE` during planning-only split.
- Do not execute design content filling in `docs/design.md` in this skill.

## Response Rules

When finishing execution, report:

- Added/updated task IDs
- Gap-to-task coverage summary
- Remaining unresolved gaps (if any)

---
name: pre-commit-fixer
description: Run `uv run pre-commit run --all-files`, fix all reported issues across the repository, and keep behavior and document meaning intact. Use when users ask to execute pre-commit checks and repair findings without waiver or bypass comments/config changes.
---

# Pre-commit Fixer

Execute repository-wide pre-commit checks and resolve findings with safe, lossless edits.

## Inputs

Required input:

- Confirmation to run `uv run pre-commit run --all-files`

Optional input:

- Scope constraints (specific directories/files)
- Priority constraints (for example fix blockers first)

## Hard Constraints

- Do not add waiver/bypass comments (for example `# noqa`, `# type: ignore`, `# pylint: disable`, `// eslint-disable`).
- Do not weaken lint or hook configs to silence failures.
- Do not drop semantics or textual meaning to satisfy checks.
- Do not remove substantive content unless the user explicitly requests deletion.

## Workflow

### 1. Run Baseline Scan

- Run `uv run pre-commit run --all-files`.
- Capture failed hooks, files, and error messages.

### 2. Classify Findings

- Group failures by hook type:
  - formatting
  - lint/static analysis
  - docs/text validation
  - file hygiene (EOF/newline/whitespace/permissions)
- For each failure, identify the smallest safe edit surface.

### 3. Apply Safe Fixes

- Prefer deterministic, minimal patches that satisfy hook rules.
- Keep code behavior unchanged:
  - preserve control flow and data contracts
  - preserve error handling intent
  - preserve public interfaces unless hook explicitly requires compatible refactor
- Keep documentation meaning unchanged:
  - reflow or reformat text instead of deleting content
  - preserve headings, examples, links, and constraints
- If a finding cannot be fixed without semantic risk, stop and request user decision.

### 4. Verify Iteratively

- Re-run targeted checks first when possible:
  - `uv run pre-commit run <hook-id> --files <files...>`
- After targeted fixes pass, re-run full scan:
  - `uv run pre-commit run --all-files`
- Repeat until all hooks pass or a blocked item requires user input.

### 5. Report Outcome

- Report:
  - changed files
  - key fixes by hook
  - residual blockers (if any) and why they require user decision
- Explicitly confirm no waiver comments and no semantic/content loss were introduced.

## Edit Rules

- Make focused edits only in files implicated by hook failures.
- Keep style consistent with surrounding project conventions.
- Avoid broad refactors unrelated to failing hooks.

---
name: commit-msg
description: Analyze current repository changes and draft concise, accurate commit messages aligned with existing commit history style. Use when users ask for commit message writing based on staged/working-tree diffs, request style consistency with previous commits, or explicitly want message suggestions without executing git commit.
---

# Commit Message Writer

## Overview

Generate short, precise commit message suggestions from current changes. Match repository-specific style by learning from recent commit subjects before drafting.

## Inputs

Required input:

- Repository with current changes (staged and/or unstaged)

Optional input:

- Preferred language (`中文` or `English`)
- Preferred style (`Conventional Commits`, sentence style, etc.)
- Length constraint (for example `<= 50 chars`)

If constraints are not provided, infer from recent commit history.

## Workflow

### 1. Inspect Current Changes

- Check staged and unstaged changes.
- Read diff summaries first, then key hunks to understand the real intent.
- If there is no effective change, report that no commit message can be drafted yet.

### 2. Learn Local Commit Style

- Read recent commit subjects from local history.
- Infer conventions such as:
  - prefix pattern (`feat:`, `fix:`, none)
  - language style (Chinese or English)
  - subject length and tone (imperative, past tense, noun phrase)
- Prefer repository-local style over generic best practices.

### 3. Map Changes to Intent

- Identify the dominant change intent (feature, fix, refactor, docs, test, chore, etc.).
- Focus on observable behavior or artifact changes, not speculative implementation details.
- If multiple unrelated intents exist, produce a concise umbrella subject and note split-commit risk.

### 4. Draft Concise Message

- Produce one primary commit subject line that is accurate and short.
- Keep wording specific; avoid vague phrases like `update files` or `misc changes`.
- Match inferred local style unless user constraints override it.

### 5. Output Format

- Return a single best message first.
- Optionally include 1-2 alternates only if ambiguity is high.
- Do not execute `git commit`, `git push`, or branch operations in this skill.

## Quality Rules

- Accuracy first: every key phrase must map to actual diffs.
- Brevity second: keep subject compact and readable.
- Style consistency: align with recent history unless explicitly overridden.
- Safety: this skill only drafts text; it never performs commit actions.

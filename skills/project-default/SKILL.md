---
name: project-default
description: Project-local working conventions for this repository. Use when Codex is working inside this project and needs default guidance for reading the codebase, making scoped edits, validating changes, or consulting project-specific notes stored in this skill.
---

# Project Default

## Workflow

Start by inspecting the repository state and nearby files before changing code. Prefer existing project patterns over new abstractions.

Keep edits scoped to the user's request. Do not refactor unrelated files or rewrite project metadata unless the task requires it.

Use fast local discovery commands such as `rg` or `rg --files` when available. Read only the files needed to understand the change.

Validate changed behavior with the smallest relevant command available in the project, such as a targeted test, linter, typecheck, or build.

## Project Context

Read `references/project-context.md` when the task depends on repository-specific conventions, architecture notes, validation commands, or domain vocabulary.

Update `references/project-context.md` when stable project knowledge is discovered and would help future project-local work.

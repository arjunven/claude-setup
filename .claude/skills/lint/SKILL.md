---
name: lint
description: Run pre-commit linting and type-checking on changed files. Use after writing code and before committing.
allowedTools:
  - Bash(git diff --name-only *)
  - Bash(git ls-files --others --exclude-standard)
  - Bash(uv run pre-commit run *)
---

# Lint changed files

Run the project's pre-commit hooks against files that have been modified in the current working tree.

## Steps

1. Get changed files (staged + unstaged + untracked) in a single command: `git diff --name-only HEAD; git ls-files --others --exclude-standard`. Collect all filenames from the output.
2. Run `uv run pre-commit run --files <file1> <file2> ...` with all collected files.
3. Report the results:
   - If all hooks pass, confirm success.
   - If any hooks fail, summarize which hooks failed and what needs fixing.
   - If a linter auto-fixed files (e.g. ruff-format), note which files were modified.

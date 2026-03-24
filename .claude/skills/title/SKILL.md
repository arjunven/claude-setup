---
name: title
description: Set the terminal tab title and chat name to reflect the current task. Detects cmux vs plain Ghostty automatically. Use proactively once the task is identified.
allowedTools:
  - Bash(cmux rename-workspace *)
  - Bash(printf '\033]0;*)
---

# Set terminal tab title and chat name

Update the terminal tab title **and** Claude Code chat name to reflect the current task, so the user can identify which tab and conversation is working on what.

## When to use

Call this **proactively** as soon as you identify the task you're working on — don't wait for the user to ask. This is especially useful when the user has multiple Claude Code sessions open.

## Steps

1. Derive a **short Title Case label** (max 30 chars) from the task context:
   - Linear ticket: ticket ID + Title Case summary, e.g. `MER-381 Datasheet Validation`
   - Free-form task: Title Case summary in 3-5 words, e.g. `Fix Login Redirect Bug`
   - Worktree: Title Case of the worktree slug, e.g. `MER-455 MOSFET Pricing`

2. **Detect the terminal** and set the title:

   Check whether `CMUX_WORKSPACE_ID` is set in the environment (it is auto-set inside cmux terminals). If `CMUX_WORKSPACE_ID` is set but `cmux` is not on PATH, fall back to the OSC 0 method.

   **cmux** (`CMUX_WORKSPACE_ID` is set and `cmux` is available):
   ```
   cmux rename-workspace "<Label>"
   ```
   This renames the workspace (the unit visible in cmux's sidebar/switcher). Do NOT use `cmux rename-tab` — it renames the surface tab inside the workspace, not the workspace itself.

   **Plain Ghostty / other terminal** (`CMUX_WORKSPACE_ID` is not set):
   ```
   printf '\033]0;%s\007' "<Label>"
   ```

3. **Rename the chat** so the conversation is identifiable in `/resume`:

   ```
   /rename <Label>
   ```

   Use the same label derived in step 1.

## Examples

- `cmux rename-workspace "MER-455 MOSFET Pricing"` + `/rename MER-455 MOSFET Pricing`
- `cmux rename-workspace "Fix Search Empty State"` + `/rename Fix Search Empty State`
- `printf '\033]0;Test Auth Middleware\007'` + `/rename Test Auth Middleware`

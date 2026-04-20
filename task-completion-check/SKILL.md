---
name: task-completion-check
description: >-
  Verify whether a coding task is complete by checking plan todos, file changes,
  TypeScript compilation, and linter errors. Use when the user asks to check if
  a task is done, verify completion, review progress, or validate implementation
  against the plan.
---

# Task Completion Check

Systematically verify that all plan items are implemented, code compiles, and nothing is missed.

## Step 1: Locate the Active Plan

Find the current plan file:

```
# Check .cursor/plans/ in the workspace for .plan.md files
Glob: .cursor/plans/*.plan.md

# Also check ~/.cursor/plans/ for user-level plans
Glob: ~/.cursor/plans/*.plan.md
```

Read the plan file. Extract:
- **Overview**: what the task is about
- **Todos**: each todo item with its id, content, and status
- **Acceptance criteria**: any explicit requirements in the plan body

If no plan file is found, ask the user which task to verify.

## Step 2: Check Todo Status

For each todo in the plan frontmatter:

1. **completed** - Verify the work actually exists:
   - Search for the files/changes mentioned in the todo
   - Confirm the implementation matches the todo's description
   - If the file doesn't exist or the change is missing, flag as **false positive**

2. **in_progress** - Report as incomplete, note what appears done vs remaining

3. **pending** - Report as not started

## Step 3: Verify Code Quality

Run these checks (in parallel where possible):

```bash
# TypeScript compilation check (no emit)
task typecheck 2>&1 | tail -20

# If task typecheck is not available, fall back to:
# npx tsc --noEmit 2>&1 | tail -20
```

```bash
# Lint check on changed files only
git diff --name-only HEAD | head -20
```

Then use ReadLints on any files that were created or modified as part of this task.

## Step 4: Check for Uncommitted Work

```bash
# Show working tree status
git status --short

# Show what branch we're on and if we're ahead/behind
git status --branch --short
```

Verify:
- All new files are tracked (not accidentally left untracked)
- No unintended file modifications outside the task scope

## Step 5: Generate Report

Present a concise completion report:

```
## Task Completion Report

**Plan:** [plan name]
**Branch:** [current branch]
**Status:** [Complete / Incomplete / Blocked]

### Todos
- [x] todo-1: description (verified)
- [x] todo-2: description (verified)
- [ ] todo-3: description (NOT DONE - reason)

### Code Quality
- TypeScript: [Pass / X errors]
- Linter: [Pass / X warnings, Y errors]

### Issues Found
1. [Issue description and suggested fix]

### Remaining Work
1. [What still needs to be done]
```

## Rules

- **Do NOT fix issues automatically** - only report them. The user decides what to do.
- **Do NOT make any file changes** - this is a read-only verification skill.
- If the user asks to fix issues after seeing the report, proceed with fixes in a separate step.
- When re-checking after fixes, re-run all steps from the beginning to ensure nothing regressed.

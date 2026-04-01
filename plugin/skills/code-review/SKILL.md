---
name: code-review
description: Request code review by dispatching a code-reviewer subagent on a git diff range. Use after completing a task, feature, or before merging.
argument-hint: Optional description of what was implemented
---

# Requesting Code Review

You are dispatching a code review for recent changes. This skill launches the **code-reviewer** subagent with the correct git diff range and context.

## When to Request Review

**Mandatory:**
- After completing a feature or task
- Before merging to main branch
- After complex bug fixes

**Recommended:**
- After each task in subagent-driven development
- When unsure about implementation quality
- After significant refactoring

## Process

### Step 1: Determine Git Range

Find the base and head commits for the review scope:

```bash
# Head is always current HEAD
HEAD_SHA=$(git rev-parse HEAD)

# Base: find where current branch diverged from main
BASE_SHA=$(git merge-base main HEAD)
```

If on `main` branch or no divergence point, use the last meaningful commit as base:
```bash
# Fallback: review last N commits
BASE_SHA=$(git rev-parse HEAD~N)
```

Ask the user if the range is unclear.

### Step 2: Gather Context

1. Run `git diff --stat $BASE_SHA..$HEAD_SHA` to see scope of changes
2. Use `$ARGUMENTS` if provided, or summarize from git log:
   ```bash
   git log --oneline $BASE_SHA..$HEAD_SHA
   ```

### Step 3: Dispatch Code Reviewer

Launch the **code-reviewer** subagent using the Task tool with the following prompt. Fill in all placeholders:

```
Review the code changes described below.

## What Was Implemented
{DESCRIPTION — what was built or changed, from $ARGUMENTS or git log}

## Git Range
Base: {BASE_SHA}
Head: {HEAD_SHA}

## Review Instructions
1. Run `git diff --stat {BASE_SHA}..{HEAD_SHA}` to see changed files
2. Run `git diff {BASE_SHA}..{HEAD_SHA}` to see full diff
3. Read modified files in full for context where needed
4. Review against your checklist
5. Categorize findings by severity
6. Give a clear merge verdict
```

### Step 4: Handle Review Results

When the code-reviewer returns:

- **Critical issues**: Fix immediately before proceeding
- **Warnings**: Fix before merging (discuss with user if trade-offs involved)
- **Suggestions**: Present to user, let them decide

If the reviewer finds issues you disagree with, apply the **receiving-code-review** skill principles — verify before implementing, push back with technical reasoning if wrong.

## Red Flags — Never Do This

- Skip review for "simple" changes
- Ignore Critical issues
- Proceed with unfixed Warnings without user approval
- Dismiss valid feedback without technical reasoning

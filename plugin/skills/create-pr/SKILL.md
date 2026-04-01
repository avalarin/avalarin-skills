---
name: create-pr
description: Push current branch and create a pull request. Provider-agnostic — works with GitHub, Bitbucket, GitLab, etc. via pluggable provider documents. Use when the user says "push and create PR", "open a PR", "submit my changes", or /create-pr.
---

# Create PR

Push the current branch and create a pull request, monitoring CI and fixing issues along the way.

This skill is provider-agnostic. Provider-specific instructions (how to create PRs, monitor CI, etc.) are loaded from a separate provider document.

## Workflow

### Step 0: Load Provider

Find and read the provider document. Check these locations in order, use the **first one found**:

1. `.claude/create-pr-provider.md` in the project root
2. `~/.claude/create-pr-provider.md` (user-level default)
3. Auto-detect by git remote domain:
   - Run `git remote get-url origin` and extract the hostname (e.g., `github.com`, `stash.msk.avito.ru`, `gitlab.example.com`)
   - Check if `~/.claude/create-pr-providers/<hostname>.md` exists
4. `<skill-path>/references/github.md` (built-in fallback)

Read the provider document. It contains 5 sections you will reference throughout this workflow:
- **Commit & Branch Rules** — branch/commit validation, base branch name, commit format
- **Create or Find PR** — how to check/create PRs
- **Monitor CI** — how to track CI status
- **Get CI Logs** — how to retrieve failure logs
- **Summary Format** — how to present results

Extract the `base-branch` value from the provider (e.g., `main` or `master`). You will use it in multiple steps below.

Tell the user which provider was loaded.

### Step 1: Pre-flight checks

1. **Check for uncommitted changes** — run `git status --porcelain`. If there are uncommitted or untracked files, commit them automatically:
   - Stage files with `git add <specific-files>` (avoid `git add -A` to prevent accidentally staging sensitive files like `.env`)
   - Use the commit message format from the provider's **Commit & Branch Rules**
   - If staging/committing fails, stop and tell the user what's pending.

2. **Get current branch** — run `git branch --show-current`.

3. **Validate branch and commits** — follow the provider's **Commit & Branch Rules** section. Run any validation checks it specifies (branch naming, commit message format, etc.). If validation fails, stop with the error message described by the provider.

4. **Run any provider prerequisites** — if the provider specifies prerequisite checks (e.g., CLI tools that must be available), run them now. Stop if prerequisites are not met.

### Step 2: Code Review

Before pushing, run a code review on all changes that will go into the PR.

1. Determine the git range using the base branch from the provider:
   ```bash
   BASE_SHA=$(git merge-base <base-branch> HEAD)
   HEAD_SHA=$(git rev-parse HEAD)
   ```

2. If there are commits beyond the base branch (`git log --oneline $BASE_SHA..$HEAD_SHA` is non-empty), dispatch the **code-reviewer** subagent with the range.

3. Handle review results:
   - **Critical issues**: Fix before pushing. Commit fixes (using provider's commit format), then re-run the review.
   - **Warnings**: Present to the user via AskUserQuestion — "The reviewer found these warnings. Fix before pushing, or proceed?"
   - **Suggestions**: Note them in the summary but don't block.

4. If on the base branch (no divergence), skip this step.

### Step 3: Push to origin

```bash
git push -u origin HEAD
```

If push fails (e.g., no upstream, rejected), diagnose and report to the user. Never force-push.

### Step 4: Check for merge conflicts

If on the base branch, skip this step.

```bash
git fetch origin <base-branch>
git merge --no-commit --no-ff origin/<base-branch>
```

If there are **no conflicts**, abort and proceed:
```bash
git merge --abort
```

If there **are conflicts**:

1. Abort the test merge: `git merge --abort`
2. **Ask the user for permission** via AskUserQuestion — show which files conflict.
3. If approved, use a sub-agent (software-engineer) to resolve conflicts.
4. **Ask the user for permission to commit and push**. Use the provider's commit format.
5. If approved, commit and push. Go back to Step 6 (Monitor CI).
6. If declined, abort the merge and report.

### Step 5: Create Pull Request

If on the base branch, skip — tell the user: "Pushed directly to <base-branch>. No PR created." and go to Step 8.

Follow the provider's **Create or Find PR** section:
- Check if a PR already exists
- If not, create one using the provider's instructions
- Store the PR URL and PR ID for later steps

### Step 6: Monitor CI

Follow the provider's **Monitor CI** section:
- Use the polling strategy described by the provider
- Determine the result: passed, failed (with build ID), running, or not found
- If CI is not found after retries, ask the user whether to wait longer or skip

If CI passes, go to Step 8.
If CI fails, go to Step 7.

### Step 7: Handle CI failures

1. **Get failure details** — follow the provider's **Get CI Logs** section to retrieve logs.

2. **Ask the user for permission** before fixing. Show them:
   - Which checks/builds failed
   - A brief summary of the errors
   - "Should I try to fix these issues?"

3. If the user agrees:
   - Analyze the error logs and make minimal code fixes
   - Run local verification if possible (lint, build, test — see CLAUDE.md for commands)

4. **Ask the user for permission to commit and push**:
   - Show what changed and why
   - Use the provider's commit format

5. If approved, commit and push. Go back to Step 6.

If the user declines, stop and report.
If CI keeps failing after 3 fix attempts, stop and tell the user.

### Step 8: Summary

Follow the provider's **Summary Format** section. Always include:

- **PR link** (if created or found)
- **CI status**: passed / passed after fixes / skipped / not found
- **Issues fixed** (if any): brief list
- **Commits included**: list of commits

Keep the summary concise and scannable.

## Important Rules

- Never force-push. If a normal push is rejected, ask the user what to do.
- Always ask for user permission before making any code changes or commits (via AskUserQuestion).
- If CI keeps failing after 3 fix attempts, stop and tell the user — don't loop forever.
- When fixing issues, follow the project's code standards from CLAUDE.md.
- All auto-created commits must follow the format specified by the provider's Commit & Branch Rules.

# GitHub Provider

## Commit & Branch Rules

base-branch: main

No special branch naming requirements.

No special commit message format required. Use descriptive messages following the repo's existing style (check `git log --oneline -5`).

If on `main`, skip merge conflict check and PR creation steps, but still push and monitor CI.

**Prerequisites**: The `gh` CLI must be available. Run `which gh` to check. If missing, stop and tell the user:
```
brew install gh
gh auth login
```

## Create or Find PR

**Check if PR exists:**
```bash
gh pr view --json url 2>/dev/null
```
If a PR already exists, skip creation and report the existing PR URL.

**Analyze commits for PR content:**
```bash
git log main..HEAD --oneline
git diff main...HEAD --stat
```

**Create PR:**
```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullet points>

## Test plan
<how to verify>
EOF
)"
```

Use a HEREDOC for the body to preserve formatting. Keep the title under 70 characters.

## Monitor CI

CI runs take a few seconds to appear after a push. Wait and retry before concluding there are no runs.

**First check** (after 5 second delay):
```bash
sleep 5 && gh run list --branch $(git branch --show-current) --limit 5 --json databaseId,name,status,conclusion,event,createdAt
```

**Retry** if empty (after 10 more seconds):
```bash
sleep 10 && gh run list --branch $(git branch --show-current) --limit 5 --json databaseId,name,status,conclusion,event,createdAt
```

If still empty after the second attempt, skip CI monitoring and proceed.

**Wait for completion:**
```bash
gh run watch <run-id> --exit-status
```

Use `--exit-status` so the command exits with non-zero if the run fails. Set a reasonable timeout (10 minutes).

**Status interpretation:**
- Command exits 0 → CI passed
- Command exits non-zero → CI failed, extract `<run-id>` for log retrieval

## Get CI Logs

**Get failure summary:**
```bash
gh run view <run-id> --log-failed
```

This shows the failed steps and their output. Use this to diagnose the issue.

## Summary Format

Present the summary as:

- **PR**: <PR URL> (clickable GitHub link)
- **CI status**: passed / passed after fixes / skipped
- **Issues fixed** (if any): brief list of what went wrong and how it was resolved
- **Commits included**: list of commits in the PR

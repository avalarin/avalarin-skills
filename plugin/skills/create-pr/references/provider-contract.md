# PR Provider Contract

A provider is a markdown document that tells `create-pr` how to interact with a specific git hosting service (GitHub, Bitbucket, GitLab, etc.).

## Required Sections

Your provider document must contain these 5 sections, using the exact `##` headings below.

### `## Commit & Branch Rules`

Define how commits and branches should be validated and formatted.

**Must specify:**
- `base-branch`: the default target branch name (e.g., `main`, `master`)
- Branch name validation rules (if any). If the branch name is invalid, tell the user and stop.
- Commit message validation rules (if any). Describe what to check and what error to show.
- Commit message format for auto-created commits (fixes, merge resolutions, etc.)

**Example for a Jira-based workflow:**
```markdown
## Commit & Branch Rules

base-branch: master

Branch name must match Jira ticket pattern: `^[A-Z]+-[0-9]+` (e.g., `SEC-123`, `SEC-123-fix-auth`).
If on `master` or `main`, STOP with error.

All commit messages must start with the ticket ID followed by a colon (e.g., `SEC-123: description`).
Run `git log master..HEAD --oneline` and verify every commit. If any commit violates this, STOP and list the offending commits.

Auto-created commits must use format: `<TICKET-ID>: <description>` (e.g., `SEC-123: fix linting errors`).
```

### `## Create or Find PR`

Define how to check for existing PRs and create new ones.

**Must specify:**
- How to check if a PR already exists for the current branch
- How to create a new PR (command, MCP tool, API call, etc.)
- What parameters to pass (title, description, target branch, reviewers, etc.)
- How to extract the PR URL/ID from the result

### `## Monitor CI`

Define how to monitor CI/CD pipeline status after push or PR creation.

**Must specify:**
- How to check CI status (command, MCP tool, polling endpoint)
- Polling strategy: initial delay, polling interval, max attempts
- How to determine if CI is: passed, failed, still running, or not found
- How to extract a build/run ID for failed builds (needed by "Get CI Logs")

### `## Get CI Logs`

Define how to retrieve failure logs from a failed CI build.

**Must specify:**
- How to get failure logs given a build/run ID
- How to search logs for specific errors (if supported)

### `## Summary Format`

Define how to format the final summary for the user.

**Must specify:**
- How to format the PR link (URL pattern, clickable format)
- Any additional provider-specific info to include (build links, review status, etc.)

## Provider Lookup Order

The `create-pr` skill looks for a provider document in this order:
1. `.claude/create-pr-provider.md` in the project root
2. `~/.claude/create-pr-provider.md` (user-level default)
3. `~/.claude/create-pr-providers/<hostname>.md` — auto-detected from `git remote get-url origin` (e.g., `github.com.md`, `stash.msk.avito.ru.md`, `gitlab.example.com.md`)
4. Built-in `references/github.md` (fallback)

**How to set up:**
- Per-project override: place provider at `.claude/create-pr-provider.md` in the project
- User-level default: place at `~/.claude/create-pr-provider.md`
- Per-hosting-service: place at `~/.claude/create-pr-providers/<hostname>.md` — this automatically applies to all repos hosted on that domain

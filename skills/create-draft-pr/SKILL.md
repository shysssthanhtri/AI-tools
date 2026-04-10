---
name: create-draft-pr
description: 'Create a GitHub draft pull request for a Jira ticket. Use when asked to "create a PR", "open a draft PR", "submit a pull request", or "create a draft pull request" after finishing implementation. Reads local ticket details, summarises changes, creates a draft PR via GitHub MCP, and assigns it to the current user. Requires setup-new-work to have been run first.'
allowed-tools: mcp_com_atlassian_getAccessibleAtlassianResources mcp_com_atlassian_getJiraIssue mcp_io_github_git_create_pull_request mcp_io_github_git_get_me mcp_git_git_diff mcp_git_git_log mcp_git_git_status mcp_git_git_branch
---

# Create Draft PR

Create a well-structured GitHub draft pull request for a Jira ticket after implementation is complete.

## Role

You are a senior software engineer who writes clear, informative pull requests. You read the ticket, understand what changed, and produce a PR that gives reviewers everything they need to understand the work.

## When to Use This Skill

- User says "create a PR", "open a draft PR", "submit a pull request", or "create a PR for CM-1234"
- Implementation work is complete (or in progress) and is ready to be reviewed
- The worktree and branch were created by the `setup-new-work` skill

## Prerequisites

- `setup-new-work` skill has been run — the worktree exists at `./worktrees/<ticket-id>-<platform>/` and the ticket document exists at `docs/<ticket-id>/<ticket-id>.md`
- Changes have been committed to the feature branch
- GitHub MCP is available (`mcp_io_github_git_*` tools)
- Atlassian MCP is available (to enrich ticket details if needed)

## Parameters

| Parameter    | Required | Description                        | Example   |
|-------------|----------|------------------------------------|-----------|
| `ticket-id` | Yes      | The Jira ticket key                | `CM-4873` |

---

## Workflow

### Step 1 — Validate Input & Environment

1. The user **must** provide a Jira ticket key (e.g. `CM-4873`). If not provided, check the current branch name — if it matches `feature/<ticket-id>-*`, extract the ticket ID from it. If still not determinable, ask the user.

2. Detect the platform and locate the worktree:

```bash
if [ -n "$CURSOR_TRACE_DIR" ] || [ -n "$CURSOR_CHANNEL" ]; then
  platform="cursor"
elif [ -n "$VSCODE_GIT_ASKPASS_NODE" ] || [[ "${TERM_PROGRAM:-}" == *vscode* ]]; then
  platform="copilot"
else
  platform="ai"
fi
repo_root="$(git rev-parse --show-toplevel)"
worktree="$repo_root/worktrees/<ticket-id>-$platform"
```

3. Verify the worktree directory exists. If not, stop and instruct the user to run the `setup-new-work` skill first.
4. `cd` into the worktree:

```bash
cd "$worktree"
```

5. Confirm there is at least one commit ahead of the default branch. If not, warn the user that there may be no changes to review.

---

### Step 2 — Read Ticket Details

1. Read `docs/<ticket-id>/<ticket-id>.md` completely from the worktree.
2. Extract:
   - **Summary** — one-line ticket title
   - **Description** — full problem context
   - **Acceptance Criteria** — definition of done
   - **Ticket URL** — construct from ticket key: `https://theiconic.atlassian.net/browse/<ticket-id>`

If the local file is missing or lacks detail, fetch from Jira:

```
mcp_com_atlassian_getAccessibleAtlassianResources  → get cloudId
mcp_com_atlassian_getJiraIssue(cloudId, issueIdOrKey=<ticket-id>, responseContentFormat=markdown)
```

---

### Step 3 — Gather Change Details

Run these git commands inside the worktree to understand what changed:

```bash
# Current branch name
git branch --show-current

# Commits ahead of master/main
git log origin/master..HEAD --oneline

# Files changed
git diff origin/master..HEAD --name-status

# Full diff (for context, do NOT include in PR body verbatim)
git diff origin/master..HEAD --stat
```

Use the output to:
- List the **files changed** (grouped by area, e.g. controllers, services, tests)
- Summarise the **nature of changes** (added, modified, deleted)
- Identify the **approach taken** from the code changes

---

### Step 4 — Identify the GitHub Repository

Call `mcp_io_github_git_get_me` to get the current authenticated GitHub user's login.

Determine the repository owner and name from `git remote -v` output or from repo context:
- owner: `theiconic`
- repo: extracted from the remote URL

---

### Step 5 — Build the PR Title

Format: `<ticket-id>: <summary>`

Rules:
- The summary must be derived from the Jira ticket title
- Maximum **10 words** for the summary portion
- Remove filler words if needed to stay under 10 words
- Use sentence case (capitalise first word only)
- Do NOT include the branch name or file names

Example: `CM-4873: Fix internal structure exposure in user response`

---

### Step 6 — Build the PR Description

Use this template exactly:

```markdown
## Jira Ticket

[<ticket-id>](<ticket-url>) — <one-sentence ticket summary>

---

## Context / Problem

<2–4 sentences describing the current state and why it is a problem. Derived from the ticket description.>

---

## Solution

<2–4 sentences describing what was changed and how it solves the problem. Derived from your analysis of the code diff.>

**Files changed:**

- `path/to/file` — <what changed and why>
- `path/to/file` — <what changed and why>

---

## How to Test

<Step-by-step instructions a reviewer can follow to verify the changes work correctly. Be specific — include endpoints, request payloads, expected responses, or commands to run.>

1. <step>
2. <step>
3. <step>

---

## Checklist

- [ ] Code follows project conventions
- [ ] Tests added or updated
- [ ] No unintended files included
```

Fill in each section using the ticket details and code change analysis from Steps 2–3.

---

### Step 7 — Create the Draft PR

Call `mcp_io_github_git_create_pull_request` with:

| Field | Value |
|-------|-------|
| `owner` | repository owner (e.g. `theiconic`) |
| `repo` | repository name (e.g. `bridge-keeper`) |
| `title` | PR title from Step 5 |
| `body` | PR description from Step 6 |
| `head` | current feature branch name |
| `base` | default branch (usually `master`) |
| `draft` | `true` |

---

### Step 8 — Assign PR to Current User

After the PR is created:

1. Get the current user's GitHub login from `mcp_io_github_git_get_me`.
2. Use `mcp_io_github_git_update_pull_request` or the appropriate assignees call to assign the PR to yourself.

If the MCP tool supports `assignees` directly in `create_pull_request`, pass it during creation to avoid an extra call.

---

### Step 9 — Confirm and Report

After the PR is created and assigned, report back to the user:

```
Draft PR created:

Title:    <PR title>
URL:      <PR URL from response>
Branch:   <feature branch>
Base:     master
Assigned: <your GitHub username>
Status:   Draft
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Ticket ID not found | Provide it explicitly or check `git branch --show-current` |
| Worktree not found | Run the `setup-new-work` skill first |
| No commits ahead of base | Commit your changes before creating a PR |
| MCP GitHub auth error | Ensure the GitHub MCP server is connected and authenticated |
| PR creation fails with 422 | A PR may already exist for this branch — check open PRs |

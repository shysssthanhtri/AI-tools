---
name: setup-new-work
description: "Use when starting work on a Jira ticket. Fetches ticket details via Atlassian MCP, creates an isolated git worktree workspace, and stores a summarized ticket document locally for reference."
allowed-tools: Bash(git:*) mcp_com_atlassian_getAccessibleAtlassianResources mcp_com_atlassian_getJiraIssue
---

# Setup New Work

Set up a clean, isolated workspace for a Jira ticket by creating a git worktree and storing summarized ticket details locally.

## Role

You are a senior software engineer who values clean, isolated workspaces. Before writing any code you want to fully understand the ticket, have an isolated branch and worktree, and keep a local copy of the ticket details for quick reference.

## Objectives

1. **Prepare a clean, isolated workspace** — Create a new git worktree so work is physically separated from other in-progress tasks.
2. **Understand the ticket** — Fetch and read the full Jira ticket (summary, description, acceptance criteria, subtasks, comments).
3. **Store ticket details locally** — Write a structured Markdown summary into a well-defined docs folder inside the worktree.

## Rules

### Ticket ID

- The user **must** provide a Jira ticket key (e.g. `CM-4873`). If not provided, ask for it before proceeding.
- Preserve the ticket key casing exactly as Jira returns it.

### Worktree & Branch

- The worktree folder must be located at `./worktrees/<ticket-id>-<platform>` relative to the current repository root, where `<platform>` is the detected or user-specified AI platform name (see [Parameters](#parameters)).
  - Example: if the repo is at `/Users/dev/repos/bridge-keeper` and platform is `copilot`, the worktree is at `/Users/dev/repos/bridge-keeper/worktrees/CM-4873-copilot`.
  - Example: if platform is `cursor`, the worktree is at `/Users/dev/repos/bridge-keeper/worktrees/CM-4873-cursor`.
- The branch name format is: `feature/<ticket-id>-<slug>` where `<slug>` is a 3–5 word summary derived from the Jira ticket summary.
- Slug rules (same as create-branch skill):
  - Lowercase.
  - Replace `&` with `and`.
  - Remove punctuation/symbols; keep only letters, digits, and spaces.
  - Remove filler words (`a`, `an`, `the`, `to`, `for`, `of`, `in`, `on`, `with`).
  - Replace spaces with `-`, collapse repeated dashes, trim leading/trailing dashes.
  - Keep first 3–5 meaningful words. Max slug length 32 characters.
  - If slug is empty after normalisation, use `work-item`.

### Ticket Documentation

- Ticket summary document must be saved at `docs/<ticket-id>/<ticket-id>.md` inside the new worktree.
- The document must follow the template defined in the [Ticket Summary Template](#ticket-summary-template) section.

## Workflow

### Step 1 — Resolve Jira Cloud

Call `mcp_com_atlassian_getAccessibleAtlassianResources` to obtain the `cloudId`.

### Step 2 — Fetch Ticket Details

Call `mcp_com_atlassian_getJiraIssue` with:
- `cloudId` from Step 1
- `issueIdOrKey` = the ticket key provided by the user
- `responseContentFormat` = `markdown`

Extract from the response:
- **Summary** (title)
- **Description**
- **Status**
- **Priority**
- **Issue Type**
- **Assignee**
- **Reporter**
- **Labels**
- **Sprint** (if present)
- **Acceptance Criteria** (often in description or a custom field)
- **Subtasks / Linked Issues** (if any)

If the Jira lookup fails, stop and ask the user to verify the ticket key.

### Step 3 — Generate the Branch Slug

Use this snippet to generate a deterministic slug from the Jira summary:

~~~bash
summary="<jira-summary-text>"

slug="$(
	printf '%s' "$summary" \
	| tr '[:upper:]' '[:lower:]' \
	| sed -E 's/&/ and /g' \
	| sed -E 's/[^a-z0-9 ]+/ /g' \
	| tr -s ' ' \
	| sed -E 's/^ +| +$//g' \
	| awk '{
			out_count=0;
			for (i=1; i<=NF; i++) {
				w=$i;
				if (w=="a" || w=="an" || w=="the" || w=="to" || w=="for" || w=="of" || w=="in" || w=="on" || w=="with") {
					continue;
				}
				out[++out_count]=w;
				if (out_count==5) break;
			}
			for (i=1; i<=out_count; i++) {
				printf "%s%s", out[i], (i<out_count?"-":"");
			}
		}' \
	| sed -E 's/-+/-/g; s/^-+|-+$//g' \
	| cut -c1-32 \
	| sed -E 's/-+/-/g; s/^-+|-+$//g'
)"

if [ -z "$slug" ]; then
	slug="work-item"
fi

echo "$slug"
~~~

### Step 4 — Detect Platform

Determine the `<platform>` value. If the user provided one explicitly, use it. Otherwise auto-detect:

- If the environment variable `CURSOR_TRACE_DIR` or `CURSOR_CHANNEL` is set → `cursor`
- If the environment variable `VSCODE_GIT_ASKPASS_NODE` is set or `TERM_PROGRAM` contains `vscode` → `copilot`
- Otherwise → ask the user.

```bash
if [ -n "$CURSOR_TRACE_DIR" ] || [ -n "$CURSOR_CHANNEL" ]; then
  platform="cursor"
elif [ -n "$VSCODE_GIT_ASKPASS_NODE" ] || [[ "${TERM_PROGRAM:-}" == *vscode* ]]; then
  platform="copilot"
else
  platform="ai"  # fallback; ask user if needed
fi
echo "$platform"
```

### Step 5 — Create the Git Worktree

1. Determine the repository root:

	```bash
	repo_root="$(git rev-parse --show-toplevel)"
	```

2. Create a new branch and worktree in one command:

	```bash
	mkdir -p "$repo_root/worktrees"
	git worktree add -b "feature/<ticket-id>-<slug>" "$repo_root/worktrees/<ticket-id>-<platform>" HEAD
	```

	This creates the branch from the current HEAD and checks it out in the new worktree directory.

3. Verify the worktree was created:

	```bash
	git worktree list
	```

### Step 6 — Store Ticket Details

1. Create the docs directory inside the new worktree:

	```bash
	mkdir -p "$repo_root/worktrees/<ticket-id>-<platform>/docs/<ticket-id>"
	```

2. Write the ticket summary document at `docs/<ticket-id>/<ticket-id>.md` using the template below.

### Step 7 — Confirm Setup

Print a summary to the user:

```
✅ Workspace ready

  Ticket:    <ticket-id> — <summary>
  Platform:  <platform>
  Worktree:  ./worktrees/<ticket-id>-<platform>
  Branch:    feature/<ticket-id>-<slug>
  Docs:      docs/<ticket-id>/<ticket-id>.md
```

## Ticket Summary Template

```markdown
# <ticket-id>: <Summary>

| Field       | Value            |
|-------------|------------------|
| Status      | <status>         |
| Priority    | <priority>       |
| Type        | <issue-type>     |
| Assignee    | <assignee>       |
| Reporter    | <reporter>       |
| Labels      | <labels>         |
| Sprint      | <sprint>         |

## Description

<Full description from Jira, in Markdown>

## Acceptance Criteria

<Acceptance criteria extracted from Jira, as a checklist>

- [ ] Criterion 1
- [ ] Criterion 2

## Subtasks / Linked Issues

| Key       | Summary              | Status   |
|-----------|----------------------|----------|
| <key>     | <summary>            | <status> |

## Notes

<Empty section for the engineer to add working notes>
```

## Configuration Reference

### Parameters

| Parameter    | Required | Description                                           | Example    |
|-------------|----------|-------------------------------------------------------|------------|
| `ticket-id` | Yes      | The Jira ticket key                                   | `CM-4873`  |
| `platform`  | No       | AI platform name. Auto-detected if not provided.      | `copilot`, `cursor` |

### Defaults

| Setting             | Value                                          |
|---------------------|------------------------------------------------|
| Worktree folder     | `./worktrees/<ticket-id>-<platform>`           |
| Branch format       | `feature/<ticket-id>-<slug>`                   |
| Docs path           | `docs/<ticket-id>/<ticket-id>.md`              |
| Max slug length     | 32 characters                                  |
| Slug word count     | 3–5 meaningful words                           |
| Platform detection  | Auto: `cursor` or `copilot` from env vars      |

## Examples

### Example 1 — Ticket `CM-4873`, platform `copilot`

Summary: "Add retry logic for payment webhooks"

```
Worktree:  ./worktrees/CM-4873-copilot
Branch:    feature/CM-4873-add-retry-logic-payment-webhooks
Docs:      docs/CM-4873/CM-4873.md
```

### Example 2 — Ticket `ENG-101`, platform `cursor`

Summary: "Fix the broken & flaky CI pipeline tests"

```
Worktree:  ./worktrees/ENG-101-cursor
Branch:    feature/ENG-101-fix-broken-and-flaky-ci
Docs:      docs/ENG-101/ENG-101.md
```

## Final Checklist

Before reporting completion, verify all of the following:

- [ ] Jira ticket was fetched successfully and details are accurate.
- [ ] Git worktree exists at `./worktrees/<ticket-id>-<platform>`.
- [ ] Branch `feature/<ticket-id>-<slug>` is checked out in the worktree.
- [ ] `docs/<ticket-id>/<ticket-id>.md` exists in the worktree with complete ticket details.
- [ ] Platform was correctly detected or provided.
- [ ] No uncommitted or unrelated changes were carried into the new worktree.

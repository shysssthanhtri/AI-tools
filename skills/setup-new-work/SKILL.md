---
name: setup-new-work
description: "Use when starting work on a Jira ticket. Fetches ticket details via Atlassian MCP (for branch naming and a local story snapshot), creates an isolated git worktree with branch feature/<ticket-id>-<slug>-<platform>, and writes docs/<ticket-id>/<ticket-id>.md in the worktree."
allowed-tools: Bash(git:*) mcp_com_atlassian_getAccessibleAtlassianResources mcp_com_atlassian_getJiraIssue
---

# Setup New Work

Set up a clean, isolated workspace for a Jira ticket by creating a git worktree and a feature branch that includes the AI platform in the name.

## Role

You are a senior software engineer who values clean, isolated workspaces. Before writing any code you want an isolated branch and worktree. You fetch the ticket to derive a branch slug from the summary **and** to persist a local story snapshot next to that work. You do **not** create `plan.md`, `progress.md`, or other planning artefacts here—the **plan-jira-ticket** skill owns those.

## Objectives

1. **Prepare a clean, isolated workspace** — Create a new git worktree so work is physically separated from other in-progress tasks.
2. **Name the branch from the ticket** — Fetch the Jira issue to read the summary and build a deterministic slug; use it with the ticket id and platform in the branch name.
3. **Store story details locally** — Write `docs/<ticket-id>/<ticket-id>.md` inside the new worktree with a team-usable brief (summary, description, status, priority, type, people, labels, parent/epic, subtasks, links, timestamps, and notes). Match the intent of **plan-jira-ticket** Step 1b so planning can assume the file exists.

## Rules

### Ticket ID

- The user **must** provide a Jira ticket key (e.g. `CM-4873`). If not provided, ask for it before proceeding.
- Preserve the ticket key casing exactly as Jira returns it.

### Worktree & Branch

- The worktree folder must be located at `./worktrees/<ticket-id>-<platform>` relative to the current repository root, where `<platform>` is the detected or user-specified AI platform name (see [Parameters](#parameters)).
  - Example: if the repo is at `/Users/dev/repos/bridge-keeper` and platform is `copilot`, the worktree is at `/Users/dev/repos/bridge-keeper/worktrees/CM-4873-copilot`.
  - Example: if platform is `cursor`, the worktree is at `/Users/dev/repos/bridge-keeper/worktrees/CM-4873-cursor`.
- The branch name format is: `feature/<ticket-id>-<slug>-<platform>` where `<slug>` is a 3–5 word summary derived from the Jira ticket summary, and `<platform>` matches the worktree folder suffix.
- Slug rules (same as create-branch skill):
  - Lowercase.
  - Replace `&` with `and`.
  - Remove punctuation/symbols; keep only letters, digits, and spaces.
  - Remove filler words (`a`, `an`, `the`, `to`, `for`, `of`, `in`, `on`, `with`).
  - Replace spaces with `-`, collapse repeated dashes, trim leading/trailing dashes.
  - Keep first 3–5 meaningful words. Max slug length 32 characters.
  - If slug is empty after normalisation, use `work-item`.

### Out of scope

- Do **not** create `plan.md` or `progress.md`, or run a full codebase planning pass—that is **plan-jira-ticket**.
- **In scope:** the single ticket snapshot file `docs/<ticket-id>/<ticket-id>.md` inside the worktree (not optional).

## Workflow

### Step 1 — Resolve Jira Cloud

Call `mcp_com_atlassian_getAccessibleAtlassianResources` to obtain the `cloudId`.

### Step 2 — Fetch Ticket Details

Call `mcp_com_atlassian_getJiraIssue` with:
- `cloudId` from Step 1
- `issueIdOrKey` = the ticket key provided by the user
- `responseContentFormat` = `markdown`

From the response, capture at least **Summary** (for the slug), **Status**, and enough fields to populate `docs/<ticket-id>/<ticket-id>.md` (e.g. description, priority, type, assignee, reporter, labels, parent, subtasks, issuelinks, components, fixVersions, created/updated). Request additional `fields` from the API if the default payload is thin.

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
	git worktree add -b "feature/<ticket-id>-<slug>-<platform>" "$repo_root/worktrees/<ticket-id>-<platform>" HEAD
	```

	This creates the branch from the current HEAD and checks it out in the new worktree directory. The branch name includes the same `<platform>` suffix as the worktree folder.

3. Verify the worktree was created:

	```bash
	git worktree list
	```

### Step 5b — Write local story snapshot

Inside the **worktree** (not the main repo root unless they are the same), create the ticket brief:

1. `mkdir -p "$repo_root/worktrees/<ticket-id>-<platform>/docs/<ticket-id>"`
2. Write `$repo_root/worktrees/<ticket-id>-<platform>/docs/<ticket-id>/<ticket-id>.md` using the Jira data from Step 2. Structure it as a readable brief: title, summary, type/status/priority, people, description (or “none”), labels, parent epic, subtasks, linked issues, sprint/acceptance criteria if present in the API, Jira timestamps, and a short **Notes** section (e.g. regenerate if Jira changes).

If a file already exists at that path, overwrite only when re-running setup for the same ticket and the user expects a refresh; otherwise skip or ask.

### Step 6 — Confirm Setup

Print a summary to the user:

```
✅ Workspace ready

  Ticket:    <ticket-id> — <summary>
  Platform:  <platform>
  Worktree:  ./worktrees/<ticket-id>-<platform>
  Branch:    feature/<ticket-id>-<slug>-<platform>
  Story:     ./worktrees/<ticket-id>-<platform>/docs/<ticket-id>/<ticket-id>.md
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
| Branch format       | `feature/<ticket-id>-<slug>-<platform>`          |
| Max slug length     | 32 characters                                  |
| Slug word count     | 3–5 meaningful words                           |
| Platform detection  | Auto: `cursor` or `copilot` from env vars      |

## Examples

### Example 1 — Ticket `CM-4873`, platform `copilot`

Summary: "Add retry logic for payment webhooks"

```
Worktree:  ./worktrees/CM-4873-copilot
Branch:    feature/CM-4873-add-retry-logic-payment-webhooks-copilot
```

### Example 2 — Ticket `ENG-101`, platform `cursor`

Summary: "Fix the broken & flaky CI pipeline tests"

```
Worktree:  ./worktrees/ENG-101-cursor
Branch:    feature/ENG-101-fix-broken-and-flaky-ci-cursor
```

## Final Checklist

Before reporting completion, verify all of the following:

- [ ] Jira ticket was fetched successfully (at least summary for slug).
- [ ] Git worktree exists at `./worktrees/<ticket-id>-<platform>`.
- [ ] Branch `feature/<ticket-id>-<slug>-<platform>` is checked out in the worktree.
- [ ] Platform was correctly detected or provided.
- [ ] `docs/<ticket-id>/<ticket-id>.md` exists in the worktree with story details from Jira.
- [ ] No `plan.md` or `progress.md` was created by this skill (those belong to plan-jira-ticket).
- [ ] No uncommitted or unrelated changes were carried into the new worktree.

---
name: create-branch
description: "Use when the user asks to create a git branch for a Jira ticket. Creates a branch from the current branch using feature/ticket-id-branch-name format with a short, meaningful title slug from Jira summary."
allowed-tools: Bash(git:*) mcp_com_atlassian_getAccessibleAtlassianResources mcp_com_atlassian_getJiraIssue
---

# Create Branch From Jira Ticket

## Goal

Create a new branch from the current checked-out branch for a Jira ticket.

## Required Output Format

Use this exact format:

`feature/ticket-id-branch-name`

Examples:

- `feature/BK-123-add-login-endpoint`
- `feature/ENG-88-fix-cache-invalidation`

## Workflow

1. Ask for the Jira ticket key if the user did not provide it (for example `BK-123`).
2. Read the current branch:

	```bash
	git branch --show-current
	```

3. Get Jira context using Atlassian MCP:
	- Call `mcp_com_atlassian_getAccessibleAtlassianResources` to resolve a valid Jira cloud.
	- Call `mcp_com_atlassian_getJiraIssue` with `issueIdOrKey=<ticket-key>` and `responseContentFormat=markdown`.
	- Extract the issue summary.
4. Build a short branch slug from the Jira summary:
	- Lowercase text.
	- Replace `&` with `and`.
	- Remove punctuation and symbols; keep only letters, digits, and spaces.
	- Remove common filler words when possible (`a`, `an`, `the`, `to`, `for`, `of`, `in`, `on`, `with`).
	- Replace spaces with `-`.
	- Collapse repeated dashes and trim leading/trailing dashes.
	- Keep the first 3-6 meaningful words.
	- Enforce max slug length of 32 characters.
	- If slug becomes empty after normalization, use `work-item`.
5. Build branch name as:

	`feature/<ticket-id>-<slug>`

6. Create the branch from the current branch explicitly:

	```bash
	git checkout -b "feature/<ticket-id>-<slug>" "<current-branch>"
	```

7. Verify the result:

	```bash
	git branch --show-current
	```

## Rules

- Always branch out from the current branch, not from the default branch unless current branch is already the default.
- Keep the ticket ID exactly as Jira key (for example `BK-123`).
- Keep the slug concise and readable; do not copy the full ticket title if it is long.
- Keep the ticket ID casing exactly as Jira returns it; do not lowercase or rewrite it.
- Do not allow trailing dash, duplicate dash, or empty slug segments.
- If Jira issue lookup fails, stop and ask the user to confirm the ticket key before creating a branch.

## Slug Examples

- Jira summary: `Add OAuth2 login for admin portal`
	- Slug: `add-oauth2-login-admin-portal`
- Jira summary: `Fix: API timeout in payment retry flow`
	- Slug: `fix-api-timeout-payment-retry`
- Jira summary: `UI & UX polish for checkout`
	- Slug: `ui-and-ux-polish-checkout`

## Deterministic Slug Snippet

Use this snippet to generate the slug in a repeatable way from Jira summary text:

~~~bash
summary="UI & UX polish for checkout"

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
				if (out_count==6) break;
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

Then build the branch name:

~~~bash
branch_name="feature/${ticket_id}-${slug}"
~~~

## Response Template

Use this style in the final user response:

- Current branch: `<current-branch>`
- Jira ticket: `<ticket-id>`
- Jira summary: `<ticket-summary>`
- New branch created: `feature/<ticket-id>-<slug>`


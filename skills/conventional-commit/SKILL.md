---
name: conventional-commit
description: 'Prompt and workflow for generating conventional commit messages using a structured XML format. Guides users to create standardized, descriptive commit messages with an optional Jira ticket ID extracted from the current branch when possible.'
---

### Instructions

```xml
	<description>This file contains a prompt template for generating conventional commit messages. It provides instructions, examples, and formatting guidelines to help users write standardized, descriptive commit messages with an optional Jira ticket ID in the subject line.</description>
```

### Workflow

**Follow these steps:**

1. Run `git diff --cached` to inspect **only staged changes** — this is the sole basis for the commit message.
2. If there are no staged changes, stop and inform the user: no staged changes detected. Suggest running `git add <file>` first.
3. Inspect the current branch name (`git branch --show-current`) and extract the ticket ID when the branch matches `feature/{ticket-id}-ticket-title`.
4. If a ticket ID is found, use it as `<ticket-id>` (for example, branch `feature/BK-123-add-login` -> ticket ID `BK-123`).
5. If no ticket ID is found (for example, branch `feature/commit-message`), skip the ticket ID and generate the commit message without scope: `type: description`.
6. Construct your commit message **based solely on the staged diff** using the following XML structure.
7. After generating your commit message, Copilot will automatically run the following command in your integrated terminal (no confirmation needed):

```bash
git commit -m "type(ticket-id): description" # when ticket ID exists
git commit -m "type: description"            # when ticket ID is unavailable
```

8. Just execute this prompt and Copilot will handle the commit for you in the terminal.

### Commit Message Structure

```xml
<commit-message>
	<type>feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert</type>
	<ticket-id>(optional) Jira issue key, for example BK-123</ticket-id>
	<description>A short, imperative summary of the change</description>
	<body>(optional: more detailed explanation)</body>
	<footer>(optional: e.g. BREAKING CHANGE: details, or issue references)</footer>
</commit-message>
```

### Examples

```xml
<examples>
	<example>feat(BK-123): add ability to parse arrays</example>
	<example>fix(BK-456): correct button alignment</example>
	<example>docs(BK-789): update README with usage instructions</example>
	<example>refactor(BK-321): improve performance of data processing</example>
	<example>chore(BK-654): update dependencies</example>
	<example>chore: improve commit message workflow</example>
	<example>feat!(BK-987): send email on registration</example>
</examples>
```

### Validation

```xml
<validation>
	<type>Must be one of the allowed types. See <reference>https://www.conventionalcommits.org/en/v1.0.0/#specification</reference></type>
	<ticket-id>Optional. First try to extract from the current branch using feature/{ticket-id}-ticket-title. If unavailable, omit ticket-id and use the format type: description.</ticket-id>
	<description>Required. Use the imperative mood (e.g., "add", not "added").</description>
	<body>Optional. Use for additional context.</body>
	<footer>Use for breaking changes or issue references.</footer>
</validation>
```

### Final Step

```xml
<final-step>
	<cmd>git commit -m "type(ticket-id): description" | git commit -m "type: description"</cmd>
	<note>Use type(ticket-id): description when ticket-id is available; otherwise use type: description. Include body and footer if needed.</note>
</final-step>
```

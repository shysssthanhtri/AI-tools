---
name: work-on-ticket-plan
description: 'Execute an existing Jira ticket implementation plan phase by phase. Use when asked to "work on ticket CM-1234", "implement the plan for CM-1234", "start working on the plan", or "execute the next phase". Reads the plan from docs/<ticket-id>/plan.md, presents each phase for user approval before coding, updates progress after each phase, and operates inside the dedicated git worktree. After the last phase is complete, presents a full change summary for user review and then creates a GitHub draft PR using the create-draft-pr skill.'
---

# Work on Ticket Plan

Execute an existing implementation plan for a Jira ticket phase by phase. Requires `setup-new-work` and `plan-jira-ticket` skills to have been run first.

## Role

You are a senior software engineer who follows a plan carefully. You read each phase, explain exactly what you will do, wait for approval, implement the changes, and update progress before moving to the next phase.

## When to Use This Skill

- User says "work on ticket CM-1234", "implement plan for CM-1234", or "start the next phase"
- User wants to execute a pre-existing plan located at `docs/<ticket-id>/plan.md`
- User wants phase-by-phase implementation with approval gates

## Prerequisites

- `setup-new-work` skill has been run — the worktree exists at `./worktrees/<ticket-id>-<platform>/`
- `plan-jira-ticket` skill has been run — both `docs/<ticket-id>/plan.md` and `docs/<ticket-id>/progress.md` exist in the worktree

## Parameters

| Parameter    | Required | Description                        | Example   |
|-------------|----------|------------------------------------|-----------|
| `ticket-id` | Yes      | The Jira ticket key                | `CM-4873` |

---

## Workflow

### Step 1 — Validate Input & Environment

1. The user **must** provide a Jira ticket key (e.g. `CM-4873`). If not provided, ask for it.
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
4. Verify `docs/<ticket-id>/plan.md` exists inside the worktree. If not, stop and instruct the user to run the `plan-jira-ticket` skill first.
5. Verify `docs/<ticket-id>/progress.md` exists inside the worktree. If not, stop and instruct the user to run the `plan-jira-ticket` skill first.
6. `cd` into the worktree:

```bash
cd "$worktree"
```

---

### Step 2 — Read and Understand the Plan

1. Read `docs/<ticket-id>/plan.md` in full.
2. Read `docs/<ticket-id>/progress.md` to determine which phases/tasks are already completed.
3. Identify all phases defined in the plan (Section 3: Proposed Solution → Implementation Phases).
4. Determine the **next incomplete phase** based on the progress tracker.
5. Build a clear mental model of the full scope before proceeding.

---

### Step 3 — Present Phase Details for Approval

For the **next incomplete phase**, present a structured summary to the user **before writing any code**:

```
## Phase <N>: <Phase Title>

**Goal:** <GOAL-xxx description>

**Tasks to implement:**

| Task     | Description                     | File(s)              |
|----------|---------------------------------|----------------------|
| TASK-xxx | <what will be done>             | `path/to/file`       |
| TASK-xxx | <what will be done>             | `path/to/file`       |

**Tests to add/update:**

| Test     | Type        | Description                     |
|----------|-------------|---------------------------------|
| TEST-xxx | Unit        | <what it verifies>              |

**Files that will be created or modified:**

| File              | Action   | Reason                          |
|-------------------|----------|---------------------------------|
| `path/to/file`    | Modify   | <why>                           |
| `path/to/newfile` | Create   | <why>                           |

---
Do you approve this phase? Reply **yes** to proceed, or provide feedback to adjust the plan.
```

**Do NOT write any code until the user explicitly approves.**

---

### Step 4 — Implement the Approved Phase

Once the user approves:

1. Update `docs/<ticket-id>/progress.md`:
   - Change tasks in this phase from 🔵 Planned → 🟡 In Progress.
   - Update the top-level status to `🟡 In Progress`.
   - Add an entry to the Daily Log.

2. Implement each task in the phase sequentially:
   - Follow the existing code conventions observed in the plan's Codebase Analysis.
   - Make atomic, focused changes — one task at a time.
   - Do not modify files outside the scope of the current phase.

3. After all tasks in the phase are implemented:
   - Run the relevant tests to verify correctness:

   ```bash
   # Run tests relevant to modified files
   ```

4. Update `docs/<ticket-id>/progress.md`:
   - Change tasks in this phase from 🟡 In Progress → 🟢 Completed.
   - Update the date column for each completed task.
   - Add evidence/artefacts if applicable (test output, etc.).
   - Add a Daily Log entry summarising what was done.

---

### Step 5 — Phase Completion Report

After implementing the phase, present a completion summary to the user:

```
## ✅ Phase <N> Complete: <Phase Title>

**Tasks completed:**
- [x] TASK-xxx — <description>
- [x] TASK-xxx — <description>

**Tests run:** <pass/fail summary>

**Files changed:**
- `path/to/file` — <what changed>

**Progress:** Phase <N> of <total> complete.

---
Please review the changes. When ready, reply **commit** to commit this phase's changes, or provide feedback.
```

**Do NOT start the next phase until the phase changes have been committed.**

---

### Step 5b — Commit Phase Changes

After the user approves the phase output:

1. Stage only the files changed in this phase:

   ```bash
   git add <files changed in this phase>
   ```

2. Load and follow the `conventional-commit` skill to generate and execute the commit message.
   - The commit should cover only this phase's changes.
   - The commit type should reflect the nature of the work (e.g. `feat`, `fix`, `refactor`, `test`).

3. After the commit is made, present a brief confirmation:

   ```
   ✅ Phase <N> changes committed.

   Ready to proceed to Phase <N+1>. Reply **next** to continue, or provide feedback.
   ```

**Do NOT start the next phase until the user explicitly replies to proceed.**

---

### Step 6 — Repeat for Each Phase

Repeat Steps 3–5b for each subsequent phase, always:
- Presenting task details and waiting for approval before coding.
- Updating progress both at the start and end of each phase.
- Committing only the current phase's changes using the `conventional-commit` skill before advancing.
- Waiting for user sign-off before starting the next phase.

---

### Step 7 — Final Summary (After Last Phase)

Once all phases have been completed and committed, present a comprehensive summary of all changes made across the entire implementation:

```
## 🎉 Implementation Complete: <ticket-id>

All phases of the implementation plan have been completed.

---

### Summary of Changes

**Phases completed:** <N> of <N>

**All files changed:**

| File | Action | Phase | Description |
|------|--------|-------|-------------|
| `path/to/file` | Modified | Phase 1 | <what changed> |
| `path/to/file` | Created  | Phase 2 | <what changed> |

**Tests added/updated:**

| Test file | Type | What it verifies |
|-----------|------|-----------------|
| `path/to/test` | Unit | <description> |

**Key decisions made:**
- <any notable implementation decision or deviation from the original plan>

---

Please review the complete summary above.
Reply **create PR** to open a draft pull request, or provide feedback if further changes are needed.
```

**Do NOT create the PR until the user explicitly approves the summary.**

---

### Step 8 — Create Draft PR

Once the user approves the final summary (replies "create PR" or equivalent):

Load and follow the `create-draft-pr` skill to create the GitHub draft pull request.

Pass the ticket ID so the skill can:
- Build the PR title and description from the ticket details and code diff
- Create the PR as a draft
- Assign it to the current user

After the PR is created, confirm with the URL and status reported by the `create-draft-pr` skill.

---

### Step 7 — Final Completion Report

When all phases are complete:

1. Update `docs/<ticket-id>/progress.md`:
   - Set top-level status to `🟢 Completed`.
   - Add a final Daily Log entry.

2. Present the final summary:

```
## 🎉 Implementation Complete: <ticket-id>

All phases have been implemented successfully.

**Summary:**
- Phases completed: <N>
- Tasks completed: <M>
- Tests added/updated: <K>

**Files changed:**
- `path/to/file` — <summary of change>

**Next steps:**
- Review `docs/<ticket-id>/progress.md` for full details.
- Run the full test suite to confirm nothing is broken.
- Open a pull request.
```

---

## Progress Update Rules

Apply these rules every time `docs/<ticket-id>/progress.md` is updated:

1. Update task status emoji:
   - 🔵 Planned → 🟡 In Progress (when phase begins)
   - 🟡 In Progress → 🟢 Completed (when task is done)
   - 🟡 In Progress → 🔴 Blocked (when a blocker is encountered)
2. Set the top-level status line to reflect the current overall state.
3. Fill in the date column with `YYYY-MM-DD` when a task changes state.
4. Append a new entry to the **Daily Log** section.
5. Add notes for blocked tasks explaining the blocker.
6. Link evidence/artefacts (test results, screenshots) as they are produced.

---

## Approval Gate Rules

- **Never write code before user approves a phase.**
- **Never start the next phase before user confirms the previous phase is reviewed.**
- If the user provides feedback instead of approving, update the plan/approach accordingly and re-present the phase summary.
- If a blocker is found during implementation, stop, mark the task 🔴 Blocked in progress.md, and report it to the user before continuing.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Worktree not found | Run the `setup-new-work` skill first |
| Plan not found | Run the `plan-jira-ticket` skill first |
| Progress file missing | Run the `plan-jira-ticket` skill first |
| All phases already completed | Inform user — no further work needed |
| User skips a phase | Warn that phases may have dependencies; proceed only if user confirms |
| Tests fail after implementation | Stop, report failures, do not mark phase as complete until resolved |

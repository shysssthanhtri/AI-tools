---
name: plan-jira-ticket
description: "Create a work plan for a Jira ticket. Use when asked to plan work, create a plan for a ticket, break down a Jira issue, or prepare implementation steps for a Jira ticket ID like CM-1234. Reads local ticket details, scans the codebase, produces an implementation plan, and tracks progress under docs/<ticket-id>/."
---

# Plan Jira Ticket

Create a comprehensive, shareable work plan for a Jira ticket. The plan captures requirements, codebase analysis, proposed solution, task breakdown, and progress — all written to `./docs/<ticket-id>/` so the team can review and collaborate.

## Role

You are a senior software engineer who plans before coding. You read the ticket thoroughly, study the relevant parts of the codebase, design a clear solution, and document everything so any teammate can pick up the work.

## When to Use This Skill

- User says "plan ticket CM-1234", "create a plan for CM-1234", or "break down CM-1234"
- User wants to understand what a Jira ticket involves before writing code
- User wants a documented plan to share with the team

## Prerequisites

- The `setup-new-work` skill must have been run first. It creates an isolated git worktree at `./worktrees/<ticket-id>-<platform>/` with a dedicated branch and a local copy of the ticket details.
- The Jira ticket details must already exist locally at `docs/<ticket-id>/<ticket-id>.md` inside the worktree (created by the `setup-new-work` skill). If the file does not exist, tell the user to run the `setup-new-work` skill first.

## Important: Work Inside the Worktree

All planning, implementation, and documentation **must** happen inside the worktree directory prepared by the `setup-new-work` skill:

```
./worktrees/<ticket-id>-<platform>/
```

This ensures:
- Work is isolated from other in-progress tasks on different branches.
- All file paths in the plan (e.g. `docs/<ticket-id>/plan.md`) are relative to the worktree root.
- The dedicated branch (`feature/<ticket-id>-<slug>`) is already checked out.

If the current working directory is **not** the worktree, `cd` into it before proceeding:

```bash
cd "$(git rev-parse --show-toplevel)/worktrees/<ticket-id>-<platform>"
```

## Parameters

| Parameter    | Required | Description                        | Example   |
|-------------|----------|------------------------------------|-----------|
| `ticket-id` | Yes      | The Jira ticket key                | `CM-4873` |

## Workflow

### Step 1 — Validate Input

- The user **must** provide a Jira ticket key (e.g. `CM-4873`). If not provided, ask for it.
- Verify the worktree directory exists at `./worktrees/<ticket-id>-<platform>/`. If it does not, stop and instruct the user to run the `setup-new-work` skill first.
- Ensure the current working directory is the worktree root. If not, `cd` into it.
- Verify that `docs/<ticket-id>/<ticket-id>.md` exists inside the worktree. If it does not, stop and instruct the user to run the `setup-new-work` skill first.

### Step 2 — Understand the Ticket

1. Read `docs/<ticket-id>/<ticket-id>.md` completely.
2. Extract and internalise:
   - **Summary** — what is being asked
   - **Description** — full context
   - **Acceptance Criteria** — the definition of done
   - **Subtasks / Linked Issues** — related work
   - **Priority & Type** — urgency and nature of work
3. Write a concise **Requirements Summary** (bullet points) capturing every discrete requirement and acceptance criterion. Save it in the plan document (Step 5).

### Step 3 — Scan the Codebase

Investigate the repository to understand what exists and what needs to change:

1. **Identify affected areas** — Based on the ticket requirements, search for relevant files, classes, functions, routes, configs, and tests.
2. **Read key files** — Open and read the most relevant source files to understand current behaviour.
3. **Map dependencies** — Note which components interact with each other and how changes might ripple.
4. **Check existing tests** — Find tests covering the affected code paths.
5. **Note conventions** — Observe coding patterns, naming conventions, and architectural patterns used in the project.

Document findings as a **Codebase Analysis** section in the plan.

### Step 4 — Design the Solution

Based on the requirements and codebase analysis:

1. **Propose an approach** — Describe the solution at a high level (what changes, where, and why).
2. **List alternatives considered** — Briefly note other approaches and why they were rejected.
3. **Identify risks** — Call out anything uncertain, complex, or potentially breaking.
4. **Break into phases** — Divide the work into sequential implementation phases, each with a clear goal.
5. **Define tasks** — Within each phase, create atomic, actionable tasks with specific file paths and descriptions.
6. **Define tests** — List tests that must be added or updated.

### Step 5 — Write the Plan Document

Create the plan file at `docs/<ticket-id>/plan.md` using the template below.

### Step 6 — Create the Progress Tracker

Create a progress file at `docs/<ticket-id>/progress.md` using the progress template below. This file tracks task-level status and is updated as work proceeds.

### Step 7 — Confirm to User

Print a summary:

```
✅ Plan ready for <ticket-id>

  Plan:      docs/<ticket-id>/plan.md
  Progress:  docs/<ticket-id>/progress.md

  Phases:    <N> phases, <M> tasks total
  Next step: Review the plan, then start implementation.
```

## Plan Document Template

```markdown
---
ticket: <ticket-id>
goal: "<Jira summary>"
date_created: <YYYY-MM-DD>
last_updated: <YYYY-MM-DD>
status: 'Planned'
tags: [<comma-separated tags e.g. feature, bug, refactor>]
---

# Plan: <ticket-id> — <Jira Summary>

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

## 1. Requirements Summary

<Concise bullet points distilled from the ticket description and acceptance criteria.>

- **REQ-001**: <Requirement 1>
- **REQ-002**: <Requirement 2>
- **AC-001**: <Acceptance criterion 1>
- **AC-002**: <Acceptance criterion 2>

## 2. Codebase Analysis

<Summary of findings from scanning the repository.>

### Affected Files & Components

| # | File / Component | Role | Impact |
|---|-----------------|------|--------|
| 1 | `path/to/file.php` | <What it does> | <What needs to change> |

### Current Behaviour

<Describe how the system works today in the affected area.>

### Existing Test Coverage

| Test File | Covers |
|-----------|--------|
| `tests/path/to/Test.php` | <What it tests> |

### Conventions & Patterns Observed

- <Pattern 1>
- <Pattern 2>

## 3. Proposed Solution

<High-level description of the approach.>

### Implementation Phases

#### Phase 1: <Phase Title>

- **GOAL-001**: <Goal description>

| Task | Description | File(s) |
|------|-------------|---------|
| TASK-001 | <Atomic task description> | `path/to/file` |
| TASK-002 | <Atomic task description> | `path/to/file` |

#### Phase 2: <Phase Title>

- **GOAL-002**: <Goal description>

| Task | Description | File(s) |
|------|-------------|---------|
| TASK-003 | <Atomic task description> | `path/to/file` |
| TASK-004 | <Atomic task description> | `path/to/file` |

## 4. Alternatives Considered

- **ALT-001**: <Alternative approach and why it was rejected>
- **ALT-002**: <Alternative approach and why it was rejected>

## 5. Testing Strategy

| # | Test | Type | Description |
|---|------|------|-------------|
| TEST-001 | <Test name> | Unit | <What it verifies> |
| TEST-002 | <Test name> | Integration | <What it verifies> |

## 6. Dependencies

- **DEP-001**: <Dependency description>

## 7. Risks & Assumptions

- **RISK-001**: <Risk description and mitigation>
- **ASSUMPTION-001**: <Assumption description>

## 8. Files Affected

| # | File | Action | Description |
|---|------|--------|-------------|
| FILE-001 | `path/to/file` | Modify | <What changes> |
| FILE-002 | `path/to/new-file` | Create | <What it does> |

## 9. References

- Ticket: docs/<ticket-id>/<ticket-id>.md
- <Any other relevant links or documents>
```

## Progress Tracker Template

```markdown
# Progress: <ticket-id> — <Jira Summary>

> Last updated: <YYYY-MM-DD>

## Status: 🔵 Planned

Status legend: 🔵 Planned | 🟡 In Progress | 🟢 Completed | 🔴 Blocked

## Task Progress

| Task | Description | Status | Notes | Date |
|------|-------------|--------|-------|------|
| TASK-001 | <Description> | 🔵 | | |
| TASK-002 | <Description> | 🔵 | | |
| TASK-003 | <Description> | 🔵 | | |

## Evidence & Artefacts

<Link to test results, screenshots, logs, or any supporting evidence collected during implementation.>

| # | Artefact | Description | Path |
|---|----------|-------------|------|
| 1 | | | |

## Daily Log

### <YYYY-MM-DD>

- <What was done today>

## Open Questions

- [ ] <Question 1>
- [ ] <Question 2>

## Decisions Made

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | | | |
```

## Updating Progress

When work is in progress, update `docs/<ticket-id>/progress.md`:

1. Change task status emoji (🔵 → 🟡 → 🟢 or 🔴).
2. Add notes for completed or blocked tasks.
3. Fill in the date column.
4. Append entries to the Daily Log section.
5. Record any decisions in the Decisions Made table.
6. Link evidence/artefacts as they are produced.
7. Update the top-level status line to reflect overall progress.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Ticket doc not found | Run the `setup-new-work` skill first to fetch and store ticket details |
| Plan seems incomplete | Re-run Step 3 with broader codebase scanning |
| Too many tasks | Merge related tasks or split into multiple tickets |
| Requirements unclear | Add open questions to progress.md and discuss with the team |

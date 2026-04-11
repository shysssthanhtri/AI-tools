---
name: plan-jira-ticket
description: "Create a work plan for a Jira ticket. Use when asked to plan work, create a plan for a ticket, break down a Jira issue, or prepare implementation steps for a Jira ticket ID like CM-1234. Reads local ticket details, deeply analyzes requirements and root causes, proposes up to three optimized solutions (not workarounds) with pros and cons, asks the user to choose, then finalizes the plan with that choice under docs/<ticket-id>/."
---

# Plan Jira Ticket

Create a comprehensive, shareable work plan for a Jira ticket. The plan captures **fully understood requirements**, **evidence-based root cause or constraint analysis**, **up to three legitimate optimized solution options** (with pros and cons), a **user decision checkpoint**, and—only after the user chooses—the **detailed implementation plan**, tasks, and progress tracking under `./docs/<ticket-id>/` so the team can review and collaborate.

## Role

You are a senior software engineer who plans before coding. You:

1. **Understand requirements completely** — Read the ticket as a product engineer would: explicit asks, implicit expectations, acceptance criteria, edge cases, and what “done” means.
2. **Investigate deeply** — Do not stop at surface symptoms. Search the codebase and (where relevant) configs, logs, and dependencies to find **root causes** (bugs/regressions) or **true constraints and leverage points** (features/refactors). Prefer evidence (file paths, call chains, failing tests) over guesses.
3. **Propose real solutions** — Each option must be a **sound, optimized approach** aligned with the project’s architecture and long-term maintainability. **Do not treat shortcuts, hacks, or workarounds as peer “solutions.”** If a workaround is the only viable path temporarily, document it explicitly as **technical debt** with a follow-up, not as Option A/B/C.
4. **Let the user decide** — Present up to **three** options with clear **pros and cons**, then **stop and ask the user to choose**. The skill’s planning work is **not complete** until they choose.
5. **Finalize the plan** — After the user selects an option (or a justified hybrid), **update `plan.md`** to lock in **Chosen Solution**, phases, tasks, tests, and risks. Then confirm completion.

## When to Use This Skill

- User says "plan ticket CM-1234", "create a plan for CM-1234", or "break down CM-1234"
- User wants to understand what a Jira ticket involves before writing code
- User wants a documented plan to share with the team

## Prerequisites

- The `setup-new-work` skill must have been run first. It creates an isolated git worktree at `./worktrees/<ticket-id>-<platform>/`, checks out branch `feature/<ticket-id>-<slug>-<platform>`, and writes `docs/<ticket-id>/<ticket-id>.md` (local story snapshot). This skill still creates `plan.md` and `progress.md` under `docs/<ticket-id>/`, and runs Step 1b only if the ticket summary file is missing (e.g. legacy worktrees).

## Important: Work Inside the Worktree

All planning, implementation, and documentation **must** happen inside the worktree directory prepared by the `setup-new-work` skill:

```
./worktrees/<ticket-id>-<platform>/
```

This ensures:
- Work is isolated from other in-progress tasks on different branches.
- All file paths in the plan (e.g. `docs/<ticket-id>/plan.md`) are relative to the worktree root.
- The dedicated branch (`feature/<ticket-id>-<slug>-<platform>`) is already checked out.

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

### Step 1b — Ensure ticket summary file

If `docs/<ticket-id>/<ticket-id>.md` does not exist inside the worktree, create it:

1. Call `mcp_com_atlassian_getAccessibleAtlassianResources` for `cloudId`, then `mcp_com_atlassian_getJiraIssue` with `issueIdOrKey` = ticket key and `responseContentFormat` = `markdown`.
2. `mkdir -p docs/<ticket-id>` and write `docs/<ticket-id>/<ticket-id>.md` with the issue summary, description, status, priority, type, people, labels, sprint, acceptance criteria, subtasks/links, and a short Notes section — matching the level of detail you need for planning (mirror the structure you would use for a team-facing ticket brief).

### Step 2 — Understand Requirements (Deep)

1. Read `docs/<ticket-id>/<ticket-id>.md` completely.
2. Extract and internalise:
   - **Summary** — what is being asked
   - **Description** — full context
   - **Acceptance Criteria** — the definition of done
   - **Subtasks / Linked Issues** — related work
   - **Priority & Type** — urgency and nature of work
3. Go beyond keywords: note **implicit requirements**, **out of scope** boundaries, **user-visible vs internal** changes, and **non-functional** expectations (performance, security, compatibility) if stated or clearly implied.
4. Write a concise **Requirements Summary** (bullet points) capturing every discrete requirement and acceptance criterion. Save it in the plan document (Step 5a).

### Step 3 — Deep Search: Codebase & Root Cause / Constraints

Investigate the repository until you can explain **why** the gap exists (bug) or **where** the right extension points are (feature):

1. **Identify affected areas** — Based on the ticket requirements, search for relevant files, classes, functions, routes, configs, and tests. Use multiple search strategies (symbols, strings, Git history if useful).
2. **Read key files** — Open and read the most relevant source files to understand current behaviour and data flow.
3. **Map dependencies** — Note which components interact and how changes might ripple.
4. **Root cause (defects)** — Trace from symptom to cause: reproduction path, failure mode, and the **underlying** defect (logic error, wrong assumption, missing validation, race, config drift, etc.). Cite evidence.
5. **Constraints & leverage (non-defects)** — For features/refactors, state architectural constraints, APIs you must use, and the **smallest correct change surface** that satisfies requirements.
6. **Check existing tests** — Find tests covering affected paths; note gaps.
7. **Note conventions** — Observe patterns the codebase already uses.

Document findings as **Codebase Analysis** and a dedicated **Root Cause Analysis** (bugs) or **Problem & Constraint Analysis** (non-bugs) section in the plan.

### Step 4 — Solution Options (Up to Three, Optimized — Not Workarounds)

1. **Generate up to three distinct solution strategies.** Each must be a **legitimate** approach you could defend in design review: maintainable, aligned with existing architecture, and **optimized** for the stated goals (correctness, clarity, performance, operational cost—as relevant). Prefer fewer than three if only one or two are genuinely viable.
2. **Exclude workarounds from the option list.** Do not present “patch over symptoms,” duplicate logic to avoid refactoring, silent failures, or hard-coded env-specific hacks as Options A/B/C. If the team might still need a temporary mitigation, put it under **Risks / Mitigations** or **Not recommended**, not as a peer solution.
3. For **each** option, provide:
   - **Summary** — What you would build or change.
   - **Pros** — Bullet list (e.g. maintainability, testability, performance, risk, time to ship).
   - **Cons** — Bullet list (trade-offs, complexity, migration cost, coupling).
4. Optionally add a **Recommendation** (one paragraph) if one option is clearly superior on evidence—still **require** user confirmation before locking the plan.
5. **Do not** write full implementation phases/tasks for a single chosen path yet (that comes after user choice). You may note high-level effort or risk per option.

### Step 5a — First Write: Analysis + Options (Before User Choice)

Create or update `docs/<ticket-id>/plan.md` using the **first-write template** below (sections 1–3, 4 for solution options, plus testing/risks placeholders as needed). Set frontmatter `status: 'Awaiting decision'` until the user chooses.

Create or update `docs/<ticket-id>/progress.md` with status indicating planning is waiting on solution choice (see progress template).

### Step 6 — User Choice (Mandatory Checkpoint)

Present the solution options clearly in the chat:

- Number options **Solution 1**, **Solution 2**, **Solution 3** (or fewer).
- Repeat **pros** and **cons** in a scannable form (or summarize with “see plan.md §3”).
- **Ask explicitly**: which option should be adopted (by number), or whether to combine aspects (describe how).

**Stop here until the user responds.** Do not mark the skill complete and do not finalize implementation phases until you have their choice.

### Step 7 — Finalize Plan After User Choice

When the user chooses (or specifies a hybrid with enough detail):

1. **Update `docs/<ticket-id>/plan.md`**:
   - Set `status: 'Planned'` (or `'Ready for implementation'`) and refresh `last_updated`.
   - Add section **Chosen Solution** naming the selected option and **why** (one short paragraph, referencing user decision).
   - Move non-selected options to **Alternatives Not Selected** (brief bullets: what they were, why not chosen).
   - Complete **Implementation Phases**, task tables, **Testing Strategy**, **Files Affected**, **Dependencies**, **Risks & Assumptions** for the **chosen** path only.
2. **Update `docs/<ticket-id>/progress.md`** — Reset task table to match the chosen plan; set overall status to 🔵 Planned (or 🟡 In Progress if implementation already started).
3. **Confirm to user** with the completion summary (Step 8).

### Step 8 — Confirm Completion

After the plan is updated with the user’s choice:

```
✅ Plan finalized for <ticket-id>

  Plan:      docs/<ticket-id>/plan.md
  Progress:  docs/<ticket-id>/progress.md

  Chosen:    Solution <N> — <short label>
  Phases:    <N> phases, <M> tasks total
  Next step: Review the plan, then start implementation.
```

If the user has not yet chosen a solution in this session, use instead:

```
⏸️ Plan awaiting your decision — <ticket-id>

  Plan:      docs/<ticket-id>/plan.md

  Please choose Solution 1, 2, or 3 (or describe a hybrid).
  The implementation breakdown will be added after you decide.
```

## Plan Document Template (First Write — Awaiting Decision)

Use this structure until the user selects a solution. After selection, merge into the **full template** below.

```markdown
---
ticket: <ticket-id>
goal: "<Jira summary>"
date_created: <YYYY-MM-DD>
last_updated: <YYYY-MM-DD>
status: 'Awaiting decision'
tags: [<comma-separated tags e.g. feature, bug, refactor>]
---

# Plan: <ticket-id> — <Jira Summary>

![Status: Awaiting decision](https://img.shields.io/badge/status-Awaiting%20decision-yellow)

## 1. Requirements Summary

- **REQ-001**: <Requirement 1>
- **AC-001**: <Acceptance criterion 1>
- <Implicit requirements or scope notes as needed>

## 2. Codebase Analysis

<Summary of findings.>

### Affected Files & Components

| # | File / Component | Role | Impact |
|---|-----------------|------|--------|
| 1 | `path/to/file` | <What it does> | <What needs to change> |

### Current Behaviour

<How the system works today in the affected area.>

### Root Cause Analysis (bugs) / Problem & Constraint Analysis (non-bugs)

- **Evidence**: <paths, flows, tests>
- **Root cause or core constraint**: <concise>
- **Why superficial fixes fail**: <if relevant>

### Existing Test Coverage

| Test File | Covers |
|-----------|--------|
| `tests/...` | <What it tests> |

### Conventions & Patterns Observed

- <Pattern 1>

## 3. Solution Options (Choose One)

You must select one option (or approve a stated hybrid) before implementation tasks are finalized.

### Solution 1 — <Short name>

**Summary:** <2–4 sentences.>

**Pros**
- <pro 1>
- <pro 2>

**Cons**
- <con 1>
- <con 2>

### Solution 2 — <Short name>

**Summary:** …

**Pros** / **Cons** — …

### Solution 3 — <Short name>

(Include only if there is a third genuine option.)

**Summary:** …

**Pros** / **Cons** — …

### Recommendation (Optional)

<If one option is clearly stronger, say why—user still confirms.>

## 4. Testing Strategy (Preview)

<High-level: what kinds of tests will matter regardless of option; detailed table after choice.>

## 5. Risks & Assumptions (Preview)

- **RISK-001**: …

## 6. References

- Ticket: docs/<ticket-id>/<ticket-id>.md
```

## Plan Document Template (After User Choice — Full)

After the user chooses, ensure `plan.md` includes everything from the first-write template **plus** the sections below, with `status: 'Planned'`.

```markdown
---
ticket: <ticket-id>
goal: "<Jira summary>"
date_created: <YYYY-MM-DD>
last_updated: <YYYY-MM-DD>
status: 'Planned'
tags: [<tags>]
---

# Plan: <ticket-id> — <Jira Summary>

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

## 1. Requirements Summary

<As before>

## 2. Codebase Analysis

<As before>

## 3. Chosen Solution

**Selected:** Solution <N> — <Short name>

**Decision:** <User chose this option / hybrid because … one short paragraph.>

### Implementation Phases

#### Phase 1: <Phase Title>

- **GOAL-001**: <Goal description>

| Task | Description | File(s) |
|------|-------------|---------|
| TASK-001 | <Atomic task description> | `path/to/file` |

#### Phase 2: <Phase Title>

…

## 4. Alternatives Not Selected

- **Solution X**: <One line what it was; why not chosen (reference user or trade-offs).>

## 5. Testing Strategy

| # | Test | Type | Description |
|---|------|------|-------------|
| TEST-001 | <Test name> | Unit | <What it verifies> |

## 6. Dependencies

- **DEP-001**: <Dependency description>

## 7. Risks & Assumptions

- **RISK-001**: <Risk and mitigation>

## 8. Files Affected

| # | File | Action | Description |
|---|------|--------|-------------|
| FILE-001 | `path/to/file` | Modify | <What changes> |

## 9. References

- Ticket: docs/<ticket-id>/<ticket-id>.md
```

## Progress Tracker Template

```markdown
# Progress: <ticket-id> — <Jira Summary>

> Last updated: <YYYY-MM-DD>

## Status: 🔵 Planned

(Status may be "Awaiting solution choice" before the user picks an option.)

Status legend: 🔵 Planned | 🟡 In Progress | 🟢 Completed | 🔴 Blocked

## Task Progress

| Task | Description | Status | Notes | Date |
|------|-------------|--------|-------|------|
| TASK-001 | <Description> | 🔵 | | |

## Evidence & Artefacts

| # | Artefact | Description | Path |
|---|----------|-------------|------|
| 1 | | | |

## Daily Log

### <YYYY-MM-DD>

- <What was done>

## Open Questions

- [ ] <Question 1>

## Decisions Made

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | Chose Solution <N> | User selection / team criteria | <date> |
```

## Updating Progress

When work is in progress, update `docs/<ticket-id>/progress.md`:

1. Change task status emoji (🔵 → 🟡 → 🟢 or 🔴).
2. Add notes for completed or blocked tasks.
3. Fill in the date column.
4. Append entries to the Daily Log section.
5. Record any decisions in the Decisions Made table (including solution choice).
6. Link evidence/artefacts as they are produced.
7. Update the top-level status line to reflect overall progress.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Ticket doc not found | Run Step 1b (fetch Jira and write `docs/<ticket-id>/<ticket-id>.md`) or ensure the worktree path is correct |
| Plan seems incomplete | Re-run Step 3 with broader codebase scanning and deeper tracing |
| Too many tasks | Merge related tasks or split into multiple tickets |
| Requirements unclear | Add open questions to progress.md and discuss with the team |
| User wants a fourth option | Add only if it is a distinct optimized approach; otherwise refine the three or explain why options are exhausted |
| Only workarounds seem possible | Document as risk/mitigation and technical-debt follow-up; do not label workarounds as full "solutions" |

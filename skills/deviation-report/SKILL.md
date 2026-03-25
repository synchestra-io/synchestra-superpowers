---
name: deviation-report
description: Use after plan execution to compare what was planned vs what actually happened
---

# Deviation Report

## Overview

After executing a plan, generate a deviation report comparing the original plan against actual task execution. This reveals unplanned work, skipped tasks, and scope changes — critical feedback for improving future plans.

**Announce at start:** "I'm using the deviation-report skill to compare plan vs actual execution."

## Prerequisites

- Synchestra must be installed and initialized
- A plan must have been executed with Synchestra task tracking
- Tasks must be in terminal states (completed, failed, or still blocked)

```bash
synchestra task list
```

If the command fails, tell the user:
> "Synchestra is not initialized in this project. Use `superpowers:install-synchestra` to get started."

## Generate Report

### Step 1: Identify the plan

Find the plan file and its corresponding Synchestra parent task:

```bash
# List plan-level tasks
synchestra task list --status completed
synchestra task list --status in_progress
```

Read the original plan file from `docs/superpowers/plans/`.

### Step 2: Gather task state

```bash
# Get all tasks under the plan's parent task
synchestra task info <parent-task-id>
```

### Step 3: Compare

For each planned task, check:
- Was it completed? At what status?
- Were the planned files actually touched? (`git diff --name-only` against the branch)
- Were there unplanned commits?

### Step 4: Generate report

Format as a deviation report:

```markdown
# Deviation Report: <Feature Name>

**Plan:** `docs/superpowers/plans/<filename>.md`
**Executed:** <date range>

## Summary
- Planned tasks: N
- Completed: X
- Failed: Y
- Blocked: Z
- Skipped: W
- Unplanned work: U items

## Task-by-Task

| # | Task | Planned | Actual | Delta |
|---|------|---------|--------|-------|
| 1 | Set up models | completed | completed | On plan |
| 2 | Add endpoints | completed | completed | +1 extra endpoint added |
| 3 | Auth middleware | completed | failed | Blocked by missing dep |
| 4 | Tests | completed | skipped | Depended on Task 3 |

## Unplanned Work
- Fixed CI configuration (not in plan)
- Added logging middleware (discovered during Task 2)

## Observations
- Task 3 failure cascaded to Task 4 — consider adding fallback paths
- Unplanned logging work suggests plan missed observability requirements
```

## When to Use

- After completing a plan execution (or partially completing)
- During retrospectives
- When deciding whether to re-plan remaining work
- To improve future planning (what do plans consistently miss?)


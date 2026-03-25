# Synchestra Superpowers Rebranding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove all optional/fallback Synchestra logic, rebrand synchestra-superpowers as its own product with Synchestra mandatory, and rewrite onboarding to handle the full 3-layer detection sequence.

**Architecture:** Synchestra is now mandatory at all layers (CLI, core skills, workflow skills). Every "if synchestra available" conditional and "without Synchestra" fallback is removed. The `synchestra-state` skill is deleted (its behavior becomes the default). The `install-synchestra` skill becomes `onboarding` with 3-layer detection and project topology choice. README, plugin.json, and package.json get Synchestra branding.

**Tech Stack:** Markdown skill files, JSON config files

---

### Task 1: Delete `synchestra-state` skill

**Files:**
- Delete: `skills/synchestra-state/SKILL.md`

- [ ] **Step 1: Remove the skill directory**

```bash
rm -rf skills/synchestra-state/
```

- [ ] **Step 2: Verify deletion**

```bash
ls skills/synchestra-state/ 2>&1
# Expected: "No such file or directory"
```

- [ ] **Step 3: Commit**

```bash
git add -A skills/synchestra-state/
git commit -m "remove: delete synchestra-state skill — its behavior becomes the default"
```

---

### Task 2: Rewrite `writing-plans` Synchestra integration from optional to mandatory

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1: Replace the "Synchestra Integration (Optional)" section**

Replace the entire section starting at `## Synchestra Integration (Optional)` (lines 127-144) with:

```markdown
## Synchestra Task Creation

After saving the plan file, create persistent Synchestra tasks:

1. Create a parent task for the plan:
   ```bash
   synchestra task new --title "Plan: <feature-name>"
   ```
2. For each `### Task N:` section, create a child task:
   ```bash
   synchestra task new --title "Task N: <component>" --parent <plan-task-id>
   ```
3. Enqueue all tasks so agents can discover them:
   ```bash
   synchestra task enqueue <id>
   ```

This enables cross-session continuity and multi-agent coordination.
```

- [ ] **Step 2: Verify the file reads correctly**

Read through the modified file to confirm no broken references or orphaned text.

- [ ] **Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "update: make Synchestra task creation mandatory in writing-plans"
```

---

### Task 3: Rewrite `executing-plans` Synchestra integration from optional to mandatory

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

- [ ] **Step 1: Replace the "Synchestra Integration (Optional)" section**

Replace the entire section from `## Synchestra Integration (Optional)` through `If Synchestra is unavailable, skip all synchestra task commands — TodoWrite alone is sufficient.` (lines 65-81) with:

```markdown
## Synchestra Task State

Use persistent Synchestra task state alongside TodoWrite:

**At plan load (Step 1):**
- Check for existing Synchestra tasks matching the plan
- If found, use their state to resume (skip already-completed tasks)
- If not found, create tasks now (see writing-plans skill for task creation)

**During execution (Step 2):**
- Before each task: `synchestra task claim <id>` then `synchestra task start <id>`
- After verification: `synchestra task complete <id>`
- On blocker: `synchestra task block <id> "reason"` and stop

**Cross-session resume:** If a previous session left tasks `in_progress` or `queued`, this session picks up where it left off without re-reading the plan.
```

- [ ] **Step 2: Remove the `synchestra-state` reference from the Integration section**

In the `## Integration` section (lines 83-91), remove the `**Optional:**` block:

```markdown
**Optional:**
- **superpowers:synchestra-state** - Persistent task state across sessions
```

Replace with nothing (delete those two lines entirely).

- [ ] **Step 3: Verify the file reads correctly**

Read through the modified file to confirm coherence and no orphaned references.

- [ ] **Step 4: Commit**

```bash
git add skills/executing-plans/SKILL.md
git commit -m "update: make Synchestra task state mandatory in executing-plans"
```

---

### Task 4: Rewrite `subagent-driven-development` Synchestra integration from optional to mandatory

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

- [ ] **Step 1: Replace the "Synchestra Integration (Optional)" section**

Replace the entire section from `## Synchestra Integration (Optional)` through `If Synchestra is unavailable, skip all synchestra task commands — TodoWrite alone is sufficient.` (lines 265-285) with:

```markdown
## Synchestra Task State

Use persistent Synchestra task state alongside TodoWrite:

**At plan extraction:**
- Check for existing Synchestra tasks matching the plan (created by `writing-plans`)
- If found, use Synchestra task IDs to track state persistently
- If not found, create tasks now (see writing-plans skill for task creation)

**Per-task lifecycle:**
1. Before dispatching implementer: `synchestra task claim <id>` then `synchestra task start <id>`
2. On `DONE` + reviews passed: `synchestra task complete <id>`
3. On `BLOCKED`: `synchestra task block <id> "reason"`
4. On failure: `synchestra task fail <id> "reason"`
5. On `NEEDS_CONTEXT` re-dispatch: task stays `in_progress`

**After all tasks:**
- Synchestra state persists — if session ends mid-plan, the next session can see exactly where work stopped
- Enables deviation reports comparing planned tasks vs actual completion
```

- [ ] **Step 2: Remove the `synchestra-state` reference from the Integration section**

In the `## Integration` section (lines 287-301), remove the `**Optional:**` block:

```markdown
**Optional:**
- **superpowers:synchestra-state** - Persistent task state across sessions
```

Replace with nothing (delete those two lines entirely).

- [ ] **Step 3: Verify the file reads correctly**

Read through the modified file to confirm coherence.

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "update: make Synchestra task state mandatory in subagent-driven-development"
```

---

### Task 5: Rewrite `dispatching-parallel-agents` Synchestra integration from optional to mandatory

**Files:**
- Modify: `skills/dispatching-parallel-agents/SKILL.md`

- [ ] **Step 1: Replace the "Synchestra Integration (Optional)" section**

Replace the entire section from `## Synchestra Integration (Optional)` through `If Synchestra is unavailable, parallel dispatch works exactly as before — you just don't get cross-session coordination.` (lines 160-181) with:

```markdown
## Synchestra Task Claiming

Use Synchestra for atomic task claiming when dispatching parallel agents:

**Before dispatching agents:**
```bash
# Each agent's task should be a Synchestra task
synchestra task claim <task-id>   # Atomic — prevents two agents claiming same task
synchestra task start <task-id>
```

**After agent returns:**
```bash
synchestra task complete <task-id>  # or fail/block as appropriate
```

**Why this matters for parallel agents:**
- Git-backed optimistic locking prevents two agents from working on the same task
- If two sessions dispatch agents for the same project, Synchestra's claim protocol prevents conflicts
- Task board shows real-time view of which agents are working on what
```

- [ ] **Step 2: Verify the file reads correctly**

Read through the modified file to confirm coherence.

- [ ] **Step 3: Commit**

```bash
git add skills/dispatching-parallel-agents/SKILL.md
git commit -m "update: make Synchestra task claiming mandatory in dispatching-parallel-agents"
```

---

### Task 6: Remove fallback messaging from `task-board`

**Files:**
- Modify: `skills/task-board/SKILL.md`

- [ ] **Step 1: Replace the Prerequisites section**

Replace the entire `## Prerequisites` section (lines 14-23):

```markdown
## Prerequisites

Synchestra must be installed and the project initialized. Check:

```bash
command -v synchestra >/dev/null 2>&1 && ls synchestra.yaml 2>/dev/null
```

If not available, tell the user:
> "Synchestra is not set up in this project. Use `superpowers:install-synchestra` to enable persistent task state."
```

With:

```markdown
## Prerequisites

Synchestra must be installed and the project initialized:

```bash
synchestra task list >/dev/null 2>&1
```

If the command fails, invoke the `superpowers:onboarding` skill to set up the Synchestra stack.
```

- [ ] **Step 2: Verify the file reads correctly**

Read through the modified file to confirm coherence.

- [ ] **Step 3: Commit**

```bash
git add skills/task-board/SKILL.md
git commit -m "update: replace fallback messaging with onboarding redirect in task-board"
```

---

### Task 7: Remove "Without Synchestra" fallback from `deviation-report`

**Files:**
- Modify: `skills/deviation-report/SKILL.md`

- [ ] **Step 1: Replace the Prerequisites section**

Replace the existing `## Prerequisites` section (lines 16-25):

```markdown
## Prerequisites

- Synchestra must be installed and initialized
- A plan must have been executed with Synchestra task tracking
- Tasks must be in terminal states (completed, failed, or still blocked)

```bash
command -v synchestra >/dev/null 2>&1 && ls synchestra.yaml 2>/dev/null
```

If not available:
> "Deviation reports require Synchestra task state. Use `superpowers:install-synchestra` to enable."
```

With:

```markdown
## Prerequisites

- Synchestra must be installed and the project initialized
- A plan must have been executed with Synchestra task tracking
- Tasks must be in terminal states (completed, failed, or still blocked)

```bash
synchestra task list >/dev/null 2>&1
```

If the command fails, invoke the `superpowers:onboarding` skill to set up the Synchestra stack.
```

- [ ] **Step 2: Delete the "Without Synchestra" section**

Remove the entire `## Without Synchestra` section at the end of the file (lines 98-105):

```markdown
## Without Synchestra

If Synchestra is not available, a limited deviation report can still be generated by:
1. Reading the plan file's checkboxes (`- [x]` vs `- [ ]`)
2. Comparing against git log on the feature branch
3. Noting commits that don't map to any plan task

This is less precise than Synchestra-tracked state but still valuable.
```

Delete this entire block.

- [ ] **Step 3: Verify the file reads correctly**

Read through the modified file to confirm coherence.

- [ ] **Step 4: Commit**

```bash
git add skills/deviation-report/SKILL.md
git commit -m "update: remove 'without Synchestra' fallback from deviation-report"
```

---

### Task 8: Add onboarding detection to `using-superpowers`

**Files:**
- Modify: `skills/using-superpowers/SKILL.md`

- [ ] **Step 1: Add onboarding check section**

After the `</SUBAGENT-STOP>` block (line 8) and before the `<EXTREMELY-IMPORTANT>` block (line 10), insert:

```markdown

## Onboarding Check

Before anything else, verify the Synchestra stack is ready:

```bash
command -v synchestra >/dev/null 2>&1 && synchestra task list >/dev/null 2>&1
```

If either check fails, invoke `superpowers:onboarding` before proceeding with any other skill.

```

- [ ] **Step 2: Verify the file reads correctly**

Read through the modified file to confirm the new section flows naturally between the subagent stop block and the importance block.

- [ ] **Step 3: Commit**

```bash
git add skills/using-superpowers/SKILL.md
git commit -m "update: add onboarding detection as first step in using-superpowers"
```

---

### Task 9: Rename `install-synchestra` to `onboarding` with full rewrite

**Files:**
- Delete: `skills/install-synchestra/SKILL.md`
- Create: `skills/onboarding/SKILL.md`

- [ ] **Step 1: Create the new `skills/onboarding/` directory**

```bash
mkdir -p skills/onboarding
```

- [ ] **Step 2: Write the new `skills/onboarding/SKILL.md`**

Create `skills/onboarding/SKILL.md` with this exact content:

```markdown
---
name: onboarding
description: Use to set up the Synchestra stack — detects and installs CLI, core skills, and initializes the project
---

# Onboarding

## Overview

Synchestra Superpowers requires three layers to function: the Synchestra CLI, Synchestra core skills, and an initialized project. This skill detects which layers are missing and walks the user through setup.

**Announce at start:** "I'm using the onboarding skill to verify the Synchestra stack is ready."

## Detection Sequence

Run each check in order. Stop at the first failure and resolve it before continuing.

### 1. Synchestra CLI installed?

```bash
command -v synchestra >/dev/null 2>&1 && synchestra --version
```

**If not installed:**

Detect the platform and download the pre-built binary from the latest GitHub release:

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "$ARCH" in
  x86_64) ARCH="amd64" ;;
  aarch64|arm64) ARCH="arm64" ;;
esac
echo "Detected: ${OS}/${ARCH}"
```

Supported platforms: `darwin/arm64`, `darwin/amd64`, `linux/amd64`, `linux/arm64`, `windows/amd64`.

Present the install plan to the user:
> "Synchestra CLI is not installed. I'll download the latest release for ${OS}/${ARCH} from GitHub and place it in ~/.local/bin. OK to proceed?"

**If user confirms:**

```bash
mkdir -p ~/.local/bin
LATEST=$(curl -sL https://api.github.com/repos/synchestra-io/synchestra/releases/latest | grep '"tag_name"' | head -1 | cut -d'"' -f4)
curl -sL "https://github.com/synchestra-io/synchestra/releases/download/${LATEST}/synchestra-${OS}-${ARCH}.tar.gz" | tar xz -C ~/.local/bin synchestra
chmod +x ~/.local/bin/synchestra
```

Verify:
```bash
~/.local/bin/synchestra --version
```

If `~/.local/bin` is not in PATH, inform the user:
> "Add ~/.local/bin to your PATH to use synchestra globally: `export PATH=\"$HOME/.local/bin:$PATH\"`"

**If user declines:** Explain that Synchestra Superpowers requires the CLI and stop. Do not nag.

No Go toolchain required. Pre-built binaries only.

### 2. Synchestra core skills available?

```bash
# Core skills are installed as a Claude plugin from the synchestra ai-plugin
# Check if synchestra-* skills are loadable
```

Attempt to invoke a core skill (e.g., check if the synchestra ai-plugin is registered). If core skills are not available:

> "Synchestra core skills are not installed. I'll download the ai-plugin from the same release. OK to proceed?"

**If user confirms:**

```bash
curl -sL "https://github.com/synchestra-io/synchestra/releases/download/${LATEST}/ai-plugin.zip" -o /tmp/synchestra-ai-plugin.zip
# Install as plugin per the platform's plugin mechanism
```

**If user declines:** Explain which skills won't work (task management, feature tracking) and stop.

### 3. Project initialized?

```bash
synchestra task list >/dev/null 2>&1
```

**If already initialized:** Show task board summary and proceed.

```bash
synchestra task list
```

**If not initialized:** Present the project topology choice:

> How is your project set up?
>
> A) Single repo, default settings are fine
> B) Multiple repos in this project
> C) Dedicated state repo (for continuity across environments and parallel workstreams)

**Option A — Single repo defaults:**

No explicit initialization needed. Synchestra uses sensible defaults:
- State stored on orphan branch `synchestra-state` in the current repo
- Spec root: `spec/`
- Docs root: `docs/`

Inform the user:
> "Using default settings: state on orphan branch `synchestra-state`, specs in `spec/`, docs in `docs/`. You can reconfigure later with `synchestra project init`."

**Options B and C — Multi-repo or dedicated state repo:**

Invoke the `synchestra-project-setup` skill (from Synchestra core skills) for guided project topology configuration.

### 4. Ready

> "Synchestra stack is ready."

Show task board summary:

```bash
synchestra task list
```

## Principles

- Each step is a clear action with a single command
- User confirms each step — no silent installs
- If the user declines any step, explain what won't work and stop — don't nag
- On subsequent sessions, detection is instant (all checks are local filesystem)
- Option A users can upgrade to B/C later as needs grow

## Uninstall

If the user wants to remove Synchestra:

```bash
# Remove worktree
git worktree remove .synchestra

# Delete state branch locally
git branch -D synchestra-state

# Delete remote branch (optional)
git push origin --delete synchestra-state

# Remove marker file
rm synchestra.yaml

# Remove .gitignore entry
# (edit .gitignore manually to remove the .synchestra line)
```

State is just a git branch — fully reversible.
```

- [ ] **Step 3: Delete the old `install-synchestra` skill**

```bash
rm -rf skills/install-synchestra/
```

- [ ] **Step 4: Verify the new skill reads correctly and old skill is gone**

```bash
ls skills/install-synchestra/ 2>&1
# Expected: "No such file or directory"
ls skills/onboarding/SKILL.md
# Expected: file exists
```

- [ ] **Step 5: Commit**

```bash
git add -A skills/install-synchestra/ skills/onboarding/
git commit -m "rename: install-synchestra → onboarding with 3-layer detection and project topology choice"
```

---

### Task 10: Rewrite `hooks/session-start` Synchestra detection from optional to mandatory

**Files:**
- Modify: `hooks/session-start`

- [ ] **Step 1: Replace the optional Synchestra detection block (lines 20-33)**

Replace lines 20-33:

```bash
# Check for Synchestra availability and gather state context
synchestra_context=""
if command -v synchestra >/dev/null 2>&1; then
    if [ -f "synchestra.yaml" ] || [ -d ".synchestra" ]; then
        synchestra_state=$(synchestra task list 2>/dev/null || true)
        if [ -n "$synchestra_state" ]; then
            synchestra_context="\n\n<synchestra-state>\nSynchestra persistent task state is active in this project.\nCurrent task board:\n${synchestra_state}\n\nUse superpowers:task-board for details, superpowers:synchestra-state for task operations.\n</synchestra-state>"
        else
            synchestra_context="\n\n<synchestra-state>\nSynchestra is initialized in this project (no tasks yet). Use superpowers:synchestra-state for task operations.\n</synchestra-state>"
        fi
    else
        synchestra_context="\n\n<synchestra-available>\nSynchestra CLI is available but not initialized in this project. The user can run superpowers:install-synchestra to enable persistent task state.\n</synchestra-available>"
    fi
fi
```

With:

```bash
# Gather Synchestra state context
synchestra_context=""
if command -v synchestra >/dev/null 2>&1 && synchestra task list >/dev/null 2>&1; then
    synchestra_state=$(synchestra task list 2>/dev/null || true)
    if [ -n "$synchestra_state" ]; then
        synchestra_context="\n\n<synchestra-state>\nSynchestra persistent task state is active in this project.\nCurrent task board:\n${synchestra_state}\n\nUse superpowers:task-board for details.\n</synchestra-state>"
    else
        synchestra_context="\n\n<synchestra-state>\nSynchestra is initialized in this project (no tasks yet).\n</synchestra-state>"
    fi
else
    synchestra_context="\n\n<synchestra-setup-needed>\nSynchestra stack is not fully set up. The onboarding skill will run automatically to configure it.\n</synchestra-setup-needed>"
fi
```

Key changes:
- Removes reference to `superpowers:synchestra-state` (deleted skill)
- Removes reference to `superpowers:install-synchestra` (renamed to `onboarding`)
- Replaces three-tier optional detection with a simple binary check: either Synchestra works or onboarding is needed

- [ ] **Step 2: Verify the hook runs without errors**

```bash
bash -n hooks/session-start
# Expected: no output (syntax OK)
```

- [ ] **Step 3: Commit**

```bash
git add hooks/session-start
git commit -m "update: rewrite session-start hook — remove optional Synchestra detection, reference onboarding"
```

---

### Task 12: Update `.claude-plugin/plugin.json` with Synchestra branding

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Replace the plugin.json content**

Replace the entire content of `.claude-plugin/plugin.json` with:

```json
{
  "name": "synchestra-superpowers",
  "description": "Synchestra's workflow engine for AI agents: TDD, planning, subagent-driven development, and multi-agent coordination — powered by persistent Synchestra state",
  "version": "5.0.5",
  "author": {
    "name": "Synchestra",
    "email": "hello@synchestra.io"
  },
  "homepage": "https://github.com/synchestra-io/synchestra-superpowers",
  "repository": "https://github.com/synchestra-io/synchestra-superpowers",
  "license": "MIT",
  "keywords": ["skills", "tdd", "debugging", "collaboration", "best-practices", "workflows", "synchestra", "multi-agent"]
}
```

- [ ] **Step 2: Verify valid JSON**

```bash
cat .claude-plugin/plugin.json | python3 -m json.tool >/dev/null && echo "Valid JSON"
```

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "update: rebrand plugin.json to Synchestra Superpowers"
```

---

### Task 13: Update `package.json` with Synchestra branding

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Replace the package.json content**

Replace the entire content of `package.json` with:

```json
{
  "name": "synchestra-superpowers",
  "version": "5.0.4",
  "type": "module",
  "main": ".opencode/plugins/superpowers.js"
}
```

- [ ] **Step 2: Verify valid JSON**

```bash
cat package.json | python3 -m json.tool >/dev/null && echo "Valid JSON"
```

- [ ] **Step 3: Commit**

```bash
git add package.json
git commit -m "update: rebrand package.json to synchestra-superpowers"
```

---

### Task 14: Rewrite `README.md` with Synchestra Superpowers positioning

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the entire README.md**

Replace the full content of `README.md` with:

```markdown
# Synchestra Superpowers

Synchestra Superpowers is Synchestra's workflow engine for AI coding agents. It provides a complete development workflow built on composable skills — from brainstorming through planning, implementation, review, and delivery — all backed by Synchestra's persistent, git-backed state.

**Synchestra is required.** This is not a fork of superpowers with optional extras. Synchestra Superpowers is designed around persistent state from the ground up: cross-session task continuity, multi-agent coordination, and deviation reports are first-class features, not bolted-on integrations.

## Three-Layer Stack

```
+----------------------------------+
|   Synchestra Superpowers         |  <-- workflow skills (this repo)
|   (brainstorming, TDD, plans,    |
|    subagent-driven-dev, etc.)    |
+----------------------------------+
|   Synchestra Core Skills         |  <-- task/feature skills (synchestra ai-plugin)
|   (task-new, claim-task,         |
|    feature-info, spec-search)    |
+----------------------------------+
|   Synchestra CLI                 |  <-- the binary
|   (synchestra project/task/      |
|    feature commands)             |
+----------------------------------+
```

| Component | Repo | Installs as |
|---|---|---|
| Synchestra CLI | `synchestra-io/synchestra` | Pre-built binary from GitHub releases |
| Core skills | `synchestra-io/synchestra` ai-plugin | Claude plugin / skill pack |
| Workflow skills | `synchestra-io/synchestra-superpowers` | Claude plugin |

Synchestra Superpowers does not bundle core skills. It declares them as a dependency. The onboarding skill detects and installs missing layers.

## Getting Started

### Install the plugin

**Claude Code Official Marketplace:**
```bash
/plugin install synchestra-superpowers@claude-plugins-official
```

**Claude Code (via Plugin Marketplace):**
```bash
/plugin marketplace add synchestra-io/synchestra-superpowers-marketplace
/plugin install synchestra-superpowers@synchestra-superpowers-marketplace
```

### Automatic onboarding

On first session, the `onboarding` skill runs automatically if any layer of the stack is missing:

1. **Synchestra CLI** — detects OS/arch, downloads pre-built binary from latest GitHub release (no Go toolchain needed)
2. **Core skills** — downloads ai-plugin from same release, installs as plugin
3. **Project setup** — presents three options:
   - **A) Single repo** — sensible defaults, zero config (state on orphan branch `synchestra-state`)
   - **B) Multiple repos** — guided topology configuration
   - **C) Dedicated state repo** — for continuity across environments and parallel workstreams

Each step asks for confirmation. If you decline a step, the skill explains what won't work and stops.

## The Workflow

1. **brainstorming** — Refines rough ideas through questions, explores alternatives, saves design document
2. **using-git-worktrees** — Creates isolated workspace on a new branch
3. **writing-plans** — Breaks work into bite-sized tasks (2-5 min each) with exact file paths, complete code, and verification steps. Creates Synchestra tasks for cross-session tracking.
4. **subagent-driven-development** or **executing-plans** — Dispatches fresh subagent per task with two-stage review (spec compliance, then code quality), or executes inline with checkpoints. Synchestra tracks task state persistently.
5. **test-driven-development** — Enforces RED-GREEN-REFACTOR cycle
6. **requesting-code-review** — Reviews against plan, reports issues by severity
7. **finishing-a-development-branch** — Verifies tests, presents options (merge/PR/keep/discard), cleans up

**Persistent state throughout:** Tasks survive across sessions. Multiple agents coordinate via atomic claiming. Deviation reports compare plan vs actual execution.

The agent checks for relevant skills before any task. Mandatory workflows, not suggestions.

## Skills Library

**Workflow**
- **brainstorming** — Socratic design refinement
- **writing-plans** — Detailed implementation plans with Synchestra task creation
- **executing-plans** — Batch execution with Synchestra state tracking
- **subagent-driven-development** — Fresh subagent per task with two-stage review
- **dispatching-parallel-agents** — Concurrent subagent workflows with atomic task claiming
- **finishing-a-development-branch** — Merge/PR decision workflow
- **using-git-worktrees** — Parallel development branches

**Quality**
- **test-driven-development** — RED-GREEN-REFACTOR cycle
- **systematic-debugging** — 4-phase root cause process
- **verification-before-completion** — Ensure it's actually fixed
- **requesting-code-review** — Pre-review checklist
- **receiving-code-review** — Responding to feedback

**Synchestra**
- **onboarding** — 3-layer stack setup (CLI, core skills, project)
- **task-board** — View current task state across all collaborators
- **deviation-report** — Compare planned vs actual execution

**Meta**
- **writing-skills** — Create new skills following best practices
- **using-superpowers** — Introduction to the skills system

## Philosophy

- **Test-Driven Development** — Write tests first, always
- **Systematic over ad-hoc** — Process over guessing
- **Complexity reduction** — Simplicity as primary goal
- **Evidence over claims** — Verify before declaring success
- **Persistent state** — Tasks survive sessions, agents coordinate, nothing gets lost

## Relationship to upstream superpowers

Synchestra Superpowers is forked from [obra/superpowers](https://github.com/obra/superpowers). Core skill logic (brainstorming, TDD, debugging, etc.) stays in sync with upstream where it makes sense. Synchestra integration is woven into the skills, not bolted on. The fork diverges from upstream patterns when Synchestra enables a better UX.

## Contributing

1. Fork the repository
2. Create a branch for your skill
3. Follow the `writing-skills` skill for creating and testing new skills
4. Submit a PR

See `skills/writing-skills/SKILL.md` for the complete guide.

## Updating

```bash
/plugin update synchestra-superpowers
```

## License

MIT License - see LICENSE file for details
```

- [ ] **Step 2: Verify the file reads correctly**

Read through the full README to confirm formatting, links, and section flow.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "rewrite: README.md with Synchestra Superpowers positioning and three-layer stack"
```

---

### Task 15: Update cross-references in `using-superpowers/references/gemini-tools.md`

**Files:**
- Modify: `skills/using-superpowers/references/gemini-tools.md`

- [ ] **Step 1: Check for fallback language about subagents**

The file at line 21 contains: "Skills that rely on subagent dispatch (`subagent-driven-development`, `dispatching-parallel-agents`) will fall back to single-session execution via `executing-plans`."

This is about Gemini CLI platform limitations (no Task tool), not Synchestra availability. **Leave this as-is** — it is a platform capability fallback, not a Synchestra fallback.

- [ ] **Step 2: No changes needed — skip this task**

This fallback is about Gemini CLI lacking a `Task` tool equivalent, which is unrelated to the Synchestra rebranding.

---

### Task 16: Verify no remaining references to `install-synchestra` or `synchestra-state` skill

**Files:**
- Verify across entire repo

- [ ] **Step 1: Search for stale references**

```bash
grep -r "install-synchestra" --include='*.md' --include='*.json' --include='*.sh' .
grep -r "synchestra-state" --include='*.md' --include='*.json' --include='*.sh' . | grep -v docs/superpowers/
grep -r "Synchestra Integration (Optional)" .
grep -r "If Synchestra is unavailable" .
grep -r "without Synchestra" .
grep -r "command -v synchestra" . | grep -v skills/onboarding/
```

Search is repo-wide (including `hooks/`, `commands/`, and any other top-level directories), not just `skills/`.

Expected: zero matches for all five searches.

- [ ] **Step 2: Fix any remaining references found**

If any matches are found, update them:
- `install-synchestra` references should become `onboarding`
- `synchestra-state` skill references should be removed
- "Optional" conditionals should become mandatory language
- "If Synchestra is unavailable" fallback text should be removed
- "without Synchestra" sections should be removed

- [ ] **Step 3: Commit any fixes**

```bash
git add -A
git commit -m "fix: remove remaining stale references to install-synchestra and synchestra-state"
```

(Skip this step if no matches were found in Step 1.)

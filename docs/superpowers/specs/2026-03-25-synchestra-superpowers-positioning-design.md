# Synchestra Superpowers — Naming, Positioning & Architecture

## Context

synchestra-superpowers originated as a fork of obra/superpowers. The original goal was to integrate Synchestra as the persistent state layer and eventually PR upstream. That upstream goal is no longer a constraint — the fork is now its own product.

The current implementation makes Synchestra optional: every skill gracefully degrades when the Synchestra CLI is absent. This creates two code paths, dilutes the brand identity, and prevents designing a holistic UX around persistent state.

## Decision

**Synchestra is mandatory.** synchestra-superpowers is Synchestra's workflow engine for AI agents. It requires the Synchestra CLI and core skills. There is no degraded mode.

### Why

- **UX superiority is the priority.** The experience with persistent state (cross-session continuity, deviation reports, multi-agent coordination) must be clearly better than vanilla superpowers. Maintaining fallback paths forces compromises in both modes.
- **Stickiness comes from accumulated state.** Every session deepens the task history. Switching away means losing that continuity.
- **Virality comes from shared state.** When a teammate sees the task board or an agent claims a task, they're already in the Synchestra ecosystem.
- **Target audience is superpowers users** who already use obra/superpowers and hit the "my tasks disappear between sessions" wall. Secondary audience is multi-agent power users who need a coordination layer.

### What it is NOT

- Not a fork of superpowers with optional extras
- Not superpowers with a Synchestra adapter
- Not usable without Synchestra — that's the point

### Relationship to upstream superpowers

- Core skill logic (brainstorming, TDD, debugging, etc.) stays in sync with upstream where it makes sense
- Synchestra integration is woven into the skills, not bolted on
- Free to deviate from upstream patterns when Synchestra enables a better UX
- The repo stays forked from obra/superpowers to keep upstream updates easy and regular

## Architecture

### Three-layer dependency stack

```
┌─────────────────────────────────┐
│   Synchestra Superpowers        │  ← workflow skills (this repo)
│   (brainstorming, TDD, plans,   │
│    subagent-driven-dev, etc.)   │
├─────────────────────────────────┤
│   Synchestra Core Skills        │  ← task/feature skills (synchestra ai-plugin)
│   (task-new, claim-task,        │
│    feature-info, spec-search)   │
├─────────────────────────────────┤
│   Synchestra CLI                │  ← the binary
│   (synchestra project/task/     │
│    feature commands)            │
└─────────────────────────────────┘
```

### What lives where

| Component | Repo | Installs as |
|---|---|---|
| Synchestra CLI | `synchestra-io/synchestra` | Pre-built binary from GitHub releases |
| Core skills (synchestra-*) | `synchestra-io/synchestra` ai-plugin | Claude plugin / skill pack |
| Workflow skills | `synchestra-io/synchestra-superpowers` | Claude plugin |

Synchestra Superpowers does NOT bundle core skills. It declares them as a dependency. The onboarding skill detects and installs missing layers.

## Onboarding Flow

The `onboarding` skill (replaces `install-synchestra`) runs automatically on first session if any layer is missing.

### Detection sequence

```
1. Synchestra CLI installed?
   NO  → detect OS/arch, download binary from latest GitHub release
         (darwin/arm64, darwin/amd64, linux/amd64, linux/arm64, windows/amd64)
         extract to ~/.local/bin or appropriate location
   YES ↓
2. Synchestra core skills available?
   NO  → download ai-plugin.zip from same release, install as plugin
   YES ↓
3. Project initialized?
   YES → use project config
   NO  → present project topology choice (see below)
         ↓
4. Ready — show task board summary
```

No Go toolchain required. Pre-built binaries are downloaded directly from GitHub releases.

### Project topology choice

When a project is not initialized, present three options:

> How is your project set up?
>
> A) Single repo, default settings are fine
> B) Multiple repos in this project
> C) Dedicated state repo (for continuity across environments and parallel workstreams)

**Option A:** No initialization needed. Synchestra uses sensible defaults:
- State stored on orphan branch `synchestra-state` in the current repo
- Spec root: `spec/`
- Docs root: `docs/`

The user is shown these defaults and informed they can reconfigure later.

**Options B and C:** Invoke the `synchestra-project-setup` skill (lives in `synchestra-io/synchestra` ai-plugin alongside other `synchestra-*` skills) for guided project topology configuration.

### Principles

- Each step is a clear action with a single command
- User confirms each step — no silent installs
- If the user declines any step, explain what won't work and stop — don't nag
- On subsequent sessions, detection is instant (all checks are local filesystem)
- Option A users can upgrade to B/C later as needs grow

## Prerequisite: Config-less Mode in Synchestra CLI

The current Synchestra CLI requires `synchestra-spec-repo.yaml` or `synchestra-state-repo.yaml` to exist before any command works. Without it, commands error with exit code 3. This blocks the "Option A: just use defaults" onboarding path.

### What already exists

- Embedded state infrastructure (orphan branch + worktree via `synchestra project init`)
- Worktree detection in `resolve.StateRepoPath()` (`worktree://` scheme)
- Default directory paths in the project-definition spec (`spec/`, `docs/`)
- Idempotent `synchestra project init`

### What needs to change (in synchestra-io/synchestra)

| Component | Change |
|---|---|
| `pkg/cli/resolve/resolve.go` | `StateRepoPath()` falls back to `.synchestra/` worktree with defaults when no config file found. Auto-creates the orphan branch and worktree on first use if the user consented during onboarding. |
| `spec/features/embedded-state/README.md` | Document config-less mode as a supported use case with defined default values |
| `spec/features/cli/command-environments.md` | Clarify that task commands work with default state store when no config is present |

The fix is localized: once `resolve.StateRepoPath()` can fall back to defaults, all task commands work without changes — they already operate on whatever state store the resolver provides.

## Codebase Changes

### Skills to modify

| Skill | Change |
|---|---|
| `using-superpowers` | Add onboarding detection as first step |
| `install-synchestra` → rename to `onboarding` | Full rewrite: 3-layer detection, binary download, core skills install, project topology choice |
| All workflow skills (writing-plans, executing-plans, subagent-driven-development, etc.) | Remove "if synchestra available" conditionals — Synchestra is always present |
| `task-board` | Remove "Synchestra not available" fallback messaging |
| `deviation-report` | Remove "without Synchestra" fallback section |

### Skills to delete

| Skill | Reason |
|---|---|
| `synchestra-state` | Its logic becomes the default behavior, not a separate opt-in skill |

### Skills to add (in synchestra-io/synchestra ai-plugin)

| Skill | Purpose |
|---|---|
| `synchestra-project-setup` | Guided project topology configuration. Invoked from onboarding options B/C, or anytime later. |

The `synchestra-project-setup` skill walks the user through:
- Paths to specifications directory
- Paths to source code directory(ies)
- Whether other repos are involved; if yes, local path and/or URL for each
- Dedicated state repo setup (if chosen)

Full specification for this skill will live as a feature draft in `synchestra-io/synchestra` at `spec/features/agent-skills/project-setup/README.md`.

### Other files to change

| File | Change |
|---|---|
| `README.md` | Full rewrite — new positioning, dependency stack, onboarding description |
| `.claude-plugin/plugin.json` | Update name, description, author to Synchestra branding |
| `package.json` | Update name and metadata |

### What stays the same

- Core skill logic (brainstorming, TDD, debugging, writing-plans, etc.) — content stays in sync with upstream where relevant
- Skill file structure and conventions
- Git worktree workflow

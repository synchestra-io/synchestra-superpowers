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

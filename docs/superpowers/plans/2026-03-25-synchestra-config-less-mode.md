# Config-less Mode & Project Setup Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable Synchestra CLI commands to work without any config file by falling back to sensible defaults (orphan branch + `.synchestra/` worktree), and add a `synchestra-project-setup` skill for guided multi-repo configuration.

**Architecture:** `resolve.StateRepoPath()` gains a final fallback: when no config file is found in any parent directory, it looks for a `.synchestra/` worktree in the git repo root. If the worktree exists, it returns that path. If it does not exist, it returns a new `NeedsInit` error type (exit code 3) that tells the caller to run `synchestra project init`. The `synchestra-project-setup` skill is a new skill in `ai-plugin/skills/` that guides users through multi-repo project topology configuration.

**Tech Stack:** Go (standard library + `gopkg.in/yaml.v3`), Markdown (skill files, spec docs)

**Spec:** `/Users/alexandertrakhimenok/projects/synchestra-io/synchestra-superpowers/docs/superpowers/specs/2026-03-25-synchestra-superpowers-positioning-design.md` (section "Prerequisite: Config-less Mode in Synchestra CLI")

**Target repo:** `synchestra-io/synchestra` at `/Users/alexandertrakhimenok/projects/synchestra-io/synchestra`

---

## File Map

- **Modify:** `pkg/cli/resolve/resolve.go:23-70` — add config-less fallback to `StateRepoPath()`
- **Create:** `pkg/cli/resolve/resolve_test.go` — tests for `StateRepoPath()` including config-less scenarios
- **Modify:** `spec/features/embedded-state/README.md:1-178` — document config-less mode as a supported use case
- **Modify:** `spec/features/cli/command-environments.md:183` — clarify task commands work with default state store
- **Create:** `ai-plugin/skills/synchestra-project-setup/README.md` — skill directory README
- **Create:** `ai-plugin/skills/synchestra-project-setup/SKILL.md` — skill instructions
- **Modify:** `ai-plugin/skills/README.md` — add `synchestra-project-setup` to the skills index
- **Create:** `spec/features/agent-skills/project-setup/README.md` — feature draft for the skill

---

### Task 1: Add tests for existing StateRepoPath() behavior

**Files:**
- Create: `pkg/cli/resolve/resolve_test.go`

- [ ] **Step 1: Create `resolve_test.go` with tests covering the three existing code paths**

These tests pin the current behavior before we modify anything. Each test sets up a temp directory structure and calls `StateRepoPath()`.

```go
package resolve

// Features implemented: embedded-state

import (
	"errors"
	"os"
	"path/filepath"
	"testing"

	"github.com/synchestra-io/synchestra/pkg/cli/exitcode"
)

func TestStateRepoPath_SpecRepoWithWorktree(t *testing.T) {
	// Setup: dir with synchestra-spec-repo.yaml pointing to worktree://synchestra-state
	// and a .synchestra/ directory present.
	dir := t.TempDir()
	specYAML := []byte("state_repo: \"worktree://synchestra-state\"\n")
	if err := os.WriteFile(filepath.Join(dir, "synchestra-spec-repo.yaml"), specYAML, 0644); err != nil {
		t.Fatal(err)
	}
	wtDir := filepath.Join(dir, ".synchestra")
	if err := os.MkdirAll(wtDir, 0755); err != nil {
		t.Fatal(err)
	}

	got, err := StateRepoPath(dir)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got != wtDir {
		t.Errorf("got %q, want %q", got, wtDir)
	}
}

func TestStateRepoPath_SpecRepoWorktreeMissing(t *testing.T) {
	// Setup: dir with synchestra-spec-repo.yaml pointing to worktree://synchestra-state
	// but NO .synchestra/ directory.
	dir := t.TempDir()
	specYAML := []byte("state_repo: \"worktree://synchestra-state\"\n")
	if err := os.WriteFile(filepath.Join(dir, "synchestra-spec-repo.yaml"), specYAML, 0644); err != nil {
		t.Fatal(err)
	}

	_, err := StateRepoPath(dir)
	if err == nil {
		t.Fatal("expected error, got nil")
	}
	var ee *exitcode.Error
	if !errors.As(err, &ee) {
		t.Fatalf("expected exitcode.Error, got %T: %v", err, err)
	}
	if ee.ExitCode() != exitcode.NotFound {
		t.Errorf("exit code = %d, want %d", ee.ExitCode(), exitcode.NotFound)
	}
}

func TestStateRepoPath_StateRepoYAML(t *testing.T) {
	// Setup: dir with synchestra-state-repo.yaml (direct detection).
	dir := t.TempDir()
	if err := os.WriteFile(filepath.Join(dir, "synchestra-state-repo.yaml"), []byte("title: test\n"), 0644); err != nil {
		t.Fatal(err)
	}

	got, err := StateRepoPath(dir)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got != dir {
		t.Errorf("got %q, want %q", got, dir)
	}
}

func TestStateRepoPath_NoConfig(t *testing.T) {
	// Setup: empty dir with no config files. Should return NotFound.
	dir := t.TempDir()

	_, err := StateRepoPath(dir)
	if err == nil {
		t.Fatal("expected error, got nil")
	}
	var ee *exitcode.Error
	if !errors.As(err, &ee) {
		t.Fatalf("expected exitcode.Error, got %T: %v", err, err)
	}
	if ee.ExitCode() != exitcode.NotFound {
		t.Errorf("exit code = %d, want %d", ee.ExitCode(), exitcode.NotFound)
	}
}
```

- [ ] **Step 2: Run the tests and verify they all pass**

```bash
cd /Users/alexandertrakhimenok/projects/synchestra-io/synchestra
go test ./pkg/cli/resolve/ -v -run TestStateRepoPath
```

Expected: all 4 tests pass (pinning current behavior).

- [ ] **Step 3: Commit**

```bash
git add pkg/cli/resolve/resolve_test.go
git commit -m "Add tests for existing StateRepoPath() behavior"
```

---

### Task 2: Add config-less fallback to StateRepoPath()

**Files:**
- Modify: `pkg/cli/resolve/resolve.go:23-70`
- Test: `pkg/cli/resolve/resolve_test.go`

- [ ] **Step 1: Add failing tests for the config-less fallback**

Append to `resolve_test.go`:

```go
func TestStateRepoPath_ConfigLess_WorktreeExists(t *testing.T) {
	// Setup: a git repo with no config files, but .synchestra/ directory exists
	// at the repo root. Should return the .synchestra/ path.
	dir := t.TempDir()

	// Initialize a git repo so we can find the root.
	cmd := exec.Command("git", "init", dir)
	if out, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("git init: %v\n%s", err, out)
	}

	wtDir := filepath.Join(dir, ".synchestra")
	if err := os.MkdirAll(wtDir, 0755); err != nil {
		t.Fatal(err)
	}

	// Call from a subdirectory to test walking up.
	subDir := filepath.Join(dir, "src", "pkg")
	if err := os.MkdirAll(subDir, 0755); err != nil {
		t.Fatal(err)
	}

	got, err := StateRepoPath(subDir)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got != wtDir {
		t.Errorf("got %q, want %q", got, wtDir)
	}
}

func TestStateRepoPath_ConfigLess_NoWorktree(t *testing.T) {
	// Setup: a git repo with no config files and no .synchestra/ directory.
	// Should return NotFound with a message suggesting 'synchestra project init'.
	dir := t.TempDir()

	cmd := exec.Command("git", "init", dir)
	if out, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("git init: %v\n%s", err, out)
	}

	_, err := StateRepoPath(dir)
	if err == nil {
		t.Fatal("expected error, got nil")
	}
	var ee *exitcode.Error
	if !errors.As(err, &ee) {
		t.Fatalf("expected exitcode.Error, got %T: %v", err, err)
	}
	if ee.ExitCode() != exitcode.NotFound {
		t.Errorf("exit code = %d, want %d", ee.ExitCode(), exitcode.NotFound)
	}
	if !strings.Contains(err.Error(), "synchestra project init") {
		t.Errorf("error message should suggest 'synchestra project init', got: %s", err.Error())
	}
}
```

Add the missing imports at the top of the test file (`"os/exec"`, `"strings"`).

- [ ] **Step 2: Run the tests and verify the new tests fail**

```bash
cd /Users/alexandertrakhimenok/projects/synchestra-io/synchestra
go test ./pkg/cli/resolve/ -v -run TestStateRepoPath_ConfigLess
```

Expected: `TestStateRepoPath_ConfigLess_WorktreeExists` fails (currently returns NotFound error instead of the worktree path). `TestStateRepoPath_ConfigLess_NoWorktree` may pass or fail depending on the current error message.

- [ ] **Step 3: Implement the config-less fallback in `resolve.go`**

Replace the final `return` statement at the end of the `for` loop (the "reached filesystem root" case) with a git-root fallback. The modified `StateRepoPath()` function:

```go
// StateRepoPath finds the state repo path for the current project.
// It walks up from startDir looking for:
//   - synchestra-spec-repo.yaml (reads state_repo field; worktree:// for embedded state)
//   - synchestra-state-repo.yaml (direct detection)
//
// If no config file is found, it falls back to config-less mode:
//   - Finds the git repo root
//   - Checks for .synchestra/ worktree directory
//   - Returns the worktree path if it exists
//   - Returns NotFound with setup instructions if it doesn't
func StateRepoPath(startDir string) (string, error) {
	current, err := filepath.Abs(startDir)
	if err != nil {
		return "", fmt.Errorf("resolving path: %w", err)
	}

	for {
		// Check for spec repo config (spec repo -> state repo via state_repo field)
		specPath := filepath.Join(current, "synchestra-spec-repo.yaml")
		if _, err := os.Stat(specPath); err == nil {
			data, err := os.ReadFile(specPath)
			if err != nil {
				return "", fmt.Errorf("reading %s: %w", specPath, err)
			}
			var cfg specRepoConfig
			if err := yaml.Unmarshal(data, &cfg); err != nil {
				return "", fmt.Errorf("parsing %s: %w", specPath, err)
			}
			if cfg.StateRepo == "" {
				return "", exitcode.NotFoundErrorf("no state_repo field in %s", specPath)
			}

			// Check for worktree:// scheme (embedded state).
			if strings.HasPrefix(cfg.StateRepo, "worktree://") {
				worktreePath := filepath.Join(current, ".synchestra")
				if info, statErr := os.Stat(worktreePath); statErr == nil && info.IsDir() {
					return worktreePath, nil
				}
				return "", exitcode.NotFoundErrorf("embedded state configured in %s but .synchestra/ worktree is missing; run 'synchestra project init' to set up", specPath)
			}

			// TODO: Resolve state_repo URL to local path using repos_dir convention
			return cfg.StateRepo, nil
		}

		// Check for state repo config (direct detection)
		statePath := filepath.Join(current, "synchestra-state-repo.yaml")
		if _, err := os.Stat(statePath); err == nil {
			return current, nil
		}

		parent := filepath.Dir(current)
		if parent == current {
			// No config found anywhere. Fall back to config-less mode.
			return configLessFallback(startDir)
		}
		current = parent
	}
}

// configLessFallback attempts to find a .synchestra/ worktree at the git repo
// root when no config file exists. This enables zero-config usage after
// 'synchestra project init'.
func configLessFallback(startDir string) (string, error) {
	repoRoot, err := findGitRoot(startDir)
	if err != nil {
		return "", exitcode.NotFoundError("project not found: no synchestra-spec-repo.yaml or synchestra-state-repo.yaml in any parent directory")
	}

	worktreePath := filepath.Join(repoRoot, ".synchestra")
	if info, statErr := os.Stat(worktreePath); statErr == nil && info.IsDir() {
		return worktreePath, nil
	}

	return "", exitcode.NotFoundErrorf("no Synchestra project found; run 'synchestra project init' to set up embedded state in this repository")
}

// findGitRoot walks up from dir looking for a .git directory or file (worktree).
func findGitRoot(dir string) (string, error) {
	current, err := filepath.Abs(dir)
	if err != nil {
		return "", err
	}
	for {
		gitPath := filepath.Join(current, ".git")
		if _, err := os.Stat(gitPath); err == nil {
			return current, nil
		}
		parent := filepath.Dir(current)
		if parent == current {
			return "", fmt.Errorf("not a git repository")
		}
		current = parent
	}
}
```

Add `"os/exec"` is NOT needed in the production code — `findGitRoot` uses `os.Stat` to avoid a dependency on `git` binary in the resolver hot path.

- [ ] **Step 4: Run the full validation suite**

```bash
cd /Users/alexandertrakhimenok/projects/synchestra-io/synchestra
gofmt -w pkg/cli/resolve/resolve.go pkg/cli/resolve/resolve_test.go
go vet ./pkg/cli/resolve/
go test ./pkg/cli/resolve/ -v
golangci-lint run ./pkg/cli/resolve/
go build ./...
```

Expected: all tests pass, no lint errors, build succeeds.

- [ ] **Step 5: Commit**

```bash
git add pkg/cli/resolve/resolve.go pkg/cli/resolve/resolve_test.go
git commit -m "Add config-less fallback to StateRepoPath()

When no synchestra-spec-repo.yaml or synchestra-state-repo.yaml is found,
fall back to checking for .synchestra/ worktree at the git repo root.
This enables zero-config usage after 'synchestra project init'."
```

---

### Task 3: Update embedded-state spec to document config-less mode

**Files:**
- Modify: `spec/features/embedded-state/README.md`

- [ ] **Step 1: Add a "Config-less Mode" section after the "Configuration" section (after line 97)**

Insert the following section between "### Configuration" and "### State Store Backend":

```markdown
### Config-less Mode

Config-less mode allows Synchestra task commands to work without any configuration file (`synchestra-spec-repo.yaml` or `synchestra-state-repo.yaml`) on the main branch. This is the zero-friction path for "Option A" onboarding — the user runs `synchestra project init` and immediately starts using task commands.

**How it works:**

1. `resolve.StateRepoPath()` walks up the directory tree looking for config files (existing behavior)
2. If no config file is found, it falls back to finding the git repo root
3. If `.synchestra/` exists at the repo root, it is returned as the state store path
4. If `.synchestra/` does not exist, a `NotFound` error suggests running `synchestra project init`

**Defaults in config-less mode:**

| Setting | Default value |
|---|---|
| State branch | `synchestra-state` |
| Worktree path | `.synchestra/` |
| Spec root | `spec/` |
| Docs root | `docs/` |

**When config-less mode is used:**

- After `synchestra project init` (which creates the orphan branch and worktree)
- When onboarding selects "Option A: single repo, default settings"
- The `synchestra-spec-repo.yaml` file is still created by `project init` for explicit configuration, but task commands no longer require it to exist

**Upgrading from config-less to configured:**

Running `synchestra project init` always writes `synchestra-spec-repo.yaml` with `state_repo: worktree://synchestra-state`. If a user later needs custom configuration (multi-repo, dedicated state repo), they can edit or replace this file. The config-less fallback only activates when no config file exists at all.
```

- [ ] **Step 2: Commit**

```bash
git add spec/features/embedded-state/README.md
git commit -m "Document config-less mode in embedded-state spec"
```

---

### Task 4: Update command-environments spec

**Files:**
- Modify: `spec/features/cli/command-environments.md`

- [ ] **Step 1: Add a note to the Coordination section (after line 137)**

After the "Environment requirements" block in section "3. Coordination", add:

```markdown
**Config-less operation:** Coordination commands work with the default embedded state store when no `synchestra-spec-repo.yaml` or `synchestra-state-repo.yaml` is present, provided that `synchestra project init` has been run (creating the `.synchestra/` worktree). See [embedded-state: Config-less Mode](../embedded-state/README.md#config-less-mode).
```

- [ ] **Step 2: Update the `project info` row in the Command Environment Matrix (line 183)**

Change the "Requires" column for `project info` from:

```
`synchestra-spec-repo.yaml`
```

to:

```
`synchestra-spec-repo.yaml` or `.synchestra/` worktree
```

- [ ] **Step 3: Commit**

```bash
git add spec/features/cli/command-environments.md
git commit -m "Clarify config-less operation in command-environments spec"
```

---

### Task 5: Create the project-setup feature spec draft

**Files:**
- Create: `spec/features/agent-skills/project-setup/README.md`

- [ ] **Step 1: Create the directory and feature spec**

```bash
mkdir -p /Users/alexandertrakhimenok/projects/synchestra-io/synchestra/spec/features/agent-skills/project-setup
```

Write the file:

```markdown
# Feature: Project Setup Skill

**Status:** Draft

## Summary

The `synchestra-project-setup` skill provides guided project topology configuration for Synchestra. It walks users through multi-repo project setup — specifying paths to specifications, source code directories, additional repositories, and dedicated state repo configuration.

This skill is invoked from the onboarding flow (options B and C) or anytime a user needs to reconfigure their project topology.

## Problem

Synchestra supports three project topologies: single-repo with embedded state, multi-repo with embedded state, and multi-repo with a dedicated state repo. The `synchestra project init` command handles the first case, and `synchestra project new` handles creating a full dedicated setup from scratch. But there is no guided path for:

1. **Multi-repo discovery** — a user has 2-3 repos and needs help configuring `synchestra-spec-repo.yaml` with the correct `repos` list
2. **Topology upgrade** — a user started with embedded state and now needs to add more repos or switch to a dedicated state repo
3. **Agent-driven setup** — an AI agent needs structured instructions for configuring project topology on behalf of the user

## Design

### Skill behavior

The skill is conversational — it asks questions and uses the answers to invoke the appropriate CLI commands:

1. **Spec directory** — "Where are your specifications?" (default: `spec/`)
2. **Source directories** — "Where is your source code?" (default: current repo root)
3. **Additional repos** — "Are there other repos involved in this project?" If yes, collect local path and/or URL for each
4. **State storage** — "Where should coordination state live?"
   - Same repo (embedded) — uses `synchestra project init`
   - Dedicated repo — uses `synchestra project new` with collected paths

### CLI commands used

| Step | Command |
|---|---|
| Single-repo embedded setup | `synchestra project init` |
| Multi-repo setup | `synchestra project new --title <title>` followed by `synchestra project code add` for each code repo |
| Add code repo | `synchestra project code add --url <url>` or `synchestra project code add --path <path>` |

### Trigger conditions

- Onboarding detects no existing project and user selects option B or C
- User explicitly asks to set up or reconfigure their project topology
- User asks about adding repos to an existing Synchestra project

## Dependencies

- [cli/project/init](../../cli/project/init/README.md) — for embedded state setup
- [cli/project/new](../../cli/project/new/README.md) — for dedicated state repo setup
- [cli/project/code/add](../../cli/project/code/add/README.md) — for adding code repos
- [embedded-state](../../embedded-state/README.md) — config-less mode and worktree setup

## Outstanding Questions

None at this time.
```

- [ ] **Step 2: Commit**

```bash
git add spec/features/agent-skills/project-setup/README.md
git commit -m "Add project-setup feature spec draft"
```

---

### Task 6: Create the synchestra-project-setup skill

**Files:**
- Create: `ai-plugin/skills/synchestra-project-setup/README.md`
- Create: `ai-plugin/skills/synchestra-project-setup/SKILL.md`
- Modify: `ai-plugin/skills/README.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p /Users/alexandertrakhimenok/projects/synchestra-io/synchestra/ai-plugin/skills/synchestra-project-setup
```

- [ ] **Step 2: Create `README.md`**

```markdown
# synchestra-project-setup

Guided project topology configuration for Synchestra. Use when setting up a multi-repo project, upgrading from single-repo to multi-repo, or configuring a dedicated state repo.

See [SKILL.md](SKILL.md) for full instructions, parameters, exit codes, and examples.

## Outstanding Questions

None at this time.
```

- [ ] **Step 3: Create `SKILL.md`**

```markdown
---
name: synchestra-project-setup
description: Guided project topology configuration for Synchestra. Use when setting up a multi-repo project or configuring a dedicated state repo.
---

# Skill: synchestra-project-setup

Walk the user through Synchestra project topology configuration. This skill is conversational — ask questions, then invoke the appropriate CLI commands based on answers.

**Feature reference:** [project-setup](../../spec/features/agent-skills/project-setup/README.md)

## When to use

- Onboarding detected no existing project and the user selected multi-repo (option B) or dedicated state repo (option C)
- A user asks to set up a new Synchestra project with multiple repositories
- A user wants to add repositories to an existing project
- A user wants to switch from embedded state to a dedicated state repo

## Workflow

### Step 1: Gather project information

Ask the user:

1. **Project title** — "What is the name of this project?"
2. **Spec directory** — "Where are your specifications stored?" (suggest `spec/` as default)
3. **Source directories** — "Where is your source code?" (suggest current repo as default)
4. **Additional repos** — "Are there other repos involved?" If yes, collect local path and/or remote URL for each
5. **State storage preference** — "Should coordination state live in this repo (embedded) or in a dedicated repo?"

### Step 2: Execute setup based on answers

**If single-repo with embedded state (option A):**

```bash
synchestra project init --title "<title>"
```

**If multi-repo with embedded state (option B):**

```bash
synchestra project init --title "<title>"
# Then for each additional code repo:
synchestra project code add --url "<repo-url>"
```

**If dedicated state repo (option C):**

```bash
synchestra project new --title "<title>"
# Then for each code repo:
synchestra project code add --url "<repo-url>"
```

### Step 3: Verify setup

```bash
synchestra project info
```

Confirm the output shows the expected project topology.

## Exit codes

These are the exit codes from the underlying CLI commands:

| Exit code | Meaning | What to do |
|---|---|---|
| `0` | Command succeeded | Continue to next step |
| `1` | Conflict (e.g., project already exists) | Show the conflict and ask the user how to proceed |
| `2` | Invalid arguments | Check parameter values and retry |
| `3` | Not found (e.g., repo path doesn't exist) | Ask the user to verify the path |
| `10+` | Unexpected error | Log the error and escalate |

## Examples

### Multi-repo setup

```
User: I have three repos — this one for the backend, one for the frontend, and one for shared libs.

Agent: I'll set up Synchestra for your project. Let me configure embedded state in this repo and register the other repos.

> synchestra project init --title "My Project"
> synchestra project code add --url "https://github.com/org/frontend"
> synchestra project code add --url "https://github.com/org/shared-libs"
> synchestra project info

Your project is configured with 3 repos. Task state is stored on the synchestra-state branch in this repo.
```

### Upgrade from single-repo

```
User: I started with just this repo but now I have a second one. How do I add it?

Agent: I'll add the new repo to your Synchestra project.

> synchestra project code add --url "https://github.com/org/new-service"

Done. Your project now tracks 2 repos.
```

## Notes

- This skill is conversational — it requires user input at each decision point. Do not assume defaults without asking.
- If the user is unsure about which topology to choose, recommend embedded state (option A or B) as the simpler starting point. They can upgrade to a dedicated state repo later.
- After setup, suggest the user run `synchestra-task-new` to create their first task.
```

- [ ] **Step 4: Add the skill to the skills index in `ai-plugin/skills/README.md`**

Find the skills table in the README and add a row for `synchestra-project-setup` in alphabetical order (after `synchestra-project-init`):

```markdown
| [`synchestra-project-setup`](synchestra-project-setup/) | Guided project topology configuration — multi-repo setup, dedicated state repo |
```

- [ ] **Step 5: Commit**

```bash
git add ai-plugin/skills/synchestra-project-setup/README.md ai-plugin/skills/synchestra-project-setup/SKILL.md ai-plugin/skills/README.md
git commit -m "Add synchestra-project-setup skill for guided project topology configuration"
```

---

### Task 7: Run full validation

**Files:**
- None (validation only)

- [ ] **Step 1: Run Go validation suite**

```bash
cd /Users/alexandertrakhimenok/projects/synchestra-io/synchestra
gofmt -w pkg/cli/resolve/resolve.go pkg/cli/resolve/resolve_test.go
go vet ./...
go test ./...
golangci-lint run ./...
go build ./...
```

Expected: all pass with zero errors.

- [ ] **Step 2: Verify all new files have correct structure**

Check that:
- `spec/features/agent-skills/project-setup/README.md` has an "Outstanding Questions" section
- `ai-plugin/skills/synchestra-project-setup/README.md` has an "Outstanding Questions" section
- `ai-plugin/skills/synchestra-project-setup/SKILL.md` has YAML frontmatter with `name` and `description`
- `pkg/cli/resolve/resolve.go` has the feature reference comment after the package declaration

- [ ] **Step 3: Final commit if any formatting was needed**

```bash
git add -A
git diff --cached --quiet || git commit -m "Apply formatting fixes from validation"
```

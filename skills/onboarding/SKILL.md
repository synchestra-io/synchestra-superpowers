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
> "Add ~/.local/bin to your PATH to use synchestra globally: `export PATH="$HOME/.local/bin:$PATH"`"

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

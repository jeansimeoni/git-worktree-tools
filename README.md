# Git Worktrees with Auto-Symlinks

Opinionated CLI for handling my workflows with Git worktrees.

Git worktrees let you check out multiple branches simultaneously in separate
directories, all sharing the same `.git` history. This is useful for reviewing
PRs, running tests on another branch, or working on a hotfix without stashing
your current work.

The problem: untracked files like `.env`, `.claude/`, or `node_modules/` only
exist in the main worktree. New worktrees start clean. The helpers in this repo
solve that by automatically symlinking those files.

## Installation

Clone this repo to your XDG data directory (recommended):

```bash
git clone https://github.com/jeansimeoni/git-worktree-tools \
    "${XDG_DATA_HOME:-$HOME/.local/share}/git-worktree-tools"
```

Or anywhere else you prefer.

Add `bin/` to your PATH in your shell config (bash or zsh):

```bash
# ~/.zshrc / ~/.bashrc
export PATH="${PATH}:${XDG_DATA_HOME:-${HOME}/.local/share}/git-worktree-tools/bin"
```

If you use [Rotz](https://github.com/volllly/rotz) as your dotfile manager, add
a `dot.yaml` entry that clones and self-updates the repo:

```yaml
darwin:
  installs:
    cmd: |
      GWT_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/git-worktree-tools"
      if [[ -d "$GWT_DIR/.git" ]]; then
        git -C "$GWT_DIR" pull --ff-only
      else
        git clone https://github.com/jeansimeoni/git-worktree-tools "$GWT_DIR"
      fi
linux:
  installs:
    cmd: |
      GWT_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/git-worktree-tools"
      if [[ -d "$GWT_DIR/.git" ]]; then
        git -C "$GWT_DIR" pull --ff-only
      else
        git clone https://github.com/jeansimeoni/git-worktree-tools "$GWT_DIR"
      fi
```

## Quick Reference

```bash
# Create a worktree (standard git)
git worktree add ../my-feature feature-branch

# Create from current branch (new branch)
git worktree add ../hotfix -b hotfix/fix-login

# List worktrees
git worktree list

# Set up auto-symlinks for a project (run once)
gwtree setup

# Set up and add created files to .git/info/exclude (hide from git status)
gwtree setup --do-not-track

# Manually trigger symlinks (from inside a worktree)
gwtree link

# Remove a worktree (with Claude conversation migration)
gwtree remove ../my-feature

# Remove with --force (passes through to git)
gwtree remove ../my-feature --force

# Remove worktree hook configuration from a project
gwtree clear-config

# Pull the latest version from GitHub
gwtree upgrade
```

## Setting Up a Project

### 1. Create `.worktree-links`

List the files and directories to symlink, one per line:

```
# .worktree-links
.env
.env.local
```

- Paths are relative to the repo root
- Lines starting with `#` are comments
- Missing source files are silently skipped

### 2. Run the setup

```bash
gwtree setup
```

This detects your project type and configures accordingly:

| Project type | What it does                                                                                           |
| ------------ | ------------------------------------------------------------------------------------------------------ |
| **Standard** | Copies hook to `.githooks/worktree-links.sh`, creates `.githooks/post-checkout`, sets `core.hooksPath` |
| **Husky**    | Places script in `.githooks/worktree-links.sh`, adds call to `.husky/post-checkout`                    |

In both cases, a guarded block is appended to the relevant `post-checkout` hook
(between `# git-worktree-tools:start` and `# git-worktree-tools:end` markers).
This block is safe on machines without the full setup installed — it silently
does nothing if `worktree-links.sh` is absent.

By default, created files appear as untracked in `git status` so you can commit
them if needed. Pass `--do-not-track` to add them to `.git/info/exclude` instead
— useful when you do not plan to commit the hook files.

### 3. Commit the hook files (optional)

If the whole team should benefit from the setup, commit the hook files. Run
setup without `--do-not-track` so the files appear as untracked and can be
staged:

```bash
gwtree setup

# Standard project
git add .githooks/ .worktree-links
git commit -m "Add worktree auto-symlink support"

# Husky project
git add .githooks/worktree-links.sh .husky/post-checkout .worktree-links
git commit -m "Add worktree auto-symlink support"
```

### 4. Team members

After cloning, team members run one command:

```bash
gwtree setup
```

For non-Husky projects this sets `core.hooksPath`. For Husky projects this is
only needed if `.husky/post-checkout` wasn't committed (the setup is
idempotent).

You can also add it to `package.json` alongside Husky's prepare script:

```json
{
  "scripts": {
    "prepare": "husky && gwtree setup 2>/dev/null || true"
  }
}
```

Or in a `Makefile`:

```makefile
setup:
	gwtree setup
```

## Removing the Configuration

To fully reverse `gwtree setup` on a project:

```bash
gwtree clear-config
```

This strips the guarded block from any hook that was modified, removes
`.githooks/worktree-links.sh`, removes the `.git/info/exclude` entries, and
cleans up `.githooks/` and `core.hooksPath` if the directory is now empty.

## Claude Code Conversation Migration

When you delete a worktree, Claude Code conversations tied to it become
inaccessible (they live under `~/.claude*/projects/<encoded-path>/`). The
`gwtree remove` command migrates them to the main worktree's project directory
before deletion so they remain visible in Claude Code's history.

```bash
# Instead of: git worktree remove ../my-feature
gwtree remove ../my-feature

# Works with --force too
gwtree remove ../my-feature --force
```

**How it works:**

- Scans all `~/.claude*` directories (e.g. `~/.claude`, `~/.claude-personal`)
- Looks for `projects/<encoded-worktree-path>/*.jsonl` in each
- Copies any found conversations to `projects/<encoded-main-path>/` in the
  **same** config dir — so work and personal history never cross-contaminate
- Then runs `git worktree remove` as normal

No environment variables or dotfile setup required — it infers everything from
the conversation files that actually exist.

> **Lazygit note**: Lazygit's built-in worktree removal calls
> `git worktree remove` directly and bypasses this command. Use `gwtree remove`
> from the terminal when you want conversation migration.

## Lazygit

Lazygit has built-in worktree support. Press `w` to open the worktree menu. When
you create a worktree through Lazygit, it calls `git worktree add` under the
hood, so the `post-checkout` hook fires and symlinks are created automatically.

To remove a worktree with Claude conversation migration, you need to add a
custom command that overrides Lazygit's built-in `D` binding on the worktrees
panel. Add the following to your Lazygit config
(`~/.config/lazygit/config.yml`):

```yaml
customCommands:
  - key: "D"
    context: "worktrees"
    description: "Remove worktree (migrates Claude conversations)"
    command: 'gwtree remove "{{.SelectedWorktree.Path}}"'
    subprocess: true
    prompts:
      - type: confirm
        title: "Remove worktree"
        body: "Remove '{{.SelectedWorktree.Path}}'?"
```

This overrides Lazygit's built-in deletion with `gwtree remove`, so
conversations are migrated before the worktree is deleted. The
`subprocess: true` flag keeps the terminal visible so you can see the migration
output.

If you also want a force-remove binding, add a second entry with a different
key:

```yaml
- key: "<c-d>"
  context: "worktrees"
  description: "Force-remove worktree (migrates Claude conversations)"
  command: 'gwtree remove --force "{{.SelectedWorktree.Path}}"'
  subprocess: true
  prompts:
    - type: confirm
      title: "Force-remove worktree"
      body: "Force-remove '{{.SelectedWorktree.Path}}'?"
```

## Commands

### `gwtree setup [--do-not-track]`

Run once per project to configure the auto-symlink hook. Detects Husky
automatically and integrates without conflicting with existing hooks. Appends a
guarded block to the relevant `post-checkout` rather than overwriting it.

By default, created files are left as untracked so they can be committed. Pass
`--do-not-track` to add them to `.git/info/exclude` and hide them from
`git status` when you do not plan to commit the hook files.

### `gwtree clear-config`

Reverses `gwtree setup`. Strips the guarded block from any hook that was
modified, removes `.githooks/worktree-links.sh`, removes the `.git/info/exclude`
entries, and cleans up `.githooks/` and `core.hooksPath` if the directory is now
empty.

### `gwtree remove <path> [--force]`

Drop-in replacement for `git worktree remove`. Before removing the worktree,
migrates any Claude Code conversation files (`*.jsonl`) from the worktree's
Claude project directory to the main worktree's project directory. Handles
multiple Claude accounts automatically by scanning all `~/.claude*` dirs. Also
cleans up `.git/info/exclude` entries when the last worktree is removed.

### `gwtree link`

Manually trigger symlink creation from inside a worktree. Useful if:

- You added new entries to `.worktree-links`
- The hook didn't fire for some reason
- You created the worktree before setting up the hook

### `gwtree upgrade`

Pulls the latest version of git-worktree-tools from GitHub using
`git pull --ff-only`. Run this to get new features and bug fixes.

## How It Works

The `post-checkout` hook runs whenever `git worktree add` creates a new worktree
(it triggers a branch checkout internally). The hook:

1. Checks if it's running inside a worktree (not the main working tree)
2. Reads `.worktree-links` from the main worktree
3. Creates symlinks for each listed path that exists in the main worktree
4. Skips entries where the source is missing or the target already exists
5. Logs everything to `.git/worktrees/<name>/worktree-links.log`

The hook is fully idempotent. Running it multiple times has no side effect.

## Debugging

Check the log file for a specific worktree:

```bash
cat .git/worktrees/<worktree-name>/worktree-links.log
```

Verify `core.hooksPath` is set correctly:

```bash
git config core.hooksPath
# Should output: .githooks (standard) or .husky (Husky projects)
```

## Common Workflows

### Review a PR without leaving your branch

```bash
git fetch origin
git worktree add ../review-pr-42 origin/feature/new-login
cd ../review-pr-42
# .env and other files are already symlinked
npm test
# When done (migrates Claude conversations before removal):
cd -
gwtree remove ../review-pr-42
```

### Work on a hotfix while keeping current work

```bash
git worktree add ../hotfix -b hotfix/urgent-fix main
cd ../hotfix
# Fix the issue, commit, push
cd -
gwtree remove ../hotfix
```

### Run two branches side by side

```bash
git worktree add ../branch-a feature-a
git worktree add ../branch-b feature-b
# Both have symlinked .env, node_modules, etc.
```

### Work on multiple features with isolated Claude sessions

Each worktree has a unique path, so Claude Code automatically tracks a separate
conversation history for each one (stored under
`~/.claude/projects/<encoded-path>/`). This means you can have Claude helping
with a feature in one terminal and a bug fix in another with zero context bleed.

To share project-level Claude config (CLAUDE.md, custom commands, settings)
across all worktrees, add `.claude/` to your `.worktree-links`:

```
# .worktree-links
.env
.claude/
```

Each worktree then symlinks `.claude/` from the main worktree, so they all share
the same instructions and slash commands — but their conversation histories
remain completely separate.

```bash
git worktree add ../feature-a feature-a
git worktree add ../feature-b feature-b

# Terminal 1 — Claude works on feature-a with its own context
cd ../feature-a && claude

# Terminal 2 — Claude works on feature-b with its own context
cd ../feature-b && claude
```

## Acknowledgements

- [Lazygit](https://github.com/jesseduffield/lazygit) — a fantastic terminal UI
  for git that makes worktree management a pleasure.
- [Rotz](https://github.com/volllly/rotz) — a cross-platform dotfile manager
  which is my tool of choice for managing my configuration across Linux and
  MacOS.

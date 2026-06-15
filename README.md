# Git Worktrees with Auto-Symlinks

Opinionated scripts for handling my workflows with Git worktrees.

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

Or anywhere else you prefer — just point `GIT_WORKTREE_TOOLS_DIR` at it.

Source the functions in your shell config (bash or zsh):

```bash
# ~/.zshrc / ~/.bashrc
export GIT_WORKTREE_TOOLS_DIR="${XDG_DATA_HOME:-${HOME}/.local/share}/git-worktree-tools"
source "${GIT_WORKTREE_TOOLS_DIR}/functions/git-worktree.sh"
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

# Remove a worktree (with Claude conversation migration)
git-worktree-remove ../my-feature

# Remove with --force (passes through to git)
git-worktree-remove ../my-feature --force

# Set up auto-symlinks for a project (run once)
git-worktree-setup

# Set up and add created files to .git/info/exclude (hide from git status)
git-worktree-setup --do-not-track

# Manually trigger symlinks (from inside a worktree)
git-worktree-link

# Remove worktree hook configuration from a project
git-worktree-clear-config
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
git-worktree-setup
```

This detects your project type and configures accordingly:

| Project type | What it does |
| ------------ | ------------ |
| **Standard** | Copies hook to `.githooks/worktree-links.sh`, creates `.githooks/post-checkout`, sets `core.hooksPath` |
| **Husky**    | Places script in `.githooks/worktree-links.sh`, adds call to `.husky/post-checkout` |

In both cases, a guarded block is appended to the relevant `post-checkout` hook
(between `# git-worktree-tools:start` and `# git-worktree-tools:end` markers).
This block is safe on machines without the full setup installed — it silently
does nothing if `worktree-links.sh` is absent.

By default, created files appear as untracked in `git status` so you can
commit them if needed. Pass `--do-not-track` to add them to `.git/info/exclude`
instead — useful when you do not plan to commit the hook files.

### 3. Commit the hook files (optional)

If the whole team should benefit from the setup, commit the hook files. Run
setup without `--do-not-track` so the files appear as untracked and can be
staged:

```bash
git-worktree-setup

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
git-worktree-setup
```

For non-Husky projects this sets `core.hooksPath`. For Husky projects this is
only needed if `.husky/post-checkout` wasn't committed (the setup is
idempotent).

You can also add it to `package.json` alongside Husky's prepare script:

```json
{
  "scripts": {
    "prepare": "husky && git-worktree-setup 2>/dev/null || true"
  }
}
```

Or in a `Makefile`:

```makefile
setup:
	git-worktree-setup
```

## Removing the Configuration

To fully reverse `git-worktree-setup` on a project:

```bash
git-worktree-clear-config
```

This strips the guarded block from any hook that was modified, removes
`.githooks/worktree-links.sh`, removes the `.git/info/exclude` entries, and
cleans up `.githooks/` and `core.hooksPath` if the directory is now empty.

## Claude Code Conversation Migration

When you delete a worktree, Claude Code conversations tied to it become
inaccessible (they live under `~/.claude*/projects/<encoded-path>/`). The
`git-worktree-remove` helper migrates them to the main worktree's project
directory before deletion so they remain visible in Claude Code's history.

```bash
# Instead of: git worktree remove ../my-feature
git-worktree-remove ../my-feature

# Works with --force too
git-worktree-remove ../my-feature --force
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
> `git worktree remove` directly and bypasses this helper. Use
> `git-worktree-remove` from the terminal when you want conversation migration.

## Lazygit

Lazygit has built-in worktree support. Press `w` to open the worktree menu. When
you create a worktree through Lazygit, it calls `git worktree add` under the
hood, so the `post-checkout` hook fires and symlinks are created automatically.

To remove a worktree with Claude conversation migration from inside Lazygit,
press `D` (capital) on a worktree in the `w` panel. This runs `git-worktree-remove`
instead of the built-in removal, so conversations are migrated first. Lazygit
will ask for confirmation before proceeding.

## Shell Helpers

### `git-worktree-setup [--do-not-track]`

Run once per project to configure the auto-symlink hook. Detects Husky
automatically and integrates without conflicting with existing hooks. Appends
a guarded block to the relevant `post-checkout` rather than overwriting it.

By default, created files are left as untracked so they can be committed.
Pass `--do-not-track` to add them to `.git/info/exclude` and hide them from
`git status` when you do not plan to commit the hook files.

### `git-worktree-clear-config`

Reverses `git-worktree-setup`. Strips the guarded block from any hook that
was modified, removes `.githooks/worktree-links.sh`, removes the
`.git/info/exclude` entries, and cleans up `.githooks/` and `core.hooksPath`
if the directory is now empty.

### `git-worktree-remove`

Drop-in replacement for `git worktree remove`. Before removing the worktree,
migrates any Claude Code conversation files (`*.jsonl`) from the worktree's
Claude project directory to the main worktree's project directory. Handles
multiple Claude accounts automatically by scanning all `~/.claude*` dirs.
Also cleans up `.git/info/exclude` entries when the last worktree is removed.

### `git-worktree-link`

Manually trigger symlink creation from inside a worktree. Useful if:

- You added new entries to `.worktree-links`
- The hook didn't fire for some reason
- You created the worktree before setting up the hook

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
git-worktree-remove ../review-pr-42
```

### Work on a hotfix while keeping current work

```bash
git worktree add ../hotfix -b hotfix/urgent-fix main
cd ../hotfix
# Fix the issue, commit, push
cd -
git-worktree-remove ../hotfix
```

### Run two branches side by side

```bash
git worktree add ../branch-a feature-a
git worktree add ../branch-b feature-b
# Both have symlinked .env, node_modules, etc.
```

## Files

| File | Purpose |
| ---- | ------- |
| `hooks/git-worktree-hook` | Hook script copied into projects as `.githooks/worktree-links.sh` |
| `functions/git-worktree.sh` | Shell helpers (`git-worktree-setup`, `git-worktree-clear-config`, `git-worktree-link`, `git-worktree-remove`) |
| `.worktree-links` (per project) | List of paths to symlink |
| `.githooks/worktree-links.sh` (per project) | The active hook script |
| `.githooks/post-checkout` (per project) | Minimal wrapper — standard projects |
| `.husky/post-checkout` (per project) | Modified to call the hook — Husky projects |

# CLAUDE.md — git-worktree-tools

Opinionated scripts for handling Git worktree workflows.

## Repo structure

```
git-worktree-tools/
├── functions/
│   └── git-worktree.sh   Shell functions sourced by the user's shell config
├── hooks/
│   └── git-worktree-hook Bash hook template copied into target repos
├── README.md
└── CLAUDE.md
```

## Shell functions

All public functions follow the `git-worktree-*` naming convention. Private
helpers are prefixed `_gwt_` to avoid polluting the user's namespace.

### `git-worktree-setup [--do-not-track]`

Run once per project. Detects Husky and integrates accordingly. Always uses
the two-file approach:

1. Copies `hooks/git-worktree-hook` to `.githooks/worktree-links.sh`
2. Appends a guarded block to the relevant `post-checkout` hook

The guarded block is wrapped in `# git-worktree-tools:start` /
`# git-worktree-tools:end` markers for clean removal. It's also safe on
machines without the setup — the `if [ -f ... ]` guard means it does nothing
if `worktree-links.sh` is absent.

With `--do-not-track`, adds created files to `.git/info/exclude` (hides them from `git status`). Without the flag, files are left as untracked so they can be committed.

### `git-worktree-clear-config`

Reverses `git-worktree-setup`. Strips guard blocks from hooks, removes
`.githooks/worktree-links.sh`, cleans up `.githooks/` + `core.hooksPath`
when empty, and removes the `.git/info/exclude` managed block.

### `git-worktree-link`

Manually re-runs symlink creation inside a worktree.

### `git-worktree-remove <path-or-name> [--force]`

Wrapper around `git worktree remove`. Migrates Claude Code conversation files
before removal. Cleans up the `.git/info/exclude` block when the last
non-main worktree is removed.

## Private helpers

| Helper | Purpose |
| ------ | ------- |
| `_gwt_append_guard <file>` | Appends the guarded block to a hook file |
| `_gwt_strip_guard <file>` | Removes the guarded block from a hook file |
| `_gwt_exclude_update <git-dir> <pattern>...` | Writes/replaces `#git-worktree:start`…`#git-worktree:end` in `.git/info/exclude` |
| `_gwt_exclude_remove <git-dir>` | Strips the managed block from `.git/info/exclude` |

## Development guidelines

- **Bash-compatible**: `functions/git-worktree.sh` must work when sourced by
  both bash and zsh. Avoid zsh-specific constructs:
  - No glob qualifiers like `(/N)` — use a `for d in .../; do [[ -d "$d" ]]` loop
  - No 1-indexed array access — use `[[ ${#arr[@]} -eq 0 ]]` to check emptiness
  - Use `local -a arr=()` + `for` loop to build arrays from globs
- **Hook script** (`hooks/git-worktree-hook`) runs inside target project repos —
  keep it dependency-free and POSIX-safe
- **sed portability** — the `sed -i` flag differs between macOS and Linux. Always
  use the temp-file pattern: `sed '...' file > tmp && mv tmp file`
- **Idempotent** — every setup step must be safe to run multiple times

## Key env var

`GIT_WORKTREE_TOOLS_DIR` — must point to this repo's root so `git-worktree-setup`
can find the hook template at `${GIT_WORKTREE_TOOLS_DIR}/hooks/git-worktree-hook`.
Set it in the shell config before sourcing `functions/git-worktree.sh`.

## Testing changes

```bash
# Create a scratch repo and test setup/clear-config round-trip
mkdir /tmp/test-gwt && cd /tmp/test-gwt && git init
touch .worktree-links
git-worktree-setup
cat .git/info/exclude          # managed block should NOT be present (no --do-not-track)
git status                     # .worktree-links + .githooks/ will show as untracked
git-worktree-setup --do-not-track
cat .git/info/exclude          # should now show managed block
git-worktree-clear-config
cat .git/info/exclude          # managed block should be gone
ls .githooks/ 2>/dev/null      # should not exist

# Test with existing post-checkout
mkdir -p .githooks && echo '#!/usr/bin/env bash' > .githooks/post-checkout
chmod +x .githooks/post-checkout && git config core.hooksPath .githooks
git-worktree-setup
grep 'git-worktree-tools:start' .githooks/post-checkout  # guard present
cat .githooks/post-checkout    # original shebang preserved, guard appended
```

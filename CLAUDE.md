# CLAUDE.md — git-worktree-tools

Opinionated CLI for handling Git worktree workflows.

## Repo structure

```
git-worktree-tools/
├── bin/
│   └── gwtree            Single CLI entry point — all commands and helpers
├── hooks/
│   └── git-worktree-hook Bash hook template copied into target repos
├── README.md
└── CLAUDE.md
```

## CLI entry point: `bin/gwtree`

All public commands are dispatched from the `case` block at the bottom of
`bin/gwtree`. The script infers its own location via `$0` and sets
`GWT_TOOLS_DIR` accordingly — no environment variable setup required from
the user.

### Public commands

| Command | Description |
| ------- | ----------- |
| `gwtree setup [--do-not-track]` | Configure current repo to use worktree hooks |
| `gwtree clear-config` | Reverse `gwtree setup` |
| `gwtree link` | Manually trigger symlink creation inside a worktree |
| `gwtree remove <path> [--force]` | Remove worktree, migrating Claude conversations first |
| `gwtree upgrade` | Pull latest version from GitHub |
| `gwtree help` | Print usage |

### `gwtree setup [--do-not-track]`

Run once per project. Detects Husky and integrates accordingly. Always uses
the two-file approach:

1. Copies `hooks/git-worktree-hook` to `.githooks/worktree-links.sh`
2. Appends a guarded block to the relevant `post-checkout` hook

The guarded block is wrapped in `# git-worktree-tools:start` /
`# git-worktree-tools:end` markers for clean removal. It's also safe on
machines without the setup — the `if [ -f ... ]` guard means it does nothing
if `worktree-links.sh` is absent.

With `--do-not-track`, adds created files to `.git/info/exclude` (hides them from `git status`). Without the flag, files are left as untracked so they can be committed.

### `gwtree clear-config`

Reverses `gwtree setup`. Strips guard blocks from hooks, removes
`.githooks/worktree-links.sh`, cleans up `.githooks/` + `core.hooksPath`
when empty, and removes the `.git/info/exclude` managed block.

### `gwtree link`

Manually re-runs symlink creation inside a worktree. Falls back through:
1. `.githooks/worktree-links.sh` (primary — always present after setup)
2. `.githooks/post-checkout` (legacy single-file setup)
3. `${GWT_TOOLS_DIR}/hooks/git-worktree-hook` (bundled fallback)

### `gwtree remove`

Wrapper around `git worktree remove`. Migrates Claude Code conversation files
before removal. Does not touch `.git/info/exclude` — that is managed
exclusively by `setup` and `clear-config`.

### `gwtree upgrade`

Runs `git -C "$GWT_TOOLS_DIR" pull --ff-only` to update the repo in-place.

## Private helpers

All defined inside `bin/gwtree`.

| Helper | Purpose |
| ------ | ------- |
| `_gwt_append_guard <file>` | Appends the guarded block to a hook file |
| `_gwt_strip_guard <file>` | Removes the guarded block from a hook file |
| `_gwt_exclude_update <git-dir> <pattern>...` | Writes/replaces `#git-worktree:start`…`#git-worktree:end` in `.git/info/exclude` |
| `_gwt_exclude_remove <git-dir>` | Strips the managed block from `.git/info/exclude` |

## Development guidelines

- **`bin/gwtree` is a script, not sourced** — use `exit 1` for errors, not `return 1`
- **`GWT_TOOLS_DIR` is inferred** — set at the top of `bin/gwtree` from `$0`; users
  do not need to set any environment variable
- **Hook script** (`hooks/git-worktree-hook`) runs inside target project repos —
  keep it dependency-free and POSIX-safe
- **sed portability** — the `sed -i` flag differs between macOS and Linux. Always
  use the temp-file pattern: `sed '...' file > tmp && mv tmp file`
- **Idempotent** — every setup step must be safe to run multiple times
- **Array compatibility** — `bin/gwtree` runs under `#!/usr/bin/env bash` so
  bash-specific array syntax is fine; still avoid zsh-only constructs for
  portability (no glob qualifiers like `(/N)`, no 1-indexed array access)

## Testing changes

```bash
# Add bin/ to PATH from repo root
export PATH="$PATH:$(pwd)/bin"

# Verify help
gwtree help

# Create a scratch repo and test setup/clear-config round-trip
mkdir /tmp/test-gwt && cd /tmp/test-gwt && git init
touch .worktree-links
gwtree setup
cat .git/info/exclude          # managed block should NOT be present (no --do-not-track)
git status                     # .worktree-links + .githooks/ will show as untracked
gwtree setup --do-not-track
cat .git/info/exclude          # should now show managed block
gwtree clear-config
cat .git/info/exclude          # managed block should be gone
ls .githooks/ 2>/dev/null      # should not exist

# Test with existing post-checkout
mkdir -p .githooks && echo '#!/usr/bin/env bash' > .githooks/post-checkout
chmod +x .githooks/post-checkout && git config core.hooksPath .githooks
gwtree setup
grep 'git-worktree-tools:start' .githooks/post-checkout  # guard present
cat .githooks/post-checkout    # original shebang preserved, guard appended

# Test upgrade
gwtree upgrade                 # should git pull --ff-only in the tools repo
```

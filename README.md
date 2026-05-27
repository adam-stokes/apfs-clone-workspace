# apfs-clone-workspace

A Claude Code plugin for fast, isolated agent workspaces using git-aware APFS copy-on-write clones.

## Why

Git worktrees share index files and have branch lock restrictions. When you dispatch multiple agents in the same monorepo, they fight over `index.lock`, can't work on the same branch, and generally make a mess.

Naive `cp -c -R` copies everything — including node_modules (200k+ files) and db snapshots (50GB+) — making the tree walk crawl.

`apfs-clone` uses `git ls-files` to identify tracked files, then `cp -c` each one with APFS CoW. You get instant, near-zero-disk-cost clones of only the code that matters.

## Install

In Claude Code, run:

```
/plugin marketplace add adam-stokes/apfs-clone-workspace
/plugin install apfs-clone-workspace
```

## How it works

The `scripts/apfs-clone` script:

1. `cp -c -R .git/` — clone full git state via APFS CoW
2. `git ls-files` — enumerate only tracked files
3. `cp -c` each file in parallel — APFS CoW per file
4. Reset index so clone starts clean

A 5,000-file repo clones in ~2 seconds, regardless of how much untracked bloat exists.

## Usage

```bash
# Clone a repo
apfs-clone /path/to/repo /tmp/my-clone

# Work in the clone
cd /tmp/my-clone
npm install  # install deps (tracked files only, no node_modules)
git checkout -b feature/my-work
# ... make changes, commit, push ...

# Cleanup
rm -rf /tmp/my-clone
```

## Comparison

| | apfs-clone | cp -c -R | git clone --local | Git Worktree |
|---|---|---|---|---|
| Speed | Fast (tracked only) | Slow on bloated repos | Fast (hardlinks) | Instant |
| Skips untracked | Yes | No | Yes | Yes |
| Disk cost | Near-zero (CoW) | Near-zero (CoW) | Low (hardlinks) | Zero (shared) |
| Independence | Full | Full | Full | Shared index/locks |
| Same branch | Yes | Yes | Yes | No |
| Concurrent agents | No conflicts | No conflicts | No conflicts | Lock conflicts |
| Platform | macOS (APFS) | macOS (APFS) | Any OS | Any OS |

## License

MIT

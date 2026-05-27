# apfs-clone-workspace

A Claude Code plugin for fast, isolated agent workspaces using `git clone --local`.

## Why

Git worktrees share index files and have branch lock restrictions. When you dispatch multiple agents in the same monorepo, they fight over `index.lock`, can't work on the same branch, and generally make a mess.

`git clone --local` creates fast clones that hardlink git objects and only check out tracked files — no node_modules, no db snapshots, no worktree bloat. Each agent gets fully independent git state.

## Install

In Claude Code, run:

```
/plugin marketplace add adam-stokes/apfs-clone-workspace
/plugin install apfs-clone-workspace
```

## What it does

Provides the `apfs-clone-workspace` skill that teaches agents to:

1. Create fast local clones of your repo for isolated workspaces
2. Work independently with full git state (status, stash, rebase — all independent)
3. Push work to remote or merge back via local fetch
4. Clean up clones when done

## Comparison

| | git clone --local | APFS cp -c | Git Worktree |
|---|---|---|---|
| Speed | Fast (hardlinks) | Instant per-block, slow on large trees | Instant |
| Skips untracked | Yes | No — copies everything | Yes |
| Disk cost | Low (hardlinked objects) | Zero until diverge | Zero (shared) |
| Independence | Full | Full | Shared index/locks |
| Same branch | Yes | Yes | No |
| Concurrent agents | No conflicts | No conflicts | Lock conflicts |
| Needs dep install | Yes | No | Yes |
| Platform | Any OS | macOS only | Any OS |

## License

MIT

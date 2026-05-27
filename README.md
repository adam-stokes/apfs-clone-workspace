# apfs-clone-workspace

A Claude Code plugin that provides instant, isolated agent workspaces using macOS APFS copy-on-write clones.

## Why

Git worktrees share index files and have branch lock restrictions. When you dispatch multiple agents in the same monorepo, they fight over `index.lock`, can't work on the same branch, and generally make a mess.

APFS clones (`cp -c`) create instant, zero-disk-cost copies of your entire repo. Each agent gets a fully independent git state — no shared anything.

## Install

In Claude Code, run:

```
/plugin marketplace add adam-stokes/apfs-clone-workspace
/plugin install apfs-clone-workspace
```

## What it does

Provides the `apfs-clone-workspace` skill that teaches agents to:

1. Create instant APFS clones of your repo for isolated workspaces
2. Work independently with full git state (status, stash, rebase — all independent)
3. Push work to remote or merge back via local fetch
4. Clean up clones when done

## Requirements

- macOS with APFS filesystem (default since High Sierra)
- That's it

## Comparison

| | APFS Clone | Git Worktree |
|---|---|---|
| Speed | Instant | Instant |
| Disk cost | Zero until diverge | Zero (shared objects) |
| Independence | Full | Shared index/locks |
| Same branch | Yes | No |
| Concurrent agents | No conflicts | Lock conflicts |
| Platform | macOS only | Any OS |

## License

MIT

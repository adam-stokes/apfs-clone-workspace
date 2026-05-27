---
name: apfs-clone-workspace
description: Use when dispatching parallel agents that need isolated copies of a repo, when git worktrees cause lock conflicts or branch restrictions, or when agents in a monorepo need full independent git state without shared index files
---

# APFS Clone Workspaces

## Overview

Create instant, isolated repo copies using macOS APFS copy-on-write clones. Each agent gets a fully independent git repo — no shared index, no lock files, no branch restrictions.

**Core principle:** Use `apfs-clone` — a git-aware APFS clone that only copies tracked files + `.git/`. Skips node_modules, db snapshots, worktrees, and all untracked bloat automatically.

## The Script

Located at: `scripts/apfs-clone` in this plugin directory.

Find it dynamically:
```bash
APFS_CLONE="$(find ~/.claude/plugins/cache -path '*/apfs-clone-workspace/scripts/apfs-clone' -type f 2>/dev/null | head -1)"
```

Usage:
```bash
$APFS_CLONE /path/to/repo /tmp/clone-name
```

What it does:
1. `cp -c -R .git/` — full git state via APFS CoW
2. `git ls-files` — get only tracked files
3. `cp -c` each tracked file in parallel — APFS CoW per file
4. Resets index so clone starts clean

## When to Use

- Dispatching 2+ agents that need to edit files in the same repo
- Monorepo where worktrees hit lock conflicts or `index.lock` errors
- Agents need to be on the same branch simultaneously
- Need full `git status`/`git stash`/`git rebase` independence per agent

**Use worktrees instead when:** Single agent, simple branch isolation, or you need the worktree to share reflog/stash with the main repo.

## Quick Reference

| Operation | Command |
|-----------|---------|
| Create clone | `$APFS_CLONE /path/to/repo /tmp/clone-name` |
| Pull work back | `git fetch /tmp/clone-name branch-name && git merge FETCH_HEAD` |
| Cherry-pick back | `git fetch /tmp/clone-name branch-name && git cherry-pick FETCH_HEAD` |
| Cleanup | `rm -rf /tmp/clone-name` |

## Dispatching Parallel Agents

Include clone setup in each agent's prompt:

```
Agent({
  description: "Fix issue #123",
  prompt: "
    First, create your isolated workspace:
      APFS_CLONE=$(find ~/.claude/plugins/cache -path '*/apfs-clone-workspace/scripts/apfs-clone' -type f 2>/dev/null | head -1)
      bash $APFS_CLONE /path/to/repo /tmp/repo-issue-123
      cd /tmp/repo-issue-123
      # Install deps if needed: make node_modules / npm install / etc.

    Then do your work:
      1. Investigate and fix the issue
      2. Run tests
      3. Commit on a new branch: git checkout -b fix/issue-123
      4. git add <files> && git commit -m 'fix: description'
      5. Push: git push -u origin HEAD

    Report: branch name, what changed, any blockers.
  "
})
```

**Launch all agents in a single message** for concurrency.

## Merging Work Back

```bash
# Option A: Fetch + merge from clone path (no remote needed)
git fetch /tmp/repo-issue-123 fix/issue-123
git merge FETCH_HEAD

# Option B: If agents pushed to remote, just pull normally
git pull origin fix/issue-123

# Option C: Cherry-pick specific commits
git fetch /tmp/repo-issue-123 fix/issue-123
git cherry-pick FETCH_HEAD~2..FETCH_HEAD
```

## Cleanup

```bash
rm -rf /tmp/repo-issue-123   # single workspace
rm -rf /tmp/repo-*            # all workspaces for a project
```

## Compared to Alternatives

| | apfs-clone | cp -c -R | git clone --local | Git Worktree |
|---|---|---|---|---|
| Speed | Fast (tracked files only) | Slow on bloated repos | Fast (hardlinks) | Instant |
| Skips untracked | Yes — git-aware | No — copies everything | Yes | Yes |
| Disk cost | Near-zero (CoW) | Near-zero (CoW) | Low (hardlinks) | Zero (shared) |
| Independence | Full | Full | Full | Shared .git/index/locks |
| Same branch | Yes | Yes | Yes | No |
| Concurrent agents | No conflicts | No conflicts | No conflicts | Lock conflicts |
| Needs dep install | Yes | No | Yes | Yes |
| Platform | macOS (APFS) | macOS (APFS) | Any OS | Any OS |

## Common Mistakes

**Forgetting to push before cleanup** — Deleting the clone loses unpushed work. Always push to remote OR fetch from the clone path first.

**Skipping dependency install** — The clone only has tracked files. Agents must install deps (node_modules, venv, etc.). Include the install command in the agent prompt.

**Non-APFS fallback** — The script detects non-APFS volumes and falls back to regular copy automatically.

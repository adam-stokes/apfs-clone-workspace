---
name: apfs-clone-workspace
description: Use when dispatching parallel agents that need isolated copies of a repo, when git worktrees cause lock conflicts or branch restrictions, or when agents in a monorepo need full independent git state without shared index files
---

# APFS Clone Workspaces

## Overview

Create instant, zero-cost isolated repo copies using macOS APFS copy-on-write clones. Each agent gets a fully independent git repo — no shared index, no lock files, no branch restrictions.

**Core principle:** `cp -c` on APFS is O(1) regardless of repo size. Only changed blocks consume disk.

## When to Use

- Dispatching 2+ agents that need to edit files in the same repo
- Monorepo where worktrees hit lock conflicts or `index.lock` errors
- Agents need to be on the same branch simultaneously
- Need full `git status`/`git stash`/`git rebase` independence per agent

**Use worktrees instead when:** Single agent, simple branch isolation, or you need the worktree to share reflog/stash with the main repo.

## Quick Reference

| Operation | Command |
|-----------|---------|
| Create clone | `cp -c -R /path/to/repo /tmp/clone-name` |
| Verify APFS clone | `stat -f '%T' /tmp/clone-name` (should show APFS volume) |
| Pull work back | `git -C /path/to/repo pull /tmp/clone-name branch-name` |
| Cherry-pick back | `git -C /path/to/repo fetch /tmp/clone-name branch-name && git cherry-pick FETCH_HEAD` |
| Cleanup | `rm -rf /tmp/clone-name` |

## Creating a Workspace

```bash
# Generate a unique workspace path
WORKSPACE="/tmp/$(basename $(pwd))-agent-$$-$(date +%s)"

# Instant APFS clone (milliseconds, zero disk cost)
cp -c -R "$(pwd)" "$WORKSPACE"

# Agent works here
cd "$WORKSPACE"
git checkout -b feature/my-work
# ... make changes, commit ...
```

## Dispatching Parallel Agents

When launching multiple agents, create one clone per agent. Each clone is fully independent.

```
# For each agent, include clone setup in the prompt:

Agent({
  description: "Fix issue #123",
  prompt: "
    First, create your isolated workspace:
      cp -c -R /path/to/project /tmp/project-issue-123
      cd /tmp/project-issue-123

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

After agents finish, pull their branches into the main repo:

```bash
# Option A: Fetch + merge from clone path (no remote needed)
git fetch /tmp/project-issue-123 fix/issue-123
git merge FETCH_HEAD

# Option B: If agents pushed to remote, just pull normally
git pull origin fix/issue-123

# Option C: Cherry-pick specific commits
git fetch /tmp/project-issue-123 fix/issue-123
git cherry-pick FETCH_HEAD~2..FETCH_HEAD
```

## Cleanup

```bash
# Remove a single workspace
rm -rf /tmp/project-issue-123

# Remove all workspaces for a project
rm -rf /tmp/project-*

# List active workspaces
ls -d /tmp/project-* 2>/dev/null
```

Clones share APFS blocks with the original — deleting a clone only frees blocks that diverged.

## Compared to Worktrees

| | APFS Clone | Git Worktree |
|---|---|---|
| Speed | Instant (filesystem CoW) | Instant (git metadata) |
| Disk cost | Zero until files diverge | Zero (shared objects) |
| Independence | Full — separate .git | Shared .git, index, locks |
| Same branch | Yes | No — branch locked |
| Concurrent agents | No conflicts | index.lock, branch locks |
| Merge back | fetch from path or push | Already same repo |
| Cleanup | `rm -rf` | `git worktree remove` |
| Platform | macOS only (APFS) | Any OS |

## Common Mistakes

**Forgetting to push before cleanup** — If the agent committed locally but didn't push, deleting the clone loses the work. Always have agents push to remote OR fetch from the clone path before removing it.

**Using on non-APFS volumes** — `cp -c` silently falls back to a full copy on HFS+/NFS. The clone will still work but takes longer and uses real disk. Verify with `diskutil info / | grep "Type"` (should show `apfs`).

**Not setting remote tracking** — Clones copy the full `.git` including remotes. Agents can push directly to origin. No extra config needed.

---
name: apfs-clone-workspace
description: Use when dispatching parallel agents that need isolated copies of a repo, when git worktrees cause lock conflicts or branch restrictions, or when agents in a monorepo need full independent git state without shared index files
---

# Isolated Agent Workspaces

## Overview

Create fast, isolated repo copies for parallel agent work. Each agent gets a fully independent git repo — no shared index, no lock files, no branch restrictions.

**Core principle:** Use `git clone --local` for agent dispatch. It hardlinks git objects (fast, low disk), only checks out tracked files (skips node_modules, snapshots, worktrees), and gives full git independence.

## When to Use

- Dispatching 2+ agents that need to edit files in the same repo
- Monorepo where worktrees hit lock conflicts or `index.lock` errors
- Agents need to be on the same branch simultaneously
- Need full `git status`/`git stash`/`git rebase` independence per agent

**Use worktrees instead when:** Single agent, simple branch isolation, or you need the worktree to share reflog/stash with the main repo.

## Quick Reference

| Operation | Command |
|-----------|---------|
| Create clone | `git clone --local /path/to/repo /tmp/clone-name` |
| Pull work back | `git -C /path/to/repo fetch /tmp/clone-name branch-name && git merge FETCH_HEAD` |
| Cherry-pick back | `git -C /path/to/repo fetch /tmp/clone-name branch-name && git cherry-pick FETCH_HEAD` |
| Cleanup | `rm -rf /tmp/clone-name` |

## Creating a Workspace

```bash
# Fast local clone — hardlinks objects, checks out only tracked files
git clone --local "$(pwd)" "/tmp/$(basename $(pwd))-agent-$$"
cd "/tmp/$(basename $(pwd))-agent-$$"

# Install deps if needed (project-specific)
# make node_modules / npm install / pip install etc.

git checkout -b feature/my-work
# ... make changes, commit ...
```

**Why not `cp -c -R`?** APFS clones are O(1) per block but still walk the entire file tree. Untracked bloat (node_modules, db snapshots, worktree dirs) gets copied too — a 50GB snapshot makes it crawl. `git clone --local` skips all untracked files automatically.

## Dispatching Parallel Agents

When launching multiple agents, create one clone per agent. Each clone is fully independent.

```
Agent({
  description: "Fix issue #123",
  prompt: "
    First, create your isolated workspace:
      git clone --local /path/to/myrepo /tmp/myrepo-issue-123
      cd /tmp/myrepo-issue-123

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
git fetch /tmp/myrepo-issue-123 fix/issue-123
git merge FETCH_HEAD

# Option B: If agents pushed to remote, just pull normally
git pull origin fix/issue-123

# Option C: Cherry-pick specific commits
git fetch /tmp/myrepo-issue-123 fix/issue-123
git cherry-pick FETCH_HEAD~2..FETCH_HEAD
```

## Cleanup

```bash
# Remove a single workspace
rm -rf /tmp/myrepo-issue-123

# Remove all workspaces for a project
rm -rf /tmp/myrepo-*

# List active workspaces
ls -d /tmp/myrepo-* 2>/dev/null
```

## Compared to Alternatives

| | git clone --local | APFS cp -c | Git Worktree |
|---|---|---|---|
| Speed | Fast (hardlinks objects) | Instant per-block, slow on large trees | Instant |
| Skips untracked | Yes — only tracked files | No — copies everything | Yes |
| Disk cost | Low (hardlinked objects) | Zero until diverge | Zero (shared objects) |
| Independence | Full — separate .git | Full — separate .git | Shared .git, index, locks |
| Same branch | Yes | Yes | No — branch locked |
| Concurrent agents | No conflicts | No conflicts | index.lock, branch locks |
| Needs dep install | Yes | No | Yes |
| Platform | Any OS | macOS only (APFS) | Any OS |

## When to Use cp -c Instead

If you need uncommitted/staged changes in the clone, or the project has no heavy untracked files, APFS `cp -c -R` is still viable:

```bash
cp -c -R /path/to/repo /tmp/clone-name
```

## Common Mistakes

**Forgetting to push before cleanup** — If the agent committed locally but didn't push, deleting the clone loses the work. Always have agents push to remote OR fetch from the clone path before removing it.

**Skipping dependency install** — `git clone --local` only gets tracked files. If the project needs node_modules, venv, etc., the agent must install them. Include the install command in the agent prompt.

**Not setting remote tracking** — Local clones copy the git config including remotes. Agents can push directly to origin. No extra config needed.

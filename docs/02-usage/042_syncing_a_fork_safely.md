# Syncing a Fork Safely Without Losing Local Changes

This guide explains how to update a fork from upstream while keeping local changes, custom documentation, patches, and fork-specific configuration intact.

## Table of Contents

1. [Core Rule](#core-rule)
2. [Terminology](#terminology)
3. [Recommended Branch Layout](#recommended-branch-layout)
4. [One-Time Remote Setup](#one-time-remote-setup)
5. [Before Syncing](#before-syncing)
6. [Safe Sync Workflow](#safe-sync-workflow)
7. [Using GitHub's Sync Fork Button](#using-githubs-sync-fork-button)
8. [Handling Conflicts](#handling-conflicts)
9. [Keeping Fork-Specific Changes Separate](#keeping-fork-specific-changes-separate)
10. [What Not To Do](#what-not-to-do)
11. [Recovery Options](#recovery-options)
12. [Quick Checklist](#quick-checklist)

(core-rule)=
## Core Rule

Never sync a fork while important local work is only in the working tree.

Before syncing, make sure your local work is either:

- committed on a branch;
- stashed with untracked files included; or
- copied to another worktree/backup branch.

Git can usually preserve committed changes during merge or rebase. Uncommitted changes are the easiest to overwrite accidentally.

(terminology)=
## Terminology

In a fork setup, the remotes usually mean:

| Name | Meaning |
| --- | --- |
| `origin` | Your fork on GitHub |
| `upstream` | The original repository you forked from |

Check your remotes:

```bash
git remote -v
```

Expected shape:

```text
origin    git@github.com:your-user/serena.git (fetch)
origin    git@github.com:your-user/serena.git (push)
upstream  git@github.com:oraios/serena.git (fetch)
upstream  git@github.com:oraios/serena.git (push)
```

(recommended-branch-layout)=
## Recommended Branch Layout

Use branches to separate upstream code from your fork-specific changes:

| Branch | Purpose |
| --- | --- |
| `main` | Tracks your fork's main branch and receives upstream syncs |
| `fork/custom-docs` | Your custom documentation and local fork notes |
| `fork/hardening` | Security hardening or behavior changes specific to your fork |
| `feature/<name>` | Short-lived feature work |

This makes future syncs much safer because upstream changes and fork-specific changes are not mixed blindly.

(one-time-remote-setup)=
## One-Time Remote Setup

If `upstream` is missing, add it:

```bash
git remote add upstream https://github.com/oraios/serena.git
git fetch upstream
```

If the upstream URL is different for your fork, replace it with the real original repository URL.

(before-syncing)=
## Before Syncing

Start by checking the working tree:

```bash
git status --short
```

If there are changes you want to keep, commit them:

```bash
git add docs/02-usage/041_per_project_index_isolation_vi.md
git add docs/02-usage/042_syncing_a_fork_safely.md
git commit -m "docs: add fork operation guides"
```

Or stash everything, including untracked files:

```bash
git stash push -u -m "wip before upstream sync"
```

Use stash only for short-term work. For important changes, a commit on a branch is safer.

(safe-sync-workflow)=
## Safe Sync Workflow

Fetch upstream first:

```bash
git fetch upstream
```

Switch to the branch you want to update:

```bash
git switch main
```

Merge upstream into your branch:

```bash
git merge upstream/main
```

Then push the updated branch to your fork:

```bash
git push origin main
```

This preserves your commits and creates a merge commit if needed.

If your team prefers a linear history, use rebase instead:

```bash
git fetch upstream
git switch main
git rebase upstream/main
git push --force-with-lease origin main
```

Use `--force-with-lease`, not `--force`. It refuses to overwrite remote work you do not have locally.

(using-githubs-sync-fork-button)=
## Using GitHub's Sync Fork Button

GitHub's **Sync fork** button updates your fork on GitHub from upstream.

Important details:

- It updates the remote fork branch on GitHub.
- It does not directly modify uncommitted files on your local machine.
- When you later run `git pull`, your local branch receives those remote changes.
- Conflicts can still happen if your fork changed the same files or lines as upstream.

Safe flow when using the GitHub button:

```bash
git status --short
git stash push -u -m "wip before pulling synced fork"
git pull origin main
git stash pop
```

If your work is already committed:

```bash
git pull origin main
```

(handling-conflicts)=
## Handling Conflicts

If Git reports conflicts, inspect them:

```bash
git status
```

Open each conflicted file and resolve the conflict markers:

```text
upstream change
```

After resolving:

```bash
git add <resolved-file>
git commit
```

For rebase:

```bash
git add <resolved-file>
git rebase --continue
```

If you are unsure, stop and inspect before continuing. Conflicts are where local work is most often lost by accident.

(keeping-fork-specific-changes-separate)=
## Keeping Fork-Specific Changes Separate

For long-lived fork customizations, prefer this pattern:

```bash
git switch main
git fetch upstream
git merge upstream/main

git switch fork/custom-docs
git rebase main
```

This keeps `main` close to upstream and replays your fork-specific changes on top.

If the fork-specific changes should always exist in `main`, commit them clearly and keep them small. Good commit messages make future conflicts easier to understand:

```bash
git commit -m "docs: document local Serena fork workflow"
git commit -m "security: disable remote dashboard news by default"
```

(what-not-to-do)=
## What Not To Do

Avoid these unless you have a backup and deliberately want to discard local changes:

```bash
git reset --hard upstream/main
git checkout -- .
git clean -fd
```

These commands can remove local edits, untracked files, or fork-specific work.

If you must reset, create a backup branch first:

```bash
git branch backup/before-reset
git reset --hard upstream/main
```

(recovery-options)=
## Recovery Options

If something goes wrong, check the reflog:

```bash
git reflog
```

Find the commit before the bad sync, then create a recovery branch:

```bash
git switch -c recovery/before-bad-sync <commit-sha>
```

If you used stash:

```bash
git stash list
git stash show -p stash@{0}
git stash pop stash@{0}
```

If a file was committed before, you can recover it from a previous commit:

```bash
git restore --source <commit-sha> -- path/to/file
```

(quick-checklist)=
## Quick Checklist

Before syncing:

1. Run `git status --short`.
2. Commit or stash all local changes.
3. Run `git fetch upstream`.
4. Merge or rebase upstream into your branch.
5. Resolve conflicts carefully.
6. Run tests or at least a focused smoke check.
7. Push to your fork.

Safe default:

```bash
git status --short
git stash push -u -m "wip before upstream sync"
git fetch upstream
git switch main
git merge upstream/main
git stash pop
```

For important fork work, use commits instead of stash.

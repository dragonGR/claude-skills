# Recovering lost work

Read this when work seems lost: after a reset, a rebase that went wrong, an amend, a dropped stash, a deleted branch, commits made on a detached HEAD, or a force push that overwrote something.

Committed work almost always survives. Git keeps unreachable commits until garbage collection removes them, and the reflog records every position `HEAD` and each branch have had (by default for 90 days for reachable entries and 30 days for unreachable ones). Uncommitted work is different: changes that were never staged or committed and were then overwritten are not in Git at all.

Before recovering anything:

1. Stop running commands that move refs (no more resets, checkouts or rebases), and do not run `git gc` or `git prune`.
2. Make a backup of the current state: `git branch backup/before-recovery-<timestamp>`.
3. Recover into a new branch, inspect it, and only then move the original branch, with the user's approval.

## Find the commit

```sh
git reflog -n 50
git reflog show <branch>
git log -g --oneline --date=iso -n 50
```

Each entry shows where `HEAD` or the branch pointed and which command moved it (`reset: moving to`, `rebase (start)`, `commit (amend)`, `checkout: moving from`). The commit you want is usually the entry just before the operation that lost it.

Inspect before restoring:

```sh
git show <sha> --stat
git log --oneline <sha> -n 10
git diff <current-branch> <sha>
```

## Common cases

**After `reset --hard` or a bad rebase.** Find the entry before the reset or before `rebase (start)` in `git reflog show <branch>`, then:

```sh
git branch recovered/<what> <sha>
```

Compare it with the current branch. To put the branch back where it was, `git reset --hard <sha>` on that branch moves it back; it discards the current state of the working tree, so confirm with the user and make sure nothing uncommitted is lost first.

`ORIG_HEAD` points at where the branch was before the last reset, rebase or merge: `git show ORIG_HEAD` is often the fastest way to see it.

**After `commit --amend`.** The pre-amend commit is the previous reflog entry of the branch (`git reflog show <branch>`, the line before `commit (amend)`).

**Deleted branch.** `git branch -D` prints the tip it deleted. If that output is gone, search the reflog for the branch's last commit (`git reflog | rg '<branch name>'`), then recreate it: `git branch <name> <sha>`.

**Detached HEAD commits.** `git reflog` shows them as `commit:` entries made while detached. Create a branch at the last one.

**Dropped or cleared stash.** Stashes are commits that are no longer referenced once dropped. Git's own documentation suggests listing unreachable stash commits like this:

```sh
git fsck --unreachable | grep commit | cut -d' ' -f3 | xargs git log --merges --no-walk --grep=WIP
```

Then apply the one you need: `git stash apply <sha>`. Stashes created with a custom message match on that message instead of `WIP`.

**Staged but never committed.** Staged content was written to the object database as blobs. `git fsck --lost-found` writes dangling blobs to `.git/lost-found/other/`; their contents are there without file names. Unstaged edits that were never added cannot be recovered by Git; check the editor's local history instead.

## A force push overwrote commits on the remote

The overwritten commits still exist in the clones of everyone who had them, including your own reflog if you ever fetched them.

1. On a machine that had the commits, find them: `git reflog show origin/<branch>` lists every position the remote-tracking ref had locally.
2. Create a branch at the right commit and compare it with the current remote state.
3. Restore by pushing that branch to a new remote branch and opening a pull request, or, with agreement from everyone using the branch, push it back with `--force-with-lease=<branch>:<current remote sha>`.

Hosting platforms also keep activity logs of pushes, and some can restore a branch or show the previous head; check before rewriting anything else.

## After recovery

- Verify the recovered branch builds and its tests pass.
- Keep the backup refs until the user confirms everything is back, then delete them with approval.
- If the loss came from an agent or a script, fix the cause: add the missing approval step, the backup ref or the dry run.

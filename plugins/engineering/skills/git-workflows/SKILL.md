---
name: git-workflows
description: Safe Git for real repositories: reflog recovery, interactive rebase and fixups, cherry-pick, reverting merges, conflicts and rerere, bisect, worktrees, force pushes with lease, secrets in history, line endings, LFS and submodules. Load it before any history rewrite, destructive command, recovery or tricky merge, or when work seems lost.
license: MIT
metadata:
  author: Alex Tsanis
---

# Git workflows

Git rarely loses work, but it makes it very easy for a person or an agent to lose track of it, and it makes a few operations permanent for everyone else: a force push over a teammate's commits, a published secret, a rewritten shared branch. The working tree, the stash, local branches and the reflog belong to the user. Treat them that way: look before acting, make a backup ref before anything risky, and ask before anything that discards or rewrites.

Check the version first (`git --version`); a few options below need a recent Git, and the repository's own conventions (merge or rebase, commit format, protected branches) override general advice.

## Safety contract

These rules hold for every Git operation an AI agent performs.

1. **Look first.** Before any change, read `git status`, `git stash list`, the current branch and its upstream (`git status -sb`), and recent history (`git log --oneline --graph -n 20`). Uncommitted or untracked work you did not create belongs to the user.
2. **Ask before anything that discards, rewrites or publishes.** That includes `reset --hard`, `clean -f`, `checkout -- <path>` or `restore` over changes, `stash` in any form (push, pop, drop, clear), `branch -D`, `rebase`, `commit --amend` on pushed commits, `filter-repo`, any force push, `push` of any kind and `commit` when the user did not ask for one. State the exact command and what it will change, then wait for approval.
3. **Back up before rewriting.** Create a ref that points at the current state before a rebase, reset, amend or history rewrite: `git branch backup/<what>-<timestamp>`. The reflog also records this, but a named ref survives garbage collection and is easy to find.
4. **Dry-run what can be dry-run.** `git clean -n` before `-f`, `git push --dry-run` before a real push, `git rebase` on a backup branch before the real one when the outcome is uncertain.
5. **Never rewrite shared history without agreement.** Commits on `main`, release branches or any branch someone else has pulled stay as they are. Fix forward with new commits or `git revert`.
6. **Force push only your own branch, only with a lease.** `git push --force-with-lease --force-if-includes origin <branch>`. A plain `--force` overwrites whatever is on the remote, including commits you have never seen.
7. **Stage deliberately.** Review `git status` and `git diff --staged` before committing. `git add -A` or `git add .` sweeps in `.env` files, keys, build output and other people's work in progress.
8. **Do not bypass checks.** No `--no-verify` to skip hooks the project relies on, no disabling of signing or branch protection to get a push through.
9. **Do not change global configuration.** `git config --global` and system config belong to the user. Repository-local settings only when the task needs them, and say so.

## Failure catalogue

**Work lost to `reset --hard` or `checkout --`.** Committed work is recoverable from the reflog for weeks. Uncommitted changes that were never staged are gone. That is why the contract says look first and back up first. Recovery steps: [references/recovery.md](references/recovery.md).

**Force push over someone else's commits.** `git push --force` after a rebase replaces the remote branch with your local copy. Commits a teammate pushed in the meantime disappear from the branch. `--force-with-lease` refuses if the remote moved since you last fetched; `--force-if-includes` also refuses if a background fetch updated your remote-tracking ref without you integrating it. Use both.

**Rebasing a shared branch.** Everyone who pulled the old commits now has a diverged history, and their next push brings the old commits back or creates duplicate merges. Rebase only branches that are yours alone, or coordinate first.

**Reverting a merge, then merging the branch again.** `git revert -m 1 <merge>` undoes the merge's changes, but Git still considers those commits merged. Merging the fixed branch later brings in only the new commits, not the reverted ones. To reapply, revert the revert first, then merge the fixes.

**Conflict resolved by taking one side wholesale.** `git checkout --ours .` or `--theirs .` during a merge or rebase silently drops the other side's changes in every conflicting file. Resolve file by file, then build and test before continuing. Remember that during a rebase "ours" is the branch being rebased onto and "theirs" is your commit.

**Secret committed and "fixed" with another commit.** The secret stays in history, in every clone, in forks and in the hosting provider's caches. Rotate or revoke it first; rewriting history afterwards only reduces exposure. Procedure and limits: [references/secrets-and-hygiene.md](references/secrets-and-hygiene.md).

**Amending or rebasing commits that are already pushed.** The remote and the local branch diverge, and the next push needs a force. Amend only unpushed commits; for pushed ones, add a fixup commit and squash only if the branch is yours and the user agrees.

**Huge, mixed commits.** One commit with a refactor, a feature, a dependency bump and formatting changes cannot be reviewed, bisected or reverted in part. Split before pushing: `git add -p` to stage hunks, interactive rebase with `edit` to split existing commits.

**Line-ending churn.** A file rewritten with different line endings shows every line changed, breaks blame and causes conflicts. Set `.gitattributes` (`* text=auto`, plus explicit `eol` rules where a tool needs them) and renormalize once in its own commit.

**Detached HEAD work abandoned.** Commits made on a detached HEAD are not on any branch and disappear from view on the next checkout. Create a branch before switching away; if it is too late, find them in the reflog.

**Hooks treated as security.** Client-side hooks run only if installed and are skipped with `--no-verify`. Secret scanning, signing requirements and checks that must hold belong in CI and server-side branch protection.

**Submodules out of sync.** A submodule pointer committed to a commit that was never pushed in the submodule repository breaks every fresh clone. Push the submodule first, then the pointer, and clone with `--recurse-submodules`.

## Decision rules

- **Rebase or merge:** rebase your own unpushed or unshared branch onto the target to keep history linear; merge (or use the platform's merge) for shared branches and for integrating long-lived branches. Follow the repository's convention when it has one.
- **Revert or rewrite:** revert anything that reached a shared branch; rewrite only private history.
- **Squash or keep commits:** keep commits that each build, pass tests and have a reason to exist on their own; squash fixups and noise. Bisect and revert work best on small, coherent commits.
- **Cherry-pick or merge:** cherry-pick a specific fix onto a release branch with `-x` so the origin is recorded; merge when you want the whole branch.
- **Worktree or stash:** use a worktree to work on or inspect another branch without touching the current working tree; stash only briefly, and never as long-term storage.
- **Recover or redo:** try the reflog before rewriting anything by hand.

## Checklist

- Did you read status, stash list, branch, upstream and recent history before changing anything?
- Is there a backup ref for every rewrite, and did the user approve every destructive or publishing command?
- Is every force push to your own branch, with `--force-with-lease --force-if-includes`?
- Was nothing on a shared branch rewritten?
- Was every conflict resolved file by file, then built and tested?
- Did you review exactly what is staged, with no secrets, build output or unrelated changes?
- Are commits small, coherent and buildable, with messages that say what changed and why?
- For a leaked secret: rotated first, then removed, with the limits of rewriting explained?

## References

- [references/recovery.md](references/recovery.md): read when work seems lost: after a reset, rebase, amend, dropped stash, deleted branch, detached HEAD or bad force push.
- [references/history-and-conflicts.md](references/history-and-conflicts.md): read for interactive rebase, fixups and autosquash, splitting commits, stacked branches, cherry-picks, reverting merges, conflicts and rerere, bisect and worktrees.
- [references/secrets-and-hygiene.md](references/secrets-and-hygiene.md): read when a secret was committed, and for line endings, large files and LFS, submodules, large repositories, signing and blame hygiene.

Related skills: human-writing for commit messages and pull request descriptions, security-engineering for secret rotation and CI, ai-code-audit before declaring a change done.

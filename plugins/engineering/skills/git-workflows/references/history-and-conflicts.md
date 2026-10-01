# History, conflicts and finding regressions

Read this for interactive rebase, fixup commits, splitting commits, stacked branches, cherry-picks, reverting merges, conflicts, `rerere`, `bisect` and worktrees. Everything here rewrites or creates history, so the safety contract in SKILL.md applies: look first, back up, and get approval for rewrites and pushes.

## Interactive rebase on your own branch

```sh
git branch backup/<branch>-<timestamp>
git rebase -i $(git merge-base HEAD origin/main)
```

In the todo list: `pick` keeps a commit, `reword` edits its message, `edit` stops to change it, `squash` and `fixup` fold it into the one above (`fixup` drops its message), `drop` removes it. Reorder lines to reorder commits.

After the rebase, compare the result with the backup before pushing:

```sh
git range-diff backup/<branch>-<timestamp>...HEAD
git diff backup/<branch>-<timestamp> HEAD
```

`git diff` should be empty when the rebase only reorganized commits. If it is not, the rebase changed code, and you need to know why.

## Fixup commits and autosquash

When review asks for a change to an earlier commit on your branch:

```sh
git commit --fixup=<sha-of-commit-to-fix>
git rebase -i --autosquash $(git merge-base HEAD origin/main)
```

`--autosquash` moves each `fixup!` commit under its target and marks it `fixup`. Set `rebase.autoSquash=true` in the repository config if the team works this way. `git commit --fixup=amend:<sha>` also replaces the target's message.

## Splitting a commit

Mark the commit `edit` in an interactive rebase. When the rebase stops:

```sh
git reset HEAD^
git add -p
git commit -m "<first logical change>"
git add -p
git commit -m "<second logical change>"
git rebase --continue
```

`git reset HEAD^` here only moves the branch back one commit and keeps every change in the working tree. Each resulting commit should build and pass tests on its own; `git rebase -i --exec "<test command>"` runs a command after every commit and stops at the first that fails.

## Stacked branches

When branch B is built on branch A and A gets rewritten, rebase B with `--update-refs` from the top of the stack so every branch ref in the stack moves with it:

```sh
git rebase -i --update-refs origin/main
```

To move a branch from one base to another, keeping only its own commits: `git rebase --onto <new-base> <old-base> <branch>`.

## Cherry-pick

```sh
git cherry-pick -x <sha>
git cherry-pick -x <first-sha>^..<last-sha>
```

`-x` appends "(cherry picked from commit …)" so the origin stays traceable. In `A..B` the commit `A` itself is excluded, which is why the range above starts at `<first-sha>^`. Cherry-picking a merge commit needs `-m <parent-number>`; usually it is better to pick the individual commits.

## Reverting

```sh
git revert <sha>
git revert -m 1 <merge-sha>
```

`-m 1` reverts a merge relative to its first parent (the branch it was merged into). Git then still considers the merged commits part of history. If the branch is fixed and merged again, only the new commits come in. To bring the reverted work back, revert the revert commit first, then merge the fixes.

## Conflicts

1. Read the conflict before resolving it. `git diff` shows both sides; `git log --merge -p <file>` shows the commits on each side that touched the file.
2. Remember the sides. In a merge, "ours" is the branch you are on and "theirs" is the branch being merged. In a rebase they swap: "ours" is the base you are replaying onto, "theirs" is your commit.
3. Resolve each hunk by understanding both intents. Taking one side for a whole file (`--ours`, `--theirs`) is correct only when you have checked that the other side's change is not needed.
4. Build and run the tests before `git rebase --continue` or the merge commit. A conflict-free result can still be broken.
5. If it goes wrong: `git merge --abort` or `git rebase --abort` returns to the state before you started.

`git rerere` (reuse recorded resolution) remembers how you resolved a conflict and applies the same resolution when it appears again, which helps with long-lived branches and repeated rebases. Enable it per repository with `git config rerere.enabled true`. Review what it applied; it reapplies a resolution, it does not check that the resolution is still correct.

## Finding a regression with bisect

```sh
git bisect start <bad-sha> <good-sha>
git bisect run ./scripts/check-regression.sh
git bisect reset
```

The script must exit 0 when the commit is good, 125 when the commit cannot be tested (it does not build, or the test infrastructure is missing), any other value from 1 to 127 when the commit is bad, and Git aborts the bisect on values above 127. Write it so that only the regression decides the result: build failures and unrelated test failures should return 125, not 1.

- `git bisect skip` marks a commit as untestable by hand.
- `git bisect start --first-parent` follows only the first parent of merges, useful when feature branches contain broken intermediate commits.
- `git bisect log` records the session; `git bisect replay <file>` repeats it.
- Bisect checks out commits in the working tree. Run it in a separate worktree so the user's working tree is untouched.

## Worktrees

```sh
git worktree add ../<repo>-<purpose> <branch-or-sha>
git worktree list
git worktree remove ../<repo>-<purpose>
git worktree prune
```

A worktree is a second working directory sharing the same repository. Use it to review a pull request, run a bisect, test against the base branch, or work on a hotfix without stashing. A branch can be checked out in only one worktree at a time. `git worktree remove` refuses when the worktree has changes; do not add `--force` without checking what would be lost.

## Searching history

```sh
git log -S '<exact string>' --oneline
git log -G '<regex>' --oneline
git log --follow -p -- <path>
git blame -w -C -C <path>
```

`-S` finds commits that change how often a string occurs (added or removed); `-G` finds commits whose diff matches a regex. `blame -w -C -C` ignores whitespace and follows code moved between files.

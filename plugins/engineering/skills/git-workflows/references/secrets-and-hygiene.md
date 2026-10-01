# Secrets in history and repository hygiene

Read this when a secret was committed, and for line endings, ignored files, large files and LFS, submodules, large repositories, commit signing and blame noise.

## A secret was committed

Order matters. Rewriting history first and rotating later leaves a window in which the secret works and is public.

1. **Rotate or revoke the secret now.** Issue a new key, update every consumer, revoke the old one, and confirm the old one no longer authenticates. Assume it was copied the moment it was pushed: public repositories are scanned for keys within minutes.
2. **Check what it could reach.** Review the provider's access logs for the period it was exposed.
3. **Remove it from the working tree** and add the file pattern to `.gitignore` (a `.env`, a key file) or move the value to the secret store.
4. **Decide whether to rewrite history.** After rotation, the old value is useless, and rewriting is about hygiene and scanners, not safety. Rewriting a shared branch needs agreement from everyone who uses it, because every clone must be replaced.
5. **If you rewrite,** use `git filter-repo` (a separate tool, recommended in Git's own documentation over `filter-branch`), in a fresh clone:

   ```sh
   git filter-repo --invert-paths --path <path/to/secret-file>
   git filter-repo --replace-text <expressions-file>
   ```

   The expressions file lists the literal values or patterns to replace. `filter-repo` removes the `origin` remote to stop an accidental push; add it back deliberately, force push every rewritten branch and tag, and tell every collaborator to re-clone.
6. **Know the limits.** Rewriting does not remove the secret from existing clones, forks, open or merged pull request refs, CI logs, caches or anything already downloaded. Hosting providers may keep cached views until their support removes them.
7. **Prevent the next one:** a secret scanner in a pre-commit hook for fast feedback, and the same scanner in CI plus the platform's push protection, because local hooks can be skipped.

## Ignored files

`.gitignore` only affects untracked files. A file committed before it was ignored stays tracked: remove it from the index with `git rm --cached <path>` and commit. Keep `.env`, keys, local config, build output and dependency folders out from the first commit; review `git status` before every commit to catch new ones.

## Line endings and attributes

```gitattributes
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.png binary
```

`text=auto` stores text files with LF in the repository and converts on checkout according to platform settings. Add explicit `eol` rules for files whose tools require one ending. After adding or changing attributes, renormalize once in a commit of its own:

```sh
git add --renormalize .
git commit -m "Normalize line endings"
```

Add that commit to `.git-blame-ignore-revs` so blame skips it.

## Blame noise

Formatting and renormalization commits make `git blame` point at the wrong author. List those commit hashes in `.git-blame-ignore-revs` and enable it locally:

```sh
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

## Large files and LFS

Large binaries in normal Git history make every clone slower forever, and removing them later requires a rewrite.

- Track large binary types with Git LFS before they are first committed: `git lfs track "*.psd"` writes the rule to `.gitattributes`, which you commit.
- Converting existing history to LFS (`git lfs migrate import`) rewrites history, with the same coordination cost as any rewrite.
- Build output, dependencies and generated artifacts do not belong in Git at all.

## Submodules

```sh
git clone --recurse-submodules <url>
git submodule update --init --recursive
```

- A submodule is pinned to a commit. Push the submodule's commit before pushing the parent commit that points to it, or every fresh clone fails.
- `git config submodule.recurse true` makes checkout and pull update submodules, which avoids stale pointers in local work.
- Review submodule pointer changes in diffs like code changes; a moved pointer can change a lot of code invisibly.

## Large repositories

```sh
git clone --filter=blob:none <url>
git sparse-checkout set <dir> <dir>
git maintenance start
```

A blobless partial clone downloads file contents on demand. Sparse checkout limits the working tree to the directories you need. `git maintenance start` schedules background maintenance tasks such as prefetch and commit-graph updates.

## Signing commits

```sh
git config gpg.format ssh
git config user.signingkey <path-to-public-key>
git config commit.gpgsign true
git log --show-signature -n 5
```

Signing proves a commit came from a key you control. It matters only if the platform verifies it: require signed commits in branch protection, and register the signing key with the platform. Set these in the repository or ask the user before changing global configuration.

## Hooks

Client-side hooks are useful for fast feedback (formatting, a quick secret scan, commit message format) and are not enforcement: they are not cloned with the repository, each developer installs them, and `--no-verify` skips them. Anything that must hold for every commit belongs in CI and branch protection.

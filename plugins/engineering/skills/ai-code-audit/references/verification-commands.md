# Verification commands

Read this when collecting an AI change and checking it against the repository. The commands are read-only: they inspect, they never reset, stash, check out over the working tree or push. Searches use `rg` (ripgrep); adjust globs to the repository's layout.

## Collect the whole change

```sh
base=$(git merge-base HEAD origin/main)
git diff --stat "$base"...HEAD
git diff --name-status "$base"...HEAD
git status --porcelain
git diff "$base"...HEAD -- '*lock*' 'package.json' 'pyproject.toml' 'Cargo.toml' 'foundry.toml' 'remappings.txt'
```

- `git status --porcelain` shows uncommitted and untracked files. An agent's real change often sits in an untracked file the diff never shows.
- Look at deleted (`D`) and renamed (`R`) entries first. Deletions of tests, migrations, checks and config are where the damage hides.
- For a pull request from someone else, fetch it into a separate worktree (`git worktree add /tmp/audit-pr <branch>`) and audit there.

## Packages

For every dependency the change adds or bumps:

```sh
git diff "$base"...HEAD -- package.json pyproject.toml requirements*.txt Cargo.toml | rg '^\+'
```

Then check each new name against its registry:

```sh
curl -s "https://registry.npmjs.org/<name>" | jq '{created: .time.created, latest: ."dist-tags".latest, maintainers: [.maintainers[].name], repo: .repository.url}'
curl -s "https://pypi.org/pypi/<name>/json" | jq '{name: .info.name, home: .info.project_urls, releases: (.releases | keys | length)}'
cargo info <name>
```

Red flags: the package was created recently, has one release, has a name one character away from a popular package, has no repository or a repository that does not match, or has install scripts. If the model named the package and you cannot find it in the project's docs, an upstream README or the lockfile of a well-known project using it, treat it as invented until shown otherwise.

Confirm the installed version is the one the code assumes:

```sh
rg -n '"<name>"' package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null | head
rg -n '^name = "<name>"' -A2 Cargo.lock uv.lock poetry.lock 2>/dev/null
```

## APIs

Find every external symbol the change calls and locate it in the installed source:

```sh
rg -n 'export (declare )?(async )?(function|const|class|interface|type) <symbol>\b' node_modules/<pkg> -g '*.d.ts'
python -c "import inspect, <module>; print(inspect.signature(<module>.<function>))"
rg -n 'pub (async )?fn <symbol>\b' ~/.cargo/registry/src/*/<crate>-<version>/src
rg -n 'function <symbol>\(' lib/<dependency>/contracts
```

- Check options and parameter names, not only the function. Invented options are passed in object literals and type checking may not catch them when the parameter type is loose.
- Check removed and renamed APIs in the dependency's changelog between the version the code was written for and the version in the lockfile.
- Run the type checker on the whole project (`npx tsc --noEmit`, `pyright` or `mypy`, `cargo check --all-targets`, `forge build`), not only on changed files.

## Configuration and environment

```sh
git diff "$base"...HEAD | rg -o 'process\.env\.[A-Z0-9_]+|os\.environ(\.get)?\(?\[?"[A-Z0-9_]+"|env::var\("[A-Z0-9_]+"\)|vm\.env\w*\("[A-Z0-9_]+"\)' | sort -u
```

For each variable, find where it is set: `.env.example`, deployment manifests, CI workflow files, the secret store's inventory. A variable read in code and set nowhere is a hallucination or a missing deployment step.

For each new config key, CLI flag or option in a tool's config file, find it in that tool's docs for the pinned version. Then observe its effect, for example by running the tool with a deliberately wrong value and confirming it complains or behaves differently. A key that changes nothing is either invented or ignored.

## Schema, routes and contracts

```sh
rg -n '<column_or_table>' migrations/ db/ prisma/schema.prisma -g '*.sql' -g '*.prisma'
rg -n "['\"]/api/<path>" src/ -g '!*.test.*'
forge inspect <Contract> methodIdentifiers
forge inspect <Contract> events
```

Every column the change queries exists in a migration that runs before it. Every endpoint the frontend calls is served by a route with the same method. Every contract function and event the backend or frontend uses exists in the deployed ABI, not only in the source on the branch.

## Suppressions and placeholders

Run on the added lines only, so existing debt does not drown the change:

```sh
git diff -U0 "$base"...HEAD | rg '^\+' | rg -n \
  -e '\bas any\b' -e 'as unknown as' -e '@ts-(ignore|nocheck|expect-error)' -e 'eslint-disable' \
  -e '# ?type: ?ignore' -e '# ?noqa' -e '# ?pragma: no cover' \
  -e '#\[allow\(' -e '\.unwrap\(\)' -e '\.expect\("' -e 'unimplemented!|todo!' -e '//\s*nolint' \
  -e '\b(TODO|FIXME|HACK|XXX)\b' -e 'not implemented' -e 'for now' -e 'temporar' \
  -e 'catch\s*(\(\w*\))?\s*\{\s*\}' -e '\.catch\(\(\)\s*=>\s*(\{\s*\}|null|undefined)\)' -e 'except[^:]*:\s*pass' \
  -e 'continue-on-error' -e '\|\|\s*true\b'
```

Then read each hit in context. Some are legitimate (`.unwrap()` on a value that cannot fail, `expect` with a stated invariant); the point is that each one is justified, not that none exist.

Configuration that got looser:

```sh
git diff "$base"...HEAD -- '*eslint*' 'tsconfig*.json' 'pyproject.toml' 'setup.cfg' 'clippy.toml' '.github/workflows/*' 'foundry.toml' 'vitest.config.*' 'jest.config.*'
```

Look for disabled rules, `strict` options turned off, raised thresholds, removed CI steps, `continue-on-error`, narrowed test globs and reduced fuzz or invariant runs.

## Tests

```sh
git diff --name-status "$base"...HEAD -- '*test*' '*spec*' 'test/' 'tests/'
git diff -U0 "$base"...HEAD -- '*test*' '*spec*' | rg '^-' | rg -e 'expect|assert|should|toBe|toEqual|require\(|vm\.expect'
git diff -U0 "$base"...HEAD | rg '^\+' | rg -e '\.(only|skip)\(' -e '\b(xit|xdescribe|xtest)\(' -e 'it\.todo' -e '@pytest\.mark\.(skip|xfail)' -e '#\[ignore\]' -e 'vm\.skip'
git diff --name-only "$base"...HEAD -- '*__snapshots__*' '*.snap'
```

- The second command lists removed assertion lines. Each one needs a reason: the behavior it checked was deliberately changed, and a replacement assertion exists.
- Snapshot files changed in the same diff as the code need their diff read, not accepted.
- Then follow [attack-tests.md](attack-tests.md) to prove the new tests fail without the change.

## Execute

Run what CI runs, from the project's own scripts (`package.json` scripts, `Makefile`, `justfile`, CI workflow), not a narrower command:

```sh
npm run build && npm run typecheck && npm run lint && npm test
cargo build --all-targets && cargo clippy --all-targets -- -D warnings && cargo test
forge build && forge test
```

Read the output for warnings and skipped counts, not only the exit code. When a result needs comparing (performance, a flaky test, a warning count), run the same command in a worktree at the base.

# Compiler settings, modules and dependencies

Read this when changing `tsconfig.json` or the TypeScript version, lint configuration, `package.json` module fields or `exports`, publishing a package, adding or upgrading a dependency, configuring Renovate or Dependabot, or reviewing a lockfile diff.

## Make sure types are checked at all

Most toolchains that run or bundle TypeScript strip types without checking them: esbuild, swc, Vite, tsx, Bun, ts-node in transpile-only mode, and Node's built-in type stripping. A project can build, deploy and run for months with type errors. The only check is `tsc`.

- CI runs `tsc --noEmit` (or `tsc -b` for project references) as a required step, over tests as well as source.
- Type-aware lint runs in CI with `parserOptions.projectService` (or `project`) configured; without type information, the promise and `unsafe` rules below do nothing.
- Editors are not evidence. A file nobody opened can be broken.

## TypeScript 6 and 7

TypeScript 7 (7.0.2, July 2026) is the compiler rewritten in Go, 8 to 12 times faster on full builds in the release announcement's benchmarks, and it ships no programmatic API; the TypeScript team expects a new one in 7.1. Anything that imports `typescript` as a library still needs 6: typescript-eslint and ts-jest declare peer ranges that stop before 7, and every type-aware lint rule in this file runs through typescript-eslint. Install both side by side, as the 7.0 release notes describe:

```json
{
  "devDependencies": {
    "typescript": "npm:@typescript/typescript6@^6.0.2",
    "@typescript/native": "npm:typescript@^7.0.2"
  }
}
```

`tsc` is then 7, `tsc6` is 6, and libraries that import `typescript` get 6. After the change, check which compiler the lint step actually loads. The failure to avoid is a bump to 7 that breaks type-aware lint, after which someone turns the lint step off.

Upgrade through 6.0 first. 6.0 changed defaults: `strict: true`, `module: esnext`, `target` set to the newest ECMAScript version, `types: []`, `rootDir: "."` and `noUncheckedSideEffectImports: true`. It deprecated what 7.0 turns into errors: `moduleResolution: node` (`node10`) and `classic`, `baseUrl`, `target: es5`, `downlevelIteration`, `module` set to `amd`, `umd`, `system` or `none`, `outFile`, and `esModuleInterop` or `allowSyntheticDefaultImports` set to `false`.

- `types: []` is the change that breaks most upgrades. `@types/node` and test-runner globals stop loading, so `process`, `Buffer` and `describe` become errors. List what the project uses (`"types": ["node"]`) instead of adding `declare` shims, `any` or `skipLibCheck`.
- With `rootDir` now the config's own directory, a project that never set it and keeps sources in `src/` emits to `dist/src/`. Set `rootDir` explicitly.
- `baseUrl` is gone in 7. Path aliases need `paths` entries relative to the config file, or `package.json` `imports`.
- `"ignoreDeprecations": "6.0"` silences the warnings in 6 and does nothing in 7, where the options are errors. Clear every deprecation before moving.

## Compiler flags

`strict` (on by default from TypeScript 6; set it explicitly anyway so older compilers and readers agree) enables `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `strictBuiltinIteratorReturn`, `noImplicitThis`, `useUnknownInCatchVariables` and `alwaysStrict`. It does not enable the two flags that catch the most runtime `undefined` bugs:

| Flag | Catches |
|---|---|
| `noUncheckedIndexedAccess` | `rows[0].id`, `byKey[k].total`, `match[1]` used as if always present |
| `exactOptionalPropertyTypes` | `{ timeoutMs: undefined }` passed where `timeoutMs?: number` is declared, which overwrites defaults in spreads and means "no filter" in ORMs |
| `noImplicitOverride` | a subclass method silently no longer overriding after a base rename |
| `noFallthroughCasesInSwitch` | a `case` missing its `break` or `return` |
| `verbatimModuleSyntax` | type imports written without `type`, which Node's type stripping keeps as value imports and fails on at runtime (Node's docs recommend this flag) |
| `erasableSyntaxOnly` | enums, runtime namespaces and parameter properties, which Node's type stripping cannot run |

Turning a flag on in an existing codebase produces a list of real bugs mixed with noise. Fix them in batches by module. Silencing them with `!` or `as` defeats the flag and marks the spot as reviewed when it was not.

`skipLibCheck: true` skips checking `.d.ts` files. It is common for build speed and usually fine, but it also hides a dependency's broken or conflicting declarations; turn it off temporarily when types from a package look wrong.

## Module resolution

| Code runs under | `module` / `moduleResolution` |
|---|---|
| Node directly (including Node type stripping) | `nodenext` with `"type"` set in `package.json` |
| A bundler, Bun or tsx | `esnext` or `preserve` with `bundler` |

The TypeScript docs say not to use `bundler` for code Node runs. The failure: `bundler` accepts extensionless relative imports (`import { x } from "./util"`), the build passes, and Node's ESM loader throws `ERR_MODULE_NOT_FOUND` at startup. Under `nodenext`, relative ESM imports carry the runtime extension (`./util.js`, or `./util.ts` with `rewriteRelativeImportExtensions`).

In ESM there is no `__dirname` or `require`. Use `fileURLToPath(new URL(".", import.meta.url))` and `createRequire(import.meta.url)`; don't paste a CJS shim that computes paths relative to the working directory.

## Dual packages

A package that ships both CommonJS and ESM builds can be instantiated twice in one process: once through `import`, once through `require` (yours or a dependency's). Symptoms:

- a singleton configured at startup appears unconfigured elsewhere;
- plugins or schema registries registered in one copy are missing in the other;
- `err instanceof LibError` is false for errors thrown by the other copy, so error mapping falls through to 500;
- two caches, two connection pools.

Check with `npm ls <pkg>` (or `pnpm why <pkg>`) and by logging the resolved path from both import sites. For applications, settle on one format and make every import path agree. For libraries you publish:

- Prefer ESM-only. Node 22.12 and every later line let CommonJS `require()` an ES module without a flag, as long as the module graph has no top-level `await`; otherwise `ERR_REQUIRE_ASYNC_MODULE` is thrown.
- If you must ship both, keep all state in one format and make the other a thin wrapper that re-exports it.
- Put `"types"` first in each conditions object in `exports`; conditions match in order.
- Once `exports` exists, every subpath not listed is unreachable (`ERR_PACKAGE_PATH_NOT_EXPORTED`). Adding `exports` to an existing package is a breaking change for consumers of deep imports.
- Check the published shape with `publint` and `@arethetypeswrong/cli` before release.

## Lint rules that catch production bugs

All need type information. The preset column says where a rule is already on; "none" means no preset enables it, so a config that only extends `recommended-type-checked` does not check exhaustiveness.

| Rule | Preset | Bug it catches |
|---|---|---|
| `no-floating-promises` | recommended-type-checked | un-awaited promises, including arrays of promises from `.map` |
| `no-misused-promises` | recommended-type-checked | async callbacks to `forEach` and event handlers, promises in `if` conditions |
| `await-thenable` | recommended-type-checked | `await` on a non-promise, usually a missing call |
| `no-unsafe-assignment`, `no-unsafe-member-access`, `no-unsafe-argument` | recommended-type-checked | `any` from `JSON.parse`, `res.json()` or untyped packages spreading through the code |
| `only-throw-error`, `prefer-promise-reject-errors` | recommended-type-checked | throwing strings and plain objects |
| `no-unsafe-enum-comparison` | recommended-type-checked | comparing an enum to a raw literal |
| `restrict-plus-operands` | recommended-type-checked | `"5" + 1` style concatenation in arithmetic |
| `return-await` (default `in-try-catch`) | strict-type-checked | `return promise` inside `try`, which skips the `catch` |
| `no-unnecessary-condition` | strict-type-checked | `?.` and checks on values the types say cannot be nullish, often a sign the type is wrong |
| `use-unknown-in-catch-callback-variable` | strict-type-checked | `.catch((e) => e.message)` with `e: any` |
| `switch-exhaustiveness-check` | none | a `switch` over a union that misses a member |
| `strict-boolean-expressions` with `allowNumber: false`, `allowString: false` | none | `0`, `""` and `NaN` taking the falsy branch (the defaults allow plain numbers and strings) |

An `eslint-disable` for one of these needs a one-line reason naming the invariant that makes it safe.

## Escape hatches in review

- `as T` on data from outside the process: replace with a schema parse.
- `!` non-null assertions: each is a claim that needs local evidence (a check two lines up, a constructor guarantee). Otherwise handle the empty case.
- `as unknown as T`: almost always a type that is wrong somewhere else.
- `// @ts-ignore`: replace with `// @ts-expect-error <reason>`, which fails when the error goes away.
- A type guard (`(x: unknown): x is User`) that checks one field and asserts the whole shape is worse than a cast, because it looks verified. Generate guards from schemas.
- `declare module "pkg";` shims make the whole package `any`. Write the few signatures you use.

## Dependencies and the install step

### Installing in CI and production builds

| Package manager | Frozen install |
|---|---|
| npm | `npm ci` (fails if `package.json` and the lockfile disagree) |
| pnpm | `pnpm install --frozen-lockfile` |
| Yarn (Berry) | `yarn install --immutable` |
| Bun | `bun install --frozen-lockfile` |

A plain `npm install` in CI can resolve new versions and ship code nobody reviewed.

### Install scripts

`preinstall`, `install` and `postinstall` scripts of every transitive dependency run with the permissions of whoever installs: the developer's SSH keys and cloud credentials, the CI runner's deploy tokens. Compromised maintainer accounts publishing malicious patch releases, and self-propagating worms that steal npm tokens from install scripts, have hit widely used packages.

- npm 12 skips the `preinstall`, `install` and `postinstall` scripts of any dependency not approved in the `allowScripts` field of `package.json`, including the implicit `node-gyp rebuild` for packages that ship a `binding.gyp`. The install still succeeds and only lists what it skipped, so a native package (`sharp`, `bcrypt`, `better-sqlite3`) breaks the first time it is loaded. npm 11.19, bundled with Node 24 and 26, reads the same field but still runs every script and only reports. Check `npm --version` before reasoning about either.
- Maintain the field with `npm approve-scripts <pkg>`, which pins each approval to the version you reviewed, and commit it. `npm approve-scripts --allow-scripts-pending` lists what still needs a decision; run it after every lockfile change. Set `strict-allow-scripts=true` in `.npmrc` so an install fails on a dependency nobody has ruled on instead of quietly skipping it. `--dangerously-allow-all-scripts` exists for migrating and has no place in CI. Workspace packages are not gated.
- npm 12 also refuses git dependencies (`allow-git`) and tarball-URL dependencies from hosts other than the registry (`allow-remote`) unless configured; both default to `none`. Prefer opening them up with `root`, which allows only those your own `package.json` declares, over `all`.
- pnpm does not run dependency build scripts until they are approved, and with `strictDepBuilds` (default `true`) the install exits non-zero when it meets an unreviewed one; approved packages go in `allowBuilds` (pnpm 10.26+, replacing `onlyBuiltDependencies`, which pnpm 11 removed). From pnpm 11, `minimumReleaseAge` defaults to 1440 minutes, but that built-in default falls back to a newer version when nothing old enough satisfies the range; setting it explicitly makes it strict. `blockExoticSubdeps` (default `true`) stops transitive dependencies resolving to git repositories or tarball URLs. In review, flag a config that turns either off.
- Bun does not run dependency lifecycle scripts unless the package is in `trustedDependencies` or Bun's built-in allowlist. Defining the field replaces that list rather than extending it.
- CI jobs that install dependencies should not hold publish or deploy credentials in the same step.

### Adding a dependency

- Could a few lines of code or an existing dependency do this? Each package brings its whole transitive tree and its maintainers' account security.
- Is the name exactly right? Typosquats differ by a letter, a hyphen or a scope. Check the repository link and the maintainers on the registry page.
- Does it have install scripts, a recent ownership change, or a new major version with a different publisher?
- Does it ship its own types, and are they for the version you install?
- Is it maintained, and does it work on your runtime (Node built-ins on Workers, native addons on Alpine or ARM)?

### Reviewing a lockfile diff

- `resolved` URLs pointing anywhere except the configured registry, and `git+` or tarball sources in transitive dependencies. npm 12 and pnpm block most of these by default, so one appearing means someone changed `allow-git`, `allow-remote` or `blockExoticSubdeps`.
- `integrity` hashes changing for a version that did not change.
- A `package.json` change with no lockfile change, or the reverse.
- Many new transitive packages from a minor upgrade.
- A new or widened `allowScripts` (npm) or `allowBuilds` (pnpm) entry, or a `true` for a package that was `false` before.

Use `overrides` (npm, pnpm) or `resolutions` (Yarn) to force a patched transitive version, and remove the override once the direct dependency updates.

## Upgrading dependencies

An upgrade is a production change with nothing to demo, so it gets less review than it needs. Change one thing at a time, know what changed, and keep the way back open. Moving to a new Node major is in nodejs-engineering's node-upgrades reference.

### Before touching versions

- Start from a green build, type check, lint and test run. On a red baseline you cannot tell what the upgrade broke.
- Take an inventory:

  ```sh
  npm outdated                                  # pnpm outdated
  npm ls <package>                              # every copy and where it sits
  npm explain <package>                         # why it is installed (alias: npm why); pnpm why <package>
  npm approve-scripts --allow-scripts-pending   # npm 11.19+: install scripts awaiting a decision
  ```

- Read the changelog and migration guide for every version between the current and the target, not only the latest. Minor and patch releases change defaults, timeouts, error types and transitive dependencies too, and `0.x` packages promise nothing. Note removed APIs, changed defaults, new peer ranges and the minimum Node version.
- A framework major usually needs new majors of its plugins, type packages and test tooling. Those move together, and nothing else moves with them.
- Write down the rollback before starting: the previous lockfile and image, and whether a data or config migration makes going back impossible.

### Order and size

Toolchain first when it blocks the rest (TypeScript, then the build tool and test runner), then libraries with many dependents (the web framework, the ORM) one major per pull request, then leaf libraries, which can be grouped for minor and patch updates. For a jump across several majors, go one major per step when the project publishes a migration guide per major. Deploy each step before starting the next: a pull request that moves the framework, the ORM and forty libraries together cannot be bisected or partly rolled back.

Run a framework's codemods on a clean branch, commit their output on its own, and read the whole diff. They miss dynamic usage, re-exports and wrappers.

### Peer conflicts and overrides

npm 7 and later install peer dependencies and stop with `ERESOLVE` when ranges conflict: two packages need incompatible versions of a third.

- Move both packages to versions that agree.
- If one lags, `overrides` (npm, pnpm) or `resolutions` (Yarn) can force a version. Record the reason, the upstream issue and a removal date in the pull request.
- `--legacy-peer-deps` and `--force` hide every conflict, now and later, and can leave a library running against a peer it was never tested with. Keep them out of CI and production builds.
- Afterwards, `npm ls <package>` should show one copy of anything that holds state or identity (React, framework cores, ORMs, `@types` packages).

### Testing an upgrade

Besides the suite, type check and lint: characterization tests written before the upgrade for behavior the changelog touches (serialization, dates, validation, error shapes, default timeouts); a production build deployed to staging, because bundlers and native modules break in builds rather than in tests; a clean install from the lockfile with install scripts gated, because a native dependency whose approval no longer matches fails only when loaded; and the deprecation warnings in test and build output, each of which is a break in the next major.

### Automated updates

- Group minor and patch updates of related packages; keep each major as its own pull request.
- Hold back fresh releases, which is when hijacked versions are live: Renovate `minimumReleaseAge`, Dependabot `cooldown` (a 3-day default for version updates), npm `min-release-age` in days in `.npmrc`, pnpm `minimumReleaseAge` in minutes. Security updates bypass the wait. When the wait stops `npm audit fix` from installing a fix, npm 12 keeps the vulnerable version and exits non-zero; add that package to `min-release-age-exclude` rather than dropping the setting.
- Automerge only minor and patch updates of well-tested packages, with required checks passing. Never automerge majors, the toolchain, or anything in authentication, cryptography or payments.
- Refresh the lockfile regularly so transitive updates do not pile up into one large change.

### Reviewing an upgrade pull request

- Does it move one major of one thing, plus its required companions?
- Are the changelog and migration notes linked, and is each breaking change accounted for in the diff?
- Does the lockfile diff contain only the expected packages (the lockfile checks above)?
- Are overrides, forced installs and peer conflicts documented with a reason and a removal date?
- Did CI run the full suite on the new versions, and did a production build reach staging?

### Rolling back

Revert the pull request and redeploy the previous image built from the previous lockfile. Rebuilding from a reverted `package.json` without the old lockfile does not restore the old dependency tree. If the upgrade included a data or config migration, the rollback has to undo or tolerate it, and that has to be known before the upgrade ships.

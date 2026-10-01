# Compiler settings, modules and dependencies

Read this when changing `tsconfig.json`, lint configuration, `package.json` module fields or `exports`, publishing a package, adding or upgrading a dependency, or reviewing a lockfile diff.

## Make sure types are checked at all

Most toolchains that run or bundle TypeScript strip types without checking them: esbuild, swc, Vite, tsx, Bun, ts-node in transpile-only mode, and Node's built-in type stripping. A project can build, deploy and run for months with type errors. The only check is `tsc`.

- CI runs `tsc --noEmit` (or `tsc -b` for project references) as a required step, over tests as well as source.
- Type-aware lint runs in CI with `parserOptions.projectService` (or `project`) configured; without type information, the promise and `unsafe` rules below do nothing.
- Editors are not evidence. A file nobody opened can be broken.

## Compiler flags

`strict` enables `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `strictBuiltinIteratorReturn`, `noImplicitThis`, `useUnknownInCatchVariables` and `alwaysStrict`. It does not enable the two flags that catch the most runtime `undefined` bugs:

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

- Prefer ESM-only. Node 20.19+ and 22.12+ let CommonJS `require()` an ES module without a flag, as long as the module graph has no top-level `await`; otherwise `ERR_REQUIRE_ASYNC_MODULE` is thrown.
- If you must ship both, keep all state in one format and make the other a thin wrapper that re-exports it.
- Put `"types"` first in each conditions object in `exports`; conditions match in order.
- Once `exports` exists, every subpath not listed is unreachable (`ERR_PACKAGE_PATH_NOT_EXPORTED`). Adding `exports` to an existing package is a breaking change for consumers of deep imports.
- Check the published shape with `publint` and `@arethetypeswrong/cli` before release.

## Lint rules that catch production bugs

All need type information. The first group is in `recommended-type-checked`; the second is in `strict-type-checked` or must be enabled individually.

| Rule | Bug it catches |
|---|---|
| `no-floating-promises` | un-awaited promises, including arrays of promises from `.map` |
| `no-misused-promises` | async callbacks to `forEach` and event handlers, promises in `if` conditions |
| `await-thenable` | `await` on a non-promise, usually a missing call |
| `no-unsafe-assignment`, `no-unsafe-member-access`, `no-unsafe-argument` | `any` from `JSON.parse`, `res.json()` or untyped packages spreading through the code |
| `only-throw-error`, `prefer-promise-reject-errors` | throwing strings and plain objects |
| `no-unsafe-enum-comparison` | comparing an enum to a raw literal |
| `restrict-plus-operands` | `"5" + 1` style concatenation in arithmetic |
| `switch-exhaustiveness-check` | a `switch` over a union that misses a member |
| `return-await` (default `in-try-catch`) | `return promise` inside `try`, which skips the `catch` |
| `strict-boolean-expressions` with `allowNumber: false`, `allowString: false` | `0`, `""` and `NaN` taking the falsy branch (the defaults allow plain numbers and strings) |
| `no-unnecessary-condition` | `?.` and checks on values the types say cannot be nullish, often a sign the type is wrong |
| `use-unknown-in-catch-callback-variable` | `.catch((e) => e.message)` with `e: any` |

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

- npm runs dependency scripts by default. Use `--ignore-scripts` in CI where the dependency tree allows it, and rebuild the few native packages that need it explicitly.
- pnpm does not run dependency build scripts until they are approved, and with `strictDepBuilds` (default `true`) the install exits non-zero when it meets an unreviewed one; approved packages go in `allowBuilds` (pnpm 10.26+, replacing `onlyBuiltDependencies`, which pnpm 11 removed). `minimumReleaseAge` (in minutes) holds back versions published too recently, which is when hijacked releases are live, and `blockExoticSubdeps` stops transitive dependencies from resolving to git repositories or tarball URLs.
- Bun does not run dependency lifecycle scripts unless the package is in `trustedDependencies` or Bun's built-in allowlist; setting the field replaces that list.
- CI jobs that install dependencies should not hold publish or deploy credentials in the same step.

### Adding a dependency

- Could a few lines of code or an existing dependency do this? Each package brings its whole transitive tree and its maintainers' account security.
- Is the name exactly right? Typosquats differ by a letter, a hyphen or a scope. Check the repository link and the maintainers on the registry page.
- Does it have install scripts, a recent ownership change, or a new major version with a different publisher?
- Does it ship its own types, and are they for the version you install?
- Is it maintained, and does it work on your runtime (Node built-ins on Workers, native addons on Alpine or ARM)?

### Reviewing a lockfile diff

- `resolved` URLs pointing anywhere except the configured registry, and `git+` or tarball sources in transitive dependencies.
- `integrity` hashes changing for a version that did not change.
- A `package.json` change with no lockfile change, or the reverse.
- Many new transitive packages from a minor upgrade.

Use `overrides` (npm, pnpm) or `resolutions` (Yarn) to force a patched transitive version, and remove the override once the direct dependency updates.

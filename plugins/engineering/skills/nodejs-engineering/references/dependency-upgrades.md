# Upgrading Node and dependencies

Read this before upgrading Node, a framework or any major dependency, when reviewing an upgrade pull request, or when setting up automated updates. Lockfile handling, install scripts and vetting a new package are covered in the typescript-engineering skill; this file is about changing versions of what you already use.

An upgrade is a production change with no feature to show for it, which is why it gets less review than it needs. The goal is to change one thing at a time, know what changed, and be able to undo it.

## Before touching versions

1. Start from a clean tree with a passing build, type check, lint and test suite on the current versions. If the baseline is red, you cannot tell what the upgrade broke.
2. Take an inventory:

   ```sh
   node --version
   npm outdated            # pnpm outdated, yarn outdated
   npm ls <package>        # where a package sits in the tree
   npm explain <package>   # why it is installed (alias: npm why); pnpm why <package>
   ```

3. For each package you will move across a major version, read the changelog and migration guide for every version between the current and the target, not only the latest. Note removed APIs, changed defaults, new peer dependency ranges and the minimum Node version.
4. Check what depends on what. A framework major often requires new majors of its plugins, its type packages and its test tooling. Upgrade those together, and nothing else with them.
5. Write down how you will roll back: the previous lockfile and image, and whether any data or config migration makes rollback impossible.

## Order of work

- Toolchain first when it blocks everything else: Node, then TypeScript, then the build tool and the test runner. Each in its own change.
- Then frameworks and libraries with many dependents (the web framework, the ORM, the UI library), one major at a time. For a jump across several majors, go one major per step when the project publishes a migration guide per major.
- Leaf libraries last, and these can be grouped when they are minor or patch updates with good tests.
- Deploy each step before the next. A merged but undeployed upgrade is a bigger, riskier change waiting to happen.

## Upgrading Node itself

- Production runs an active LTS release. Plan the move to the next LTS well before the current one reaches end of life; the schedule is published at nodejs.org.
- Read the release notes for each major you cross. Typical breaks: deprecated APIs removed, stricter OpenSSL defaults rejecting old keys or ciphers, changed HTTP defaults and timeouts, a new bundled npm, and native modules that need a rebuild or a newer version.
- Change the version in every place that installs Node in one change: `engines`, `.nvmrc` or `.node-version`, CI images and matrices, the Docker base image (pinned by digest), serverless runtime settings and developer tooling. A mismatch between CI and production defeats the point of testing.
- Run CI on both the old and new version for a while when the upgrade is risky, then drop the old one.
- Watch error rates, memory and event loop delay after the deploy; heap and garbage collection behavior change between V8 versions.

## Codemods

Official codemods for a framework major save time and miss things. Run them on a clean branch, read the whole diff, and expect to fix by hand what they cannot decide: dynamic usage, re-exports, wrappers and anything outside the patterns they match. Never combine a codemod with unrelated edits in the same commit.

## Peer dependencies and conflicts

npm 7 and later install peer dependencies and stop with `ERESOLVE` when ranges conflict. The error is information: two packages need incompatible versions of a third.

- First try to move both packages to versions that agree.
- If one package lags, an override (`overrides` in npm, the package manager's equivalent in pnpm or yarn) can force a version. Add a comment in the pull request with the reason, link the upstream issue and set a date to remove it.
- `--legacy-peer-deps` and `--force` hide the conflict for every package, now and in the future. Avoid them in CI and production builds.
- After any change, check for duplicate copies of packages that keep state or identity (React, framework cores, ORMs, `@types` packages): `npm ls <package>` should show one version.

## Testing an upgrade

- The full suite, type check and lint on the new versions.
- Characterization tests before the upgrade for behavior you depend on and that the changelog mentions, such as serialization formats, date handling, validation rules, error shapes and default timeouts.
- A production build and a staging deploy, not only the test run. Bundlers, native modules and runtime flags break in builds, not in tests.
- For frameworks with rendering changes, compare key pages before and after.
- Read deprecation warnings in the test and build output. Each one is a break in the next major.

## Automated updates

Renovate and Dependabot keep you close to current, which makes each upgrade small. Configure them so that speed does not become risk:

- Group minor and patch updates of related packages; keep each major as its own pull request.
- Wait before adopting new releases: Renovate `minimumReleaseAge`, Dependabot `cooldown` (Dependabot applies a three-day default to version updates). Security updates bypass the wait.
- Automerge only minor and patch updates of well-tested packages, with required checks passing. Never automerge majors, runtime upgrades or anything in authentication, cryptography, payments or the build toolchain.
- Schedule updates for working hours, so someone is around when one breaks production.
- Keep the lockfile refreshed regularly as well, so transitive updates do not pile up into one large change.

## Reviewing an upgrade pull request

- Does it change one major version of one thing (plus its required companions)?
- Are the changelog and migration notes linked, and are the breaking changes accounted for in the diff?
- Does the lockfile diff contain only the expected packages? New transitive packages, changed registries or tarball URLs, and packages from unknown publishers need an explanation.
- Are overrides, forced installs and peer conflicts documented with a reason and a removal date?
- Did CI run the full suite on the new versions, and was a production build deployed to staging?
- Is the rollback the previous lockfile and image, and does nothing in the change prevent it?

## Rolling back

Revert the pull request and redeploy the previous image built from the previous lockfile. Rebuilding from a reverted `package.json` without the old lockfile does not restore the old dependency tree. If the upgrade included a data or config migration, the rollback plan has to undo or tolerate it, and that has to be known before the upgrade ships.

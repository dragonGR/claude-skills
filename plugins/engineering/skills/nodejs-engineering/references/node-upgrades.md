# Upgrading Node

Read this before moving a service to a new Node major, when reviewing that pull request, or when choosing which Node release to run. Upgrading npm packages, lockfiles, install-script policy and Renovate or Dependabot settings are in typescript-engineering's tooling-and-supply-chain reference.

## Which release to run

Production runs a release in its LTS phase, Active or Maintenance, and moves before that release's end-of-life date. The dates are in `https://raw.githubusercontent.com/nodejs/Release/main/schedule.json` and on nodejs.org. As of October 2026: Node 20 reached end of life on 2026-04-30, Node 22 is in Maintenance until 2027-04-30, Node 24 moves to Maintenance on 2026-10-20, and Node 26 becomes LTS on 2026-10-28.

The cadence changes with Node 27. Through Node 26 there were two majors a year and only the even-numbered ones became LTS. From 27 there is one major a year, released in April, and every major becomes LTS in October after six months as Current. An Alpha channel, which may carry semver-major changes, takes over the early-testing role of the odd-numbered releases. "Run an even-numbered release" stops being a usable rule from 27 on.

## What breaks between majors

- Removed APIs and flags. Runtime deprecations become removals: `--experimental-transform-types` is gone in Node 26, so scripts that pass it fail to start. Run the test suite on the current version with `--pending-deprecation` (or `--throw-deprecation` in CI) to see what the next major removes.
- Changed defaults. `server.close()` closes idle connections since Node 19; `--permission` also blocks network access from Node 25 unless `--allow-net` is given; an unreleased change on Node's main branch raises `server.keepAliveTimeout` from 5 to 65 seconds. Print the values you depend on (`node -p "require('http').createServer().keepAliveTimeout"`) on both versions instead of trusting memory.
- OpenSSL. A newer OpenSSL rejects keys, ciphers and protocol versions the old one accepted, which shows up as TLS handshake failures against old upstreams or as `ERR_OSSL_*` errors when loading old key files.
- V8. Heap sizing and garbage collection change. Compare `heap_size_limit`, GC pauses and RSS before and after.
- The module ABI. Native addons are built for one `NODE_MODULE_VERSION` (`process.versions.modules`) and must be rebuilt or upgraded. A package may not publish prebuilt binaries for the new major yet, and then it compiles from source or fails to install; check every native dependency before you schedule the move.
- The bundled npm. Node 22 ships npm 10, Node 24 and 26 ship npm 11.19. npm 12 is installed separately and blocks dependency install scripts by default, so a CI image that runs `npm install -g npm@latest` changes install behavior without any Node change.

## Doing the upgrade

1. Start from a green build, type check, lint and test run on the current version. If the baseline is red, you cannot tell what the upgrade broke.
2. Read the changelog of every major you cross (`doc/changelogs/CHANGELOG_V<major>.md` in the Node repository), not only the target's, and list what applies to this codebase.
3. Change the version everywhere Node is installed, in one change: `engines`, `.nvmrc` or `.node-version`, CI images and matrices, the Docker base image (pinned by digest), serverless runtime settings and developer tooling. A CI version that differs from production defeats the test run.
4. Change nothing else in the same pull request. Dependency upgrades the new major forces go in their own change before or after it.
5. For a risky move, run CI on both versions for a while, then drop the old one.
6. Deploy to staging with a production build, then to production, and watch error rates, memory, event loop delay and TLS errors to upstreams.

Rolling back means redeploying the previous image. The previous Node version, the previous lockfile and the native builds made for it go back together; reverting only `.nvmrc` leaves addons compiled for the wrong ABI.

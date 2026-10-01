---
name: security-engineering
description: Security review and secure implementation for authorization, sessions, JWT, OAuth, CSRF, SSRF, injection, file handling, webhooks, secrets, CI pipelines and Kubernetes. Use when code touches auth, user input, outbound URLs, money, credentials, signatures or admin roles, or when asked for a security review, audit or threat model.
license: MIT
metadata:
  author: Alex Tsanis
---

# Security engineering

Most shipped vulnerabilities are a correct-looking check in the wrong place: authorization on the parent but not the child, a signature verified over the wrong bytes, one address validated and a different one connected to. Review by following attacker-controlled data from the entry point to the sink, and at each step ask who decides, on what state, and whether that decision still holds when the write happens. A finding needs a reachable path and has to survive an honest attempt to disprove it. Scanner output is a lead.

## How to review

1. Name the assets (funds, credentials, personal data, authority, availability) and the principals (anonymous, user, member of another tenant, admin, service, CI job, contract caller).
2. List entry points, including the forgotten ones: webhooks, queue consumers, cron jobs, uploads, import-from-URL, admin tools, GraphQL resolvers, batch and export endpoints, older API versions, contract callbacks.
3. Trace each input to its sinks (query, shell, filesystem, outbound request, template, redirect, transfer, role change). Write down where authentication, authorization and validation happen and which state they read.
4. Run the catalogue below against each path, then the second pass in [exploit-triage.md](references/exploit-triage.md) before reporting anything.

## Failure catalogue

### Authorization

**Object-level authorization missing (IDOR/BOLA).** `SELECT * FROM invoices WHERE id = $1` behind middleware that only checked membership of `:orgId`. `/orgs/A/invoices/<id from org B>` passes the check for A and returns B's row. Sequential ids make it trivial; UUIDs only slow it down. Scope every lookup by the authority you just checked, in the same query (`WHERE id = $1 AND org_id = $2`), and answer 404 outside it. Check every verb and sibling: GET gets fixed while PATCH, DELETE, export, the PDF renderer and the GraphQL `node(id)` resolver do not.

**Function-level authorization missing.** Admin or internal routes guarded only by `requireAuth`, hidden in the UI, mounted under `/internal`, or "only called by our frontend". Any logged-in user calls them. Each privileged handler checks a named permission, and the router denies routes that declare none.

**Mass assignment.** `db.update(users).set(req.body)`, `Object.assign(entity, body)`, `sql(obj)` over every key, or an input schema derived from the table model with `.partial()`. The attacker adds `role`, `orgId`, `emailVerified`, `balance`. Write one input schema per operation and caller type listing only the fields that caller may set, and reject unknown keys (`.strict()` in Zod, `extra='forbid'` in Pydantic) on privileged endpoints.

**Authorization checked outside the write (TOCTOU).** Load the row, check `row.ownerId === user.id` or `row.status === 'paid'`, await something, then `UPDATE ... WHERE id = $1`. Between check and write the membership is revoked, the row changes owner, or a concurrent request passes the same check. Put the authorization predicate and the state precondition in the write (`UPDATE ... WHERE id = $1 AND org_id = $2 AND status = 'paid' RETURNING ...`) and treat zero rows as denied or conflict, or lock with `SELECT ... FOR UPDATE` inside the transaction that writes.

**Identity or tenant taken from the request.** `X-Tenant-Id`, `userId` in the body, the subdomain, a `role` claim from an unverified token. Derive the principal from the verified session, then check membership for any tenant the request names.

### Tokens and sessions

**JWT verified loosely.** `decode` where `verify` was meant; no pinned `algorithms`, which opens HS/RS confusion (an HS256 token signed with the RSA public key) and `alg: none` on weak libraries; key selected from token-supplied `jku`, `x5u`, `jwk`, or a `kid` used as a path or query; `iss` and `aud` unchecked, so a token minted for another service or another tenant of the same IdP works; `exp` missing. Verify with an algorithm list and key source from config and require `iss`, `aud`, `exp`. `jose`'s `jwtVerify` never accepts `alg: none` and checks `exp` only when present, so add `requiredClaims: ['exp']`.

**Tokens that outlive the authority behind them.** A 30-day JWT carries `role: admin`; the user is demoted, the token keeps working. Keep access tokens to minutes, refresh against server state, and keep a per-user `session_version` that password change, role change, logout-all and disable increment, checked on refresh and on sensitive actions.

**Session fixation.** The session id from before login stays valid after it, so whoever planted it (a sibling-subdomain cookie, a shared machine) shares the logged-in session. Regenerate the id on login, privilege change and MFA completion (`req.session.regenerate` in express-session; Django's `login()` does it, hand-rolled login code often does not). Use `__Host-` cookies where possible.

**Second factor enforced by the UI.** Login marks the session authenticated, then redirects to the MFA page; the API works without it. Hold a pending state with no API authority until the factor verifies, then regenerate the session.

**OAuth and OIDC mistakes.** Authorization server matching `redirect_uri` by prefix or pattern (RFC 9700 requires exact string matching); `state` missing or not bound to the browser session; no PKCE (required for public clients, recommended for confidential ones), or `plain` instead of S256; ID token `nonce`, `iss` or `aud` unchecked; logging in or linking accounts by the `email` claim from a provider that does not assert verification or lets users edit it, which hands over existing accounts. Link identities by `(iss, sub)`.

**CSRF with cookie auth.** State-changing endpoints accept cookie-authenticated requests with no token or origin check. `SameSite=Lax` does not stop same-site attackers (a user-content or compromised subdomain), side effects on GET, or POSTs within two minutes of a cookie set without `SameSite` in browsers that default to Lax; not every browser defaults to Lax. Set `SameSite` explicitly, never change state on GET, and require a CSRF token or an allowlisted `Origin` on unsafe methods. Bearer tokens sent in a header are not ambient and need no CSRF token.

**Password reset flaws.** Link built from the request `Host` header, so a reset requested with `Host: attacker.example` mails the victim a link that delivers their token to the attacker; token derived from time or user id; token stored in plaintext; token reusable until expiry; sessions left alive after reset. Build links from a configured origin, generate at least 32 bytes from a CSPRNG, store its SHA-256 (a fast hash is right for high-entropy tokens), consume it in one conditional `UPDATE ... WHERE used_at IS NULL AND expires_at > now() RETURNING user_id`, and bump the session version.

**User enumeration.** Different messages, status codes or timing for unknown user versus wrong password on login, signup, reset and invite. The timing leak is the early return before the password hash runs. Return one response, verify against a dummy hash for unknown users, and send mail from a background job.

**Weak password storage.** SHA-256 or MD5 with or without salt; bcrypt silently ignoring bytes past 72; hashes compared with `==`. Use Argon2id (OWASP minimum 19 MiB, 2 iterations, parallelism 1) through the library's verify function; with bcrypt, reject inputs over 72 bytes.

**Rate limit keyed by the wrong identity.** `app.set('trust proxy', true)` plus an IP-keyed limiter: Express then uses the left-most `X-Forwarded-For` entry, which the client writes, so each guess gets a fresh bucket. Also: IP-only keys on login (credential stuffing from many IPs), limits on login but not on OTP, reset or GraphQL aliased and batched mutations. Set `trust proxy` to the hop count or addresses of your own proxies and key auth limits by target account as well as source.

### Untrusted input reaching sinks

**SQL injection in the parts that cannot be parameters.** Values are bound but `ORDER BY ${sort}`, table or column names, `sql.unsafe`, Prisma `$queryRawUnsafe`, Sequelize `literal` take user strings. Map identifiers through an allowlist. Tagged-template clients (postgres.js, Prisma `$queryRaw`) bind `${}` values as parameters, so that interpolation is not injection.

**NoSQL operator injection.** `users.findOne({ email, password: req.body.password })` receiving `{"password": {"$ne": null}}`. JSON bodies carry objects, and `qs`-style parsers (`express.urlencoded({ extended: true })`) turn `password[$ne]=` into one. Validate that each field is a string before it reaches the query.

**Command and argument injection.** `` exec(`convert ${name} out.png`) ``, `subprocess.run(cmd, shell=True)`, and option injection even with argv arrays when user input starts with `-` (`git`, `tar`, `curl`, `ssh`). Use argv arrays without a shell, put `--` before positional user input, and allowlist values.

**Template injection.** User text compiled as a template (`render_template_string(bio)`, Handlebars compiling user strings, `new Function` in a formula feature). Users supply data to templates, never template source.

**Header, log and CRLF injection.** User values in `Location`, `Set-Cookie`, `Content-Disposition` filenames or email headers; strings logged into a system that interprets them (Log4Shell was a JNDI lookup triggered by a logged string). Use framework header APIs that reject CR/LF, encode filenames per RFC 6266, log structured fields.

**Path traversal.** `path.join(UPLOAD_DIR, name)` with `../` or `%2e%2e%2f`; `path.resolve` and Python `os.path.join` let an absolute second argument replace the base; symlinks inside the base. Resolve the final path and check it stays inside the resolved base with a separator-aware comparison. Better, store files under generated ids and keep the user's filename as metadata.

**Zip slip and archive bombs.** Entries named `../../app/server.js`, symlink entries pointing outside, tiny archives expanding to fill the disk. Python `zipfile` strips `..` and absolute prefixes; `tarfile` defaults to the safe `data` filter only from 3.14, so pass `filter='data'`. For other libraries, check each entry's resolved target, refuse links, and cap entry count and total uncompressed bytes while streaming.

**SSRF.** Import-from-URL, customer webhook URLs, PDF and image renderers, link previews, OIDC discovery, XML external entities. Bypasses: a redirect from an allowed host to `169.254.169.254`; DNS rebinding between the check and the connect; `http://2130706433/`, `0x7f.1`, `[::ffff:127.0.0.1]`, AWS IMDS on IPv6 at `[fd00:ec2::254]`, `0.0.0.0`; validating the hostname string instead of the address. Check every resolved address against non-public ranges at connect time, connect only to a checked address, disable redirects or re-check each hop, cap response size and time. For arbitrary user URLs, send traffic through an egress proxy or network that cannot reach internal ranges. IMDSv2 and metadata headers (`Metadata-Flavor: Google`) help but do not stop an attacker who controls method and headers.

**Open redirect.** `res.redirect(req.query.next)`; `startsWith('/')` passes `//evil.example` and `/\evil.example`; `includes('example.com')` passes `example.com.evil.net`. It gives phishing links your domain and, in OAuth flows, leaks codes and tokens. Parse against your own origin and compare origins, or map an id to a stored URL.

**Unsafe deserialization.** `pickle.loads`, `yaml.load` with `yaml.Loader` or `UnsafeLoader`, Java `ObjectInputStream`, .NET `BinaryFormatter`, PHP `unserialize`, `node-serialize` on anything an attacker can influence, including cookies, cache entries and queue messages. These run code. Use JSON plus schema validation, or `yaml.safe_load`.

**XML external entities.** Parsers that resolve DTDs and external entities, common in older Java defaults and SAML stacks. Disable DTD processing; in Python parse untrusted XML with `defusedxml`.

**Prototype pollution.** Deep merge or path-set over user JSON (`merge(settings, req.body)`, `_.set(obj, req.body.path, v)`) with `__proto__` or `constructor.prototype` keys, after which `isAdmin` is truthy on every object. Validate with a schema first, reject those keys, use `Map` or `Object.create(null)` for user-keyed data.

**User uploads served from the app origin.** An SVG or HTML upload served from `app.example.com` runs script with your users' cookies. Serve user content from a separate registrable domain, set `Content-Type` from your own allowlist, send `X-Content-Type-Options: nosniff`, and `Content-Disposition: attachment` for anything not meant to render.

**CORS reflecting origins with credentials.** Echoing the request `Origin` with `Access-Control-Allow-Credentials: true`, a regex like `/example\.com$/` that also matches `evilexample.com`, or allowing `null` (sent by sandboxed iframes). Any site can then read authenticated responses. Compare against an exact configured list and send `Vary: Origin`.

### Money, races and inbound events

**Race on a limited resource.** Wallet debit, coupon or gift card redemption, invite acceptance, OTP attempt counter, stock: read, check, write as separate steps. Twenty parallel requests all pass the check. Use one conditional statement (`UPDATE ... SET remaining = remaining - 1 WHERE id = $1 AND remaining > 0`), a unique constraint such as `(coupon_id, user_id)`, or a row lock, and prove it with a concurrent test. Increment OTP attempts atomically before comparing the code.

**Amounts from the client.** Negative quantities that refund money, price or discount taken from the body, currency not matched to the stored price, floats for money. Price from your own catalogue, validate sign and bounds, integer minor units or decimals.

**Webhook verification done wrong.** HMAC computed over re-serialized JSON instead of the raw bytes, then "fixed" by removing the check; `===` or `==` comparison; the signed timestamp ignored, so a captured delivery replays forever; no dedupe by event id, so provider retries apply the effect twice; one secret shared across environments. Verify over the raw body with `crypto.timingSafeEqual` or `hmac.compare_digest`, enforce a timestamp tolerance (Stripe's library defaults to 300 seconds), insert the event id under a unique constraint in the same transaction as the effect, and refetch the object from the provider when its current state matters.

### Secrets and pipelines

**Secrets where readers should not be.** Tokens in URLs (access logs, `Referer`, history); request headers or bodies logged or sent to an error tracker; driver errors carrying connection strings returned to clients; server keys in `VITE_*` variables (or any prefix listed in Vite's `envPrefix`), which are compiled into the browser bundle; published source maps; `.env` copied into an image layer. Redact by key at the logger, return a correlation id instead of error detail, and rotate anything that shipped.

**CI running untrusted code with secrets.** `pull_request_target` or `workflow_run` jobs that check out and run the PR head with secrets and a write token; `${{ github.event.pull_request.title }}` inside `run:` (shell injection by PR title); third-party actions pinned by tag; cloud OIDC trust conditions that accept any repo, branch or `pull_request` subject.

### Smart contracts

Solidity, signed messages verified on chain, proxies, oracles, contract roles and DeFi accounting are covered by the solidity-engineering skill. Load it for any contract review; the method above (assets, principals, entry points, second-pass disproof) still applies.

## Decision rules

- **Where authorization lives.** Object access: in the query or write, scoped by the checked principal. Role and permission rules: one policy function the handler calls explicitly. Middleware that never sees the object cannot authorize access to it.
- **Sessions or JWTs.** One backend and a need for instant revocation: server-side sessions. Several services verifying: access JWTs measured in minutes plus a refresh checked against server state. A long-lived JWT with no revocation is never acceptable for privileged roles.
- **CSRF.** Cookie auth with unsafe methods: explicit `SameSite` plus a token or `Origin` allowlist. Header bearer tokens: no token needed, keep CORS closed.
- **SSRF.** Fixed partner hosts: exact host allowlist and still check the resolved address. Arbitrary user URLs: egress proxy or isolated network with no route to internal ranges, plus the in-process check.
- **Deserialization.** Never a language-native format across a trust boundary.
- **Exposed secret.** Rotate first. Deleting it from git history does not un-expose it.
- **Guard or remove.** If a privileged path has no current user, delete it instead of guarding it.

## Reporting

A finding states entry point, attacker prerequisites, the path with file and line, the property broken, concrete impact and the fix direction, and says what you checked to try to disprove it. Label it confirmed, plausible (name the runtime evidence that would settle it) or drop it. Severity follows impact and reachability, not how alarming the code looks. Procedure, false-positive list and a worked write-up: [exploit-triage.md](references/exploit-triage.md).

## Review checklist

Answer yes or no before calling a security-relevant change done.

- Does every object lookup and write include the authorized tenant or owner in the query itself?
- Does every privileged handler check a named permission, including export, batch, GraphQL and older-version routes?
- Does each write endpoint parse input with an operation-specific schema that cannot set role, owner, tenant, status or balance?
- Are authorization and state preconditions part of the conditional write, with zero rows handled?
- Are JWT algorithms, issuer, audience and expiry pinned from config, and can a role change or password reset revoke existing tokens?
- Is the session id regenerated on login and privilege change, and is MFA enforced server-side?
- Do cookie-authenticated unsafe methods require a CSRF token or allowlisted `Origin`?
- Are reset tokens random, hashed at rest, single-use, expiring, and are links built from configured origins?
- Do login, signup and reset give identical responses and similar timing for unknown accounts?
- Are rate limits keyed by an identity the client cannot forge, including per target account?
- Is every identifier in dynamic SQL allowlisted, and does no user string reach a raw or unsafe query API, shell, template compiler or deserializer?
- Do outbound fetches of user-supplied URLs check the connected address, handle redirects and cap size and time?
- Are file paths and archive entries confined to the base after resolution, with size caps?
- Do redirects accept only same-origin paths or mapped ids?
- Does CORS allow credentials only for exact configured origins?
- Are balance, coupon, OTP and stock changes single conditional statements or locked, with a concurrent test?
- Do webhooks verify raw bytes in constant time, bound the timestamp and dedupe by event id in the effect's transaction?
- Are secrets absent from URLs, logs, error responses, client bundles, images and CI logs?
- Do CI jobs with secrets run only trusted code, with actions pinned by SHA and OIDC trust scoped to repo and environment?
- Did every finding survive the second pass, and does each fix have a test that fails without it?

## References

- [authz-and-sessions.md](references/authz-and-sessions.md): read when writing or reviewing authorization, JWT verification, sessions, OAuth login, CSRF, password reset, login and rate limiting. Before/after code.
- [untrusted-input.md](references/untrusted-input.md): read when user input reaches SQL, NoSQL, shell, templates, the filesystem, archives, outbound HTTP, redirects, CORS or parsers. Guarded fetch, safe path join, redirect validation.
- [webhooks-and-races.md](references/webhooks-and-races.md): read when receiving signed webhooks or changing balances, coupons, OTP counters or other limited resources. Verification template and atomic patterns.
- [secrets-and-ci.md](references/secrets-and-ci.md): read when secrets are created, logged, shipped to clients or used in CI, or when a secret has leaked.
- [kubernetes.md](references/kubernetes.md): read when reviewing pod security, RBAC, network policy, admission or workload identity.
- [exploit-triage.md](references/exploit-triage.md): read before reporting any finding. Second-pass disproof procedure, common false positives, severity, worked write-up, safe reproduction.

Related skills: backend-architecture for idempotency and unknown outcomes, database-engineering for isolation and locking, infrastructure-ops for deployment and CI plumbing, frontend-engineering for browser-side trust boundaries, test-engineering for adversarial and concurrency tests, solidity-engineering for smart contracts.

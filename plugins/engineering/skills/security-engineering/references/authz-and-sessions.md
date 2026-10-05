# Authorization, tokens and sessions

Read this when writing or reviewing object access, role checks, JWT verification, session handling, OAuth login, CSRF defences, password reset, login or rate limiting. Examples are TypeScript with Express and postgres.js; the patterns carry to other stacks.

## Object-level authorization

The check and the lookup must use the same authority. Middleware that verified membership of `:orgId` proves nothing about `:invoiceId`.

```ts
// Before: membership checked for :orgId, invoice fetched by id alone.
app.get('/orgs/:orgId/invoices/:invoiceId', requireAuth, requireOrgMember, async (req, res) => {
  const [invoice] = await sql`SELECT * FROM invoices WHERE id = ${req.params.invoiceId}`;
  if (!invoice) return res.status(404).end();
  res.json(invoice);
});

// After: the org that was authorized is part of the query.
app.get('/orgs/:orgId/invoices/:invoiceId', requireAuth, requireOrgMember, async (req, res) => {
  const [invoice] = await sql`
    SELECT id, number, status, amount_cents, currency, issued_at
    FROM invoices
    WHERE id = ${req.params.invoiceId} AND org_id = ${req.member.orgId}`;
  if (!invoice) return res.status(404).end();
  res.json(invoice);
});
```

`req.member.orgId` comes from the membership row the middleware loaded, not from the URL again. Selecting named columns also stops internal fields leaking when someone adds a column later.

For systems with many tables, a repository layer that takes the principal and always adds the scope is less error-prone than remembering the predicate in every handler. PostgreSQL row-level security is a second layer worth having, but only if the application connects with a role that does not bypass it and sets the tenant per transaction; check both before counting it as a control.

Sweep for siblings once one route is fixed: every verb on the same resource, nested routes that take a child id, exports, file downloads, background jobs triggered by id, GraphQL `node`/`nodes` resolvers and field resolvers that load related objects, and older API versions still routed.

## Function-level authorization

```ts
type Permission = 'billing:refund' | 'members:manage' | 'org:delete';

function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.member || !can(req.member.role, permission)) return res.status(403).end();
    next();
  };
}
```

`can` is a single table from role to permissions in one module. Handlers declare what they need; a router test enumerates registered routes and fails when a non-public route has no permission declared.

## Mass assignment

```ts
// Before: the schema mirrors the table, so `role` is settable by any member.
const Member = z.object({
  displayName: z.string().max(NAME_MAX),
  title: z.string().max(TITLE_MAX).nullable(),
  role: z.enum(ROLES),
});
app.patch('/orgs/:orgId/members/me', requireAuth, requireOrgMember, async (req, res) => {
  const patch = Member.partial().parse(req.body);
  await sql`UPDATE members SET ${sql(patch)} WHERE org_id = ${req.member.orgId} AND user_id = ${req.user.id}`;
  res.status(204).end();
});

// After: one schema per operation and caller.
const UpdateOwnMembership = z
  .object({
    displayName: z.string().trim().min(1).max(NAME_MAX),
    title: z.string().max(TITLE_MAX).nullable(),
  })
  .partial()
  .strict();

const ChangeMemberRole = z.object({ role: z.enum(ROLES) }).strict();
```

The role change gets its own endpoint behind `requirePermission('members:manage')`, plus the rules people forget: an admin cannot demote the last admin, and cannot grant a role above their own. Zod strips unknown keys by default, which silently hides probing; `.strict()` turns it into a 400 you can alert on.

In Pydantic v2, `model_config = ConfigDict(extra='forbid')`. In Rails, `params.require(:member).permit(:display_name, :title)`. In ORMs with `update(req.body)`, the fix is the same: pass an object built from the validated schema.

## Authorization inside the write

```ts
// Before: role and status checked, then a separate unconditional write.
const [inv] = await sql`SELECT * FROM invoices WHERE id = ${id} AND org_id = ${orgId}`;
if (!inv || inv.status !== 'paid') return res.status(409).end();
await sql`UPDATE org_credit SET balance_cents = balance_cents + ${inv.amount_cents} WHERE org_id = ${orgId}`;
await sql`UPDATE invoices SET status = 'credited' WHERE id = ${id}`;

// After: the transition claims the row; only the winner applies the effect.
await sql.begin(async (tx) => {
  const [inv] = await tx`
    UPDATE invoices SET status = 'credited', credited_by = ${req.user.id}
    WHERE id = ${id} AND org_id = ${orgId} AND status = 'paid'
    RETURNING amount_cents`;
  if (!inv) throw new ConflictError('invoice not refundable');
  await tx`UPDATE org_credit SET balance_cents = balance_cents + ${inv.amount_cents} WHERE org_id = ${orgId}`;
});
```

When the permission itself can change mid-request (membership revoked while a long operation runs), re-read it inside the transaction with `SELECT ... FOR SHARE` on the membership row, or include it as a join in the conditional write.

## JWT verification

```ts
import { createRemoteJWKSet, jwtVerify } from 'jose';

const jwks = createRemoteJWKSet(new URL(config.auth.jwksUrl));

export async function authenticate(token: string): Promise<Principal> {
  const { payload } = await jwtVerify(token, jwks, {
    algorithms: config.auth.algorithms,
    issuer: config.auth.issuer,
    audience: config.auth.audience,
    requiredClaims: ['exp', 'sub'],
  });
  const user = await users.findById(payload.sub);
  if (!user || user.disabledAt || user.sessionVersion !== payload.sv) throw new UnauthorizedError();
  return { userId: user.id, role: user.role };
}
```

Points that matter:

- The JWKS URL, algorithms, issuer and audience come from config. Never fetch keys from a URL in the token header.
- `kid` is only a lookup key into the key set you trust. If your own code resolves `kid`, treat it as untrusted input (no file paths, no SQL).
- Role comes from the database, not the token, for anything privileged. If you must trust a role claim for latency reasons, keep the token lifetime short enough that a demotion taking effect after it is acceptable, and say so.
- `sv` (session version) is a claim you mint. Increment the stored value on password change, role change, logout-all, MFA reset and account disable.
- Refresh tokens are stored hashed, rotated on use, and a reused old refresh token revokes the whole family.

## Session handling

```ts
app.post('/login', loginLimiter, async (req, res, next) => {
  const { email, password } = LoginInput.parse(req.body);
  const user = await users.findByEmail(email);
  const ok = await argon2.verify(user?.passwordHash ?? config.auth.dummyHash, password);
  if (!user || !ok) return res.status(401).json({ error: 'invalid_credentials' });

  req.session.regenerate((err) => {
    if (err) return next(err);
    req.session.userId = user.id;
    req.session.sessionVersion = user.sessionVersion;
    req.session.mfaPending = user.mfaEnabled;
    res.status(204).end();
  });
});
```

`config.auth.dummyHash` is an Argon2id hash of a random value generated with the same parameters as real hashes, so unknown and known users cost the same. Authorization middleware rejects sessions with `mfaPending` for everything except the MFA endpoints, and the MFA success handler regenerates the session again.

Cookie settings: `__Host-` prefix (requires `Secure`, `Path=/`, no `Domain`), `HttpOnly`, explicit `SameSite`, and an idle plus absolute expiry enforced on the server, not only via cookie `Max-Age`.

## OAuth and OIDC login (client side)

1. Start: generate `state`, `nonce` and a PKCE `code_verifier` from a CSPRNG; store them in the server session with the intended return path; send `code_challenge = BASE64URL(SHA256(verifier))` with `code_challenge_method=S256`.
2. Callback: require `state` equal to the stored value, then delete the stored value so it is single-use. Reject if missing. With more than one provider configured, also store which issuer the flow started with and require the response `iss` parameter (RFC 9207) to equal it before exchanging the code; for providers that do not send `iss`, give each one its own redirect URI. Without this mix-up defence, which RFC 9700 requires for multi-provider clients, a malicious or compromised provider can obtain a code the honest one issued.
3. Exchange the code with the stored verifier at the token endpoint from discovery or config.
4. Verify the ID token: signature with the issuer's keys, `iss` equal to the configured issuer, `aud` containing your client id, `exp`, and `nonce` equal to the stored one.
5. Identify the user by `(iss, sub)`. Only use `email` to link to an existing account when the provider asserts `email_verified` and you trust that provider for the email's domain; otherwise require the user to log in to the existing account and link explicitly.
6. Redirect to the stored return path after validating it as a same-origin path (see untrusted-input.md).

```ts
const verifier = randomBytes(PKCE_VERIFIER_BYTES).toString('base64url');
const challenge = createHash('sha256').update(verifier).digest('base64url');
```

On an authorization server you operate: exact `redirect_uri` string match against registered values (RFC 9700 allows only localhost port variation for native apps), no implicit grant, no password grant, codes single-use and short-lived, reject a `code_verifier` when the authorization request had no challenge.

## CSRF

```ts
const UNSAFE_METHODS = new Set(['POST', 'PUT', 'PATCH', 'DELETE']);

export function requireTrustedOrigin(req: Request, res: Response, next: NextFunction) {
  if (!UNSAFE_METHODS.has(req.method)) return next();
  const origin = req.get('Origin');
  if (origin && config.web.allowedOrigins.has(origin)) return next();
  return res.status(403).json({ error: 'origin_not_allowed' });
}
```

Use this for cookie-authenticated APIs called by your own frontend. Server-rendered form apps use the framework's synchronizer token. Neither replaces `SameSite`; together they cover same-site subdomain attackers that `SameSite` misses. A GET that changes state (`/unsubscribe?id=`, `/approve?token=`) is outside every CSRF defence: make it a POST behind a confirmation page.

## Password reset

```ts
app.post('/password/forgot', resetLimiter, async (req, res) => {
  const { email } = ForgotInput.parse(req.body);
  await jobs.enqueue('password-reset-request', { email });
  res.status(202).end();
});

// Job: runs off the request path so timing does not reveal account existence.
async function handleResetRequest({ email }: { email: string }) {
  const user = await users.findByEmail(email);
  if (!user) return;
  const token = randomBytes(RESET_TOKEN_BYTES).toString('base64url');
  await sql`
    INSERT INTO password_resets (user_id, token_hash, expires_at)
    VALUES (${user.id}, ${sha256Hex(token)}, now() + ${config.auth.resetTtl}::interval)`;
  const link = new URL('/reset', config.web.publicOrigin);
  link.searchParams.set('token', token);
  await mailer.sendReset(user.email, link.toString());
}

app.post('/password/reset', resetLimiter, async (req, res) => {
  const { token, newPassword } = ResetInput.parse(req.body);
  const hash = await argon2.hash(newPassword);
  const done = await sql.begin(async (tx) => {
    const [row] = await tx`
      UPDATE password_resets SET used_at = now()
      WHERE token_hash = ${sha256Hex(token)} AND used_at IS NULL AND expires_at > now()
      RETURNING user_id`;
    if (!row) return false;
    await tx`
      UPDATE users SET password_hash = ${hash}, session_version = session_version + 1
      WHERE id = ${row.user_id}`;
    await tx`UPDATE password_resets SET used_at = now() WHERE user_id = ${row.user_id} AND used_at IS NULL`;
    return true;
  });
  if (!done) return res.status(400).json({ error: 'invalid_or_expired' });
  res.status(204).end();
});
```

The reset page should not load third-party scripts while the token is in the URL, and should send `Referrer-Policy: no-referrer`, since the token otherwise leaks through `Referer`.

## Rate limiting

- Behind one load balancer: `app.set('trust proxy', 1)`. Behind a known chain: the proxy addresses or subnets. `true` makes `req.ip` whatever the client wrote first in `X-Forwarded-For`.
- Key login, OTP and reset limits on both source and target account (`login:acct:<normalized email>`), and make the account limit lock progressively rather than permanently, so an attacker cannot lock out a victim at will.
- Limit in a shared store (Redis, database) when you run more than one instance; per-process counters multiply by replica count.
- GraphQL: count operations, not HTTP requests. Aliases and batched queries put hundreds of login or OTP attempts in one request.
- OTP: bound total attempts per code, invalidate the code after the bound, and increment the attempt counter atomically before comparing.

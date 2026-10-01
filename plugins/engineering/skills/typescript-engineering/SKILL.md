---
name: typescript-engineering
description: TypeScript and Node.js: runtime validation, promises and async, fetch timeouts, bigint and money, dates, tsconfig strictness, ESM/CJS, npm supply chain, Cloudflare Workers. Load it before writing or reviewing any TS or JS service, route, script or library, even a short snippet, and when debugging unhandled rejections, leaks, hangs or wrong numbers.
license: MIT
metadata:
  author: Alex Tsanis
---

# TypeScript engineering

TypeScript types record what the author expected, and the runtime never sees them. The expensive bugs live where the compiler has nothing to say: JSON typed with an annotation, a promise nobody awaits, an id that lost its last digits in `JSON.parse`, a module variable shared by every request. Everything in the catalogue below compiles cleanly and passes a happy-path test.

For React, Vite and React Router architecture also load frontend-engineering.

## Before you judge or change anything

- Runtime and version: `engines`, `.nvmrc`, the Docker base image, `wrangler.jsonc`/`wrangler.toml`, Bun or Deno config. Globals, module loading and what survives past a response differ between Node, Workers, Bun, Deno and browsers.
- Who type-checks. esbuild, swc, Vite, tsx, Bun and Node's built-in type stripping all erase types without checking them. Find the `tsc --noEmit` or `tsc -b` step in CI. If there isn't one, type errors ship.
- The flags actually in effect, following `extends`. `strict` does not turn on `noUncheckedIndexedAccess` or `exactOptionalPropertyTypes`.
- Module format: `"type"` in `package.json`, `module`/`moduleResolution`, and the `exports` map of packages involved.
- Library majors where advice diverges: Zod 3 or 4, Express 4 or 5, the ORM and its version.
- For a claimed bug: the input or interleaving that triggers it, and whether a schema, DB constraint, lock or later check already stops it. No reachable path, no finding.

## Failure catalogue

### Trust boundaries

**Annotating untrusted data.** `JSON.parse` and Express's `req.body` are `any` (so is `res.json()` under the DOM lib), so `const body: Transfer = req.body` compiles with no cast and checks nothing. Missing fields, strings where numbers belong and attacker-chosen extras all flow on. Parse at the boundary with a schema and infer the type from it. The same applies to webhooks, queue messages, `postMessage`, `localStorage`, cookies and your own database's JSON columns.

```ts
// Before: `amount` may be a float or a string; `userId` is whatever the client sent
const body: TransferRequest = req.body;
await transfers.create(body);

// After (Zod 4)
const TransferRequest = z.strictObject({
  toAccountId: z.uuid(),
  amountMinor: z.string().regex(DECIMAL_INTEGER).transform(BigInt),
  idempotencyKey: z.uuid(),
});

const parsed = TransferRequest.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ error: "invalid_request" });
await transfers.create({ ...parsed.data, userId: session.userId });
```

**Validating, then using the raw object.** `z.object` strips unknown keys from its output in Zod 3 and 4, so the usual mass-assignment bug is code that parses and then writes `req.body`, or schemas built with `.passthrough()`, `z.looseObject` or `z.record(z.string(), z.any())`. On privileged writes (roles, prices, ownership, plan, limits) use `z.strictObject` (Zod 4; `.strict()` in Zod 3, legacy in 4) so a probe with `role` fails loudly, write only `parsed.data`, and keep identity and tenant fields out of client schemas entirely.

**Coercion that invents values.** `z.coerce.number()` calls `Number(input)`, so `""` becomes `0`. `z.coerce.boolean()` turns `"false"` into `true`, as does `Boolean(process.env.FLAG)`. `parseInt("10abc")` is `10`. For query strings use `z.string().regex(DIGITS).transform(Number)` with bounds, and for boolean strings `z.stringbool()` (Zod 4) or `z.enum(["true", "false"])`.

**Configuration read where it is used.** `process.env.JWT_SECRET!`, `?? "dev-secret"`, `Number(process.env.TIMEOUT_MS)` (NaN when unset), `if (process.env.SKIP_AUTH)` (true for `"false"`). Parse the environment once at startup, fail to boot on anything missing, and export a frozen typed object. `z.object` rather than strict here, because the environment carries many unrelated keys. Workers get configuration from the `env` binding instead (`references/runtimes.md`).

### Promises and async

**Floating promises.** `audit.record(event)` with no `await`, or `.then()` with no rejection handler. In Node an unhandled rejection is raised as an uncaught exception by default, so one failed email kills every in-flight request on the process. On Workers the runtime may cancel the work when the invocation ends. Await it, return it, or give it an owner (a tracked set drained on shutdown, `ctx.waitUntil`, a durable queue). Turn on `@typescript-eslint/no-floating-promises` and `no-misused-promises`; both are in `recommended-type-checked` and need type information.

**Async functions where the caller ignores the result.** `forEach(async ...)` returns before anything finishes and loses every rejection. `filter(async ...)` keeps every element, because a promise is truthy. An async guard used without `await` always passes:

```ts
// Before: canEdit is async; a Promise is truthy, so nobody is ever refused
if (!canEdit(user, doc)) throw new ForbiddenError();

// After
if (!(await canEdit(user, doc))) throw new ForbiddenError();
```

Event listeners and Express 4 handlers declared `async` have the same shape: nothing awaits them, so a rejection is unhandled. Express 5 forwards rejected promises from handlers to `next`; Express 4 needs `.catch(next)` or a wrapper.

**`Promise.all` treated as a transaction.** It rejects on the first failure and the other operations keep running. Cleanup that starts in the `catch` races writes that land afterwards, which is how provisioning code leaves orphaned resources. When each outcome matters, use `Promise.allSettled` and compensate only after everything settled; when siblings should stop, share an `AbortController` and abort it on first failure.

**No deadline on outbound calls.** Node's `fetch` (undici) waits up to 300 seconds for headers and 300 seconds between body chunks by default. Pass `signal: AbortSignal.timeout(config.upstreamTimeoutMs)`, combined with the incoming request's signal through `AbortSignal.any` when there is one. A timed-out write is an unknown outcome: the server may have applied it, so a retry needs an idempotency key (backend-architecture). `fetch` also resolves on 4xx and 5xx, so check `res.ok`, and consume or cancel every body you don't read; undici leaves the connection to the garbage collector otherwise and the pool can stall.

**`Promise.race` as a timeout.** The losing operation keeps running and holding its socket, and unless the code clears the timer, every call leaves one pending until it expires. Cancel through an `AbortSignal` the operation honours.

**`return promise` inside `try`.** Without `await`, the rejection happens after the function has left the `try`: the `catch` never sees it, and `finally` releases the lock or connection while the operation is still running. Write `return await` inside `try` blocks (`@typescript-eslint/return-await`, default `in-try-catch`).

**Check-then-act across `await`.** Anything read before an `await` can be stale after it. Two requests that both read `remaining = 1` both redeem. Make the check and the write one atomic statement and branch on the affected row count:

```ts
// Before
const coupon = await db.coupon.findUniqueOrThrow({ where: { code } });
if (coupon.remaining < 1) throw new ConflictError("coupon exhausted");
await db.coupon.update({ where: { code }, data: { remaining: coupon.remaining - 1 } });

// After (Prisma)
const { count } = await db.coupon.updateMany({
  where: { code, remaining: { gte: 1 } },
  data: { remaining: { decrement: 1 } },
});
if (count === 0) throw new ConflictError("coupon exhausted");
```

In-process, the same bug is a cache stampede: store the in-flight promise in the map, not the resolved value, and delete it on rejection.

**Request state in module scope.** `let currentUser` set by middleware, a singleton with `setTenant()`, a module-level array collecting per-request data. Request B overwrites it while A is suspended at an `await`, and A acts as B. Workers reuse isolates across requests, so the same leak happens there. Pass context as arguments; use `AsyncLocalStorage` for logging and tracing context. A module-level rate limiter, dedupe set or lock also exists once per process, so it multiplies with instances and resets on deploy.

**Errors from `catch`.** Under `strict`, `e` is `unknown`, and `(e as Error).message` is `undefined` for a thrown string and throws inside the handler when `null` or `undefined` was thrown. `e instanceof Error` is false for errors from another realm (`vm`, iframes), and `err instanceof HttpError` or `instanceof ZodError` is false for an error thrown by a second copy of the same package, so the mapping silently turns a 400 into a 500. Branch on a `code` or `name` field or the library's own guard, wrap with `new Error(msg, { cause })`, and never send `e.message` to a client. `process.on("uncaughtException")` is for synchronous cleanup before exiting; Node's docs say resuming is not safe. An `EventEmitter` that emits `'error'` with no listener throws and exits the process.

### Resources and the event loop

**Listener, interval and timer leaks.** A `setInterval` with no `clearInterval`, `emitter.on` per request on a long-lived emitter (the symptom is `MaxListenersExceededWarning` past 10 listeners), listeners added to a long-lived `AbortSignal`. Each needs an owner that removes it. Use `{ signal }` on `addEventListener`, `once` where one event is enough, and `timer.unref()` for background timers that should not hold the process open at shutdown.

**Long `setTimeout` delays.** A delay above 2147483647 ms (about 24.8 days) is set to 1 ms. `setTimeout(refresh, expiresAt - Date.now())` for a 30-day token fires immediately, and if the callback reschedules, it spins. Clamp and re-arm, or hand long schedules to a job scheduler.

**Ignored backpressure.** `readable.on("data", (c) => out.write(c))` ignores `write()` returning `false`, and Node buffers until the process runs out of memory. `.pipe()` does not close the destination when the source errors. Use `pipeline` from `node:stream/promises`, which propagates errors and destroys every stream:

```ts
// Before: a client disconnect or read error leaks file descriptors
fs.createReadStream(file).pipe(zlib.createGzip()).pipe(res);

// After
await pipeline(fs.createReadStream(file), zlib.createGzip(), res);
```

**Blocking the event loop.** `pbkdf2Sync`, `scryptSync`, `readFileSync` in a handler, `JSON.parse` of an unbounded body, a catastrophic regex. Every request on the process waits and health checks fail. Use the async APIs, a worker thread for CPU-bound work, and body size limits.

### Numbers, money and time

**Integers above 2^53.** `JSON.parse` rounds `9007199254740993` to `...992` without an error. Snowflake ids, Postgres `int8` and token amounts in wei break this way. node-postgres returns `int8` as a string on purpose; `setTypeParser(20, parseInt)` reintroduces the corruption. Keep such values as strings or `bigint` end to end, and send them over JSON as decimal strings, because `JSON.stringify` throws a `TypeError` on `bigint`. Mixing `bigint` and `number` in arithmetic throws; `Number(big)` silently rounds. If a producer sends a bare JSON number above 2^53, the value is already wrong after parsing: fix the producer.

**Float money.** `0.1 + 0.2 !== 0.3`, `Math.round(1.005 * 100)` is `100`, `(1.005).toFixed(2)` is `"1.00"`. Money is integer minor units (`bigint` when totals can pass 2^53) or a decimal library, never `number` with a decimal point, and split amounts allocate the remainder so parts sum to the total.

**Date parsing.** `new Date("2024-03-10")` is UTC midnight, `new Date("2024-03-10T00:00")` is local midnight, and non-ISO strings are implementation-specific. On a server west of UTC the first one's `getDate()` is the 9th. Invalid input gives an Invalid Date that throws only later, in `toISOString()`. Instants travel as ISO strings with `Z` or an offset (`z.iso.datetime()` in Zod 4 accepts only `Z` unless `offset: true`); calendar dates stay `YYYY-MM-DD` strings; display goes through `Intl.DateTimeFormat` with an explicit `timeZone`. Details in `references/numbers-money-time.md`.

### Types that hide bugs

**Truthiness on numbers and strings.** `if (!amount)`, `limit || DEFAULT_LIMIT`, `discount && applyDiscount()`: a legitimate `0` or `""` takes the "missing" branch. Use `??` and explicit `=== undefined` checks. `strict-boolean-expressions` flags the rest only with `allowNumber: false` and `allowString: false`; its defaults allow plain numbers and strings.

**Optional chaining that fails open.** `?.` turns "this should never be missing" into `undefined`, and the comparison decides which way that goes.

```ts
// Before: a missing or unloaded subscription grants access
if (account.subscription?.status === "canceled") throw new PaymentRequiredError();

// After: only an explicit active state grants access
if (account.subscription?.status !== "active") throw new PaymentRequiredError();
```

Use `?.` where absence is a legitimate state. Where absence is a bug, throw.

**`undefined` in ORM filters.** Prisma treats a `where` field set to `undefined` as no filter, so `deleteMany({ where: { tenantId: input.tenantId } })` with a missing `tenantId` deletes every tenant's rows. Validate required filter values before the query, or enable the `strictUndefinedChecks` preview and use `Prisma.skip` for intentional omission. Check how your ORM or driver treats `undefined` before trusting an optional value in a filter.

**Spreads that overwrite defaults with `undefined`.** `{ ...DEFAULTS, ...overrides }` where `overrides.timeoutMs` is present but `undefined` wipes the default. `exactOptionalPropertyTypes` makes `timeoutMs?: number` reject an explicit `undefined`.

**Unchecked indexing.** `rows[0].id` and `byId[key].total` are typed as defined. `noUncheckedIndexedAccess` adds `| undefined` and forces the empty case.

**Non-exhaustive `switch`.** A new union member falls into `default` or off the end and returns `undefined`. End with `const unhandled: never = value; throw new Error(...)`, since data from a database or API can carry a value the type does not know. Use `satisfies Record<Status, Handler>` for lookup tables.

**Enums.** Numeric enums have reverse mappings, so `Object.values(Role)` returns names and numbers; an allowlist built from it contains strings you never meant. `const enum` values are inlined and go stale when a dependency changes, and `isolatedModules` rejects references to a `const enum` declared in a `.d.ts`. Node's type stripping and `erasableSyntaxOnly` reject enums outright. For new code use an `as const` object and a union type.

### Injection and platform security

**`child_process.exec` with input.** `exec`, `execSync` and `spawn(..., { shell: true })` hand the string to a shell. Use `execFile` or `spawn` with an argument array, end options before user values (`--`, or `--end-of-options` for git revisions), and set `timeout` and `maxBuffer`.

**Path traversal.** `path.join(root, "../../etc/passwd")` normalizes to a path outside `root`; `path.resolve(root, "/etc/passwd")` ignores `root`; `target.startsWith(root)` accepts `/srv/uploads-evil`. Resolve, then check `path.relative`. Better, look files up by id with an ownership check.

**Prototype pollution.** Recursive merge or `set(obj, path, value)` over user-controlled keys reaches `__proto__` or `constructor.prototype`. `Object.assign(target, JSON.parse(body))` with a `"__proto__"` key replaces `target`'s prototype. Lookups are the other half: `handlers[req.query.type]` with `type=constructor` returns a function from `Object.prototype`. Validate shape first, use `Map` or `Object.create(null)` for user-keyed data, and check `Object.hasOwn` before indexing.

**ReDoS.** Nested or overlapping quantifiers (`(a+)+`, `(\w|\d)+$`, hand-written email patterns) over untrusted input pin the event loop on one request. Cap the input length before matching, rewrite the pattern, or use a linear-time engine such as the `re2` package.

**Secret comparison and randomness.** Compare HMACs and tokens with `timingSafeEqual`, which throws a `RangeError` when lengths differ; with attacker-controlled input, check lengths first or compare fixed-length digests. Verify webhook signatures over the raw body bytes, not `JSON.stringify(req.body)`. `Math.random` is predictable; tokens, ids and codes come from `crypto.randomBytes`, `randomUUID`, `getRandomValues` or `randomInt`.

### Builds, modules and dependencies

**Types never checked.** See the first section: a green build under esbuild, swc, Vite or Node type stripping says nothing about types.

**Dual package hazard.** A package that ships both CJS and ESM can load twice in one process when one caller imports and another requires it. Two instances mean two registries, two caches, two configs, and `instanceof` failing across them. Check `npm ls <pkg>` for duplicate copies and log the resolved path from each import site, pick one format per app, and for libraries prefer ESM-only now that `require(esm)` works unflagged (Node 20.19+ and 22.12+, for modules without top-level `await`).

**Install scripts and lockfiles.** CI installs with `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable` or `bun install --frozen-lockfile`, never a plain install that can rewrite the lock. Dependency `postinstall` scripts run with the developer's or CI runner's credentials; npm runs them by default, recent pnpm and Bun only for allowlisted packages. Review lockfile diffs for `resolved` URLs off the registry and for new transitive packages. Details in `references/tooling-and-supply-chain.md`.

**Runtime mismatch.** Code written for Node deployed to Workers or another edge runtime: module state shared across requests, promises cancelled after the response unless passed to `ctx.waitUntil`, Node built-ins missing when the compatibility date and flags leave Node compatibility off, bodies buffered past the memory limit. See `references/runtimes.md`.

## Decision rules

- Unknown keys: strict schemas for anything you persist or that grants authority; default strip for third-party responses, validating only the fields you read, so an added field upstream does not break you. Loose objects only for pass-through proxies that never interpret the data.
- Concurrency: `await` in a `for...of` loop when order matters or the input is small; bounded concurrency (`references/async-and-lifetimes.md`) for data-sized input; `Promise.all` only for all-or-nothing work whose siblings are reads or abortable; `allSettled` when each outcome is reported or compensated.
- Numeric representation: `number` only when the producer guarantees values below 2^53 (an `int4` column, a counter); `bigint` when you do arithmetic on 64-bit or token-sized values; a string when it is an identifier.
- Money: integer minor units for fixed-exponent currencies that only add and subtract; a decimal library with explicit rounding points when rates, percentages or proration are involved.
- Errors: throw for defects and unexpected states; return a typed result for expected domain outcomes (not found, conflict, declined) the caller must branch on.
- Request context: explicit parameters for anything that authorizes an action; `AsyncLocalStorage` for logging and tracing.
- Enums: `as const` object plus union type. Keep an existing enum rather than churning a codebase, but never add `const enum` to a published package.

## Review checklist

- Is every value from the network, disk, environment, storage or `JSON.parse` parsed by a schema before use, with the type inferred from it?
- Do privileged writes use strict schemas and only the parsed output, with user, tenant and role taken from the session?
- Does any coercion turn `""`, `"false"` or partial numbers into valid-looking values?
- Is configuration parsed once at startup, with no defaults for secrets or security switches?
- Is every promise awaited, returned or owned, and are there no async callbacks passed to `forEach`, `filter`, conditionals or Express 4 handlers?
- Does every outbound call have a timeout signal, a `res.ok` check and a consumed or cancelled body, and is a timed-out write treated as an unknown outcome?
- Does any read-then-write span an `await` without an atomic statement, constraint or lock behind it?
- Is any request-scoped data stored in module scope or a singleton?
- Does every `catch` handle a non-`Error` value, preserve `cause` and keep internals out of responses?
- Is every interval, listener, stream and timer released by an owner, and are streams joined with `pipeline`?
- Are ids and amounts that can exceed 2^53 strings or `bigint` from the database to the JSON response?
- Is money free of fractional `number` arithmetic?
- Are dates parsed from ISO strings with an explicit offset, and formatted with an explicit time zone?
- Could `0`, `""` or a missing object take a permissive branch through `||`, `!` or `?.`?
- Can any ORM filter value be `undefined`?
- Does every `switch` over a union end in a `never` check that throws?
- Is untrusted input kept out of shells, file paths, object keys, merges and regexes?
- Are secrets compared in constant time with length handled, and generated with `crypto`?
- Does CI run `tsc --noEmit` and type-aware lint, and install from the lockfile?
- Does the code match the runtime it deploys to?

## References

- `references/async-and-lifetimes.md`: read when writing or reviewing promise-heavy code, fan-out, timeouts, cancellation, streams, timers or graceful shutdown.
- `references/boundaries.md`: read when code handles untrusted input: Zod patterns, environment parsing, webhooks, shell, file paths, prototype pollution, regexes, crypto.
- `references/numbers-money-time.md`: read for ids, token amounts, bigint serialization, money, rounding, dates and time zones.
- `references/tooling-and-supply-chain.md`: read when touching `tsconfig.json`, lint config, module format, package publishing, lockfiles or new dependencies.
- `references/runtimes.md`: read when the code runs on Cloudflare Workers, another edge runtime, Bun, Deno or Node's type stripping.

Related skills: nodejs-engineering for the Node runtime in production (shutdown, server timeouts, memory, thread pool, debugging, upgrades); security-engineering for authorization and threat modelling; backend-architecture for idempotency, retries and reconciliation; database-engineering for transactions and locking; frontend-engineering for React and Vite; solidity-engineering for contracts and Ethereum hashing; test-engineering for test strategy; performance-benchmarking before optimizing.

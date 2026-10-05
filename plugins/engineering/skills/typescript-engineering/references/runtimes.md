# Runtime differences that break TypeScript code

Read this when code runs somewhere other than a long-lived Node process: Cloudflare Workers or another edge runtime, Node with built-in type stripping, Bun, or the browser, or when code is shared between them.

Code moved between runtimes usually still compiles, because the types for Node, Workers and the DOM overlap. The differences are in what persists between requests, what is cancelled when a response is sent, and which APIs exist.

## Cloudflare Workers

### Isolates are reused across requests

Module-level variables persist from one request to the next in the same isolate, and many requests can be in flight in one isolate. A variable holding the current user, tenant, request id or a per-request client leaks between users. Holding request-bound objects (a stream, a response, something created from one request's context) globally and touching them from another request fails with "Cannot perform I/O on behalf of a different request".

```ts
// Before: the second concurrent request overwrites the first one's tenant
let tenantId: string | undefined;

export default {
  async fetch(request, env, ctx): Promise<Response> {
    tenantId = await resolveTenant(request, env);
    return handle(request, env);
  },
} satisfies ExportedHandler<Env>;

// After: request-scoped values travel as arguments
export default {
  async fetch(request, env, ctx): Promise<Response> {
    const tenantId = await resolveTenant(request, env);
    return handle(request, env, ctx, { tenantId });
  },
} satisfies ExportedHandler<Env>;
```

Module scope is still right for things that are safe to share and expensive to build: compiled schemas, lookup tables, a parsed static config. A module-level `Map` used as a cache is per isolate: several isolates run at once and each starts empty, so it cannot back rate limits, dedupe or locks. Use Durable Objects, KV or a database for shared state (database-engineering covers consistency).

### Work after the response

A promise that is neither awaited nor passed to `ctx.waitUntil()` can be cancelled when the invocation ends: logs dropped, writes half-done, no error anywhere. `ctx.waitUntil` extends the invocation for up to 30 seconds after the response for HTTP-triggered Workers. Work that may take longer, or must not be lost, goes to a Queue.

```ts
// Before: the audit write may never happen
auditLog(env, event);
return Response.json(result);

// After
ctx.waitUntil(auditLog(env, event).catch((err: unknown) => console.error("audit write failed", err)));
return Response.json(result);
```

Call it as `ctx.waitUntil(...)`. Destructuring `const { waitUntil } = ctx` loses `this` and throws "Illegal invocation".

### Configuration and bindings

- Secrets, variables and bindings arrive on the `env` argument of the handler. Validate the parts you need at the top of the handler (or once per isolate, cached in module scope, since `env` does not change between requests of one deployment).
- Node built-ins (`node:crypto`, `node:buffer`, `node:stream`, `AsyncLocalStorage`) need Node compatibility. It is on by default for a `compatibility_date` of 2026-08-04 or later; older dates need `compatibility_flags: ["nodejs_compat"]`, and without it the import fails at runtime. Moving the compatibility date forward changes runtime behavior (from 2026-02-10 with Node compatibility on, global `setTimeout` returns a Node `Timeout` object instead of a number), so treat it like a dependency upgrade and read the flags the new date turns on.
- `fetch()` and access to request context are not allowed during script startup. Keep module scope free of I/O.

### Memory and bodies

A Worker has a 128 MB memory limit. `await request.text()`, `arrayBuffer()` or `json()` on a large upload, or buffering a large upstream response, crashes the invocation. Stream bodies through (`new Response(upstream.body, upstream)`), and check `Content-Length` against a configured maximum before reading, remembering that the header is client-supplied and may be absent or wrong.

### Security primitives

- Compare secrets with `crypto.subtle.timingSafeEqual`, over digests of both values so the comparison does not reveal the expected length.
- Generate tokens with `crypto.randomUUID()` or `crypto.getRandomValues()`.
- `passThroughOnException()` fails open by sending the request to the origin when the Worker throws. Do not use it on a Worker that enforces authentication or authorization.

## Node with built-in type stripping

Node 22.18 and every later line run `.ts` files directly by erasing type annotations, unflagged; it is stable from 24.12 and 25.2. Node does not type-check and does not read `tsconfig.json`.

- Syntax that needs code generation is not supported: `enum`, namespaces containing values, constructor parameter properties (`constructor(private readonly db: Db)`) and import aliases. Set `erasableSyntaxOnly` so `tsc` rejects them before Node does. Decorators are a parse error in Node as well.
- Type-only imports must say so (`import type { User }` or `import { type User }`). Otherwise Node keeps the import, and the module it names exports nothing at runtime, so startup fails. `verbatimModuleSyntax` enforces this in `tsc`.
- `paths` aliases from `tsconfig.json` do not apply. Use `package.json` `imports` (`#db`) or relative paths.
- Relative imports name the file as it exists on disk (`./util.ts`, not `./util`). `.tsx` files are not supported.
- Node refuses to strip types in files under `node_modules`. A package published as `.ts` source fails for every consumer; publish compiled JavaScript.
- Node 26 removed `--experimental-transform-types`, and a script that still passes it fails to start. Code that needed it (enums, parameter properties) has to be rewritten as erasable syntax or run through tsx.
- `tsc --noEmit` in CI is the only type check.

## Bun

- Bun runs TypeScript without type-checking; CI still needs `tsc --noEmit`.
- Bun does not run dependency lifecycle scripts unless the package is in `trustedDependencies` or Bun's default allowlist. A dependency that relies on its `postinstall` (native builds, binary downloads) installs without error and breaks when first used. Add that package to `trustedDependencies` deliberately rather than turning scripts back on for everything. Defining the field replaces Bun's default allowlist instead of extending it, so also list the default-trusted packages the project relies on (the default list is `src/install/default-trusted-dependencies.txt` in the Bun repository).

## Browser and shared code

- Code shared between server and browser must not read secrets or `process.env` at module scope. Bundlers inline only explicitly public variables; anything else is either `undefined` in the browser or, worse, a secret shipped to it (frontend-engineering covers the framework rules).
- `AbortSignal.timeout` counts active time. In a suspended worker or a page in the back-forward cache the timer pauses, so a client-side deadline is not a server-side guarantee.
- `structuredClone`, `postMessage` and storage round-trips return plain objects: class instances lose their prototype, so methods and `instanceof` checks fail after the round-trip. Validate the data again on the receiving side.
- `instanceof Error` is false for errors created in another realm (iframes, `vm` contexts). `Error.isError` checks the internal brand instead (Node 24.3+, current Chrome, Firefox and Safari), but Safari and Bun return `false` for a `DOMException`, which is what an aborted or timed-out `fetch` rejects with. Branch on `name` or a `code` field in shared code.

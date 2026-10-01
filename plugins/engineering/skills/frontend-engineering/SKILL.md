---
name: frontend-engineering
description: React 19, Vite and React Router SPAs: TanStack Query data fetching, Zustand state, effects, forms and Actions, env and build config, code splitting, auth against an API, and wallet UIs. Load it before writing or reviewing any React component, hook, route or Vite config, even a small one, because the bugs it lists pass review.
license: MIT
metadata:
  author: Alex Tsanis
---

# Frontend engineering

A Vite SPA is a folder of static files that anyone can download, read and run with the network tab open. Every request it makes can be replayed, edited or sent by a script that never loaded the app. The bugs that cost money or leak data are rarely about markup: they are trust placed in the client, secrets compiled into the bundle, state with two owners, and async code that assumes each response arrives once, in order, and means what it says.

Read `package.json`, `vite.config.*`, the router setup and the `QueryClient` setup before writing code. Check which React Router mode the app uses (`<BrowserRouter>` is declarative mode; `createBrowserRouter` with `<RouterProvider>` is data mode), because loaders, `errorElement`, route `lazy` and `<ScrollRestoration>` exist only in data mode. Match the installed majors and minors, not memory: React Router 8 removed the `react-router-dom` package, and recent TanStack Query v5 minors deprecate `ensureQueryData`, `fetchQuery` and `prefetchQuery` in favor of `queryClient.query`.

## Failure catalogue

Each entry: what it looks like in a diff, why it breaks, what to do instead. Longer before/after code lives in the reference files.

### The SPA is an untrusted client

**API trusts client-computed values.** The request body carries `price`, `total`, `discount`, `role`, `userId`, `isAdmin` or an `ownerId`, and the Worker writes it. Anyone can POST any value. The API derives identity from its session, prices from its own catalogue and permissions from its own records. The client sends intent only: SKUs, quantities, the id of the thing to act on.

**Route guard treated as authorization.** `<RequireAdmin>` wrapping a route, a loader that redirects non-admins, or `if (user.role !== 'admin') return <Navigate to="/" />`. This hides a page; it protects nothing. The admin chunk is still a public file under `/assets/`, and its API endpoints answer whoever calls them. Every API endpoint authenticates the request and authorizes it against the specific resource (ownership, tenant, role) on its own. A role decoded from a JWT in the browser is for choosing what to render, never for deciding what is allowed.

**Secret compiled into the bundle.** `VITE_STRIPE_SECRET`, `VITE_ADMIN_TOKEN`, an RPC or AI provider key in `import.meta.env`, `define: { 'process.env': env }` fed from `loadEnv(mode, process.cwd(), '')` (the empty prefix loads every variable), or `envPrefix` widened to something like `['VITE_', 'APP_']` so more names qualify. Vite statically replaces `import.meta.env.VITE_*` and `define` values in the built JavaScript, so the value is readable by anyone. Secrets live in the Worker's secret store; the SPA calls the Worker. A value is fine in the bundle only when its provider designed it to be public (a publishable key, an analytics site id, an origin-restricted key).

**Tokens in `localStorage` or a persisted store.** `localStorage.setItem('accessToken', token)` or a Zustand `persist` store holding the session. Any XSS, or any compromised dependency, reads it and sends it elsewhere, and it outlives the tab. Prefer an `HttpOnly`, `Secure`, `SameSite` cookie set by the API and send it with `fetch(url, { credentials: 'include' })`. That makes the API responsible for CORS (an explicit origin allowlist, never a reflected `Origin` with `Access-Control-Allow-Credentials: true`) and for CSRF on cookie-authenticated writes. If a bearer token must be readable by JavaScript, keep it in memory with a short lifetime.

**XSS through HTML, markdown or URLs.** `dangerouslySetInnerHTML={{ __html: userText }}`, markdown rendered with `rehype-raw` and no sanitizer, `href={user.website}`. Render text as text. Sanitize HTML you must render with DOMPurify (it works directly in the browser) or `rehype-sanitize`. Parse user URLs with `new URL()` and allowlist `http:`/`https:`/`mailto:`. React 19 blocks `javascript:` URLs that it renders into attributes, but not URLs your code passes to `navigate()`, `window.location`, `window.open` or a third-party widget.

**Open redirect via `?returnTo=`, `?next=`, `?redirect=`.** After login, `navigate(searchParams.get('returnTo'))` or `window.location.assign(returnTo)`. `returnTo.startsWith('/')` is not enough: `//evil.example` and `/\evil.example` both pass and resolve off-site. Resolve against your own origin, compare origins, and reject a path that starts with `//`; or map to an allowlist of route names.

**Dev server or preview exposed.** `server.host: true` (or `--host`) on a laptop on shared Wi-Fi, `server.allowedHosts: true` to make a tunnel work, or `vite preview` used as the production server. The dev server serves source, and Vite documents that `allowedHosts: true` lets any website read it through DNS rebinding. Keep the default `localhost`, list tunnel hosts explicitly in `allowedHosts`, and deploy `dist/` to a real static host. Vite's docs say `vite preview` is not meant as a production server.

**Source maps deployed.** `build.sourcemap: true` publishes original source, comments included, next to the bundle. `'hidden'` drops the `//# sourceMappingURL` comment but still writes `.map` files into `dist/`, so they deploy unless the pipeline removes them. Upload maps to the error tracker in CI and delete them before the deploy step.

### Build, config and delivery

**Build-time config treated as runtime config.** `const API_URL = import.meta.env.VITE_API_URL` is frozen when `vite build` runs. One artifact promoted from staging to production keeps calling staging. A missing variable is `undefined`, which becomes `fetch('undefined/orders')` and a request to a relative path on your own host. Validate env with a schema in one module at startup and fail loudly. Then pick one model: build per environment (`vite build --mode staging` loads `.env.staging`), or build once and fetch a validated config file at boot. See `vite-build-and-deploy.md`.

**Type errors shipped.** Vite transpiles TypeScript and does not type-check, so `vite build` succeeds with type errors. The build script runs `tsc --noEmit` (or `tsc -b` with project references) before or alongside `vite build`, and CI runs it.

**Stale chunks after a deploy.** A user with the app open clicks a lazy route after a deploy removed the old hashed chunks, and gets "Failed to fetch dynamically imported module" or a blank route. Serve `index.html` with `Cache-Control: no-cache` so new visits get new chunk names, keep the previous deploy's assets available for a while, and handle `vite:preloadError` with a reload that cannot loop.

**No real not-found.** The host's SPA fallback returns `index.html` with 200 for every unknown path, which deep links need, but the router has no `path="*"` route, so a typo renders an empty layout. Add a catch-all not-found route, and on public pages tell crawlers (Google treats a 200 "not found" view as a soft 404). If the API shares the host, make sure API paths reach the Worker and never fall through to `index.html`.

**No route-level code splitting, or splitting done wrong.** The whole app, including admin screens, chart libraries and the wallet stack, in one entry chunk. Or `const Page = lazy(() => import('./Page'))` declared inside a component, which creates a new component type each render and resets its state. Declare `lazy` at module top level, wrap lazy routes in `<Suspense>` with an error boundary (a failed chunk load throws to the nearest one), or use route `lazy` in data mode so the component and its loader load in parallel.

**Bundle bloat.** A whole icon set, date library, chart library or SDK imported through a barrel file into the entry chunk. Import specific modules, load heavy widgets with a dynamic import at the route or interaction that needs them, and compare bundle output before and after.

**Hydration bugs imported from SSR advice.** An app mounted with `createRoot` never hydrates: there is no server HTML to match, so hydration mismatches, `suppressHydrationWarning` and `getServerSnapshot` do not apply. Do not add `typeof window` guards or mount-then-render workarounds for a problem the app does not have. If the marketing site adds build-time prerendering and switches to `hydrateRoot`, every rule about time, locale, randomness and browser-only reads during the first render applies again; see `vite-build-and-deploy.md`.

**Layout shift from media.** `<img>` without `width`/`height` or an `aspect-ratio`, late-loading fonts and banners that push content down. Reserve the space up front.

### State ownership

**Server state copied into `useState` or a Zustand store.** `const [user, setUser] = useState(data.user)` or `useEffect(() => useCartStore.setState({ items: data.items }), [data])`. The copy stops tracking the source: after a refetch or invalidation the screen shows old data and a later save overwrites newer server data. Render from the query result. If the user edits a draft, keep the draft separate and key the form by the record's id and version so it resets when the record changes.

**Zustand selector that returns a new object.** `const { items, add } = useCart((s) => ({ items: s.items, add: s.add }))`. Zustand 5 documents that a selector returning a new reference may cause an infinite loop, which React reports as "Maximum update depth exceeded". Select each value separately, or wrap the selector in `useShallow` from `zustand/shallow`.

**Persisted store without a version or a boundary.** `persist` with only a `name` stores the whole store, server data and tokens included, in `localStorage`. When the shape changes, old data rehydrates into new code. Persist only client-owned preferences with `partialize`, bump `version` and write `migrate` when the shape changes (a version mismatch without `migrate` discards the stored value), and reset user-scoped stores on sign-out.

**Effect used for derived state.** `useEffect(() => setFullName(first + ' ' + last), [first, last])` renders once with stale output, then again. Compute during render. To reset state when an id changes, put `key={id}` on the component.

**Query key missing an input.** `useQuery({ queryKey: ['projects', orgId], queryFn: () => fetchProjects(orgId, status, page) })`. Changing `status` or `page` returns the cached page for the wrong filter. Every value the `queryFn` reads goes in the key. Use a key factory per resource so mutations invalidate the same shape.

**No invalidation after a mutation.** The mutation succeeds but lists, counts and detail views keep serving cached data. Invalidate in `onSettled` with the resource prefix (`invalidateQueries` matches by prefix, so `['projects']` covers `['projects', orgId, filters]`).

**Cache that outlives the account.** Keys without the user or org id, and no reset on sign-out, so the next account in the same tab briefly sees the previous one's data. Put the scope id in keys and call `queryClient.clear()` on sign-out. Create the `QueryClient` once at module level or in a lazy `useState` initializer; `new QueryClient()` in a component body gives every render an empty cache.

**URL state kept in component state.** Filters, sort, tab, page and selected item in `useState`, so refresh, back and share lose them. Use `useSearchParams`, parse values with a schema that falls back to a default (they are user input), and pass `{ replace: true }` for keystroke-level changes. The functional form of `setSearchParams` does not queue like `setState`: two calls in the same tick do not build on each other.

**Stale closures.** A `setInterval` or subscription callback created once reads the `count` or `userId` from the first render forever. Use functional updates, include the value in the effect's dependencies, or read the latest value through `useEffectEvent` (React 19.2) or a ref.

**Controlled/uncontrolled switch.** `<input value={user?.name} />` is uncontrolled while `user` loads and controlled after. Use `value={user?.name ?? ''}`, or render the form only once data exists.

**Index keys on lists that reorder, filter or delete.** With `key={index}`, deleting row 2 hands row 3's input text or open menu to row 2. Key by a stable id from the data; never generate keys in render.

### Async flows and mutations

**Effect races and missing cleanup.** A search effect fetches on every keystroke and the response for "re" lands after the one for "react". Abort the previous request in cleanup (`AbortController`), or use TanStack Query, which keys results by input. Every effect that subscribes, listens or starts a timer returns cleanup. Strict Mode's development double mount exists to expose missing cleanup; fix the effect rather than removing Strict Mode.

**Request waterfalls.** A parent `useQuery` renders a child with its own `useQuery`, which renders a grandchild with another; or a lazy route whose component must download before its data request starts. Start independent requests together, hoist data needs to the route (a data-mode loader that starts the queries into the `QueryClient` cache), and load a lazy route's component and loader in parallel.

**Double submit.** Two clicks, Enter plus click, or a retry after a slow response create two orders. Disable the control while pending, and send an idempotency key generated once per user intent so the API deduplicates. The disabled button is UX; the key is the protection.

**Non-idempotent mutation retried.** `useMutation({ retry: 3, mutationFn: createPayment })`. TanStack Query does not retry mutations by default, but queries retry three times with backoff. A retried POST without an idempotency key can charge twice. Retry writes only with the same key, after the unknown-outcome check.

**Optimistic update without rollback.** The row disappears before the API answers and never comes back on failure. With TanStack Query, either render the pending `variables` in the one place that needs them, or snapshot the cache in `onMutate`, restore it in `onError`, cancel in-flight refetches first and invalidate in `onSettled`. `useOptimistic` reverts on its own only when the real value updates inside the same transition and only on success.

**Form Action clears the user's input.** `<form action={submit}>` resets uncontrolled fields after the action function succeeds, and an action that returns validation errors has still succeeded, so the user's typing vanishes next to the error. Return the submitted values in the state and feed them back as `defaultValue`, or use controlled inputs. In `useActionState`, a thrown error cancels queued actions and goes to the error boundary; return expected failures as state.

**Success shown before the API confirms.** A "Payment sent" toast in `onMutate` or right after `fetch()` resolves. `fetch` resolves on 4xx and 5xx; check `res.ok` and parse the body. Show success only from the confirmed response.

**Unknown outcome treated as failure.** A timeout or dropped connection on a POST does not mean nothing happened. Show "checking status", re-read the authoritative record by idempotency key or returned id, and only then offer retry with the same key.

**Missing loading, error and empty states.** `if (!data) return <Spinner />` spins forever on error. Handle pending, error with retry, empty and success separately, and decide what the user sees when a background refetch fails over data already on screen.

**Client-only validation.** The form validates with a schema and the Worker trusts the payload. Share one schema: the client runs it for feedback, the API runs it as the gate, and API errors map back to fields.

### Routing and navigation

**No error boundary on routes.** In data mode, loader and render errors bubble to the closest parent `errorElement`, so with a boundary only at the root, one failing panel route replaces the whole layout. Put an `errorElement` (or `ErrorBoundary`) on the root and on routes that can fail independently, use `isRouteErrorResponse` to tell a thrown 404 from a crash, and in declarative mode wrap lazy routes in your own boundary.

**Not-found and forbidden that differ.** The UI shows "You don't have access" for another tenant's invoice and "Not found" for a missing one, confirming which ids exist. The API returns the same 404 for both, and the UI renders the same page.

**Focus left behind on navigation.** A client-side route change swaps the page without a document load, so focus stays on a removed link or falls to `<body>`, and nothing announces the new page. React Router does not manage this. Set `document.title` per route and move focus to the new page's heading on mount; the `accessibility` skill has the component.

**Exit animation that renders the wrong page.** `<AnimatePresence>` around `<Outlet />` or an unkeyed `<Routes>`: the exiting copy renders the new route. The `react-motion` skill covers the `useOutlet` and keyed `<Routes location>` patterns and their scroll and focus interactions.

### Web3 frontends

**Wrong chain.** The wallet is on a different chain than the contract. Compare the connection's chain id with the target before every write, offer a switch, and pass the target chain to the write call so a mismatch throws.

**Wallet-reported state trusted as proof.** A connected address, balance or NFT read in the browser gates something on the API. The address is a claim until the Worker verifies a signed EIP-4361 message with a server-issued nonce, domain and expiry. Balances in the UI are for display; the contract or API re-checks.

**Private RPC key in the bundle.** `http(import.meta.env.VITE_RPC_URL)` with a provider key in the URL. wagmi recommends an authenticated RPC to avoid public rate limits, but in an SPA that URL is public. Use a key the provider restricts to your origins, or proxy reads through the Worker with its own rate limit.

**Opaque signature requests.** "Sign to continue" over a hex blob, `personal_sign` for something that authorizes value, or an unlimited `permit`. Use EIP-712 typed data bound to chain and contract, and say in the UI what the signature authorizes, for how much and until when.

**Transaction treated as done on hash.** A hash means submitted. Track awaiting-wallet, submitted, included (check `receipt.status`), confirmed, replaced or cancelled, and unknown after a timeout; persist pending hashes so a refresh does not lose them.

## Two patterns to reach for

Stale responses cannot win when the effect aborts its request on cleanup:

```tsx
// Before
useEffect(() => {
  fetch(`${apiBase}/search?q=${query}`).then((r) => r.json()).then(setResults)
}, [query])

// After
useEffect(() => {
  const controller = new AbortController()
  fetch(`${apiBase}/search?${new URLSearchParams({ q: query })}`, { signal: controller.signal })
    .then((res) => {
      if (!res.ok) throw new Error(`search responded ${res.status}`)
      return res.json()
    })
    .then((body) => setResults(SearchResponse.parse(body).results))
    .catch((error: unknown) => {
      if (!controller.signal.aborted) setSearchError(error)
    })
  return () => controller.abort()
}, [query])
```

Env values are validated once, in one module, and nothing else reads `import.meta.env`:

```ts
// src/config/env.ts
import { z } from 'zod'

const PublicEnv = z.object({
  VITE_API_BASE_URL: z.url(),
  VITE_TARGET_CHAIN_ID: z.coerce.number().int().positive(),
})

export const env = PublicEnv.parse({
  VITE_API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
  VITE_TARGET_CHAIN_ID: import.meta.env.VITE_TARGET_CHAIN_ID,
})
```

Each name is written out because Vite statically replaces `import.meta.env.VITE_X` at build time, and its docs note that computed access such as `import.meta.env['BASE_URL']` is not replaced. Everything in this schema ships to the browser, so it holds only public values.

## Decision rules

Where state lives:

| Kind | Owner |
| --- | --- |
| Data from the API or chain | TanStack Query (or wagmi/viem reads, which use it) |
| Filters, sort, tab, page, selected id | URL search params |
| In-progress form input | The form (uncontrolled with `FormData`, or a form library), validated by a schema the API also runs |
| Client-only preferences shared across routes (theme, sidebar, unsent drafts) | A Zustand store, persisted with `partialize` and a `version` only if it must survive reload |
| Open/closed, hover, local drafts | Component state, lifted only as far as the nearest common owner |
| Session identity, permissions | The API session; the client holds a display copy fetched through a query |

- Build per environment or runtime config: build per environment when each environment has its own pipeline anyway. Build once and fetch a config file at boot when the same artifact must move through staging to production. Never ship both models in one app.
- Declarative or data mode: data mode when routes need loaders, per-route error boundaries, route `lazy` or `<ScrollRestoration>`. Declarative mode for small apps whose data loading lives entirely in TanStack Query. Do not run both routers.
- `useMutation` or React Actions: when the resource already lives in TanStack Query, mutate with `useMutation` so invalidation and rollback sit next to the cache. Use `useActionState` for forms whose result stays inside the form (field errors, a confirmation), and invalidate the query cache from the action when it changes server data.
- Optimistic or pessimistic: optimistic for reversible, low-stakes, usually-successful actions (like, rename, reorder, toggle). Pessimistic with an explicit pending state for payments, transfers, irreversible deletes, permission changes and anything on-chain.
- Retry: automatic retry only for reads and idempotent writes. A non-idempotent write retries only with the same idempotency key, after the unknown-outcome check.
- Effects: if the value can be computed from props and state, compute it. If it responds to a user event, put it in the handler. Use an effect only to synchronize with something outside React.
- Memoization: add `memo`, `useMemo` and `useCallback` where the Profiler shows wasted renders, or rely on React Compiler if the project has adopted it.

## Review checklist

- Does every API endpoint this change calls authenticate, schema-validate and check ownership of the specific resource itself, independent of any route guard?
- Is every amount, price, role and user id the API uses derived on the API?
- Is every `VITE_` variable, `define` value and `envPrefix` entry free of secrets, and is env read through one validated module?
- Are tokens kept out of `localStorage` and persisted stores, and does a cookie-based API have an exact CORS origin list and CSRF protection?
- Is user-controlled HTML sanitized, and does every user-supplied URL (links, `navigate`, `window.location`, `returnTo`) pass a protocol or origin check?
- Are source maps absent from the deployed `dist/`, and is the dev server bound to `localhost` with no `allowedHosts: true`?
- Does CI run `tsc` as well as `vite build`?
- Is server data rendered from its query rather than copied into `useState` or a store?
- Does each query key include every input its fetcher reads plus the user or org scope, and does each mutation invalidate the affected keys?
- Do Zustand selectors return stable values, and does each persisted store use `partialize`, `version` and a sign-out reset?
- Can a slow earlier response overwrite a newer one? Does each effect clean up?
- Is a double click harmless because the API deduplicates by idempotency key, and are mutation retries off unless the write is idempotent?
- Does each optimistic update roll back on failure, and is success shown only after confirmation?
- Is a timeout shown as "unknown, checking" rather than "failed, try again"?
- Do loading, error, empty and success states each render something deliberate?
- Are lazy routes declared at module level, wrapped in error handling, and covered by a `vite:preloadError` reload?
- Is there a catch-all not-found route, and does `index.html` revalidate on every load?
- For web3: is the chain checked before writes, success tied to a successful receipt, every signature human-readable and scoped, and no private RPC key in the bundle?

## References

- [security-boundaries.md](references/security-boundaries.md): read when code touches auth, tokens, env vars, the API contract, user HTML or markdown, links or redirects.
- [state-and-async.md](references/state-and-async.md): read when writing effects, queries, mutations, optimistic updates, Zustand stores, forms, URL state or list rendering.
- [vite-build-and-deploy.md](references/vite-build-and-deploy.md): read before changing `vite.config`, env handling, build scripts, source maps, the dev server, caching headers, the SPA fallback or prerendering.
- [react-router-spa.md](references/react-router-spa.md): read when adding or changing routes, lazy loading, loaders, error boundaries, not-found handling or navigation side effects.
- [web3-frontends.md](references/web3-frontends.md): read when the UI connects a wallet, requests signatures or sends transactions.
- [react-upgrades.md](references/react-upgrades.md): read for React, React Router, Vite, TanStack Query, Zustand or wagmi major upgrades and class-to-hook conversions.

Related skills: `security-engineering` for the API side and contract audits, `typescript-engineering` for types and runtime validation, `accessibility` for focus and semantics, `react-motion` for route and exit animations, `seo-engineering` for metadata and crawlability on the marketing site, `test-engineering` for testing async UI, `performance-benchmarking` for measurement.

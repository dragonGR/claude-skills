# Security boundaries in a client-rendered app

Read this when a change touches sign-in, tokens, env variables, the contract with the API, user-supplied HTML or markdown, links or redirects. The SPA runs in the user's browser and can be replaced by `curl`, so every decision that matters is enforced by the API (a Cloudflare Worker in this stack). Examples use TypeScript and Zod 4; the API-side snippets show what the SPA must be able to rely on, and `security-engineering` covers the API in depth.

## What an attacker can read

Everything the browser downloads: every chunk in `dist/` (lazy admin routes included), every `VITE_*` and `define` value, any deployed source map, every JSON response the app receives, and anything in `localStorage`, `sessionStorage` or IndexedDB. Hiding a field in the UI, a route behind a guard or a button behind a role check hides nothing. `vite-build-and-deploy.md` has the audit commands for the bundle.

## Env variables

- `import.meta.env.VITE_*` values and `define` values are replaced with literals in the built JavaScript.
- Secrets never get the `VITE_` prefix, never go into `define`, and never reach the client through a widened `envPrefix` or `loadEnv(mode, root, '')`.
- A provider key is acceptable in the bundle only when the provider designed it to be public: a Stripe publishable key, an analytics site id, an RPC or maps key that the provider restricts to your origins. The finding is a key whose holder can do more than an anonymous visitor should.
- When the SPA needs a privileged third-party call, the Worker makes it with a secret from its own environment and exposes a narrow endpoint.

## Sessions and tokens

Before:

```ts
const res = await fetch(`${env.VITE_API_BASE_URL}/login`, { method: 'POST', body })
const { accessToken, refreshToken } = await res.json()
localStorage.setItem('accessToken', accessToken)
localStorage.setItem('refreshToken', refreshToken)
```

Any script that runs on the origin (an XSS, a compromised npm dependency, a third-party widget) can read both tokens and use them from anywhere until they expire. The refresh token makes that weeks.

After: the API sets the session as a cookie the page cannot read, and every call sends it.

```ts
// src/api/client.ts
export async function apiFetch(path: string, init: RequestInit = {}): Promise<Response> {
  return fetch(new URL(path, env.VITE_API_BASE_URL), { ...init, credentials: 'include' })
}
```

The API's `Set-Cookie` uses `HttpOnly`, `Secure`, `SameSite=Lax` or `Strict`, a `Path`, and a lifetime it controls. The SPA learns who is signed in by calling a `/me`-style endpoint through TanStack Query, never by decoding a token.

Moving to cookies moves two duties to the API:

- CORS. For credentialed requests the browser requires `Access-Control-Allow-Credentials: true` and an explicit `Access-Control-Allow-Origin`; `*` is not accepted. The dangerous shortcut is echoing whatever `Origin` arrived, which lets any website make authenticated calls and read the responses. Compare against an allowlist from configuration.
- CSRF. Cookies ride along on cross-site requests that `SameSite` does not block. Check `Origin` (or a CSRF token) on every state-changing request. `SameSite` is per site, not per origin: `app.example.com` and `blog.example.com` are the same site, so a compromised sibling subdomain is not stopped by it.

```ts
// Worker side, before: reflects any origin with credentials
headers.set('Access-Control-Allow-Origin', request.headers.get('Origin') ?? '')
headers.set('Access-Control-Allow-Credentials', 'true')

// After
const origin = request.headers.get('Origin')
headers.append('Vary', 'Origin')
if (origin !== null && allowedOrigins.has(origin)) {
  headers.set('Access-Control-Allow-Origin', origin)
  headers.set('Access-Control-Allow-Credentials', 'true')
}
```

`allowedOrigins` is built from the Worker's configuration. `Vary: Origin` goes on every response, allowed or not; otherwise a shared cache can store the no-CORS response a disallowed origin got and serve it to the SPA.

If the architecture forces a bearer token into JavaScript (a third-party API that only accepts headers), keep it in memory, short-lived, and refreshed through an `HttpOnly` cookie endpoint. It still dies to XSS while the tab is open, so the CSP and sanitizing rules below matter more.

Sign-out clears client state as well as the session: `queryClient.clear()`, reset every user-scoped Zustand store, remove persisted entries, and then navigate. Otherwise the next person on a shared machine sees cached data until it refetches.

## The API stands alone

Every endpoint the SPA calls is reachable without the SPA. The handler authenticates, parses, authorizes against the specific resource, mutates, and returns a narrow result.

Before:

```ts
export async function handleCheckout(request: Request, deps: CheckoutDeps): Promise<Response> {
  const { userId, items, total } = await request.json()
  const order = await deps.orders.create({ userId, items, total })
  await deps.payments.charge(userId, total)
  return Response.json(order)
}
```

The buyer is whoever `userId` says, the total is whatever the client sent, `items` is unvalidated, the whole order row (with any internal columns) goes back to the browser, and a double click creates two orders.

After:

```ts
import { z } from 'zod'

const CheckoutInput = z.strictObject({
  items: z
    .array(
      z.strictObject({
        sku: z.string().min(1),
        quantity: z.int().positive().max(MAX_LINE_QUANTITY),
      }),
    )
    .min(1)
    .max(MAX_CART_LINES),
  idempotencyKey: z.uuid(),
})

export async function handleCheckout(request: Request, deps: CheckoutDeps): Promise<Response> {
  const user = await deps.auth.requireUser(request)
  if (!user) return Response.json({ error: 'unauthenticated' }, { status: 401 })

  const parsed = CheckoutInput.safeParse(await request.json().catch(() => null))
  if (!parsed.success) return Response.json({ error: 'invalid_input' }, { status: 400 })

  const result = await deps.orders.place({ userId: user.id, ...parsed.data })
  return Response.json(result, { status: result.ok ? 200 : 409 })
}
```

`orders.place` loads prices from the catalogue, computes the total, and inserts under a unique constraint on `(user_id, idempotency_key)`, so a replay returns the first order instead of creating a second. It returns `{ ok, orderId }` or a typed error, never the row.

Object-level authorization goes into the write itself:

```sql
UPDATE projects SET name = ?1 WHERE id = ?2 AND org_id = ?3
```

with `org_id` taken from the session. Zero affected rows returns the same 404 as a project that does not exist, so ids from other tenants cannot be probed. A separate "load, check owner, update" sequence leaves a gap between the check and the write.

## Route guards are UX

```tsx
export function RequireRole({ role, children }: { role: Role; children: ReactNode }) {
  const me = useQuery(meQuery)
  if (me.isPending) return <PageSkeleton />
  if (!me.data?.roles.includes(role)) return <Navigate to={FORBIDDEN_PATH} replace />
  return children
}
```

This is correct as UX: it keeps people out of screens they cannot use. It is not access control. The admin chunk downloads for anyone who requests it, and its endpoints must reject non-admins on their own. In a review, for every guarded route, find the endpoints its screens call and confirm each checks the role or ownership itself.

## What the API sends back

The SPA receives the full JSON body, whatever the UI renders. An endpoint that returns the user row sends `passwordHash`, `email`, internal flags and provider ids to anyone with devtools open. Shape each response for its screen (a DTO with only the rendered fields), and type the client's parser to that shape with a strict schema so a new column does not silently start flowing into state.

## Rendering user content

Text in JSX is escaped. The holes are the escape hatches.

HTML from users or a CMS:

```tsx
import DOMPurify from 'dompurify'

export function RichText({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
}
```

In the browser DOMPurify has a DOM and works as is. Without a DOM it does nothing and returns its input unchanged, which matters in two places: a prerender step that runs components in Node, and unit tests running in a plain Node environment. Bind DOMPurify to a jsdom window there (DOMPurify warns that old jsdom versions are unsafe and that happy-dom is not suitable), or sanitize on the API at write time. Write-time sanitizing alone misses rows stored before the sanitizer existed.

Markdown: `react-markdown` escapes raw HTML and strips `javascript:` URLs by default. Adding `rehype-raw` turns raw HTML back on, so pair it with `rehype-sanitize`. A custom `urlTransform` or custom `components` for `a` and `img` can reopen the hole; review them.

Links:

```ts
const SAFE_LINK_PROTOCOLS = new Set(['http:', 'https:', 'mailto:'])

export function safeHref(raw: string, base: string): string | null {
  let url: URL
  try {
    url = new URL(raw, base)
  } catch {
    return null
  }
  return SAFE_LINK_PROTOCOLS.has(url.protocol) ? url.href : null
}
```

`URL.canParse` would read better, but it needs Chrome 120, Firefox 115 and Safari 17, newer than Vite 8's default targets (Chrome 111, Firefox 114, Safari 16.4). Vite lowers syntax, not missing APIs, so on those browsers it throws. The try/catch works everywhere. Call it with `window.location.origin` as `base`. Render nothing, or plain text, when it returns `null`. External links from users also get `rel="noopener noreferrer"`, plus `ugc` or `nofollow` on public pages.

React 19 blocks `javascript:` URLs it renders into attributes, but not URLs your code hands to `navigate()`, `window.location`, `window.open`, an iframe `src` set imperatively, or a third-party component that sets the DOM itself. Validate before every navigation sink.

## Redirect targets

Before:

```ts
const returnTo = searchParams.get('returnTo') ?? '/'
if (returnTo.startsWith('/')) window.location.assign(returnTo)
```

`//evil.example/login` and `/\evil.example` both start with `/` and leave your site; browsers treat the backslash as a slash in URLs.

After:

```ts
export function safeRedirectPath(raw: string | null, appOrigin: string, fallback: string): string {
  if (!raw) return fallback
  let target: URL
  try {
    target = new URL(raw, appOrigin)
  } catch {
    return fallback
  }
  if (target.origin !== appOrigin || target.pathname.startsWith('//')) return fallback
  return `${target.pathname}${target.search}${target.hash}`
}
```

In the SPA, `appOrigin` is `window.location.origin`. On the Worker (an OAuth callback that redirects back to the app), it comes from configuration, never from the request's `Host` header. The origin check alone is not enough when you return a path: `/.//evil.example` resolves to your origin with the path `//evil.example`, which a browser then reads as a protocol-relative URL to another host. Rejecting a leading `//` closes it. If the set of post-login destinations is small, an allowlist of route names is simpler still.

## Content Security Policy

A CSP sent by the host limits what an injected script can load and where it can send data. It is the backstop for every sanitizer mistake above, and it is configured on the host, not in React. `vite-build-and-deploy.md` lists what a Vite build needs.

## Review questions

- Can I call this endpoint with `curl`, as another user, with a modified body, twice? What happens each time?
- Which field in the request decides who, how much or which record? Is any of them read from the body instead of the session?
- What does the browser receive from this endpoint that the UI does not display?
- Where does the session live, and what can a script running on the origin read?
- Does the API's CORS response reflect arbitrary origins, and does every cookie-authenticated write check `Origin`?
- Which strings reach a navigation or HTML sink, and which check sits in between?

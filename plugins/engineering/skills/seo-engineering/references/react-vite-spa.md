# React + Vite SPAs and crawlers

Read this when the site is a React + Vite single-page app (React Router in declarative or data mode, built to static files) and the change touches what crawlers or link-preview bots receive: `index.html`, per-route head tags, prerendering, status codes from the host, sitemaps or deploy caching.

## What each client receives

A client-rendered SPA returns the same `index.html` for every route. The title, canonical, Open Graph tags, main content and internal links of a route exist only after the bundle has downloaded, run and fetched its data.

| Client | Runs JavaScript | What it reads |
|---|---|---|
| Googlebot | Yes, in a separate render pass | The rendered DOM. Pages with status 200 go into a render queue where they "may stay ... for a few seconds, but it can take longer than that". |
| Link-preview fetchers | Do not count on it | The served HTML. Slackbot fetches "as little of the page as it can (using HTTP Range headers)" to read oEmbed, Twitter Card and Open Graph tags. Facebook's crawler cuts off Open Graph properties after the first 1 MB. |
| Other crawlers | Not unless the vendor documents it | Google's own guidance: "not all bots can run JavaScript". |

What Google's renderer does differently from a user's browser, per its JavaScript SEO docs:

- A `noindex` in the served HTML may stop rendering altogether, so JavaScript that removes it may never run.
- Cookies, `localStorage` and `sessionStorage` are cleared between page loads, and permission prompts are declined.
- It fetches over HTTP only; content that arrives over WebSockets or WebRTC needs an HTTP fallback.
- It caches resources aggressively and may ignore your cache headers, so resource URLs need content hashes.

The practical consequence: Google can index a client-rendered public page, but share previews and non-rendering crawlers only ever see the shell, and Google sees route content later and less reliably than server HTML. Public pages that must rank or be shared get prerendered HTML; the app behind login stays client-rendered.

## Concerns that do not exist here, and what replaces them

- **No server/client component boundary.** Every module runs in the browser, so there is no server-only metadata API. Head tags are either written into HTML at build time or rendered by React at runtime.
- **No server redirects from app code.** `<Navigate>` and a loader's `redirect()` run in the browser. Google follows JavaScript redirects only when rendering succeeds and recommends them only when a server or meta refresh redirect is impossible. Permanent moves belong in the host config (`_redirects`, nginx, a Worker) with 301 or 308.
- **No per-request status codes from React.** A not-found route component cannot set the HTTP status. The host decides the status, before any JavaScript runs.
- **No streaming metadata.** Nothing reaches the crawler later in the same response; it is in the HTML file or it is client-rendered.
- **Env exposure is build-time.** `import.meta.env.VITE_*` values are replaced in the bundle at build time and are public. For SEO the dangerous ones are the site origin and environment flags, because they get baked into canonicals, sitemaps and robots decisions for whichever environment the build was made for. Secrets: see security-engineering.

## The shell: what index.html may contain

Every tag in `index.html` is a claim about every URL the fallback serves. Keep it to tags that are true everywhere: charset, viewport, icons, `theme-color`, site verification tags, preconnects, and the stylesheet and script Vite injects. These do not belong in a shell served for many URLs:

- `<link rel="canonical">`: every route declares the home page canonical. A per-route canonical added later by React becomes a second canonical with a different value, which Google's guidance says not to do.
- `og:url`, `og:title`, `og:description`, `og:image`, `twitter:*`: every shared link previews as the home page.
- `<meta name="robots" content="noindex">` intended for app routes: it applies to every public route too, and Google may skip the render that would have removed it.
- A route-specific `<meta name="description">`.

A neutral `<title>` with the site name is acceptable in a shell that no indexable route depends on; once React renders per-route titles, check in the rendered DOM that only one `<title>` is in the head (see audit.md).

Vite replaces `%VITE_NAME%` placeholders in `index.html` from `import.meta.env`, and a variable that does not exist "will be ignored and not replaced". A missing `VITE_SITE_ORIGIN` ships `href="%VITE_SITE_ORIGIN%/"` literally. Read and validate origin variables in `vite.config.ts` with `loadEnv` so the build fails instead:

```ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'

function requireOrigin(value: string | undefined, name: string): string {
  if (!value) throw new Error(`${name} is not set for this build mode`)
  const url = new URL(value)
  if (url.protocol !== 'https:' || url.pathname !== '/' || url.search || url.hash) {
    throw new Error(`${name} must be a bare https origin, got ${value}`)
  }
  return url.origin
}

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_')
  requireOrigin(env.VITE_SITE_ORIGIN, 'VITE_SITE_ORIGIN')
  return { plugins: [react()] }
})
```

`vite build` runs in `production` mode by default and loads `.env.production`. A staging build made with plain `vite build` therefore carries the production origin and `import.meta.env.PROD === true`. Build staging with `--mode staging`, and never decide indexability inside the bundle: the same artifact is often promoted between environments, so the host decides per hostname (see crawl-control.md).

## Per-route head tags

### React 19 built-in tags

React 19 hoists `<title>`, `<meta>` and `<link>` rendered anywhere in the tree into `document.head`. What the docs pin down:

- `<title>` children must be a single string. `<title>Results page {page}</title>` is an array; write `` <title>{`Results page ${page}`}</title> ``.
- If more than one component renders a `<title>` at the same time, React puts all of them in the head, and "the behavior of browsers and search engines is undefined". A layout title plus a page title is exactly this.
- `<meta>` and `<link>` with `itemProp` are not hoisted. `<link>` with `onLoad` or `onError` is not hoisted. A stylesheet `<link>` gets special handling only with `precedence`.
- Deduplication is documented only for stylesheets with `precedence` (same `href`). Nothing is documented for canonicals or Open Graph tags; two components rendering `<link rel="canonical">` give you two.
- React only manages elements it rendered. Tags written in `index.html` stay where they are.

### react-helmet-async 3

On React 19, v3 renders a native element for each tag and lets React hoist it. Reading its `React19Dispatcher` source: each `<Helmet>` instance renders its own tags with no merging across instances, so the v1/v2 behavior where the innermost `<Helmet>` replaced an outer title or canonical is gone. A layout `<Helmet>` with a default title and canonical plus a page `<Helmet>` now produce two of each. `titleTemplate` and `defaultTitle` still apply within one instance. `htmlAttributes` and `bodyAttributes` are still applied by direct DOM writes. The `HelmetProvider` `context` object "will not be populated with helmet state on React 19", so a prerender script that reads `helmetContext.helmet.title.toString()` gets nothing.

Both approaches therefore need the same discipline: exactly one component owns the head for the current route.

```tsx
// Before: layout and page both render head tags; on React 19 the head ends up with two titles and two canonicals.
function SiteLayout() {
  return (
    <>
      <Helmet>
        <title>Acme</title>
        <link rel="canonical" href="https://acme.example/" />
      </Helmet>
      <Outlet />
    </>
  )
}
function PricingPage() {
  return (
    <Helmet>
      <title>Pricing | Acme</title>
      <link rel="canonical" href="https://acme.example/pricing" />
    </Helmet>
  )
}

// After: one component, rendered once per route, built from one meta object.
type RouteMeta = {
  title: string
  description: string
  path: string
  imagePath: string
  indexable: boolean
}

export function RouteHead({ meta }: { meta: RouteMeta }) {
  const url = new URL(meta.path, SITE_ORIGIN).href
  const image = new URL(meta.imagePath, SITE_ORIGIN).href
  return (
    <>
      <title>{`${meta.title} | ${SITE_NAME}`}</title>
      <meta name="description" content={meta.description} />
      <link rel="canonical" href={url} />
      {meta.indexable ? null : <meta name="robots" content="noindex" />}
      <meta property="og:type" content="website" />
      <meta property="og:url" content={url} />
      <meta property="og:title" content={meta.title} />
      <meta property="og:description" content={meta.description} />
      <meta property="og:image" content={image} />
    </>
  )
}
```

`SITE_ORIGIN` and `SITE_NAME` live in one config module; `SITE_ORIGIN` comes from the validated `VITE_SITE_ORIGIN`. `meta.path` is the route's canonical identity (slug, page number), never `location.pathname + location.search`. Layouts render no head tags. With react-helmet-async, the same rule applies: one `<Helmet>` per route, nothing in layouts.

On the client this is enough for Google's rendered DOM. It is not enough for share previews or non-rendering crawlers, which is what prerendering is for.

## Prerendering public routes

Prerender a route when it must rank, be shared with a preview, or show its canonical to crawlers that do not render. Leave routes client-only when they sit behind login or depend on wallet or user state.

React Router's `prerender` option lives in `react-router.config.ts` and is documented as framework mode only; data and declarative mode apps cannot use it. Framework mode is a Vite plugin from `@react-router/dev` with its own route module conventions. For an app using React Router in declarative or data mode, adopting it is a migration, not a config change. The options below work without it.

### Option A: render with React at build time

Build a prerender entry with Vite's SSR build (`vite build --ssr src/entry-prerender.tsx --outDir dist-prerender`) after the client build, then run a Node script that renders each public path and writes the HTML into `dist`.

- Routing: declarative mode wraps the app in `StaticRouter location={path}`; data mode uses `createStaticHandler(routes)`, `query(request)`, `createStaticRouter(dataRoutes, context)` and `StaticRouterProvider`, all from `react-router`. In data mode, `query` returns a `Response` when a loader redirects or throws one; handle that instead of writing a page.
- Rendering: `prerenderToNodeStream` from `react-dom/static` waits for all Suspense boundaries and data before resolving, which is what a build wants. `renderToString` does not wait.
- Data: with TanStack Query, create a new `QueryClient` per page, fill it, render, then embed `dehydrate(queryClient)` and wrap the client tree in `<HydrationBoundary state={...}>`. TanStack's guide says never embed it with `JSON.stringify`, use an escaping serializer, and set a default `staleTime` above 0 so the client does not refetch immediately.
- Head tags: when React renders only the `#root` fragment, there is no React-rendered `<head>` in its output, so do not assume its `<title>`, `<meta>` and `<link>` end up in the document head. Either render the whole document with React (`<html>`, `<head>`, `<body>`), take the built script and CSS URLs from Vite's manifest (`build.manifest: true`, written to `.vite/manifest.json`), and hydrate with `hydrateRoot(document, <Document />)`; or inject the app markup into the `index.html` template and write the head tags into the template's `<head>` yourself from the same `RouteMeta`. With the template approach, check where React put any `<title>`, `<meta>` and `<link>` it rendered: a canonical in `<body>` is ignored by Google.
- Mounting: if the markup came from React's server renderer and the first client render produces identical output, use `hydrateRoot`. If you call `createRoot(...).render()` on prerendered markup, React clears it and rebuilds the DOM, which the docs say is slower, resets focus and scroll, and may lose input.

Hydration mismatches in this stack come from the first client render differing from the build: reading `window`, `localStorage` or `matchMedia` during render, `Date.now()` or random values, a TanStack query that starts in `pending` because the cache was not dehydrated, and wallet state from wagmi, which exists only in the browser. On React 19.3 and later, mark browser-only and wallet-dependent subtrees with `use(browser())` (from `react-dom`) inside a `<Suspense>` boundary: the prerender keeps the fallback, and the client renders the subtree after hydration without a recoverable error. On earlier React, render them after mount with the two-pass pattern (a state flag set in an effect). Either way, keep them out of anything that must rank. Treat mismatch warnings as bugs; React recovers by client rendering, at a cost.

### Option B: snapshot the built app with a headless browser

Serve the built `dist` (`vite preview`), load each public path in Playwright, wait for the route's content, and save `document.documentElement.outerHTML` as that route's file. No SSR build, and the output is exactly what the client renders. The costs: the snapshot contains whatever the browser had (wallet prompts, consent banners, loading states, logged-out variants), it can capture an error state if the API was down during the build, and the snapshot is not React server output, so mount with `createRoot` unless you have proved the first client render matches. Fail the build when a snapshot lacks the route's `<h1>` or canonical rather than shipping an empty page.

### Head tags in the prerendered file versus React at runtime

After prerendering, the file's `<head>` has the route's tags, and React renders `RouteHead` again on mount. If both exist the head has two canonicals and two titles. Pick one of:

- Whole-document render with `hydrateRoot(document, ...)`: React owns the head from the start, so there is one set.
- Template injection: mark the injected tags (for example `data-prerendered-head`) and remove them in the client entry before the first render, so `RouteHead` becomes the single owner with identical values. Google's guidance allows JavaScript to set the canonical as long as it is the same value as the original HTML and the only one on the page.

Either way, run the audit's rendered-DOM check on a prerendered route and a client-only route before calling it done.

### Output file names and trailing slashes

Static hosts map file names to URLs. On Cloudflare Workers static assets with the default `html_handling` (`auto-trailing-slash`), `pricing.html` is served at `/pricing` and `pricing/index.html` at `/pricing/`, with the other form redirected. Write files in the form your canonicals use, or every canonical points at a redirect. If the home route is prerendered, `index.html` is no longer a neutral shell; keep the unrendered shell as a separate file for client-only routes, or client-only URLs inherit the home page's canonical and Open Graph tags.

## Status codes from a static host

The SPA fallback is a soft-404 generator. Cloudflare Workers with `not_found_handling: "single-page-application"` serve `index.html` with 200 for any unmatched path; Cloudflare Pages assumes an SPA and does the same when there is no top-level `404.html`; nginx `try_files $uri /index.html` does the same. A `path="*"` route that renders "Page not found" still arrives with 200.

Make the host know the route table:

- Every public route is a prerendered file: use the host's 404 page. On Workers static assets, `not_found_handling: "404-page"` serves the nearest `404.html` with status 404. On nginx, `try_files $uri $uri.html $uri/index.html =404;` with `error_page 404 /404.html;`, plus a `location` that serves the shell for each client-only prefix.
- Some routes are client-only: put a Worker in front that serves the shell for paths matching the client route table and the 404 page with status 404 for everything else. Generate the matcher from the same route definitions React Router uses so they cannot drift.

```ts
import { isClientRoute } from '../src/routes/client-routes'

const SHELL_PATH = '/app-shell'
const NOT_FOUND_PATH = '/404'

interface Env {
  ASSETS: Fetcher
}

export default {
  async fetch(request, env): Promise<Response> {
    const { pathname } = new URL(request.url)
    if (isClientRoute(pathname)) {
      return env.ASSETS.fetch(new URL(SHELL_PATH, request.url))
    }
    const notFound = await env.ASSETS.fetch(new URL(NOT_FOUND_PATH, request.url))
    return new Response(notFound.body, { status: 404, headers: notFound.headers })
  },
} satisfies ExportedHandler<Env>
```

Leave `not_found_handling` at its default so unmatched requests reach the Worker; files that exist (assets, prerendered pages, `robots.txt`) are served directly without invoking it. Rules in `_headers` and `_redirects` are not applied to responses the Worker returns, so set any `X-Robots-Tag` or cache header on those responses in the Worker.

A pattern match does not prove a record exists: `/products/:slug` matches a slug that was deleted. Either generate the set of valid slugs at build time, or have the Worker ask the API. If that lookup times out or fails, return the shell with 503 rather than 404; a false 404 drops a real page, a 503 only slows crawling.

When the host cannot do any of this, Google documents two client-side fallbacks: a JavaScript redirect to a URL that the server answers with 404, or rendering `<meta name="robots" content="noindex">` on the not-found view (React 19 hoists it). Render that noindex only on a confirmed not-found (the API answered 404), never on a timeout or 5xx, or a transient outage removes real pages from the index.

## Share previews for data-driven routes

When a route has too many records to prerender (one page per token, listing or profile) but links to it get shared, the shell's tags are wrong for every one of them. A Worker in front of the assets can fetch the record from the API and rewrite the shell's head with `HTMLRewriter` (`setAttribute` on existing tags). `append` and `prepend` escape content unless you pass `{ html: true }`; never pass API data with `{ html: true }`. Do this for every request to that route, not only for bot user agents, so users and crawlers get the same document. Bound the API call with a timeout and serve the unmodified shell when it fails.

## Sitemaps with vite-plugin-sitemap

`vite-plugin-sitemap` (built on `sitemap-ts`) runs in Vite's `closeBundle` hook, globs `**/*.html` in `outDir`, adds `dynamicRoutes`, removes `exclude`, and writes `sitemap.xml` and, by default, `robots.txt`. From its source:

- `hostname` defaults to `http://localhost/`. Unset, every `<loc>` and the `Sitemap:` line in robots.txt point at localhost.
- `outDir` is the plugin's own option (default `'dist'`, resolved from the working directory), not Vite's `build.outDir`. Set it explicitly whenever the build writes anywhere else, such as `dist/client` in a Worker setup, or the plugin scans and writes the wrong folder.
- A client-rendered build has one HTML file, so the scan finds only `/`. Every public route must come from `dynamicRoutes`. HTML written by a prerender script after `vite build` finishes is not scanned either.
- `404.html` and any shell file are scanned and listed as `/404` and `/app-shell`. List them in `exclude`.
- `exclude` is exact string equality on normalized routes: no globs, no prefixes, and routes lose their trailing slash (`/blog/` becomes `/blog`).
- Routes are passed through `path.parse`, so a dot in the last segment is treated as an extension: `/docs/v1.2` becomes `/docs/v1`.
- `dynamicRoutes` is not deduplicated against scanned routes; listing `/` there emits it twice.
- `lastmod` defaults to `new Date()` at build, and a route missing from a per-route `lastmod` map also gets the build time. There is no way to omit it, so feed real modification dates for every route.
- `generateRobotsTxt` defaults to `true` and writes `robots.txt` with `writeFileSync` at the end of the build, replacing the one Vite copied from `public/`. The default rules are `User-agent: *` / `Allow: /`. Set it to `false` if robots.txt is maintained by hand or per environment.
- `priority` and `changefreq` are emitted with defaults; Google ignores both.

In the same `vite.config.ts` as `requireOrigin` above:

```ts
import Sitemap from 'vite-plugin-sitemap'
import { indexableRoutes } from './src/seo/indexable-routes'

const SCANNED_ROUTES = new Set(['/'])
const NON_PAGE_ROUTES = ['/404', '/app-shell']

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_')
  const siteOrigin = requireOrigin(env.VITE_SITE_ORIGIN, 'VITE_SITE_ORIGIN')
  const routes = indexableRoutes()
  return {
    plugins: [
      react(),
      Sitemap({
        hostname: siteOrigin,
        dynamicRoutes: routes.map((r) => r.path).filter((p) => !SCANNED_ROUTES.has(p)),
        exclude: NON_PAGE_ROUTES,
        lastmod: Object.fromEntries(routes.map((r) => [r.path, r.updatedAt])),
        generateRobotsTxt: false,
      }),
    ],
  }
})
```

`indexableRoutes()` is the same list the prerender and the host's route table use, filtered to canonical, indexable pages, with `updatedAt` from the content source. If the routes come from an API at build time, the sitemap lists what existed at deploy; decide how often the site rebuilds.

## Caching and deploys

Vite fingerprints built JS and CSS file names, which is what Google asks for, since its renderer may ignore cache headers. Serve `index.html` and every prerendered HTML file with `Cache-Control: no-cache` (Vite's build guide recommends it for HTML) so neither users nor crawlers pair new HTML with old chunks. Hosts that delete the previous deploy's assets break lazy-loaded routes for anyone holding old HTML; Vite emits `vite:preloadError` when a dynamic import fails, which the app can handle by reloading. A crawler that rendered a route with a missing chunk saw an empty page, so keep the previous deploy's assets available for a while after each release where the host allows it.

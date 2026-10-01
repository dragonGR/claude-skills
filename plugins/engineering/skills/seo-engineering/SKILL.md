---
name: seo-engineering
description: Technical SEO: indexing, noindex and robots.txt, canonicals, redirects, status codes, hreflang, sitemaps, structured data, Core Web Vitals, AI crawlers, and how React + Vite SPAs render for Google and link-preview bots. Load it before building or reviewing any public page, route, redirect or head tag, and when pages drop out of Google or the wrong URL ranks.
license: MIT
metadata:
  author: Alex Tsanis
---

# SEO engineering

A crawler is an unauthenticated, stateless, non-interactive client that trusts your status codes, headers and `<head>` more than your intentions. Most SEO incidents are ordinary engineering bugs: a 200 on a missing page, a header set on the wrong environment, a canonical built from the request, a rule in one layer that another layer contradicts. Ranking advice beyond this is mostly guesswork; do not present it as fact.

Judge every change by what a crawler receives: the status line, response headers and raw HTML for the exact URL, then the rendered DOM. Fetch them (see `references/audit.md`) instead of reasoning from the component tree.

## Failure catalogue

Each entry: what it looks like, why it breaks, what to do instead.

### Index control

**Staging or preview indexed.** `staging.example.com`, a preview deployment URL or a `pr-123.` host shows up in Google, usually because the robots rule depends on a variable that nobody set on that environment, or a link from production or a public issue exposed it. Put non-production environments behind authentication (HTTP basic auth or SSO returning 401), which Google documents as the way to keep private content out of Search. As a second layer, send `X-Robots-Tag: noindex` from the server or CDN for every response, keyed on an explicit deployment environment value or hostname where unknown means noindex. A static SPA bundle cannot do this for itself: the host config decides. Cloudflare Pages adds the header to preview deployments but not to the production `<project>.pages.dev` host, which serves a public duplicate of the site until it is redirected. If it is already indexed: keep it crawlable with noindex or return 401/404 until the URLs drop, and use the Search Console removals tool for speed; a removal lasts about six months and is not a fix on its own.

**noindex hidden behind a robots.txt disallow.** `Disallow: /search` plus `<meta name="robots" content="noindex">` on the same pages. Google never fetches a disallowed URL, so it never sees the noindex, and the URL can still be indexed from links. Pick one: to remove from the index, allow crawling and send noindex; once they have dropped, add the disallow only if crawl volume matters. The same trap applies to staging: `Disallow: /` on an already indexed staging host freezes it in the index.

**noindex removed or added with JavaScript.** The server HTML says `noindex` and client code removes it after hydration, or the reverse. Google may skip rendering when the initial HTML has noindex, so removing it in JS may not work. Robots directives belong in the server response.

**Non-HTML files without a header.** PDFs, price sheets, exported CSVs and images get indexed because there is nowhere to put a meta tag. Send `X-Robots-Tag: noindex` on those responses from the server or CDN. A noindex meta on the HTML page that links to the PDF does nothing for the PDF.

**robots.txt that is not a robots file.** An SPA catch-all serves `index.html` with 200 for `/robots.txt`, which parses as no rules at all. Or `/robots.txt` returns 5xx or 429 during an incident, and Google pauses crawling of the whole site for up to 12 hours, then falls back to the last cached copy for up to 30 days. Or it returns 4xx (other than 429), which Google treats as no restrictions. Serve it as a static file with `text/plain` and monitor its status separately. Google reads only the first 500 KiB.

**Wrong user-agent group.** Adding `User-agent: GPTBot` with a single `Disallow` means GPTBot now obeys only that group; every rule under `User-agent: *` stops applying to it. The `*` group is not merged in. Repeat the shared rules in each specific group.

### Status codes and redirects

**Soft 404.** A missing product, an empty search, an unpublished article or a "no longer available" page renders a friendly message with 200. Google reports these as soft 404s and they waste crawl on thin pages. Return 404 (or 410) for things that do not exist, from the server, before streaming starts. An out-of-stock product that still exists and has content is not a 404.

**SPA returning 200 for every route.** `try_files $uri /index.html`, a Netlify `/* /index.html 200` rule, Cloudflare Workers `not_found_handling: "single-page-application"`, a Cloudflare Pages project with no top-level `404.html` (Pages then assumes an SPA), or an Express `app.get('*')` fallback. Every typo URL and every deleted page is a 200, and a React Router `path="*"` route that renders "Page not found" cannot change that. The host should know the route table and return 404 for unknown paths: a 404 page when every public route is a prerendered file, or a Worker that serves the shell only for known client routes. If it cannot, Google documents two client-side fallbacks: a JavaScript redirect to a URL that the server answers with 404, or rendering `<meta name="robots" content="noindex">` on the not-found view. Only render that noindex when the API confirmed the record is gone, never on a timeout or 5xx. Patterns in `references/react-vite-spa.md`.

**Redirect chains and loops.** `http://example.com/Shoes` to `https://example.com/Shoes` to `https://www.example.com/Shoes` to `https://www.example.com/shoes/`. Googlebot follows up to 10 hops, and each layer (CDN, load balancer, app, static host trailing-slash handling) adds its own. Loops usually come from two layers disagreeing, such as the app redirecting to https because TLS terminates at the proxy and it ignores `X-Forwarded-Proto`, or the CDN adding a slash that the framework strips. Redirect every variant straight to the final URL in one hop, and test with `curl -sIL` from outside the CDN.

**Temporary redirect for a permanent move.** A Cloudflare `_redirects` line without a status code sends 302; Express `res.redirect()` and Django `redirect()` default to 302. Google treats 302, 303 and 307 as temporary and 301 and 308 as permanent. Use 301 or 308 for migrations, slug changes and domain moves, and keep them in place for as long as old links exist.

**Redirects done by the router.** A moved page handled with React Router `<Navigate>` or a loader `redirect()` is a JavaScript redirect: no HTTP status, and Google only sees it if rendering succeeds. Google recommends JavaScript redirects only when server and meta refresh redirects are impossible. Permanent moves go in the host config (`_redirects` with an explicit 301, nginx, a Worker).

**Maintenance page with 200.** The maintenance or error template returns 200, and Google may index "We'll be back soon" as the content of every URL. Return 503. 5xx and 429 slow crawling; URLs that keep returning 5xx are eventually dropped, so keep outages short. Do not use 401 or 403 to slow crawlers: they are treated like 404 and do not affect crawl rate.

**Geo or language redirects.** Middleware redirects by IP country or `Accept-Language`. Googlebot crawls mostly from US IPs, sends no `Accept-Language`, and does not keep cookies, so it only ever sees the US or English version and the other locales go uncrawled. Give every locale its own URL, annotate with hreflang, and suggest a switch with a dismissible banner instead of forcing a redirect. Never cloak by user agent.

### Canonicals and duplicates

**Canonical pointing at a URL that is not the final page.** The canonical target redirects, returns 404, has noindex, uses `http://`, or has a different trailing slash from the served URL. Each one contradicts the canonical with another signal. A canonical target must be the absolute URL that returns 200, is indexable and declares itself canonical.

**Canonical built from the request.** `canonical = req.protocol + '://' + req.headers.host + req.originalUrl`. Behind a proxy that yields `http://`, internal hostnames, a spoofed `Host`, and every `?utm_`, `?sort=` and `?sessionid=` variant claiming to be canonical. Build it from configured origin plus the route's identity plus only the parameters that select distinct content.

**Canonical injected or changed client-side.** A React `<link rel="canonical">`, a `<Helmet>` or a `useEffect` sets the canonical, or overrides the one in the served HTML. Google's guidance is not to change the canonical with JavaScript, and if JavaScript must set it, to set the same value as the original HTML and keep it the only `rel="canonical"` on the page; a different value is a conflicting signal, and crawlers that do not render never see it. Put the canonical in the served HTML (prerendered for public routes) and make the client render the identical value.

**Canonical outside `<head>`.** Google accepts `rel="canonical"` only in the `<head>`. In the server HTML, an element that is not allowed in the head (an `<img>` or `<iframe>` pixel pasted into the head template, a `<div>`, stray text) makes the HTML parser end the head early, and every canonical, hreflang and robots tag after it is now in the body. Keep those tags first in the head, and check the parsed DOM rather than the template.

**Pagination canonicalized to page 1.** `/shoes?page=3` has `rel="canonical" href="/shoes"`. Google's guidance is the opposite: each page gets its own canonical. Pointing them all at page 1 declares the deeper pages duplicates, so items linked only from those pages lose their discovery path. Link pages sequentially with plain `<a href>`. Google no longer uses `rel="next"`/`rel="prev"`.

**Host, protocol and slash variants all serving 200.** `http` and `https`, `www` and apex, `/about` and `/about/`, `/About` and `/about` all return 200 with the same content. Pick one form per dimension, 301/308 the others in one hop, and use the same form in canonicals, sitemaps, hreflang and internal links. Mixed-content `http://` links in templates, sitemaps or canonicals on an https site are the same bug.

**noindex used to deduplicate.** Near-duplicate variants get noindex instead of a canonical. Google recommends against this within a site because it blocks the page outright rather than consolidating signals. Canonicalize duplicates; noindex is for pages that should not be in Search at all.

### International

**hreflang that is not reciprocal or omits itself.** The German page lists English and French but not itself, or English lists German while German's set was generated from a different locale list. Google requires every version to list itself and all others, and ignores a pair that does not point both ways. Generate the whole set from one locale table, identically on every version, including `x-default` for the selector or fallback page.

**Invalid codes.** `en-UK` (Google ignores the reserved `UK` region, so it is plain `en`; the code is `en-GB`), region alone (`hreflang="us"`), `jp` for Japanese (`ja`), `zh-CN` written as `cn`, or underscores (`en_GB`). Language is ISO 639-1; the optional region is ISO 3166-1 alpha-2. Validate codes against a list at build time instead of trusting CMS input.

**Cross-language canonical.** `/de/schuhe` declares `https://example.com/en/shoes` as canonical. Translated pages are not duplicates, and this tells Google the German page is a copy of the English one, which undoes the hreflang. Each locale canonicalizes to itself. hreflang targets are the canonical, 200, indexable URLs; an annotation that lands on a redirect or a page canonicalized elsewhere contradicts your other signals.

### Faceted navigation and crawl traps

**Unbounded filter combinations.** `?color=red&size=9&sort=price&brand=a&brand=b` with every facet a crawlable link multiplies into millions of URLs; add a calendar with a "next month" link forever or session ids in URLs and the crawl never ends. Crawl budget goes to junk while new products wait. Decide which facet pages deserve to rank (usually a small set with real search demand), give those clean URLs with self-canonicals and put them in the sitemap. For the rest, Google documents: disallow the facet parameters in robots.txt, or keep filters in URL fragments, and return 404 for filter combinations with no results. Keep parameter order and names fixed so the same filter set has one URL. A canonical on facet pages only slowly reduces crawling; it does not stop it.

### Rendering and links

**Links that are not links.** `<div onClick={() => navigate(url)}>`, `<a onClick>` without `href`, `<a href="javascript:void(0)">`, `<button>` for navigation, or routes in `#/fragments` (`createHashRouter`, `HashRouter`). Google crawls `<a>` elements with an `href` and cannot reliably resolve fragment URLs. Use React Router's `<Link>`, which renders a real `<a href>`, and History API routing (`createBrowserRouter` or `BrowserRouter`).

**Content that needs interaction or state.** Tabs, "read more", reviews or specs fetched only on click; lazy content triggered by `scroll` events; content gated on a cookie banner choice, a geolocation prompt, `localStorage` or login. Google does not click or scroll, declines permission prompts, and clears cookies and storage between page loads. Content that must rank is in the initial HTML or loads when it enters the viewport (`IntersectionObserver`). Infinite scroll needs a paginated URL behind every chunk, linked with `<a href>`.

**Resources blocked in robots.txt.** `Disallow: /api/` or `/assets/` (Vite's default `build.assetsDir`) or `/static/` blocks the JS or data the page needs to render, and Google won't render JavaScript from blocked files. Allow whatever the rendered page fetches, including the API origin's robots.txt when the SPA calls a separate API host.

### React + Vite SPAs

A client-rendered SPA serves one `index.html` for every route. Google renders it later in a queue; link-preview bots and crawlers that do not run JavaScript only ever see that file. Details, code and hosting patterns: `references/react-vite-spa.md`.

**Route-specific tags in the shell.** `index.html` carries `<link rel="canonical" href="https://example.com/">`, `og:url`, `og:image` or a description. The fallback serves it for every URL, so every route declares the home page canonical and every shared link previews as the home page; a per-route canonical rendered later by React is a second, conflicting one. Keep only tags that are true for every URL in the shell.

**Share previews from the shell.** Product, article or token pages are shared and show the site's default card. Slackbot fetches only the start of the HTML with Range requests to read Open Graph and Twitter Card tags, and Facebook's crawler reads Open Graph only in the first 1 MB; neither is documented to run your bundle. Prerender shareable routes, or rewrite the shell's head at the edge for data-driven routes, for every request rather than only for bots.

**noindex in the shell.** `<meta name="robots" content="noindex">` in `index.html` to hide app routes, removed by React on public routes. Google may skip rendering a page whose served HTML has noindex, so the removal never runs. Keep noindex out of a shell that any indexable route uses; send it per route or per path from the host.

**Two owners of the head.** A layout and a page both render `<title>` or `<link rel="canonical">`. React 19 puts every concurrently rendered `<title>` in the head ("behavior of browsers and search engines is undefined") and documents no deduplication for canonicals or meta tags. react-helmet-async 3 on React 19 renders plain elements per `<Helmet>` with no merging, so the old "innermost Helmet wins" assumption is gone, and its `HelmetProvider` `context` stays empty, so a prerender script reading `helmetContext.helmet` writes no tags. One component per route owns the head; layouts render none.

**Prerendered markup thrown away or mismatched.** The build writes per-route HTML, then the client calls `createRoot(...).render()` on it: React clears the markup and rebuilds it. Or it calls `hydrateRoot` while the first client render differs (TanStack Query cache not dehydrated so the page starts in `pending`, `window` or `localStorage` read during render, wagmi wallet state, `Date.now()`). Use `hydrateRoot` only when the first client render matches the prerender; render browser-only state after mount.

**Origin or environment baked into the wrong build.** `%VITE_SITE_ORIGIN%` in `index.html` is left literally in the output when the variable is undefined. `vite build` defaults to `production` mode, so a staging build made without `--mode` carries the production origin and `import.meta.env.PROD === true`. Validate origin variables in `vite.config.ts` and fail the build; decide indexability per hostname at the host, not in the bundle.

**Client-only rendering for pages that should rank.** It can be indexed, but it goes through a render queue and every failure mode above applies. Prerender indexable and shareable pages. Client rendering is fine for the app behind login and for wallet- or user-specific views. React Router's `prerender` option is framework mode only; a `react-router-dom` app in declarative or data mode prerenders with React's own `react-dom/static` APIs or a headless-browser snapshot at build time.

### Sitemaps

**Sitemap lists URLs you do not want indexed.** Redirected URLs, 404s, noindex pages, parameter variants, `http://` URLs, the other trailing-slash form, or staging hosts because the base URL came from the wrong environment. A sitemap that disagrees with canonicals is a conflicting signal. Generate it from the same query that decides indexability, from configured origin, with absolute URLs.

**lastmod that lies.** `lastmod` set to build time or `new Date()` on every URL. Google uses `lastmod` only when it is consistently and verifiably accurate, so a fake one teaches it to ignore yours. Use the content's real modification time, or omit it. Google ignores `priority` and `changefreq`.

**Limits and staleness.** A sitemap holds at most 50,000 URLs or 50 MB uncompressed; split with a sitemap index. A sitemap generated once at build lists only what existed at deploy time.

**vite-plugin-sitemap defaults.** `hostname` defaults to `http://localhost/`; `lastmod` defaults to the build time for every route, including routes missing from a per-route map; `generateRobotsTxt` defaults to `true` and overwrites the `robots.txt` copied from `public/` with `Allow: /`; it scans built HTML, so an SPA build yields only `/` plus any `404.html` or shell file. Set `hostname` from validated config, list routes in `dynamicRoutes`, exclude non-pages, pass real dates, and turn robots generation off when robots.txt is maintained elsewhere.

### Metadata and content parity

**Duplicate or boilerplate titles.** Every product under a category renders `Products | Shop`, or the title falls back to the site name when data fails to load. Google rewrites title links it finds boilerplate, half-empty or inaccurate. Generate each title from that page's own data, and treat a missing title as a render error, not a fallback.

**Mobile and desktop differ.** Google indexes the mobile version. Server-side UA sniffing that serves a lighter mobile page without the specs, reviews, structured data or robots tags of the desktop version removes them from the index.

**Canonical built from the router location.** `href={origin + location.pathname + location.search}` in a head component makes every `?utm_`, `?sort=` and `?ref=` variant claim to be canonical, and `window.location.origin` makes staging and preview hosts canonical on themselves. Build it from the configured origin plus the route's identity (slug, id, page number).

### Structured data

**JSON-LD that does not match the page.** `Product` markup with the list price while the page shows the sale price, an `aggregateRating` that is not displayed, reviews that do not exist, `availability: InStock` from a stale cache. Google's policies forbid marking up content that is not visible to readers, and violations can lead to a manual action. Build JSON-LD from the same view model the page renders, in the same request.

**Script injection through JSON-LD.** `<script type="application/ld+json">${JSON.stringify(data)}</script>` with CMS or user text in `data`. `JSON.stringify` does not escape `<`, so a product name containing `</script><script>...` runs in every visitor's browser. Escape every `<` after stringifying (`.replace(/</g, '\\u003c')` in JavaScript, which emits the JSON unicode escape), or use the template engine's JSON-for-script helper. Never build JSON-LD by string concatenation. In React the same escape goes into `dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c') }}`, because a prerender writes that string into static HTML.

**Product markup only in the client render.** Google processes JSON-LD that JavaScript adds to the DOM, but warns that dynamically generated markup makes Shopping crawls less frequent and less reliable, which matters for price and availability. Prerender product pages with their JSON-LD in the served HTML.

**Markup for rich results that no longer exist.** Google no longer shows FAQ or HowTo rich results, and it has retired several other types (Course Info, Estimated Salary, Vehicle Listing among them). Check the current structured data gallery before promising a rich result. Valid markup makes a page eligible, never guaranteed.

### Core Web Vitals

Google's "good" thresholds at the 75th percentile of page loads: LCP at most 2.5 s, INP at most 200 ms, CLS at most 0.1. Field data decides; lab runs help diagnose.

**Lazy-loaded LCP image.** `loading="lazy"` on the hero, a framework image component that is lazy by default, or the hero set as a CSS background or inserted by JavaScript so the browser discovers it late. Load the LCP image eagerly with `fetchpriority="high"` in the initial HTML; preload it if it can only be found through CSS. In a client-rendered route the hero `<img>` does not exist until the bundle, the lazy route chunk and often an API call have finished, so no attribute fixes discovery; prerendering the route puts the element in the HTML.

**Layout shift from fonts and ads.** A web font swaps in with different metrics and reflows the text; an ad, cookie banner or embed pushes content down when it loads. Reserve space for ad slots and embeds at their largest likely size. For fonts, use a metric-matched fallback (`size-adjust`, `ascent-override`, `descent-override`) or `font-display: optional`.

### AI crawlers

**Treating robots.txt as access control.** Vendors document separate tokens for training, search and user-initiated fetches, and some user-initiated fetchers (ChatGPT-User, Perplexity-User) say robots.txt may not apply. User agents are also spoofed. If content must not be fetched, enforce it at the edge or with authentication, and verify claimed crawlers against the vendor's published IP ranges or DNS.

**Blocking the wrong token.** `Google-Extended` controls use for Gemini training and grounding; it does not affect Search inclusion or ranking. AI Overviews and AI Mode are governed by Googlebot access and snippet controls (`nosnippet`, `data-nosnippet`, `max-snippet`, `noindex`), so blocking Googlebot to opt out of AI features removes the site from Search. Google states that no AI-specific text files such as `llms.txt` are needed to appear in those features. Token reference: `references/crawl-control.md`.

## Decision rules

- **Keep a URL out of Search:** it should not exist, return 404 or 410. It is private, require authentication. It exists but should not be listed, allow crawling and send noindex (header for non-HTML). Use robots.txt only to save crawl on URLs that are not indexed yet.
- **Redirect or canonical:** users should never land on the old URL, redirect (301/308). Both URLs must keep working for users (tracking parameters, print views, facet variants), canonical.
- **Facet or parameter page:** it has search demand and unique content, give it a clean URL, self-canonical and a sitemap entry. Otherwise keep it out of crawling (robots.txt parameter rules or fragments) and return 404 for empty combinations.
- **Rendering:** indexable or shareable pages are prerendered with title, canonical, robots, hreflang, Open Graph, structured data, main content and links in the served HTML. Client rendering only for pages that do not need to rank or preview (logged-in app, wallet views).
- **Prerender method for a Vite SPA:** render with React at build time (`react-dom/static` plus `StaticRouter` or `createStaticHandler`) and `hydrateRoot` when the first client render can match the build. Snapshot with a headless browser and mount with `createRoot` when it cannot, or when the site is small and an SSR build is not worth it. Moving to React Router framework mode only for its `prerender` option is a migration, decide it as one.
- **Head tag library:** React 19's built-in `<title>`, `<meta>` and `<link>` are enough when one component owns the head per route. react-helmet-async 3 on React 19 adds `titleTemplate`/`defaultTitle` and html/body attributes, not deduplication; keep it only where those are used.
- **Locale handling:** separate URLs per locale with hreflang; a banner suggests switching. Automatic redirects only from a locale-neutral root, and only if every locale stays reachable without cookies or headers.
- **AI crawlers:** decide per purpose (training, AI search, user fetches) using each vendor's documented token, and write it as its own group with the shared rules repeated.

## Review checklist

- Does every indexable template return 200, and do missing, deleted and unpublished items return 404 or 410 from the server or host?
- Does the canonical come from configured origin and route identity, resolve in one hop to a 200 indexable URL, and appear in the served HTML `<head>`?
- In an SPA, does `index.html` carry only tags true for every URL, and does the rendered DOM of each route have exactly one `<title>`, one canonical and one set of Open Graph tags?
- Do shareable routes have their Open Graph tags in the served HTML, checked with a plain `curl` rather than a browser?
- Does the host return 404 for unknown paths instead of the SPA fallback with 200?
- Are robots directives in the server response, and is nothing that carries noindex also disallowed in robots.txt?
- Do non-production environments require authentication and send `X-Robots-Tag: noindex`, keyed so an unset environment fails closed?
- Does every non-canonical host, protocol, case and slash variant redirect to the final URL in one 301/308 hop?
- Are hreflang sets reciprocal, self-referencing, valid ISO 639-1 (plus optional ISO 3166-1 region) codes, pointed at canonical 200 URLs?
- Is every navigational element an `<a href>`, and is ranking content present without clicks, scrolling, cookies or storage?
- Does the sitemap list only canonical, indexable, 200, absolute URLs on the production origin with truthful `lastmod`, and did the sitemap plugin leave robots.txt alone?
- Is JSON-LD built from the rendered data and escaped for a script context?
- Is the LCP image eager with high fetch priority, and is space reserved for fonts, ads and embeds?
- Does robots.txt serve as `text/plain` with a monitored status, and does each specific user-agent group repeat the shared rules?
- Have you fetched the real URLs (status, headers, raw HTML, rendered DOM) for a sample from every template, rather than trusting the code?

## References

- `references/audit.md`: read when auditing a site or verifying a fix; curl-based checks for status, redirects, headers, canonicals and sitemaps, plus how to report findings.
- `references/crawl-control.md`: read when editing robots.txt, robots headers, staging protection, faceted navigation rules or AI crawler policy.
- `references/international.md`: read when adding locales, hreflang or any geo or language based routing.
- `references/react-vite-spa.md`: read when the site is a React + Vite SPA; what Google and preview bots see, the shell, per-route head tags with React 19 or react-helmet-async, prerendering, real 404s from static hosts, vite-plugin-sitemap and deploy caching.

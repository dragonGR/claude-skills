# Crawl control

Read this when editing robots.txt, robots meta tags or `X-Robots-Tag` headers, protecting staging, taming faceted navigation, or setting policy for AI crawlers.

## Which mechanism does what

| Goal | Mechanism | What it does not do |
|---|---|---|
| Stop crawling of a path | robots.txt `Disallow` | Does not remove or prevent indexing; hides any noindex on those URLs |
| Keep an HTML page out of the index | `<meta name="robots" content="noindex">` in server HTML | Only works if the page is crawlable |
| Keep a PDF, image, CSV or any response out of the index | `X-Robots-Tag: noindex` response header | Same: the URL must be crawlable |
| Keep private content private | Authentication (401) | Nothing else in this table is access control |
| Hide quickly from Google results | Search Console removals tool | Temporary, about six months |
| Consolidate duplicates | 301/308 redirect, or `rel="canonical"` | Canonical is a hint; Google may pick another URL |

Google ignores `noarchive` (the cached link it controlled is gone) and `nocache`, which Google Search does not use. `none` equals `noindex, nofollow`. `unavailable_after: <date>` drops a page after a date (useful for events and expiring listings).

## robots.txt behavior that bites

- A crawler obeys exactly one group: the most specific `User-agent` that matches it. `*` rules are not merged in. When you add a named group, copy every shared rule into it.
- Google supports `*` and `$` in paths. `Disallow: /*?*sort=` blocks any URL with a `sort` parameter; `Disallow: /*.pdf$` blocks URLs ending in `.pdf`.
- Status of `/robots.txt` itself: 2xx is parsed; 4xx other than 429 means no restrictions; 5xx or 429 pauses crawling for up to 12 hours, then Google uses its cached copy for up to 30 days, then treats the site as unrestricted if it is otherwise reachable. Google follows at least five redirect hops for robots.txt and caches it for up to 24 hours, so a fix is not instant.
- Only the first 500 KiB is read.
- Each host and protocol has its own robots.txt. `https://shop.example.com/robots.txt` does not cover `https://example.com`.
- Blocking `/api/`, `/assets/` (Vite's default build output for JS and CSS), `/static/` or a JS CDN path stops Google rendering pages that need those files. Disallow API paths only when no public page fetches them during render. When the SPA calls an API on another host, that host's robots.txt governs those fetches.
- Build tools can write robots.txt for you. `vite-plugin-sitemap` does by default (`generateRobotsTxt: true`): at the end of the build it overwrites the file Vite copied from `public/` with `User-agent: *` / `Allow: /` and a `Sitemap:` line built from `hostname`, which defaults to `http://localhost/`. Turn it off or configure it, and diff `dist/robots.txt` in CI.

## Staging and preview environments

Layer the protection. Authentication is the control; the header catches the day someone turns auth off for a demo.

```nginx
server {
    server_name staging.example.com;

    auth_basic           "staging";
    auth_basic_user_file /etc/nginx/staging.htpasswd;

    add_header X-Robots-Tag "noindex, nofollow" always;

    location /assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        add_header X-Robots-Tag "noindex, nofollow" always;
    }
}
```

Two nginx details matter. `always` adds the header on 4xx and 5xx responses too. And by default any `add_header` inside a `location` block discards every `add_header` inherited from the `server` block, so the robots header (and your security headers) silently vanish from that location unless you repeat them.

In application code, decide indexability from an explicit value and fail closed:

```ts
const INDEXABLE_ENVIRONMENTS = new Set(['production'])

export function isIndexable(env: NodeJS.ProcessEnv = process.env): boolean {
  return INDEXABLE_ENVIRONMENTS.has(env.DEPLOYMENT_ENV ?? '')
}
```

The bug to avoid is the inverse, `env.DEPLOYMENT_ENV !== 'staging'`: a new preview environment, a renamed variable or a missing value makes every non-production deploy indexable.

### Static hosts and Vite builds

A Vite SPA is a set of files, so the code above runs in whatever sits in front of them (a Worker, nginx), never in the bundle. Do not key robots tags on `import.meta.env.PROD` or `MODE`: `vite build` runs in `production` mode unless given `--mode`, so a staging build made with the default command reports production, and an artifact promoted from staging to production carries whatever was baked in. Decide per request hostname at the host, with an allowlist of production hostnames.

On Cloudflare:

- Pages preview deployments (`<hash>.<project>.pages.dev` and branch aliases) get `X-Robots-Tag: noindex` by default, but they are publicly reachable unless you enable an Access policy for previews in the project settings. The production `<project>.pages.dev` host is not a preview: it serves the production site with no noindex, a full duplicate of the custom domain. Cloudflare documents a Bulk Redirect with 301 from it to the custom domain.
- Workers static assets read a `_headers` file from the assets directory. Cloudflare's own example marks every `workers.dev` hostname noindex:

```text
https://:version.:subdomain.workers.dev/*
  X-Robots-Tag: noindex
```

- `_headers` and `_redirects` rules are not applied to responses generated by Worker code, even when the URL matches. If a Worker serves the shell or the 404 page (see react-vite-spa.md), set `X-Robots-Tag` in the Worker for non-production hostnames.

Check every host the site answers on (custom domain, `www`, `*.pages.dev`, `*.workers.dev`, preview aliases) with `curl -sI`; each one is either the canonical host, a one-hop 301 to it, or noindex behind authentication.

Do not serve `Disallow: /` on a staging host that is already indexed; it locks the indexed URLs in. Serve noindex (or 401) and let Google recrawl, then request removal if speed matters.

## Non-HTML files

```nginx
location ~* \.(pdf|csv|xlsx|docx)$ {
    add_header X-Robots-Tag "noindex" always;
}
```

Like the `/assets/` block above, this `location` drops every `add_header` set at `server` level, so repeat your security headers inside it.

On object storage behind a CDN, set the header as object metadata or in a CDN response rule; there is no server to do it. Check with `curl -sI` against the public URL, not the origin bucket.

To give a PDF a canonical (for example, the HTML version of the same document), send `Link: <https://example.com/guide>; rel="canonical"` as a header on the PDF.

## Faceted navigation

1. List the facet pages that deserve to rank. Give them path-based or fixed-order URLs, self-canonicals, internal links and sitemap entries.
2. Normalize the rest: one parameter order, no duplicate keys, lowercase values, and a 301 from non-normalized forms. Unknown parameters either redirect to the clean URL or return 404; they must not produce a new 200 page each.
3. Return 404 when a filter combination has no results.
4. Keep the long tail out of the crawl. Google documents disallowing the facet parameters in robots.txt, or moving filter state into the URL fragment (`/shoes#color=red`), which Google Search generally doesn't support for crawling and indexing. `rel="nofollow"` on filter links only helps if every link to those URLs carries it.
5. Watch for other infinite spaces: calendar "next" links with no end, session ids or tracking ids in paths, relative links that stack path segments (`/a/b/a/b/...`), and search result pages linked from other search result pages.

## AI crawlers

Each vendor separates purposes. Block per purpose, in its own group, with shared rules repeated.

| Token | Vendor | Purpose | Documented robots.txt behavior |
|---|---|---|---|
| `Google-Extended` | Google | Gemini training and grounding | Control token only, no separate fetcher; no effect on Search inclusion or ranking |
| `GPTBot` | OpenAI | Model training | Honors robots.txt |
| `OAI-SearchBot` | OpenAI | ChatGPT search results | Honors robots.txt |
| `ChatGPT-User` | OpenAI | Fetches triggered by a user | "robots.txt rules may not apply" |
| `ClaudeBot` | Anthropic | Model training | Honors robots.txt |
| `Claude-SearchBot` | Anthropic | Search indexing | Honors robots.txt |
| `Claude-User` | Anthropic | Fetches in response to a user query | Anthropic states its bots honor robots.txt |
| `PerplexityBot` | Perplexity | Perplexity search results, not training | Honors robots.txt |
| `Perplexity-User` | Perplexity | Fetches triggered by a user | "generally ignores robots.txt" |
| `Applebot-Extended` | Apple | Opt out of foundation model training | Control token only; disallowing it does not remove the site from Apple search |
| `CCBot` | Common Crawl | Open web crawl dataset | Honors robots.txt |

Google's AI Overviews and AI Mode are part of Search. They are controlled by Googlebot access and by `nosnippet`, `data-nosnippet`, `max-snippet` and `noindex`, not by `Google-Extended`. Google states that no AI-specific text files such as `llms.txt` are needed to appear in those features.

Example: allow search engines and AI search, refuse training.

```text
User-agent: *
Disallow: /cart/
Disallow: /*?*sort=

User-agent: GPTBot
User-agent: ClaudeBot
User-agent: Google-Extended
User-agent: Applebot-Extended
User-agent: CCBot
Disallow: /

Sitemap: https://example.com/sitemap.xml
```

Several `User-agent` lines before one rule set form a single group. The search bots (`OAI-SearchBot`, `Claude-SearchBot`, `PerplexityBot`) have no group of their own here, so they fall under `*` and get its rules. If you later give one of them its own group, copy the `*` rules into it.

## Verifying a crawler

User agents are trivially spoofed; do not grant anything (bypassing a paywall, a rate limit or a bot challenge) on the user agent alone.

- Google: reverse DNS on the IP must resolve to `googlebot.com`, `google.com` or `googleusercontent.com`, and a forward lookup of that name must return the same IP. Google also publishes CIDR lists (`common-crawlers.json`, `special-crawlers.json`, and the user-triggered fetcher files) for automated checks.
- Anthropic publishes its crawler IPs at `claude.com/crawling/bots.json`; Common Crawl at `index.commoncrawl.org/ccbot.json`. Check other vendors' bot pages for their current lists.

Refresh published lists on a schedule; hardcoding today's ranges goes stale.

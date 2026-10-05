# SEO audit

Read this when auditing a site, reviewing an SEO-affecting change end to end, or proving that a fix works on the deployed URL.

The code shows intent; the response shows what shipped. Host rules, CDN rules, build plugins, the SPA fallback and a stale cache all sit between them. Every finding needs a fetched response as evidence.

## 1. Build the URL sample

Pick two or three real URLs per template (home, category, facet, product, article, search, locale variants, paginated page 2+) plus these synthetic ones per template:

- a slug that cannot exist (`/products/zz-missing-<random>`)
- the same URL with `?utm_source=audit`
- `http://`, the other `www` form, the other trailing-slash form, an uppercase path
- a page number past the end (`?page=9999`)

## 2. Status, redirects and headers

```sh
check() {
  curl -sS -o /dev/null -L --max-redirs 10 \
    -w '%{http_code} hops=%{num_redirects} final=%{url_effective}\n' "$1"
}
check "https://example.com/products/zz-missing-$RANDOM"   # expect 404 or 410, hops=0
check "http://example.com/Shoes/"                          # expect 200, hops=1

curl -sSIL "http://example.com/shoes" | grep -iE '^(HTTP/|location:|x-robots-tag:|link:)'
```

What to look for: more than one hop to the final URL, any 302/307 on a permanent path, a 200 on the missing slug, `X-Robots-Tag` present in production or missing in staging, and a final URL that differs from the page's own canonical.

Run from outside your network and bypass nothing: CDN rules, WAF bot challenges and geo rules only show up on the public path. If a result looks wrong, fetch again with a cache-busting parameter and compare with the origin before calling it a defect.

## 3. Raw HTML head

```sh
curl -sS "$URL" | grep -ioE '<title>[^<]*</title>|<link[^>]+rel="?(canonical|alternate)"?[^>]*>|<meta[^>]+name="?(robots|googlebot)"?[^>]*>'
```

Check: exactly one canonical, absolute, equal to the URL you would put in the sitemap; robots meta consistent with the header; hreflang set complete; title specific to this page. If the canonical or title is missing here but present in the browser, it is client-injected.

grep sees the bytes, not the parse. To catch a head that closed early (canonical parsed into `<body>`), check in a real browser:

```js
import { chromium } from 'playwright'

const url = process.argv[2]
if (!url) throw new Error('usage: node check-head.mjs <url>')

const browser = await chromium.launch()
try {
  const page = await browser.newPage()
  const response = await page.goto(url, { waitUntil: 'networkidle' })
  const report = await page.evaluate(() => ({
    canonicalsInHead: [...document.head.querySelectorAll('link[rel="canonical"]')].map((l) => l.getAttribute('href')),
    canonicalsInBody: [...document.body.querySelectorAll('link[rel="canonical"]')].map((l) => l.getAttribute('href')),
    robotsInHead: [...document.head.querySelectorAll('meta[name="robots"]')].map((m) => m.getAttribute('content')),
    robotsInBody: document.body.querySelectorAll('meta[name="robots"]').length,
    titleCount: document.head.querySelectorAll('title').length,
    ogUrls: [...document.head.querySelectorAll('meta[property="og:url"]')].map((m) => m.getAttribute('content')),
    title: document.title,
    h1Count: document.querySelectorAll('h1').length,
  }))
  console.log(JSON.stringify({ status: response?.status(), ...report }, null, 2))
} finally {
  await browser.close()
}
```

A canonical in the body in this rendered view, when the served HTML had it in the head, means something earlier in the head broke the parse. In a prerendered SPA it can also mean React's own `<link>` output was written inside `#root` by the prerender. More than one title, canonical or `og:url` in the head means two components (or the shell and a component) both own the head.

### SPA shell checks

For a React + Vite SPA, compare what the host serves for different routes before looking at anything rendered. Pass the origin and a few real paths from different templates (bash, for `$RANDOM`):

```sh
# usage: ORIGIN=https://example.com bash shell-check.sh / /pricing /blog/a-real-post
for path in "$@" "/zz-missing-$RANDOM"; do
  printf '%s ' "$path"
  curl -sS -o /dev/null -w '%{http_code} ' "$ORIGIN$path"
  curl -sS "$ORIGIN$path" | grep -ioE '<link[^>]+rel="?canonical"?[^>]*>|<meta[^>]+property="?og:(url|title)"?[^>]*>' | tr '\n' ' '
  echo
done
```

Identical canonical and `og:url` on every path means the shell carries route-specific tags. A 200 on the missing path is the SPA fallback. A shareable route with no `og:` tags in this output shows the default card everywhere it is shared, whatever the browser shows. Also grep the output for a literal `%VITE_`: an unreplaced Vite HTML placeholder.

## 4. Raw versus rendered content

Compare the main content, internal links and structured data in the curl output with the rendered DOM. Anything present only after rendering depends on Google's render queue and is invisible to crawlers that do not run JavaScript. Anything present only after a click, scroll, consent or login is invisible to Google as well.

Count `<a href>` links to other internal pages in the raw HTML. A product grid with zero `href`s to products is the JS-only links bug.

## 5. robots.txt

```sh
robots=$(mktemp)
curl -sS -D - -o "$robots" "https://example.com/robots.txt" | grep -iE '^(HTTP/|content-type:)'
head -c 300 "$robots"
```

Expect 200, `text/plain`, and robots syntax rather than `<!doctype html>`. Then read each user-agent group on its own: a crawler gets only its most specific group. Check that no URL carrying a noindex, and no JS, CSS or API path the page needs for rendering, is disallowed.

## 6. Sitemaps

```sh
curl -sS "https://example.com/sitemap.xml" \
  | grep -oE '<loc>[^<]+</loc>' | sed -e 's/<loc>//' -e 's/<\/loc>//' -e 's/&amp;/\&/g' \
  | shuf -n 200 \
  | while read -r url; do
      printf '%s %s\n' "$(curl -sS -o /dev/null -w '%{http_code}' "$url")" "$url"
    done \
  | grep -v '^200 '
```

Anything printed is a sitemap URL that is not a direct 200. If the file is a sitemap index, run this per child sitemap. For the 200s, spot-check that each page's canonical equals its `<loc>` and that it has no noindex. Compare a few `lastmod` values with the content's real edit dates; if every URL carries the same timestamp, `lastmod` is fake (the `vite-plugin-sitemap` default). For an SPA, a 200 here proves little because the fallback answers 200 for anything; check the `<loc>` hosts are the production origin, not `localhost` or a staging host, and that `/404` or a shell path is not listed.

## 7. Search Console and Google tools

Use URL Inspection's live test for the Google-rendered HTML, the Google-selected canonical versus yours, and whether indexing is allowed. The Page indexing report groups excluded URLs by reason (soft 404, duplicate without user-selected canonical, crawled but not indexed, blocked by robots.txt, excluded by noindex): read the sample URLs per reason, they usually point at one template. Validate structured data with the Rich Results Test. For Core Web Vitals, use field data (Search Console CWV report, CrUX, your RUM) at the 75th percentile, and lab tools only to find the cause.

## Reporting

For each finding: the URL and template, the exact response evidence (status line, header, HTML snippet), why it matters for crawling, indexing or ranking, and the fix at the layer that owns it (CDN, server, framework config, template). Separate defects (wrong status, canonical to a redirect, noindex behind a disallow, soft 404s, injection) from preferences (title wording, description length). Before reporting, re-fetch once more and check that the behavior is not a cache artifact, a bot challenge served only to curl, or an environment you did not mean to test.

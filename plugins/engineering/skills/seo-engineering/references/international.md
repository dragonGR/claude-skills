# International sites

Read this when adding a locale, generating hreflang, or writing any routing that depends on country, language, IP or `Accept-Language`.

## URL model

Every language or language-region version has its own stable URL: a path prefix (`/de/`, `/en-gb/`), a subdomain or a country domain. The locale comes from the URL, never only from a cookie, header or IP. A crawler with no cookies, no `Accept-Language` and a US IP must be able to fetch every locale directly.

Pages whose main content is still untranslated are duplicates. Either translate the page or leave that locale out of the hreflang set for it; do not publish a `/de/` URL that serves English content and claim it as German.

## Generating hreflang from one table

Build every page's set from one validated source, so reciprocity and self-reference hold by construction instead of by discipline.

```ts
const HREFLANG_PATTERN = /^(?<language>[a-z]{2})(?:-(?<script>[A-Z][a-z]{3}))?(?:-(?<region>[A-Z]{2}))?$/

type Locale = { hreflang: string; pathPrefix: string }

export function assertLocales(locales: readonly Locale[]): void {
  for (const { hreflang } of locales) {
    const parts = HREFLANG_PATTERN.exec(hreflang)?.groups
    if (!parts) throw new Error(`invalid hreflang code: ${hreflang}`)
    const { language = '', script, region } = parts
    if (!ISO_639_1.has(language)) throw new Error(`unknown language in ${hreflang}`)
    if (script && !ISO_15924.has(script)) throw new Error(`unknown script in ${hreflang}`)
    if (region && !ISO_3166_ALPHA2.has(region)) throw new Error(`unknown region in ${hreflang}`)
  }
}

export function hreflangLinks(
  origin: URL,
  locales: readonly Locale[],
  localizedPath: (locale: Locale) => string | null,
  xDefaultPath: string,
): { hreflang: string; href: string }[] {
  const links = locales.flatMap((locale) => {
    const path = localizedPath(locale)
    return path === null ? [] : [{ hreflang: locale.hreflang, href: new URL(path, origin).href }]
  })
  if (links.length < 2) return []
  return [...links, { hreflang: 'x-default', href: new URL(xDefaultPath, origin).href }]
}
```

`ISO_639_1`, `ISO_15924` and `ISO_3166_ALPHA2` are code sets owned in one module; the pattern alone would accept `jp` or `en-UK`. `localizedPath` returns the canonical path of this page in that locale, or `null` when the page does not exist there. The same function runs on every locale version of the page, so each version emits the same set, including itself. A page that exists in only one locale emits no hreflang at all.

Rules the generator enforces:

- Language is ISO 639-1 (`en`, `de`, `ja`, `zh`), optionally followed by an ISO 15924 script (`zh-Hant`, `zh-Hans`) and then an optional ISO 3166-1 alpha-2 region (`en-GB`, `pt-BR`, `zh-Hans-US`). Region alone is invalid, and UN M.49 regions such as `es-419` are not supported. A script and a region say different things: `zh-Hant` is Traditional Chinese anywhere, `zh-TW` is Chinese for Taiwan, so do not swap one for the other to get past a validator. Google ignores reserved codes such as `UK`, `EU` and `UN` in the region position, so `en-UK` is treated as `en`.
- Every target is an absolute, canonical URL that returns 200 and is indexable. Build it from configured origin, in the same host, protocol and slash form as your canonicals.
- Each locale page's canonical is itself. Never canonicalize `/de/...` to `/en/...`.
- `x-default` points at the language selector or the fallback page users get when no locale matches.
- Language-only and language-region entries can coexist (`es`, `es-ES`, `es-MX`); the language-only one catches users in regions you have not listed.

hreflang can live in the HTML head, in a `Link` HTTP header (the only option for PDFs), or in the sitemap with `xhtml:link` entries. Use one method per URL set. For large sites the sitemap form keeps the HTML small, but the sitemap generator then has to apply the same rules.

## Geo and language routing

```ts
// Before: every visitor from a German IP gets 302'd to /de, including on deep links,
// and a user or crawler outside Germany never reaches /de at all.
app.use((req, res, next) => {
  const country = geoip.lookup(req.ip)?.country
  if (country === 'DE' && !req.path.startsWith('/de')) return res.redirect(`/de${req.path}`)
  next()
})

// After: the URL decides the locale. The suggestion is a hint the page renders,
// and the user's explicit choice is remembered.
app.use((req, res, next) => {
  const urlLocale = localeFromPath(req.path)
  const preferred = parseLocale(req.cookies[LOCALE_COOKIE]) ?? localeFromCountry(req.get(GEO_COUNTRY_HEADER))
  res.locals.localeSuggestion = preferred && preferred !== urlLocale ? preferred : null
  next()
})
```

`parseLocale` returns a locale from the configured table or `null`. The cookie is client-controlled, so it never reaches the page unchecked.

The page renders a dismissible banner ("This page is available in Deutsch") with a real `<a href>` to the equivalent URL. That link doubles as an internal link between locale versions.

If the business insists on redirecting, restrict it to a locale-neutral root such as `/`, use 302 (the target varies by visitor), send `Vary` for whatever the decision reads, and keep every locale URL directly reachable with no redirect. Serve the same response to Googlebot as to a user with the same signals; a special case for crawlers is cloaking.

Googlebot also crawls from some non-US IPs, so do not assume a single crawler location either way.

## Checks

- Fetch one URL per locale and diff the hreflang sets: same entries, each includes itself, one `x-default`.
- For every hreflang target: 200, no redirect, no noindex, canonical equals itself.
- Fetch each locale URL with no cookies, no `Accept-Language` and from a non-matching region (or with the geo header your CDN sets overridden) and confirm no redirect.
- Confirm the `lang` attribute on `<html>` matches the page's language.

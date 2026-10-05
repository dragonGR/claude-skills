# Vite build and deploy

Read this before changing `vite.config`, env handling, build scripts, source maps, the dev server, caching headers, the host's SPA fallback or prerendering. Check the installed `vite` and `@vitejs/plugin-react` majors first: Vite 8 builds with Rolldown and Oxc, and plugin-react 6 requires Vite 8.

## What ships

`dist/` is public. Everything in it can be fetched by anyone who knows or guesses the path, regardless of which routes the UI shows them:

- every chunk, including lazy admin screens and their API calls;
- every `import.meta.env.VITE_*` value and every `define` value, inlined as literals;
- `.map` files, if the build wrote them;
- anything in `public/`, copied as is.

Audit before a release:

```bash
rg -n 'import\.meta\.env|envPrefix|loadEnv|define\s*:' --glob '!node_modules' --glob '!dist'
rg -l 'sourceMappingURL' dist
find dist -name '*.map'
```

Read each env name and each `define` key. Anything named `SECRET`, `PRIVATE`, `SERVICE_ROLE`, `ADMIN` or `WEBHOOK`, or holding a key whose holder can do more than an anonymous visitor, is a finding.

## Env variables

Vite exposes variables prefixed with `VITE_` (the default `envPrefix`) on `import.meta.env` and statically replaces them at build time. Its docs say plainly that `VITE_*` variables must not contain secrets such as API keys. Vite refuses `envPrefix: ''` with an error, but a widened prefix list or a `define` block can still leak.

Before:

```ts
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  return {
    plugins: [react()],
    define: { 'process.env': env },
  }
})
```

`loadEnv` with an empty prefix returns every variable from the `.env*` files and every variable in the process environment, and `define` writes all of them into the bundle wherever `process.env` appears. A database URL in `.env.production`, or a deploy token the CI runner exports, is now public.

After:

```ts
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_')
  return {
    plugins: [react()],
    define: { __APP_RELEASE__: JSON.stringify(env.VITE_RELEASE) },
  }
})
```

Pass the narrowest prefix to `loadEnv` and define named values one at a time. Most apps need neither: client code reads `import.meta.env.VITE_*` directly, through one validated module (see the env pattern in `SKILL.md`).

Other details that cause bugs:

- `.env` files are not loaded into `process.env` while `vite.config` is evaluated. Code in the config that reads `process.env.VITE_API_URL` sees only what the shell exported. Use `loadEnv` when the config itself needs a value.
- Mode-specific files win over generic ones: `.env.production` overrides `.env`, and `.env.[mode].local` overrides both. `vite build --mode staging` loads `.env.staging`. Keep `.env*.local` out of git. A variable already set in the shell or CI environment beats every `.env*` file, so a `VITE_API_BASE_URL` exported for the whole CI pipeline, or left over in a developer's shell, overrides `.env.staging` without a warning. Keep `VITE_*` values out of CI-wide env, or set them per job.
- A missing variable is `undefined` at runtime, not a build error. Validate in one module and let the app fail at boot with a clear message.
- Declare variables on `ImportMetaEnv` in `src/vite-env.d.ts` for editor types. Types do not prove the value exists; the schema does.
- `import.meta.env.MODE`, `BASE_URL`, `PROD`, `DEV` and `SSR` are built in. Gate debug tooling on `import.meta.env.DEV` so the minifier drops it from production builds.

## Build-time versus runtime configuration

A static SPA has no server to read env at request time. Choose one model and apply it everywhere:

| Model | How | Fits | Watch for |
| --- | --- | --- | --- |
| Build per environment | `vite build --mode <env>`, one artifact per environment | Separate pipelines per environment | The artifact tested in staging is not the one shipped to production |
| Build once, configure at boot | Fetch a config file served next to `index.html`, validate it, then render | One artifact promoted through environments | The config file must not be long-cached, and the app must not render before it loads |

Runtime config at boot:

```tsx
// src/config/runtime.ts
import { z } from 'zod'

const RuntimeConfig = z.strictObject({
  apiBaseUrl: z.url(),
  targetChainId: z.number().int().positive(),
})
export type RuntimeConfig = z.infer<typeof RuntimeConfig>

export async function loadRuntimeConfig(path: string): Promise<RuntimeConfig> {
  const res = await fetch(path, { cache: 'no-store' })
  if (!res.ok) throw new Error(`runtime config responded ${res.status}`)
  return RuntimeConfig.parse(await res.json())
}
```

```tsx
// src/main.tsx
const root = createRoot(rootElement)

loadRuntimeConfig(`${import.meta.env.BASE_URL}${RUNTIME_CONFIG_FILE}`).then(
  (config) =>
    root.render(
      <ConfigProvider value={config}>
        <App />
      </ConfigProvider>,
    ),
  (error: unknown) => {
    reportError(error)
    root.render(<ConfigError />)
  },
)
```

If the fetch or the parse fails, render a static error instead of falling back to defaults: a defaulted API URL or chain id sends real users and real transactions to the wrong place. A bare top-level `await` on the load would leave an unhandled rejection and a blank page instead.

The config file is public too. It holds endpoints and ids, never secrets.

## Type checking

Vite only transpiles `.ts`; it does not type-check. The standard script is:

```json
{ "scripts": { "build": "tsc -b && vite build" } }
```

If the project has no project references, use `tsc --noEmit`. A CI job that runs `vite build` alone ships type errors.

## Dev server and preview

| Setting | Default | Risk |
| --- | --- | --- |
| `server.host` | `'localhost'` | `true` or `0.0.0.0` listens on all addresses, LAN and public included |
| `server.allowedHosts` | `[]` (`localhost`, `.localhost` subdomains and IP addresses are allowed) | `true` lets any website read your source through DNS rebinding, per the Vite docs |
| `server.cors` | Allows only `localhost`, `127.0.0.1` and `[::1]` origins | Widening it lets other sites call the dev server from a browser |
| `server.fs.deny` | `.env`, `.env.*`, certificate and key files, `.npmrc`, `.yarnrc.yml`, `.git` | Overriding the list without re-adding these exposes them |

For a phone on the same network, use `--host` for that session only. For a tunnel, add the tunnel's hostname to `allowedHosts`. Keep Vite on the latest patch of its line: Vite's security advisories include several `server.fs.deny` bypasses and an arbitrary file read through the dev server WebSocket, which matter as soon as the dev server is reachable from another machine.

`vite preview` serves `dist/` for a local check. The Vite docs say it is not meant as a production server.

## Source maps

| `build.sourcemap` | Result |
| --- | --- |
| `false` (default) | No maps |
| `true` | `.map` files plus a `sourceMappingURL` comment in each chunk |
| `'inline'` | Map appended to each chunk as a data URI; ships the source inside the bundle |
| `'hidden'` | `.map` files without the comment; the files still sit in `dist/` |

For error tracking, build with `'hidden'`, upload the maps from CI, then delete them before the deploy step:

```bash
vite build
upload-sourcemaps dist
find dist -name '*.map' -delete
```

(`upload-sourcemaps` stands for the error tracker's CLI.) Then confirm `find dist -name '*.map'` prints nothing.

## Code splitting and stale chunks

Vite generates `<link rel="modulepreload">` for the entry and its direct imports, preloads a dynamic import's dependencies in parallel, and splits CSS used by async chunks into its own file. Route-level splitting is therefore mostly a matter of putting the dynamic `import()` at the route boundary; see `react-router-spa.md`.

After a deploy, a tab opened before it still references the old chunk names. If the host deleted those files, the next lazy route fails. Vite emits `vite:preloadError` with the import error on `event.payload`:

```ts
// src/main.tsx
const RELOAD_MARKER = 'app:chunk-reload-at'

window.addEventListener('vite:preloadError', (event) => {
  const last = Number(sessionStorage.getItem(RELOAD_MARKER) ?? 0)
  if (Date.now() - last < CHUNK_RELOAD_COOLDOWN_MS) return
  event.preventDefault()
  sessionStorage.setItem(RELOAD_MARKER, String(Date.now()))
  window.location.reload()
})
```

The cooldown stops an infinite reload when the chunk is missing for a reason a reload does not fix; in that case the error reaches the route's error boundary. Calling `preventDefault()` keeps Vite from rethrowing while the page reloads.

Also:

- Serve `index.html` with `Cache-Control: no-cache` (the Vite docs recommend this) so new visits get new chunk names. Hashed files under `assets/` can be cached long and marked immutable, because their names change when their content does.
- Keep the previous deploy's assets for a while if the host allows it.
- Check what the host returns for a missing `/assets/*.js`. A host whose SPA fallback answers every unmatched path with `index.html` and 200 turns a missing chunk into an HTML response, which fails as a MIME or syntax error instead of a 404.

## SPA fallback, API paths and real 404s

Deep links and refreshes on `/projects/42` need the host to serve `index.html` for paths that are not files. On Cloudflare Workers static assets, `not_found_handling = "single-page-application"` serves `/index.html` with 200 for requests that match no asset. Consequences:

- The host answers every unknown path with the shell and 200, so the router must render a not-found page (`path: '*'`). For public pages that need a real 404, leave `not_found_handling` unset and put a Worker in front that serves the shell only for known client routes; `seo-engineering` (`react-vite-spa.md`) has the pattern.
- If the same Worker serves the API, a browser navigation to an API path can be answered with HTML. Cloudflare's docs show this case and use `run_worker_first` with route patterns such as `/api/*` so API paths reach the Worker first.
- On public marketing pages, a 200 response with a "not found" view is a soft 404 to Google. Google's documented options: redirect with JavaScript to a URL that returns a real 404 status, or add `<meta name="robots" content="noindex">` to the error view. The `seo-engineering` skill covers metadata, `react-helmet-async` and the sitemap.

## Content Security Policy

A CSP header from the host limits what injected script can do. For a Vite build:

- The built `index.html` loads its entry as an external module script, so `script-src 'self'` plus your real third-party origins is the starting point. Inline scripts you add (theme bootstraps, analytics snippets) need a hash.
- Assets under `build.assetsInlineLimit` (4 KiB by default) are inlined as data URIs, so `img-src` and `font-src` need `data:`, or set `assetsInlineLimit: 0`.
- `html.cspNonce` adds a nonce attribute to generated tags, but a nonce must change per response; a static host serving the same `index.html` to everyone cannot use one meaningfully.
- `connect-src` lists the API origin, the RPC endpoints and the wallet relay origins the app really uses.

Roll out with `Content-Security-Policy-Report-Only` first and read the reports before enforcing.

## Prerendering and hydration

An app started with `createRoot` renders from an empty root on the client. There is no server HTML to match, so hydration mismatches do not happen, and `suppressHydrationWarning` and the `getServerSnapshot` argument of `useSyncExternalStore` have no job to do. Reading `localStorage` or `matchMedia` during render is fine in this setup.

If a build step prerenders routes to HTML and the entry switches to `hydrateRoot`, the first client render must reproduce that HTML exactly. Then:

- Format dates and numbers with an explicit locale and `timeZone` known at build time, or render the absolute value first and switch to a relative or local format after mount.
- Use `useId` for ids, never a counter or `Math.random()` in render.
- Read browser-only state through `useSyncExternalStore` with a `getServerSnapshot` that matches the prerendered output, or after mount.
- For a whole browser-only subtree (wallet UI, anything reading `localStorage`), React 19.3's `use(browser())` from `react-dom` inside a `<Suspense>` boundary makes the prerender keep the fallback and the client render the subtree after hydration, with no recoverable hydration error and no mounted-flag effect. On earlier React, render it after mount from a state flag set in an effect.
- Avoid invalid nesting (`<div>` in `<p>`, `<a>` in `<a>`), which the browser rewrites before React hydrates.
- Prerendered HTML is built once for every visitor. It must not contain anything user-specific.

## Plugin and compiler setup

`@vitejs/plugin-react` 6 no longer bundles Babel; Vite 8 handles the Fast Refresh transform with Oxc. Projects that used the plugin's `babel` option, including React Compiler setups, move to `@rolldown/plugin-babel`:

```ts
import react, { reactCompilerPreset } from '@vitejs/plugin-react'
import babel from '@rolldown/plugin-babel'

export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
})
```

`build.rollupOptions` is now an alias of `build.rolldownOptions` and the `esbuild` option is deprecated in favor of `oxc`. Rename them when touching the config.

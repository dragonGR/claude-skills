# Upgrades: React, React Router, Vite and the data layer

Read this when an app needs a major upgrade of React, React Router, Vite, `@vitejs/plugin-react`, TanStack Query, Zustand or wagmi, or when class components need converting to hooks. Each project's official upgrade guide is the authority; this file covers how to run the upgrade safely and where it usually breaks.

## Safe rollout

1. Establish a passing baseline: lockfile, `tsc`, build, unit and integration tests, and at least one browser test per critical flow (sign-in, checkout or wallet write, the main form). Without it you cannot tell upgrade breakage from existing breakage.
2. Inventory: package versions, the root API (`createRoot`, or `hydrateRoot` if anything prerenders), router mode, class components, legacy context, string refs, `findDOMNode`, deprecated lifecycle names, custom test helpers, and every library with a React, React Router or TanStack Query peer dependency.
3. Cross one compatibility boundary at a time, one package per change. For React 19, move to React 18.3 first (18.2 plus deprecation warnings) and clear its warnings. For React Router, enable the next major's future flags on the current one and fix what they break before bumping.
4. Run official codemods on a clean tree, one codemod per commit where practical, and read every hunk. Codemods cannot decide component ownership, effect dependencies, error handling or accessibility.
5. Validate production-like behavior: a production build served from `dist/` by a static server, deep links and refresh on nested routes, lazy routes, forms, async updates, error states, and development Strict Mode.
6. Remove compatibility shims once each batch is proven. Do not leave two routers, two query clients or two state paths behind.

## React 19

Codemods:

```bash
npx codemod@latest react/19/migration-recipe
npx types-react-codemod@latest preset-19 ./src
```

Removed APIs to search for:

```bash
rg -n 'ReactDOM\.(render|hydrate|unmountComponentAtNode|findDOMNode)|ref="[^"]+"|contextTypes|getChildContext|createFactory|react-dom/test-utils|react-test-renderer/shallow|\.defaultProps\s*=|\.propTypes\s*=' src
```

| Removed | Replacement |
| --- | --- |
| `ReactDOM.render`, `ReactDOM.hydrate` | `createRoot`, `hydrateRoot` from `react-dom/client` |
| `unmountComponentAtNode` | `root.unmount()` |
| `findDOMNode` | a ref on the DOM element |
| String refs | callback refs or `useRef` |
| `propTypes` checks, `defaultProps` on function components | TypeScript types, default parameter values |
| Legacy context (`contextTypes`, `getChildContext`) | `createContext` |
| `act` from `react-dom/test-utils` | `act` from `react` |

Behavior changes that tests and monitoring notice:

- Render errors are no longer re-thrown. Uncaught ones go to `window.reportError`, caught ones to `console.error`. If error reporting depended on the re-throw, wire `onUncaughtError` and `onCaughtError` on `createRoot`.
- `javascript:` URLs rendered into `href` or `src` are blocked. Any legitimate use (bookmarklets, legacy links) must become an event handler.
- `<form action={fn}>` resets uncontrolled fields after the function succeeds. Forms migrated from `onSubmit` to `action` need their error states to carry the submitted values back.

TypeScript changes after the type codemod:

- `useRef()` needs an argument: `useRef<HTMLDivElement>(null)`.
- Ref callbacks may now return a cleanup function, so an implicit return like `ref={(el) => (node = el)}` is a type error. Use a block body.
- The global `JSX` namespace is gone; use `React.JSX` or augment `JSX` inside `declare module 'react'`.

Fix these type errors properly. Loosening `strict`, adding `any` or `@ts-expect-error` to get through the upgrade leaves the bugs they were pointing at.

React 19.2 adds `useEffectEvent` and `<Activity>`. Upgrade `eslint-plugin-react-hooks` with it, or the linter tries to add Effect Events to dependency arrays. `<Activity mode="hidden">` keeps a subtree's state while unmounting its effects, which can replace hand-rolled "keep the tab mounted but hidden" code; effects in the hidden subtree stop until it is visible again, so anything that must keep running (a pending transaction watcher) does not belong there.

React 19.3 adds `<ViewTransition>` and `addTransitionType` for View Transition animations, refs on `<Fragment>`, and `browser()` in `react-dom`, which marks a subtree browser-only during server rendering or prerendering (see `vite-build-and-deploy.md`). Separate transitions now render independently, so a slow one no longer holds back an unrelated one.

## React Router 6 to 7

Requirements: Node 20, React 18 and React DOM 18 or later.

Before bumping, enable on v6 and fix what breaks: `v7_relativeSplatPath` and `v7_startTransition` (on `<BrowserRouter>` or the router), and for data routers `v7_fetcherPersist`, `v7_normalizeFormMethod`, `v7_partialHydration` (replaces `fallbackElement` with `HydrateFallback`) and `v7_skipActionErrorRevalidation`. In v7 these behaviors are the default, and `<BrowserRouter>` no longer takes a `future` prop.

- `react-router-dom` 7 re-exports `react-router`, so existing imports keep working. Prefer moving to `react-router` (and `RouterProvider` from `react-router/dom`) during this upgrade rather than later.
- `json()` and `defer()` are deprecated. Return plain objects from loaders; use `Response.json()` when a real `Response` is needed.
- Splat routes: with `v7_relativeSplatPath`, relative links inside a `path="dashboard/*"` route resolve differently. Click through every nested link in splat routes, not only the top-level ones.
- `v7_startTransition` makes router state updates transitions. React does not replace already-visible content with a Suspense fallback during a transition, so a lazy route that used to flash its fallback may now leave the old page on screen until it loads; check that a pending indicator still appears.

## React Router 7 to 8

v8 requires Node 22.22 or later and React and React DOM 19.2.7 or later. For data and declarative mode SPAs:

- The `react-router-dom` package is gone. Import everything from `react-router`, and `RouterProvider` from `react-router/dom`, then uninstall `react-router-dom`.
- Middleware is always on in v8. If the app adopted it on v7, it ran behind `future.v8_middleware`; enable that flag on v7 first and test.
- `useMatches()` entries expose `loaderData` instead of `data`.
- `react-router` is published as ESM only. Node's own `require()` loads it on the Node versions v8 supports, but a test runner with its own CommonJS module system may not: Jest in CommonJS mode fails on the `import` syntax unless it transforms the package or is a version that falls back to `require(esm)` (30.4 and later on Node 24.9 and later). Run the test suite against v8 before merging; Vitest loads it as is.

The remaining v8 flags (`v8_splitRouteModules`, `v8_viteEnvironmentApi`, `v8_passThroughRequests`, `v8_trailingSlashAwareDataRequests`) concern framework mode and server request handling; an SPA using `createBrowserRouter` or `<BrowserRouter>` does not need them.

## Vite 7 to 8 and plugin-react 6

Vite 8 replaces esbuild and Rollup with Rolldown and Oxc.

- `build.rollupOptions` becomes `build.rolldownOptions` (the old name is a deprecated alias), `worker.rollupOptions` becomes `worker.rolldownOptions`, and the `esbuild` option becomes `oxc`.
- JavaScript is minified by the Oxc minifier and CSS by Lightning CSS. Compare bundle output and visual snapshots before and after.
- Default browser targets moved up to Chrome and Edge 111, Firefox 114 and Safari 16.4. Check analytics before shipping if the audience includes older browsers.
- The default import from a CommonJS module is now handled consistently between dev and build. Packages that worked by accident in one of them can break; run the production build and the dev server.
- Check each plugin in `vite.config` against its Vite 8 compatibility notes before assuming it works under Rolldown.

`@vitejs/plugin-react` 6 requires Vite 8 and drops its Babel dependency (Oxc handles the React Refresh transform). Anything that used the plugin's `babel` option, React Compiler included, moves to `@rolldown/plugin-babel`; `vite-build-and-deploy.md` has the config.

## TanStack Query 4 to 5

React 18 or later is required. The changes most likely to break silently:

| v4 | v5 |
| --- | --- |
| `useQuery(key, fn, options)` overloads | One object: `useQuery({ queryKey, queryFn, ...options })` |
| `cacheTime` | `gcTime` |
| `status: 'loading'`, `isLoading` for "no data yet" | `status: 'pending'`, `isPending`; the new `isLoading` means `isPending && isFetching` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` |
| `onSuccess`, `onError`, `onSettled` on queries | Removed from queries (mutations keep them) |
| `useErrorBoundary` | `throwOnError` |

The renamed `isLoading` is the trap: code that kept `if (isLoading) return <Spinner />` now falls through to rendering `undefined` data whenever a query is disabled or not fetching. Search for every `isLoading` and decide whether it means `isPending`.

Code that used query-level `onSuccess` to copy data into state or a store should not be ported to an effect that does the same thing; read the query data where it is needed (see `state-and-async.md`).

TanStack ships a codemod for the overload removal. Within v5, 5.102 and later deprecate `ensureQueryData`, `fetchQuery` and `prefetchQuery` in favor of `queryClient.query`. The mapping is not a rename: `ensureQueryData(opts)` becomes `query({ ...opts, staleTime: 'static' })` (without it, `query` refetches stale data on every call), `fetchQuery(opts)` becomes `query(opts)`, and `prefetchQuery(opts)` becomes `query(opts).catch(noop)`.

## Zustand 4 to 5

React 18 and TypeScript 4.5 or later are required, and default exports are gone (`import { create } from 'zustand'`).

- A selector that returns a new object or array on every call may now loop. Search for selectors returning `{ ... }` or `[ ... ]` and either split them or wrap them in `useShallow` from `zustand/shallow`.
- `create` no longer takes an equality function. Code that passed `shallow` as the second argument moves to `createWithEqualityFn` from `zustand/traditional` (with the `use-sync-external-store` peer dependency) or to `useShallow`.
- `persist` no longer writes the initial state to storage when the store is created. Code that relied on the key existing right after startup must set state explicitly.
- With `setState(next, true)` (replace), TypeScript now requires the full state.

## wagmi 2 to 3

`useAccount`, `useAccountEffect` and `useSwitchAccount` become `useConnection`, `useConnectionEffect` and `useSwitchConnection`. Mutation hooks expose `mutate` and `mutateAsync` instead of hook-specific names. `connectors` and `chains` were removed from several hook results in favor of `useConnectors`, `useConnections` and `useChains`. Connector SDKs are optional peer dependencies that must be installed explicitly, and the minimum TypeScript version rises to 5.9.3. Re-test every wallet flow listed in `web3-frontends.md` after the upgrade.

## Security patch level

CVE-2025-55182 affects the React Server Components packages (`react-server-dom-webpack`, `-parcel` and `-turbopack`). React's advisory says an app whose React code does not use a server, or that does not use a framework, bundler or plugin supporting Server Components, is not affected. A Vite SPA rendered with `createRoot` qualifies, but confirm with the lockfile: `rg 'react-server-dom' package-lock.json pnpm-lock.yaml yarn.lock` should find nothing. Still land on the newest patch of the chosen line for `react`, `react-dom`, `react-router` and `vite`, and check each project's security advisories rather than pinning a version from memory.

## Class-to-hook conversion

Start with a leaf component with narrow observable behavior. Add characterization tests first if the behavior is unclear. Convert state and methods, then external synchronization. Keep public props, accessible names, focus behavior and error behavior unchanged unless the task says otherwise.

Lifecycle methods do not map one-to-one to effects:

- `componentDidMount` plus `componentDidUpdate` that recompute a value from props: compute during render.
- `componentDidUpdate(prevProps)` that resets state when an id changes: change the child's `key`.
- `componentDidMount` that fetches: TanStack Query, started from a route loader where the app has one. A hand-written effect needs abort-on-cleanup.
- Subscriptions, timers, listeners, imperative widgets: `useEffect` with cleanup, or `useSyncExternalStore` for external stores.
- `getSnapshotBeforeUpdate` has no hook equivalent. Leave that component as a class unless a tested layout-effect workaround preserves the behavior.
- `this.x` instance fields that do not affect rendering: `useRef`.

```tsx
useEffect(() => {
  const unsubscribe = feed.subscribe(feedId, onMessage)
  return unsubscribe
}, [feedId, onMessage])
```

If `onMessage` is recreated every render, the subscription churns; read it through `useEffectEvent` (React 19.2) or a ref instead of listing it.

Error boundaries still need a class component, a maintained wrapper such as `react-error-boundary`, or React Router's `errorElement` for route-level errors in data mode. Do not try to replace one with a hook.

For HOCs, keep a thin adapter while consumers still need the old prop-shaped API. Move the logic into a hook behind it, migrate call sites in batches, and delete the adapter when the last caller is gone.

Strict Mode in development mounts, unmounts and remounts each component once. A converted component that double-fetches, double-subscribes or leaks listeners under Strict Mode has a missing cleanup; fix the effect.

## Memoization and loading

Keep lazy loading where it protects a real bundle boundary. Place `Suspense` around independently useful regions rather than the whole app. Add `memo`, `useMemo` and `useCallback` after profiling shows referential churn or expensive recomputation, or adopt React Compiler (through `@rolldown/plugin-babel` with plugin-react 6) and remove manual memoization only after measuring. `useTransition` keeps input responsive during non-urgent updates; it does not replace request cancellation or error handling.

## Validation matrix

- `tsc` and lint with the repository's configured commands, in CI as well as locally.
- Unit and integration tests plus the browser flows from the baseline.
- Forms, keyboard interaction, focus after navigation, loading, error, empty and retry states.
- Strict Mode warnings in development and console errors in a production build.
- Deep links and refresh on nested routes against the built `dist/`, plus the not-found route.
- Lazy routes after a simulated deploy (old tab, new assets) to exercise the stale-chunk handler.
- Query behavior on the screens that changed: does data refresh after mutations, and is anything now refetching on every focus that did not before?
- Bundle output: chunk sizes before and after, no duplicate React copies, no obsolete compatibility packages.

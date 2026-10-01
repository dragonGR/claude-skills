# React Router in a client-rendered SPA

Read this when adding or changing routes, lazy loading, loaders, error boundaries, not-found handling or navigation side effects. Examples use React Router 7 in data mode (`createBrowserRouter` plus `<RouterProvider>`), with declarative mode (`<BrowserRouter>` plus `<Routes>`) called out where it differs. Framework mode, SSR and route modules are out of scope.

## Check the setup first

- Mode: `createBrowserRouter` means data mode. `<BrowserRouter>` means declarative mode, which has no loaders, actions, `errorElement`, route `lazy`, `useBlocker`, `useNavigation` or `<ScrollRestoration>`. Advice written for one mode often does not compile in the other.
- Package: in v7, `react-router-dom` re-exports everything from `react-router`, so either import works. v8 removes `react-router-dom`; new code can import from `react-router` now (and `RouterProvider` from `react-router/dom`) to make that upgrade a no-op.
- Router instance: create it once at module level. A `createBrowserRouter` call inside a component rebuilds the router and resets navigation state on every render of that component.

## Route-level code splitting

Data mode, with the component and loader fetched in parallel:

```tsx
const router = createBrowserRouter([
  {
    path: '/',
    Component: RootLayout,
    errorElement: <RootError />,
    children: [
      { index: true, Component: Home },
      {
        path: 'reports',
        lazy: async () => {
          const [{ ReportsPage }, { reportsLoader }] = await Promise.all([
            import('./routes/reports/page'),
            import('./routes/reports/loader'),
          ])
          return { Component: ReportsPage, loader: reportsLoader(queryClient) }
        },
      },
      { path: '*', Component: NotFound },
    ],
  },
])
```

`path`, `index` and `children` cannot be lazy; the router needs them to match before anything loads. Since 7.5, `lazy` also accepts an object with one loader per route property, so a small loader can arrive without the component's heavy dependencies. Check the installed version's docs before using it.

Declarative mode uses `React.lazy`:

```tsx
const ReportsPage = lazy(() => import('./routes/reports/page'))

export function AppRoutes() {
  return (
    <Routes>
      <Route element={<RootLayout />}>
        <Route
          path="reports"
          element={
            <RouteErrorBoundary>
              <Suspense fallback={<PageSkeleton />}>
                <ReportsPage />
              </Suspense>
            </RouteErrorBoundary>
          }
        />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  )
}
```

- `lazy` sits at module top level. Declared inside a component, it creates a new component type on every render and the subtree's state resets.
- The module must have a default export, or the loader must map a named export: `lazy(() => import('./page').then((m) => ({ default: m.ReportsPage })))`.
- A rejected import (network failure, chunk removed by a deploy) throws to the nearest error boundary. Without one around the route, the whole app unmounts. Pair this with the `vite:preloadError` handler in `vite-build-and-deploy.md`.
- Split at routes and at heavy, rarely used widgets (editors, charts, wallet modals). Splitting every small component adds requests without saving bytes.

## Loaders and TanStack Query

Loaders run before the route renders, so they are the place to start data requests and avoid the component-then-fetch waterfall. Keep TanStack Query as the cache and let the loader fill it:

```ts
// src/queries/projects.ts
export const projectQueries = {
  detail: (orgId: string, projectId: string) =>
    queryOptions({
      queryKey: ['projects', orgId, 'detail', projectId],
      queryFn: ({ signal }) => api.getProject(orgId, projectId, signal),
    }),
}
```

```ts
// src/routes/project/loader.ts, mounted at 'orgs/:orgId/projects/:projectId'
const ProjectParams = z.object({ orgId: z.string().min(1), projectId: z.string().min(1) })

export function projectLoader(queryClient: QueryClient) {
  return async ({ params }: LoaderFunctionArgs) => {
    const parsed = ProjectParams.safeParse(params)
    if (!parsed.success) throw data('Not found', { status: HTTP_NOT_FOUND })
    const { orgId, projectId } = parsed.data
    await queryClient.ensureQueryData(projectQueries.detail(orgId, projectId))
    return parsed.data
  }
}
```

```tsx
export function ProjectPage() {
  const { orgId, projectId } = useLoaderData<ReturnType<typeof projectLoader>>()
  const project = useQuery(projectQueries.detail(orgId, projectId))
  // project.data is already in the cache on first render; still handle the error state
}
```

- One `queryOptions` object per resource gives the loader, the component and invalidation the same key.
- `ensureQueryData` returns cached data if present and fetches otherwise. Current TanStack Query v5 docs mark it, `fetchQuery` and `prefetchQuery` as deprecated in favor of `queryClient.query(...)`; use whichever the installed minor provides and do not mix both styles in one codebase.
- Await in the loader when the page is useless without the data (the router keeps the old page visible and `useNavigation().state` is `'loading'`). Start the request without awaiting when the page can render a skeleton, so navigation is instant.
- An awaited query that fails makes the loader throw, and the route's `errorElement` renders. Decide whether that is right for the page or whether the component should render its own error state instead.
- Route params are user input. Parse them; throw a 404 for anything that does not fit.
- A loader that redirects anonymous users (`throw redirect(loginPath)`) is UX. The API still rejects the request on its own.

## Error boundaries and not-found

```tsx
export function RouteError() {
  const error = useRouteError()
  if (isRouteErrorResponse(error) && error.status === HTTP_NOT_FOUND) return <NotFound />
  reportError(error)
  return <PageError />
}
```

- Errors bubble to the closest parent `errorElement`. Give the root one so every error has a deliberate page to land on, and give independent sections their own so a failing panel does not replace the layout.
- `throw data(message, { status })` in a loader produces a route error response for `isRouteErrorResponse`. Anything else (a thrown `Error`, a rejected import) is an unexpected error: report it, show a generic page, never render `error.message` from an API response as HTML.
- The catch-all `path: '*'` route handles URLs no route matches. A loader 404 handles URLs that match a route but name a record that does not exist. Both render the same not-found page.
- The API answers "not found" and "not yours" the same way, and the UI does not distinguish them.
- Error boundaries do not catch errors in event handlers or in async code after render. Catch those and put them in state, or rethrow them during render.

## Navigation side effects

- Focus and title. A route change replaces content without a document load, so focus stays on the clicked link (now removed) or falls to `<body>`, and screen readers hear nothing. React Router does not manage focus. Set `document.title` per route and focus the new page's heading on mount. The `accessibility` skill has the component and the cases to skip (first load, query-string-only changes).
- Scroll. In data mode, render `<ScrollRestoration />` once in the root layout; it restores saved positions on back and forward and stores them in `sessionStorage`. Declarative mode has no equivalent, and a pushed route keeps the previous scroll offset unless the app resets it.
- Transitions. Animating route changes with `AnimatePresence` needs the `useOutlet` or keyed `<Routes location>` pattern; the `react-motion` skill covers it, including how exits interact with scroll restoration and focus.
- Unsaved changes. `useBlocker` (data mode) intercepts in-app navigations; a `beforeunload` listener covers reloads and tab closes. Neither replaces saving drafts.

## Redirect after login

```tsx
export function useReturnTo(fallback: string): string {
  const [searchParams] = useSearchParams()
  return safeRedirectPath(searchParams.get(RETURN_TO_PARAM), window.location.origin, fallback)
}

// after the API confirms the session
navigate(returnTo, { replace: true })
```

`safeRedirectPath` is in `security-boundaries.md`. `window.location.origin` is the document's own origin, which a request header cannot spoof. Use `replace` so Back does not return to the login form.

# State and async patterns

Read this when writing or reviewing effects, data fetching, TanStack Query keys and mutations, optimistic updates, Zustand stores, forms, URL state or anything that can be clicked twice. Examples use React 19, TanStack Query v5, Zustand 5 and React Router 8 (imports from `react-router`). `apiFetch` is the app's fetch wrapper (see `security-boundaries.md`).

## Effect races

Before:

```tsx
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<Result[]>([])

  useEffect(() => {
    apiFetch(`/search?q=${query}`)
      .then((res) => res.json())
      .then(setResults)
  }, [query])

  return <ResultList results={results} />
}
```

Typing "react" fires five requests. Whichever finishes last wins, and it is often not the last one sent. The query is not URL-encoded (`&`, `#` and `%` break it), an HTTP 500 body is rendered as results, and there is no loading or error state.

After, without a query library:

```tsx
type SearchState =
  | { status: 'loading' }
  | { status: 'error' }
  | { status: 'done'; results: Result[] }

function SearchResults({ query }: { query: string }) {
  const [state, setState] = useState<SearchState>({ status: 'loading' })

  useEffect(() => {
    const controller = new AbortController()
    setState({ status: 'loading' })

    apiFetch(`/search?${new URLSearchParams({ q: query })}`, { signal: controller.signal })
      .then(async (res) => {
        if (!res.ok) throw new Error(`search responded ${res.status}`)
        return SearchResponse.parse(await res.json())
      })
      .then((body) => setState({ status: 'done', results: body.results }))
      .catch((error: unknown) => {
        if (controller.signal.aborted) return
        reportError(error)
        setState({ status: 'error' })
      })

    return () => controller.abort()
  }, [query])

  if (state.status === 'loading') return <ResultsSkeleton />
  if (state.status === 'error') return <SearchError />
  if (state.results.length === 0) return <NoResults query={query} />
  return <ResultList results={state.results} />
}
```

After, with TanStack Query (preferred when the project has it): results are cached per key, so an old response can never land under a new query.

```tsx
const search = useQuery({
  queryKey: ['search', query],
  queryFn: ({ signal }) => searchApi(query, signal),
  enabled: query.length >= MIN_QUERY_LENGTH,
  placeholderData: keepPreviousData,
})
```

Debounce the input value, not the fetch function, so the key and the request stay in step. `useDeferredValue` is not a debounce: it defers rendering but does not reduce the number of requests.

## Server state copies

Before:

```tsx
export function ProfileForm({ profile }: { profile: Profile }) {
  const [name, setName] = useState(profile.name)
  // ...
}
```

`useState(profile.name)` reads the prop once. After a save invalidates the profile query and it refetches, `profile.name` changes and the input still shows the old value. Navigating from one profile to another keeps the first person's draft.

After: keep the draft local, and let the parent reset it when the underlying record changes.

```tsx
<ProfileForm key={`${profile.id}:${profile.updatedAt}`} profile={profile} />
```

The same bug in store form:

```tsx
const { data } = useQuery(cartQuery)
useEffect(() => {
  if (data) useCartStore.setState({ items: data.items })
}, [data])
```

Two owners now disagree whenever one updates first. Read `data` where it is needed (with `select` for derived shapes), and keep only client-owned state in the store.

## Zustand stores

A store holds state the client owns: UI preferences, a multi-step wizard's progress, an unsent draft, a selected wallet view. Data that has a server or chain copy stays in TanStack Query.

Selectors:

```tsx
// Before: a new object on every call; Zustand 5 can loop on this
const { items, addItem } = useDraftStore((s) => ({ items: s.items, addItem: s.addItem }))

// After: one selector per value
const items = useDraftStore((s) => s.items)
const addItem = useDraftStore((s) => s.addItem)

// Or, when a grouped read is clearer
const { items, addItem } = useDraftStore(useShallow((s) => ({ items: s.items, addItem: s.addItem })))
```

`useShallow` comes from `zustand/shallow`. Selectors that compute a new array (`s.items.filter(...)`) have the same problem; select the source and derive in render, or wrap in `useShallow`.

Persistence:

```ts
export const usePreferencesStore = create<PreferencesState>()(
  persist(
    (set) => ({
      density: DEFAULT_DENSITY,
      sidebarOpen: true,
      setDensity: (density) => set({ density }),
      toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
    }),
    {
      name: PREFERENCES_STORAGE_KEY,
      version: PREFERENCES_STORAGE_VERSION,
      partialize: (s) => ({ density: s.density, sidebarOpen: s.sidebarOpen }),
      migrate: migratePreferences,
    },
  ),
)
```

- `partialize` keeps actions, server data and anything sensitive out of storage. The default storage is `localStorage`, which every script on the origin can read.
- When the stored `version` does not match and there is no `migrate`, the stored value is discarded. Bump the version whenever the persisted shape changes and write the migration, or users silently lose their settings (or, without a version bump, old shapes rehydrate into new code).
- With `localStorage`, hydration is synchronous: the store already holds the persisted values when it is created.
- Zustand 5's `persist` no longer writes the initial state to storage at creation.
- Stores are module singletons. Reset user-scoped stores on sign-out, and reset stores between tests.

## Query keys and invalidation

Every value the `queryFn` reads belongs in the `queryKey`. A key factory keeps reads and invalidations in the same shape:

```ts
export const projectKeys = {
  all: ['projects'] as const,
  lists: (orgId: string) => ['projects', orgId, 'list'] as const,
  list: (orgId: string, filters: ProjectFilters) => ['projects', orgId, 'list', filters] as const,
  detail: (orgId: string, id: string) => ['projects', orgId, 'detail', id] as const,
}

useQuery({
  queryKey: projectKeys.list(orgId, filters),
  queryFn: ({ signal }) => fetchProjects({ orgId, ...filters }, signal),
})
```

Object members of a key are hashed deterministically, so `{ status, page }` and `{ page, status }` are the same key. `invalidateQueries({ queryKey: projectKeys.lists(orgId) })` matches by prefix and refreshes every filtered list for that org; pass `exact: true` only when you mean one entry. Active queries refetch in the background; inactive ones are marked stale and refetch when next used. The returned promise resolves once the matching queries have settled, which lets an action wait for fresh data.

After a mutation, invalidate every key whose data the write can change: the list, the detail, counts, and any aggregate on another screen. When the API returns the updated entity, `setQueryData` on the detail key avoids a round trip; still invalidate lists, since sort order and filters may change.

Defaults that surprise people: cached data is stale immediately (`staleTime` 0), so stale queries refetch on mount, window focus and reconnect; failed queries retry three times with exponential backoff before showing an error; mutations do not retry. Set `staleTime` per resource from how fast it really changes rather than turning refetching off globally.

Put the org or user id in keys for data scoped to them, and call `queryClient.clear()` on sign-out. Create the `QueryClient` once:

```tsx
// Before: a new, empty cache on every render of App
function App() {
  const queryClient = new QueryClient()
  return <QueryClientProvider client={queryClient}>{/* ... */}</QueryClientProvider>
}

// After
const queryClient = new QueryClient({ defaultOptions: { queries: { staleTime: DEFAULT_STALE_TIME_MS } } })

function App() {
  return <QueryClientProvider client={queryClient}>{/* ... */}</QueryClientProvider>
}
```

## Optimistic updates

Use them for reversible, low-stakes writes.

When the pending value shows in one place, render the mutation's `variables` and skip the cache entirely:

```tsx
const addTodo = useMutation({
  mutationFn: (text: string) => api.addTodo(listId, text),
  onSettled: () => queryClient.invalidateQueries({ queryKey: todoKeys.list(listId) }),
})

// in the list
{addTodo.isPending && <PendingTodo text={addTodo.variables} />}
{addTodo.isError && <FailedTodo text={addTodo.variables} onRetry={() => addTodo.mutate(addTodo.variables)} />}
```

When several parts of the screen must reflect it, write to the cache and keep a rollback:

```tsx
const queryClient = useQueryClient()
const listKey = projectKeys.list(orgId, filters)

const archive = useMutation({
  mutationFn: (id: string) => api.archiveProject(orgId, id),
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: listKey })
    const previous = queryClient.getQueryData<Project[]>(listKey)
    queryClient.setQueryData<Project[]>(listKey, (old) => old?.filter((p) => p.id !== id))
    return { previous }
  },
  onError: (_error, _id, snapshot) => {
    queryClient.setQueryData(listKey, snapshot?.previous)
    toast.error('Could not archive the project. It is back in the list.')
  },
  onSuccess: () => toast.success('Project archived'),
  onSettled: () => queryClient.invalidateQueries({ queryKey: projectKeys.lists(orgId) }),
})
```

`cancelQueries` stops an in-flight refetch from landing after the optimistic write and wiping it. The third argument of `onError` is whatever `onMutate` returned. Success feedback lives in `onSuccess`, never in `onMutate`.

React 19's `useOptimistic` shows a temporary value while an Action is pending and drops it when the Action ends. It rolls back correctly only if the real value changes on success, inside the Action, and not on failure:

```tsx
type StarButtonProps = { orgId: string; projectId: string; starred: boolean }

export function StarButton({ orgId, projectId, starred }: StarButtonProps) {
  const queryClient = useQueryClient()
  const [optimisticStarred, setOptimisticStarred] = useOptimistic(starred)
  const [failed, setFailed] = useState(false)

  async function toggle() {
    setFailed(false)
    setOptimisticStarred(!starred)
    let res: Response
    try {
      res = await apiFetch(`/projects/${encodeURIComponent(projectId)}/star`, {
        method: starred ? 'DELETE' : 'PUT',
      })
    } catch {
      setFailed(true)
      return
    }
    if (!res.ok) {
      setFailed(true)
      return
    }
    await queryClient.invalidateQueries({ queryKey: projectKeys.detail(orgId, projectId) })
  }

  return (
    <form action={toggle}>
      <button aria-pressed={optimisticStarred}>Star</button>
      {failed && <p role="alert">Could not update. Try again.</p>}
    </form>
  )
}
```

`starred` comes from the project query. `fetch` rejects on a dropped connection, and an error thrown from a `<form action>` function goes to the nearest error boundary, so without the `try` one network blip replaces the route with the error page. Awaiting `invalidateQueries` keeps the Action pending until the refetch settles, so the optimistic value is replaced by real data rather than flicking back first. Calling the optimistic setter outside an Action or `startTransition` makes React warn and show the value only briefly. State set after an `await` is not part of the transition, because React does not carry it across `await`; that is fine for an error flag, but wrap the update in `startTransition` when it should render together with the Action's result.

## Double submit and idempotency

```tsx
function PayButton({ invoiceId }: { invoiceId: string }) {
  const [idempotencyKey, setIdempotencyKey] = useState(() => crypto.randomUUID())
  const pay = useMutation({
    mutationFn: () => api.payInvoice({ invoiceId, idempotencyKey }),
    onSuccess: () => setIdempotencyKey(crypto.randomUUID()),
  })

  return (
    <button type="button" disabled={pay.isPending} onClick={() => pay.mutate()}>
      Pay
    </button>
  )
}
```

The key is created once per intent and reused by every retry of that intent, so the API can return the first result instead of charging again. A client-generated key does not survive a reload. For payments and orders, have the API create the pending operation when the page loads (an order id, a payment intent) and make the button confirm that id; a reload then resumes the same operation. `crypto.randomUUID()` only exists in secure contexts (HTTPS or localhost).

Never add `retry` to a mutation that is not idempotent. If writes to one resource must not overlap (reorder, then rename), give them the same mutation `scope.id` so TanStack Query runs them one after another.

## Unknown outcomes

A POST that times out, loses its connection, or gets a 502/504 from a gateway may have committed. Model it:

```ts
type SubmitState =
  | { kind: 'idle' }
  | { kind: 'submitting' }
  | { kind: 'confirmed'; orderId: string }
  | { kind: 'rejected'; reason: RejectReason }
  | { kind: 'unknown' }
```

- 2xx with a valid body: `confirmed`.
- 4xx with a known error body: `rejected`; the API said no, so the user can fix and resubmit.
- Network error, timeout, 502, 503 without a body you trust, 504: `unknown`. Show "Checking whether your order went through", then look it up by idempotency key or returned id with bounded backoff. Resolve to `confirmed` or `rejected` from what the API reports.
- Still unknown after the reconciliation budget: say so, link to the order history, and keep the same idempotency key if the user retries.

Never map `unknown` to "Failed, try again" with a fresh key.

## Stale closures

```tsx
// Before: `seconds` is 0 forever inside the interval.
useEffect(() => {
  const id = setInterval(() => setSeconds(seconds + 1), TICK_MS)
  return () => clearInterval(id)
}, [])

// After
useEffect(() => {
  const id = setInterval(() => setSeconds((s) => s + 1), TICK_MS)
  return () => clearInterval(id)
}, [])
```

When an effect needs the latest value of something it should not re-run for (a logging call that reads the current cart size), React 19.2 has `useEffectEvent`:

```tsx
const onVisit = useEffectEvent((visitedUrl: string) => {
  logVisit(visitedUrl, cartItemCount)
})

useEffect(() => {
  onVisit(pathname)
}, [pathname])
```

Effect Events are declared in the same component or hook as their effect, called only from effects, and never listed as dependencies; `eslint-plugin-react-hooks` at a current version knows the rules. On older React, keep the latest value in a ref updated in an effect.

Silencing `react-hooks/exhaustive-deps` is almost always hiding one of these bugs.

## Loading, error and empty

```tsx
const projects = useQuery(projectListQuery)

if (projects.isPending) return <ProjectListSkeleton />
if (projects.data === undefined) return <LoadError onRetry={() => projects.refetch()} />

return (
  <>
    {projects.isError && <StaleBanner onRetry={() => projects.refetch()} />}
    {projects.data.length === 0 ? <NoProjects /> : <ProjectTable rows={projects.data} />}
  </>
)
```

A failed background refetch leaves both `error` and the previous `data` set. Checking `isError` first would replace good data with an error page; checking only `data` would hide the failure. Decide which the screen needs and handle both.

## Forms with Actions

React 19 form Actions work in a client-only app: the function passed to `<form action>` runs in a transition with the `FormData`, and no server framework is involved. Two behaviors catch people:

- After the action function succeeds, React resets every uncontrolled field in the form. An action that returns validation errors has succeeded, so the user's input disappears unless the state carries it back.
- With `useActionState`, a thrown error cancels the queued actions and goes to the nearest error boundary. `fetch` throws on network failure, so catch it and return a state.

```tsx
const SignupInput = z.strictObject({
  email: z.email(),
  password: z.string().min(MIN_PASSWORD_LENGTH),
})

type SignupState = {
  status: 'idle' | 'invalid' | 'unknown' | 'done'
  email: string
  fieldErrors: Partial<Record<'email' | 'password', string[]>>
}

async function signupAction(_prev: SignupState, formData: FormData): Promise<SignupState> {
  const email = String(formData.get('email') ?? '')
  const parsed = SignupInput.safeParse({ email, password: formData.get('password') })
  if (!parsed.success) {
    return { status: 'invalid', email, fieldErrors: z.flattenError(parsed.error).fieldErrors }
  }

  let res: Response
  try {
    res = await apiFetch('/signup', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(parsed.data),
    })
  } catch {
    return { status: 'unknown', email, fieldErrors: {} }
  }

  if (res.ok) return { status: 'done', email, fieldErrors: {} }
  const body = SignupErrorBody.safeParse(await res.json().catch(() => null))
  if (body.success) return { status: 'invalid', email, fieldErrors: body.data.fieldErrors }
  return { status: 'unknown', email, fieldErrors: {} }
}

export function SignupForm() {
  const [state, formAction, isPending] = useActionState(signupAction, INITIAL_SIGNUP_STATE)

  return (
    <form action={formAction}>
      <input name="email" type="email" defaultValue={state.email} aria-invalid={!!state.fieldErrors.email} />
      {state.fieldErrors.email && <p>{state.fieldErrors.email.join(' ')}</p>}
      <input name="password" type="password" aria-invalid={!!state.fieldErrors.password} />
      <button disabled={isPending}>Create account</button>
      {state.status === 'unknown' && <p role="alert">We could not confirm the sign-up. Check your email before trying again.</p>}
    </form>
  )
}
```

The password is never echoed back into state. The same schema runs on the Worker as the actual gate; the client copy is for fast feedback. `useActionState` runs dispatches one after another, each receiving the previous result, so a slow request delays the next submit rather than racing it.

## URL state

```tsx
export function StatusFilter() {
  const [searchParams, setSearchParams] = useSearchParams()
  const status = ProjectStatus.catch(DEFAULT_STATUS).parse(searchParams.get('status'))

  function setStatus(next: ProjectStatusValue) {
    setSearchParams(
      (prev) => {
        const params = new URLSearchParams(prev)
        params.set('status', next)
        params.delete('page')
        return params
      },
      { replace: true },
    )
  }
  // ...
}
```

Search params are user input: parse with a schema that falls back to a default instead of throwing. Reset dependent params (page) when a filter changes. Copy `prev` instead of mutating it: the `searchParams` object is mutable, and changing it without calling the setter makes values drift from the URL. Two `setSearchParams` calls in the same tick do not build on each other, so make one call that sets everything.

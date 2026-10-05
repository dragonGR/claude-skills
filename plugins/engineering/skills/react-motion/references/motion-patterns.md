# Motion patterns

Read this when writing or reviewing presence, dialogs, animated lists, layout animation, scroll-linked effects, first-load entrances, view transitions or React Router route transitions. Each pattern shows the broken version first. Constants are named so the values live in one place; put them in the project's shared motion module.

The examples import from `motion/react`, the entry point of the `motion` package that new code should install. `framer-motion` is the same library under its old name and exports the same names, so in a codebase that already uses it, keep its import path rather than mixing the two.

## Presence: keys, placement and committed work

The list below has two bugs: keys are indexes, and the delete request waits for the exit animation.

```tsx
// Before
<ul>
  <AnimatePresence onExitComplete={flushPendingDeletes}>
    {rows.map((row, index) => (
      <motion.li key={index} exit={{ opacity: 0, x: ROW_EXIT_OFFSET_PX }}>
        {row.title}
        <button type="button" onClick={() => queueDelete(row.id)}>Delete</button>
      </motion.li>
    ))}
  </AnimatePresence>
</ul>
```

`flushPendingDeletes` never runs if the list unmounts during the exit (the user navigates away), so rows vanish on screen and come back on reload. With index keys the last row animates out instead of the deleted one.

```tsx
// After
const deletingIds = useRef(new Set<string>())

async function deleteRow(row: Row) {
  // The row stays on screen until the request settles, so a double click would send a second DELETE.
  if (deletingIds.current.has(row.id)) return
  deletingIds.current.add(row.id)
  setDeleteError(null)
  try {
    await api.deleteRow(row.id)
    setRows((current) => current.filter((r) => r.id !== row.id))
  } catch (error: unknown) {
    setDeleteError(describeDeleteFailure(error))
  } finally {
    deletingIds.current.delete(row.id)
  }
}

<ul>
  <AnimatePresence initial={false}>
    {rows.map((row) => (
      <motion.li key={row.id} layout exit={{ opacity: 0, x: ROW_EXIT_OFFSET_PX }}>
        {row.title}
        <button type="button" onClick={() => deleteRow(row)}>Delete</button>
      </motion.li>
    ))}
  </AnimatePresence>
</ul>
```

The request finishes before state changes; the exit animates state that is already true. `initial={false}` stops existing rows animating in on first render while rows added later still animate. If the product wants optimistic removal, snapshot and restore on failure (the `frontend-engineering` skill covers rollback); the rule that the mutation lives in the handler does not change.

A presence boundary wrapped in a condition never runs exits:

```tsx
// Before: AnimatePresence unmounts with the panel
{open && (
  <AnimatePresence>
    <motion.aside key="panel" exit={{ opacity: 0 }} />
  </AnimatePresence>
)}

// After
<AnimatePresence>
  {open && <motion.aside key="panel" exit={{ opacity: 0 }} />}
</AnimatePresence>
```

## Animated dialog: focus and inert exit

### Native `<dialog>`: animate in CSS

Never put a native `<dialog>` inside `AnimatePresence`. Unmounting it skips the close algorithm, so focus is not returned; the `accessibility` skill's `Modal` stays mounted for that reason. The Motion order of focus first, exit second does not work either: while the dialog is modal everything outside it is inert, so focusing the trigger before `close()` does nothing, and once `close()` runs the dialog loses `open` and, without a CSS transition on `display`, disappears in the same frame. Let `close()` return focus, and animate open and close in CSS:

```css
:root {
  --motion-surface: 150ms;
}

dialog {
  opacity: 0;
  transition:
    opacity var(--motion-surface),
    display var(--motion-surface) allow-discrete,
    overlay var(--motion-surface) allow-discrete;
}

dialog[open] {
  opacity: 1;
}

@starting-style {
  dialog[open] {
    opacity: 0;
  }
}
```

`@starting-style` gives the entrance a start value, and `allow-discrete` holds `display` (and `overlay`, which keeps the dialog in the top layer) until the fade ends. `@starting-style` and `transition-behavior` have been Baseline since August 2024; `overlay` is Chromium only. In Firefox and Safari the closing dialog leaves the top layer as soon as the exit starts and can drop behind other content, so keep the exit a short opacity fade. Put any movement (`translate`) inside `@media (prefers-reduced-motion: no-preference)`. When the dialog's action removes the trigger (Delete on a row), `close()` focuses a node that is about to be detached. Move focus to the next row or the list heading after `close()` has run, from an effect in the parent: in one commit React runs the `Modal`'s effect (which calls `close()`) before its parent's, and by then the page is no longer inert.

### Dialogs that render a plain element

The Motion pattern below is for dialogs that render an ordinary element you can keep mounted: Radix with `forceMount` on `Dialog.Portal` and `Dialog.Content` inside `AnimatePresence`, React Aria, or a custom modal. `ConfirmSurface` stands in for the content element. It is not a complete modal: it has no focus trap, no inert background and no Escape handling, and it must not ship without a primitive that provides them or your own code that meets the `accessibility` skill's full list. With Radix, leave focus return to Radix. Its focus trap stays active until React re-renders with `open={false}`, so a `focus()` call in the click handler is pulled back into the content. Radix focuses `Dialog.Trigger` when `Dialog.Content` unmounts, which is after the exit, and the rest of the page stays `aria-hidden` until then. When the trigger will be gone (the dialog deleted its row), pass `onCloseAutoFocus` that calls `event.preventDefault()` and focuses the next row or the list heading. Keep the exit short, because focus has no useful place to be until it ends.

```tsx
import { useId, useRef, useState } from 'react'
import { AnimatePresence, motion, useIsPresent } from 'motion/react'

const SURFACE_OFFSET_PX = 8

type Row = { id: string; title: string }

type ConfirmSurfaceProps = {
  titleId: string
  title: string
  onCancel: () => void
  onConfirm: () => void
}

function ConfirmSurface({ titleId, title, onCancel, onConfirm }: ConfirmSurfaceProps) {
  const isPresent = useIsPresent()

  return (
    <motion.div
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
      inert={!isPresent}
      initial={{ opacity: 0, y: SURFACE_OFFSET_PX }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: SURFACE_OFFSET_PX }}
    >
      <h2 id={titleId}>{title}</h2>
      <button type="button" onClick={onCancel} autoFocus>Cancel</button>
      <button type="button" onClick={onConfirm}>Delete</button>
    </motion.div>
  )
}

export function RowList({ rows, onDelete }: { rows: Row[]; onDelete: (row: Row) => void }) {
  const titleId = useId()
  const headingRef = useRef<HTMLHeadingElement>(null)
  const triggerRef = useRef<HTMLButtonElement | null>(null)
  const [pending, setPending] = useState<Row | null>(null)

  function cancel() {
    setPending(null)
    triggerRef.current?.focus()
  }

  function confirm(row: Row) {
    setPending(null)
    headingRef.current?.focus()
    onDelete(row)
  }

  return (
    <>
      <h2 ref={headingRef} tabIndex={-1}>Notifications</h2>
      <ul>
        {rows.map((row) => (
          <li key={row.id}>
            {row.title}
            <button
              type="button"
              onClick={(event) => {
                triggerRef.current = event.currentTarget
                setPending(row)
              }}
            >
              Delete
            </button>
          </li>
        ))}
      </ul>
      <AnimatePresence>
        {pending && (
          <ConfirmSurface
            key={pending.id}
            titleId={titleId}
            title={`Delete "${pending.title}"?`}
            onCancel={cancel}
            onConfirm={() => confirm(pending)}
          />
        )}
      </AnimatePresence>
    </>
  )
}
```

What this fixes: focus moves in the handler, before the exit starts, so it never lands on `<body>`. That only works when your own code owns the trap and the background's inertness and releases both in the same handler. Radix's trap pulls the focus back (use `onCloseAutoFocus` as above), and a native modal `<dialog>` keeps the trigger inert until `close()`. On confirm the trigger's row is about to disappear, so focus goes to the list heading instead. The leaving surface is `inert`, so a second click on Delete during the exit does nothing. `inert` is a boolean prop on React 19; on React 18 write `inert={isPresent ? undefined : ''}`. Initial focus goes to the least destructive action on mount through `autoFocus`, not after the entrance finishes.

## Layout animation

```tsx
// Before: animate and layout both own width; radius in a class distorts
<motion.div layout className="card rounded-xl" animate={{ width: expanded ? EXPANDED_PX : COLLAPSED_PX }}>
  <h3>{title}</h3>
</motion.div>

// After: state drives style, layout animates it, radius goes through style
<motion.div
  layout
  className="card"
  style={{ width: expanded ? EXPANDED_PX : COLLAPSED_PX, borderRadius: CARD_RADIUS_PX }}
>
  <motion.h3 layout="position">{title}</motion.h3>
</motion.div>
```

Inside an `overflow: auto` list, add `layoutScroll` to the scroll container (`<motion.ul layoutScroll>`). On a `position: fixed` container, add `layoutRoot`. Repeated components that share a `layoutId`:

```tsx
export function Tabs({ instanceId, tabs, selected }: TabsProps) {
  return (
    <LayoutGroup id={instanceId}>
      {tabs.map((tab) => (
        <button key={tab.id} type="button" aria-pressed={tab.id === selected}>
          {tab.label}
          {tab.id === selected && <motion.span layoutId="indicator" className="indicator" />}
        </button>
      ))}
    </LayoutGroup>
  )
}
```

## Scroll-linked values

```tsx
// Before: re-renders every frame and animates width
const { scrollYProgress } = useScroll()
const [progress, setProgress] = useState(0)
useMotionValueEvent(scrollYProgress, 'change', setProgress)
return <div className="progress" style={{ width: `${progress * 100}%` }} />

// After: no React render per frame, transform only
const { scrollYProgress } = useScroll()
return <motion.div className="progress" style={{ scaleX: scrollYProgress, originX: 0 }} />
```

A discrete state derived from scroll sets a boolean, which React skips when unchanged:

```tsx
const { scrollY } = useScroll()
const [hidden, setHidden] = useState(false)

useMotionValueEvent(scrollY, 'change', (current) => {
  const previous = scrollY.getPrevious() ?? current
  setHidden(current > previous && current > HEADER_HIDE_AFTER_PX)
})
```

Parallax needs its own reduced-motion branch because a motion value bound through `style` is not an animation and `MotionConfig reducedMotion="user"` does not stop it:

```tsx
const ref = useRef<HTMLDivElement>(null)
const reduceMotion = useReducedMotion()
const { scrollYProgress } = useScroll({ target: ref, offset: ['start end', 'end start'] })
const y = useTransform(scrollYProgress, [0, 1], [PARALLAX_RANGE_PX, -PARALLAX_RANGE_PX])

return <motion.div ref={ref} style={{ y: reduceMotion ? 0 : y }} />
```

`useScroll({ target })` measures the target's layout position and ignores transforms on it and its ancestors, so applying the parallax `y` to the tracked element does not feed back into its own progress.

## First-load entrances

The app renders on the client, so `initial` can read browser state without a hydration mismatch. The trap is that it reads it once.

```tsx
// Before: captured at render; wrong after a resize or rotation, and the JS breakpoint drifts from the CSS one
<motion.aside initial={{ x: window.innerWidth }} animate={{ x: 0 }} />
<motion.h1 initial={{ y: isMobile ? 16 : 48 }} animate={{ y: 0 }} />

// After: relative units and one value; breakpoints belong in CSS
<motion.aside initial={{ x: '100%' }} animate={{ x: 0 }} />
<motion.h1 initial={{ y: HEADLINE_RISE_PX }} animate={{ y: 0 }} />
```

Branching `initial` or markup on `useReducedMotion()` cannot cause a hydration mismatch here, because there is no server HTML to disagree with. If a page is ever prerendered, the first render must stop depending on the preference, since the prerenderer cannot know it.

The LCP element must be visible on the first frame React paints:

```tsx
// Before: invisible until the bundle renders plus the delay, and not an LCP candidate while at opacity 0
<motion.h1 initial={{ opacity: 0, y: HEADLINE_RISE_PX }} animate={{ opacity: 1, y: 0 }} transition={{ delay: HERO_DELAY_S }}>

// After: visible from the first paint; a short rise is the only entrance
<motion.h1 initial={{ y: HEADLINE_RISE_PX }} animate={{ y: 0 }}>
```

## Long lists

```tsx
import { motion, stagger, type Variants } from 'motion/react'

// Before: the 200th row waits 200 × STAGGER_STEP_S seconds
const listVariants: Variants = { visible: { transition: { delayChildren: stagger(STAGGER_STEP_S) } } }

// After: only the first rows are staggered
const MAX_STAGGERED_ROWS = 8

const rowVariants: Variants = {
  hidden: { opacity: 0, y: ROW_RISE_PX },
  visible: (index: number) => ({
    opacity: 1,
    y: 0,
    transition: { delay: Math.min(index, MAX_STAGGERED_ROWS) * STAGGER_STEP_S },
  }),
}

{rows.map((row, index) => (
  <motion.li key={row.id} custom={index} variants={rowVariants} initial="hidden" animate="visible">
    {row.title}
  </motion.li>
))}
```

Keys must be ids. If the list's parent remounts on refetch or filter change, every row animates again; key the parent by something stable or pass `initial={false}` on later renders.

## Loops that stop

WCAG 2.2.2 Pause, Stop, Hide (level A) applies to anything that moves on its own for more than five seconds next to other content. Decoration is not exempt, and `prefers-reduced-motion` is not a pause mechanism, because it only helps users who know the OS setting exists and have turned it on. A loop either stops within five seconds (a finite `repeat`) or gets a visible control:

```tsx
const ref = useRef<HTMLDivElement>(null)
const inView = useInView(ref)
const reduceMotion = useReducedMotion()
const [paused, setPaused] = useState(false)
const shouldLoop = inView && !reduceMotion && !paused

<>
  <motion.div
    ref={ref}
    aria-hidden="true"
    animate={shouldLoop ? { y: [0, -FLOAT_RANGE_PX, 0] } : { y: 0 }}
    transition={shouldLoop ? { duration: FLOAT_CYCLE_S, repeat: Infinity, ease: 'easeInOut' } : { duration: 0 }}
  />
  {reduceMotion ? null : (
    <button type="button" aria-pressed={paused} onClick={() => setPaused((value) => !value)}>
      Pause animation
    </button>
  )}
</>
```

The button sits near the animation, keeps one label with `aria-pressed` for its state, and disappears only when nothing moves. Persist the choice (`localStorage`) if the loop appears on many pages. For an attention cue, a finite `repeat` whose total runs under five seconds needs no control.

## Bundle size

```tsx
// src/motion/motion-provider.tsx
import type { ReactNode } from 'react'
import { LazyMotion, MotionConfig, domAnimation } from 'motion/react'

export function MotionProvider({ children }: { children: ReactNode }) {
  return (
    <LazyMotion features={domAnimation} strict>
      <MotionConfig reducedMotion="user">{children}</MotionConfig>
    </LazyMotion>
  )
}

// src/components/fade-in.tsx
import type { ReactNode } from 'react'
import * as m from 'framer-motion/m'

export function FadeIn({ children }: { children: ReactNode }) {
  return <m.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>{children}</m.div>
}
```

`domAnimation` covers animations, variants, exit and tap/hover/focus gestures. Layout animation and drag need `domMax`; load it asynchronously where it is used: `features={() => import('./motion-features').then((mod) => mod.default)}` with `motion-features.ts` doing `export { domMax as default } from 'motion/react'`. Check the production build output to confirm the layout features landed in a separate chunk rather than the entry.

## React Router route transitions

React Router moves no focus, and in data mode `<ScrollRestoration>` scrolls when the location changes, not when your animation finishes. Everything below works with React Router 7 and 8 in either declarative or data mode unless it says otherwise. Import from `react-router`: v8 removed the `react-router-dom` package (`RouterProvider` now comes from `react-router/dom`), and in v7 it only re-exports `react-router`.

### View transitions (data mode)

For a page-level crossfade or a shared element between pages, let the browser do it. `<Link viewTransition>`, `<NavLink viewTransition>`, `<Form viewTransition>` and `navigate(to, { viewTransition: true })` wrap the router update in `document.startViewTransition()`. The browser snapshots the old page and animates the snapshot, so nothing old stays mounted: the problems the exit pattern below has to test for (old routes rendering new content, lost loader data, two titles, `<ScrollRestoration>` firing mid-exit) do not arise. The API is Baseline since October 2025 (Firefox 144); where it is missing, React Router navigates without animating. The `viewTransition` option is ignored in declarative mode.

```tsx
import { Link } from 'react-router'

<Link to={`/invoices/${invoice.id}`} viewTransition>
  {invoice.number}
</Link>
```

```css
:root {
  --motion-page: 200ms;
}

::view-transition-group(root) {
  animation-duration: var(--motion-page);
}

@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none;
  }
}
```

The browser does not apply reduced motion to view transitions; the media query is required. Pages do not take pointer input while the transition runs, so keep it short. Shared elements get a `view-transition-name` that must be unique on the page at snapshot time; in a list, set it only on the clicked item (`useViewTransitionState(href)` or `NavLink`'s `isTransitioning` render prop). Keep the `PageHeading` focus move from the `accessibility` skill: the new page's effects run before React Router lets the transition start animating, so focus lands on the new heading.

React 19.3 made `<ViewTransition>` and `addTransitionType` stable. They animate updates made inside `startTransition`, a `<Suspense>` reveal or `useDeferredValue` (list reorders, tab content, a Suspense fallback giving way to data), and React skips the animation for updates that start from `popstate`, so Back does not animate. React calls `startViewTransition` itself and interrupts view transitions it did not start, so do not combine React Router's `viewTransition` with `<ViewTransition>` boundaries that change in the same navigation; give each navigation one owner. Motion 13.4 and later wrap React's component as `AnimateView` (`motion/react-animate-view`, React 19.3 or later) to drive those layers with Motion transitions. It does not read `MotionConfig reducedMotion`; it retimes the browser's view-transition animations, so the media query above leaves it nothing to animate.

### Enter-only (the default)

A fade on the incoming page, no exit. Nothing old stays mounted, so scroll restoration, focus and router hooks behave exactly as without animation.

```tsx
import { useEffect, useState, type ReactNode } from 'react'
import { Outlet, useLocation } from 'react-router'
import { motion } from 'motion/react'

const PAGE_FADE_S = 0.2
let firstPageRendered = false

function PageFade({ children }: { children: ReactNode }) {
  // Each navigation mounts a new PageFade, so the initializer reads the flag once per page
  const [skipEntrance] = useState(() => !firstPageRendered)

  useEffect(() => {
    firstPageRendered = true
  }, [])

  return (
    <motion.div
      initial={skipEntrance ? false : { opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: PAGE_FADE_S }}
    >
      {children}
    </motion.div>
  )
}

export function RootLayout() {
  const { pathname } = useLocation()

  return (
    <>
      <SiteHeader />
      <main id="main" tabIndex={-1}>
        <PageFade key={pathname}>
          <Outlet />
        </PageFade>
      </main>
    </>
  )
}
```

The key sits on a child component, not on `motion.div` inside `RootLayout`, because a `useState` initializer in the layout would run once for the whole session and the skip flag would never change. The first page of a session paints at full opacity, which keeps it eligible as the LCP element. Opacity only, so the wrapper never becomes a containing block for fixed descendants. `<Outlet />` is safe here because nothing outlives the navigation.

### With an exit

Only when the design needs the old page to leave visibly. The exiting copy must keep rendering the old route, so it cannot contain a bare `<Outlet />`.

```tsx
import type { ReactNode } from 'react'
import { Routes, useLocation, useOutlet } from 'react-router'
import { AnimatePresence, motion, useIsPresent } from 'motion/react'

const PAGE_ENTER_S = 0.15
const PAGE_EXIT_S = 0.1

function PageFrame({ children }: { children: ReactNode }) {
  const isPresent = useIsPresent()

  return (
    <motion.div
      inert={!isPresent}
      initial={{ opacity: 0 }}
      animate={{ opacity: 1, transition: { duration: PAGE_ENTER_S } }}
      exit={{ opacity: 0, transition: { duration: PAGE_EXIT_S } }}
    >
      {children}
    </motion.div>
  )
}

// Data mode: rendered inside a layout route
export function AnimatedOutlet() {
  const { pathname } = useLocation()
  const outlet = useOutlet()

  return (
    <AnimatePresence mode="wait" initial={false}>
      <PageFrame key={pathname}>{outlet}</PageFrame>
    </AnimatePresence>
  )
}

// Declarative mode: the exiting copy keeps matching the location it was rendered with
export function AnimatedRoutes({ children }: { children: ReactNode }) {
  const location = useLocation()

  return (
    <AnimatePresence mode="wait" initial={false}>
      <PageFrame key={location.pathname}>
        <Routes location={location}>{children}</Routes>
      </PageFrame>
    </AnimatePresence>
  )
}
```

`useOutlet()` returns the element for the child route at the moment the layout renders, and `AnimatePresence` keeps rendering the exiting child with the element it had, so the old page stays the old page. `<Routes location>` does the same in declarative mode: the prop is the location to match against, and the exiting copy holds the old one. `initial={false}` on `AnimatePresence` skips the entrance on the first page of the session. `mode="wait"` avoids two full pages stacked in the flow; it also means every navigation costs exit plus enter, so keep both short. `inert` stops clicks and Tab reaching the leaving page.

What still goes wrong with exits, and has to be tested:

- `<ScrollRestoration>` acts when the location changes, which is at the start of the exit. The leaving page jumps to the top, and on Back the saved offset is applied against the old page's height. Press Back from a long page that was reached from another long page and check the landing position.
- Inside the leaving page, `useLocation` and `useSearchParams` already reflect the new location. Anything rendered from them changes during the exit, and location effects run in a page that is going away.
- In data mode, `useLoaderData`, `useRouteLoaderData`, `useActionData` and `useMatches` read the router's current state, and the router drops loader data for routes that no longer match. The leaving page re-renders with `undefined` and throws on `data.items`, so the nearest error boundary shows mid-exit; for the same route with a new param (`/invoices/1` to `/invoices/2`) it shows the next record's data while it fades out. `useParams()` keeps the old match, because it reads the captured element's route context. A page that may exit reads its data through TanStack Query keyed from `useParams()`, with the loader only priming the cache; otherwise keep exits off that route.
- The new page mounts only after the exit, so a focus effect in the layout keyed on `pathname` runs too early. Move focus from the new page's heading on mount (the `accessibility` skill has the component) and call `focus({ preventScroll: true })` so it does not undo the restored scroll position.
- Code-split routes (`lazy` in data mode, `React.lazy` in declarative mode) can delay the incoming page further. Check the transition on a throttled connection, where the gap between exit and enter is longest.

In declarative mode there is no `<ScrollRestoration>`; the browser does not reset scroll on a client-side push, so the app must scroll to the top on new navigations itself (see the `accessibility` skill's route-change section).

## Switching between `framer-motion` and `motion`

Both packages ship the same code on the same version line: `motion/react` re-exports `framer-motion`, and `motion` depends on `framer-motion`. Motion's upgrade guide describes uninstalling `framer-motion`, installing `motion` and changing imports to `motion/react`. That is a rename. Do it only as its own change, and confirm the lockfile then resolves a single `framer-motion` (the one `motion` depends on). Staying on `framer-motion` is equally valid.

Crossing a major version is the part that changes behavior, whichever package name is used. `AnimateSharedLayout` is long gone; use `layoutId` and `LayoutGroup`. v11 moved the post-mount render to a microtask, so tests that assert right after an update need to await a frame. v12 has no breaking changes in Motion for React. v13 dropped the optional `@emotion/is-prop-valid` dependency, so styled `motion` components (Styled Components, Emotion) may start passing props to the DOM that used to be filtered; pass `isValidProp` to `MotionConfig` to restore the filter. v14 has no breaking changes for React, but `motion` and `framer-motion` now pin `framer-motion`, `motion-dom` and `motion-utils` to exact versions, so a direct `framer-motion` dependency at any other version always resolves a second copy. Keep behavior identical in the upgrade change and redesign motion separately.

# Motion patterns

Read this when writing or reviewing presence, dialogs, animated lists, layout animation, scroll-linked effects, first-load entrances or React Router route transitions with Motion. Each pattern shows the broken version first. Constants are named so the values live in one place; put them in the project's shared motion module.

The examples import from `framer-motion`. `motion/react` re-exports the same module, so every name here is available from either path; use the one the codebase already uses.

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

This covers only the part animation owns. A dialog primitive (native `<dialog>`, Radix, React Aria) still provides the focus trap, Escape and scroll locking; with Radix, render `Dialog.Portal` and `Dialog.Content` with `forceMount` inside `AnimatePresence` so they stay mounted for the exit.

```tsx
import { useId, useRef, useState } from 'react'
import { AnimatePresence, motion, useIsPresent } from 'framer-motion'

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

What this fixes: focus moves in the handler, before the exit starts, so it never lands on `<body>`. On confirm the trigger's row is about to disappear, so focus goes to the list heading instead. The leaving surface is `inert`, so a second click on Delete during the exit does nothing. `inert` is a boolean prop on React 19; on React 18 write `inert={isPresent ? undefined : ''}`. Initial focus goes to the least destructive action on mount through `autoFocus`, not after the entrance finishes.

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
import { motion, stagger, type Variants } from 'framer-motion'

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

```tsx
const ref = useRef<HTMLDivElement>(null)
const inView = useInView(ref)
const reduceMotion = useReducedMotion()
const shouldLoop = inView && !reduceMotion

<motion.div
  ref={ref}
  aria-hidden="true"
  animate={shouldLoop ? { y: [0, -FLOAT_RANGE_PX, 0] } : { y: 0 }}
  transition={shouldLoop ? { duration: FLOAT_CYCLE_S, repeat: Infinity, ease: 'easeInOut' } : { duration: 0 }}
/>
```

## Bundle size

```tsx
// src/motion/motion-provider.tsx
import type { ReactNode } from 'react'
import { LazyMotion, MotionConfig, domAnimation } from 'framer-motion'

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

`domAnimation` covers animations, variants, exit and tap/hover/focus gestures. Layout animation and drag need `domMax`; load it asynchronously where it is used: `features={() => import('./motion-features').then((mod) => mod.default)}` with `motion-features.ts` doing `export { domMax as default } from 'framer-motion'`. Check the production build output to confirm the layout features landed in a separate chunk rather than the entry.

## React Router route transitions

React Router has no page-transition component and moves no focus, and in data mode `<ScrollRestoration>` scrolls when the location changes, not when your animation finishes. Everything below works with `react-router-dom` 7 in either declarative or data mode unless it says otherwise.

### Enter-only (the default)

A fade on the incoming page, no exit. Nothing old stays mounted, so scroll restoration, focus and router hooks behave exactly as without animation.

```tsx
import { useEffect, useState, type ReactNode } from 'react'
import { Outlet, useLocation } from 'react-router-dom'
import { motion } from 'framer-motion'

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
import { Routes, useLocation, useOutlet } from 'react-router-dom'
import { AnimatePresence, motion, useIsPresent } from 'framer-motion'

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
- The new page mounts only after the exit, so a focus effect in the layout keyed on `pathname` runs too early. Move focus from the new page's heading on mount (the `accessibility` skill has the component) and call `focus({ preventScroll: true })` so it does not undo the restored scroll position.
- Code-split routes (`lazy` in data mode, `React.lazy` in declarative mode) can delay the incoming page further. Check the transition on a throttled connection, where the gap between exit and enter is longest.

In declarative mode there is no `<ScrollRestoration>`; the browser does not reset scroll on a client-side push, so the app must scroll to the top on new navigations itself (see the `accessibility` skill's route-change section).

## Switching between `framer-motion` and `motion`

Both packages ship the same code on the same version line: `motion/react` re-exports `framer-motion`, and `motion` depends on `framer-motion`. Motion's upgrade guide describes uninstalling `framer-motion`, installing `motion` and changing imports to `motion/react`. That is a rename. Do it only as its own change, and confirm the lockfile then resolves a single `framer-motion` (the one `motion` depends on). Staying on `framer-motion` is equally valid.

Crossing a major version is the part that changes behavior, whichever package name is used. `AnimateSharedLayout` is long gone; use `layoutId` and `LayoutGroup`. v11 moved the post-mount render to a microtask, so tests that assert right after an update need to await a frame. v12 has no breaking changes in Motion for React. v13 dropped the optional `@emotion/is-prop-valid` dependency, so styled `motion` components (Styled Components, Emotion) may start passing props to the DOM that used to be filtered; pass `isValidProp` to `MotionConfig` to restore the filter. Keep behavior identical in the upgrade change and redesign motion separately.

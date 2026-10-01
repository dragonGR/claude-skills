---
name: react-motion
description: Animation in React with Motion (framer-motion or motion/react) and GSAP: exits, layout animation, route transitions with React Router, scroll effects, ScrollTrigger pinning, reduced motion and frame performance. Load it before writing or reviewing any animation code, even one transition, and when an animation janks, leaks, skips its exit or breaks focus.
license: MIT
metadata:
  author: Alex Tsanis
---

# React motion

Animation bugs rarely throw. They slide the wrong row out, strand focus in a dialog that is fading away, re-render the page on every scroll frame, leak ScrollTriggers into the next route, hide the headline until JavaScript arrives, or make a user with a vestibular disorder feel sick. Write animation code for the conditions it will actually meet: interrupted, doubled, navigated away from mid-exit, reduced and running on a throttled phone. Every animation also has to earn its place: if it does not explain a change, guide attention or confirm an action, remove it.

Before writing anything, read `package.json` and the lockfile: `motion` or `framer-motion` (and whether both are present), `gsap` and `@gsap/react`, the React major (React 19 takes `inert` as a boolean), and how React Router is mounted. Declarative mode (`<BrowserRouter>` with `<Routes>`) and data mode (`createBrowserRouter` with `<RouterProvider>`) need different route-transition code, and `<ScrollRestoration>` exists only in data mode. Motion for React needs React 18.2 or later. Match the installed APIs, not memory.

`framer-motion` and `motion` are the same library. `motion/react` re-exports `framer-motion`, the `motion` package depends on `framer-motion` at its own version, and both are published on the same version line. Importing from `framer-motion` is supported; do not rewrite a codebase's imports to `motion/react` as a fix. The lazy `m` components live at `framer-motion/m`, which `motion/react-m` re-exports. Use whichever path the codebase already uses, and only one of them.

## Failure catalogue

Each entry: what it looks like in a diff, why it breaks, what to do instead. Longer before/after code is in the reference files.

### Presence and lifecycle

**Conditional outside `AnimatePresence`.** `{open && <AnimatePresence><motion.div exit={…} /></AnimatePresence>}`, or a component that returns `null` early and would otherwise render the presence boundary. The boundary unmounts together with its child, so nothing is left to run the exit. Keep `AnimatePresence` mounted and put the condition inside it.

**Index keys or missing keys inside presence.** Every direct child of `AnimatePresence` needs a unique key that stays with the logical item. With `key={index}`, dismissing the first toast makes React see the last key disappear: the last toast plays the exit while the dismissed one's content silently shifts into its slot, and per-item state (timers, inputs) attaches to the wrong item. Key by id. For a single slot that swaps content (`step === 1 ? <motion.div>A</motion.div> : <motion.div>B</motion.div>`), same element type at the same position is an update, not an exit; give it `key={step}`.

**Committed work tied to the animation lifecycle.** `onExitComplete={() => deleteItem(id)}`, `onAnimationComplete={() => navigate(destination)}`, `await animate(...)` before calling `submit()`. The work is delayed by the animation, and skipped entirely if the presence boundary or component unmounts first (route change, parent re-keyed, list refetched). The row is gone from the screen but the DELETE never left. Do the mutation or navigation in the event handler; the animation only renders state that has already changed.

**`mode="wait"` in front of something the user is waiting for.** `wait` holds the entering child until the exiting one finishes, so every step, route or result pays exit plus enter duration in series, and rapid changes queue. Default `sync` suits lists and toasts. Use `popLayout` when the exiting element should leave the layout immediately so siblings reflow; a custom component that is a direct child then has to forward its ref to the DOM node (`forwardRef`, or `ref` as a prop on React 19). Reserve `wait` for a single slot where overlap would look broken and both durations are short.

**Exiting content still interactive.** During an exit the DOM stays mounted. A second click on "Delete" in a closing dialog sends a second request, Tab reaches links in a panel that is leaving, and screen readers read it. Read `useIsPresent()` in the exiting component and set `inert` on it when not present (a boolean on React 19; on React 18 pass `inert=""` when inert and omit it otherwise).

**Focus lost when an animated modal closes or a route changes.** Focus returns to the trigger only when the dialog unmounts, which is after the exit, or never if the trigger was the row that was just deleted. Focus lands on `<body>` and keyboard and screen-reader users lose their place. Move focus in the close handler, before the exit starts: to the trigger if it still exists, otherwise to the next row, the list heading or the region that changed. On open, move focus into the dialog on mount rather than in `onAnimationComplete`. A dialog primitive still owns the focus trap and `aria-modal`; Motion only animates its visual lifetime (Radix exposes `forceMount` so the content stays mounted during the exit). React Router moves no focus on navigation; see the route entries below and the `accessibility` skill for the full dialog and route-change contract.

### Route transitions with React Router

**Animating `<Outlet />` or an unscoped `<Routes>`.** `<AnimatePresence><motion.div key={pathname}><Outlet /></motion.div></AnimatePresence>`. The exiting copy still contains `<Outlet />`, which renders whatever route matches now, so for the whole exit both copies show the new page and the old one never animates out. `<Routes>` without a `location` prop has the same bug because it matches the current location. In a layout route, capture the element with `const outlet = useOutlet()` and render `{outlet}` inside the keyed wrapper. With `<Routes>`, pass `location={location}` and put the key on it, so the exiting copy keeps matching the old location.

**Keying the transition on the wrong value.** `key={location.key}` changes on every history entry, so filter, sort and pagination changes in the query string replay the page transition and remount the page, dropping its local state. `key={location.pathname}` replays only when the path changes. When `/invoices/1` to `/invoices/2` should not replay, key on something that names the page rather than the record: in data mode, the `id` of the deepest match from `useMatches()`. Whatever the key, everything under it remounts when it changes: uncontrolled inputs, local state and effects start over.

**Exits that fight scroll restoration.** With `mode="wait"` the old page stays mounted for the whole exit, but the location has already changed when the exit starts, and `<ScrollRestoration>` (data mode) scrolls when the location changes. The leaving page jumps to the top mid-fade, and on Back the saved offset is applied against the old page's height, so an offset deeper than that page can reach is clamped and the user lands in the wrong place. Default to enter-only route transitions. Add an exit only when the design needs it, keep it short and opacity-only, and test Back on a long page.

**Focus logic in the layout instead of the page.** A `useEffect` on `location` in the animated layout runs as soon as the URL changes. Under `mode="wait"` the new page has not mounted yet, so it focuses the leaving page's heading (which is `inert` if you followed the presence rules) or nothing, and focus ends on `<body>`. Move focus from the new page's heading when it mounts, so the timing follows the animation automatically. `accessibility` has the component.

**Router-dependent hooks inside the leaving page.** With the `useOutlet` pattern the exiting page stays mounted under the same router, so any `useLocation` or `useSearchParams` call inside it re-renders with the new location during the exit. Breadcrumbs or effects derived from the location flicker to the new values, and effects keyed on the location fire in a page that is leaving. A page that renders React 19's `<title>` keeps it in `<head>` until the exit ends, so two titles are present at once, which React documents as undefined browser behavior. Render the title outside the animated subtree, for example in the layout from the deepest match's `handle` in data mode. Keep location reads in the page shallow, never start requests from a location effect in a page that may be exiting (check `useIsPresent()`), and keep exits short.

### Layout and geometry

**Animating layout properties.** ``animate={{ width: `${pct}%` }}`` on a progress bar, `left` on a drawer, `top` on a toast stack, `height` on many rows at once. Each frame runs layout and paint for the affected subtree and janks on mid-range phones. Use `scaleX` with `originX: 0` for bars, `x`/`y` for movement, and the `layout` prop for real geometry changes (it animates with transforms). One accordion animating `height: "auto"` is acceptable; a list of them opening during scroll is not.

**`layout` and `animate` fighting over geometry.** `<motion.div layout animate={{ width: expanded ? 480 : 240 }}>`. Two mechanisms own the same box, which produces snapping and jitter. Motion's rule: layout changes come from `style` or `className`; `layout` animates the result. Drop the geometric `animate`.

**Layout animation distortion.** Layout animations scale the element, so a `border-radius` or `box-shadow` set in a CSS class stretches mid-animation; set them through `style` (or an animation prop) so Motion can correct them. Text children stretch unless they get `layout` or `layout="position"`. Inside a scrollable container, add `layoutScroll` to the container or the animation starts from the wrong offset; on a `position: fixed` container, add `layoutRoot`. `display: inline` elements do not animate and SVG layout animation is not supported.

**Global `layoutId` collisions.** `layoutId` is global. Two instances of a tabs component that both use `layoutId="indicator"` make the indicator fly between them. Wrap each instance in `<LayoutGroup id={instanceId}>`.

**Transformed ancestor breaks fixed descendants.** Any `transform` (including Motion's `x`, `y`, `scale` on a page wrapper) and `will-change: transform` make the element the containing block for `position: fixed` descendants. Modals, toasts and GSAP pins inside it position against the wrapper and get clipped or jump. Portal overlays to `document.body`, keep page-transition wrappers to opacity, and never wrap a pinned section in a transformed container.

### Scroll and continuous values

**`setState` per scroll frame.** `useMotionValueEvent(scrollYProgress, "change", setProgress)` or `window.addEventListener("scroll", () => setY(window.scrollY))`, then ``style={{ width: `${progress * 100}%` }}``. Every frame re-renders the component and all its children. Bind the motion value directly (`style={{ scaleX: scrollYProgress }}`) and derive with `useTransform`. When the UI needs a discrete state (header hidden or shown), compute the boolean in the event handler and set state only with that boolean, so React bails out on unchanged values.

**Parallax and scroll-linked values ignoring reduced motion.** `MotionConfig reducedMotion="user"` disables transform and layout animations. A `y` motion value derived from `useScroll` and bound through `style` is not an animation and keeps moving. Pass `0` instead of the motion value when the user prefers reduced motion; the Motion docs use exactly this pattern.

### First render and bundle

A Vite SPA renders on the client only: `index.html` ships an empty root and `createRoot` builds everything. Hydration mismatches and server-rendered `initial` styles do not exist here. If a route is ever prerendered or server rendered, those rules come back: `initial` must then be identical on server and client.

**One-time window reads in `initial`.** `initial={{ x: window.innerWidth }}` or `initial={{ y: isMobile ? 16 : 48 }}` with `isMobile` read from `window` during render. In a SPA this does not throw, but the value is captured once: after a resize or rotation the drawer starts from the old width, and a JavaScript breakpoint drifts from the CSS one. Use relative units (`x: "100%"`) and keep breakpoint differences in CSS.

**LCP content faded in on first load.** `initial={{ opacity: 0 }}` with a delay on the hero headline or LCP image. In a SPA nothing paints until the bundle has downloaded and rendered; the fade adds its delay and duration on top of that, and Chromium does not count opacity-0 elements as LCP candidates, so LCP moves to after the reveal. Do not fade LCP content on the app's first render. Animate a small transform with opacity left at 1, or skip the entrance on the first render and keep it for later navigations (`motion-patterns.md` has the flag).

**Two Motion versions in one bundle.** `motion` depends on `framer-motion` at its own version. When `package.json` also pins `framer-motion` directly at a different version, the lockfile resolves two copies, and providers from one copy (`MotionConfig`, `LazyMotion`, `LayoutGroup`, `AnimatePresence`) are invisible to components from the other. The reduced-motion policy set at the root silently stops applying, and shared layout breaks across the boundary. Depend on one package, import from one path, and confirm the lockfile resolves a single `framer-motion` version. Motion's upgrade guide describes switching to `motion` and `motion/react`; that is a package rename with no behavior change, so do it as its own change or not at all.

**Full Motion bundle for simple effects.** The `motion` component preloads every feature. On first-load routes that only fade and slide, use `m` (`import * as m from "framer-motion/m"`, or `motion/react-m`) inside `<LazyMotion features={domAnimation}>`; load `domMax` (layout and drag) asynchronously where it is needed. Add `strict` to `LazyMotion` so a stray `motion.*` throws instead of quietly pulling the full bundle back in.

### Input, accessibility and cost

**No reduced-motion path.** Nothing checks the preference, or only `MotionConfig` does. `reducedMotion="user"` covers Motion transform and layout animations. It does not cover scroll-linked motion values, CSS animations, GSAP, autoplay video or Lottie. Parallax, large travel, zooms and infinite loops each need an explicit static or opacity alternative. State feedback stays; decoration goes.

**Animations blocking input.** `pointer-events: none` or `disabled` while `isAnimating`, click handlers that `await` an animation first, `isAnimating` flags that stick when an animation is interrupted or unmounted, `mode="wait"` on a surface the user is clicking through. Accept input immediately. Motion animations retarget from the current value when the target changes, so a rapid toggle needs no guard.

**Stagger on long lists.** `delayChildren: stagger(0.05)` over 200 rows makes the last row wait ten seconds, and the whole list replays on every refetch or filter change when keys or the parent remount. Clamp the delay (`Math.min(index, MAX_STAGGERED) * STEP`), stagger only what is on screen, and use `AnimatePresence initial={false}` so existing rows do not animate on first render.

**Infinite animations draining battery.** `repeat: Infinity`, GSAP `repeat: -1` or CSS `infinite` on decorative blobs, gradients and shimmer that keep running offscreen and for users who asked for less motion. Run loops only while in view (`useInView`, or ScrollTrigger `toggleActions`) and not under reduced motion; prefer a finite number of repeats for attention cues.

**`will-change` left on.** `will-change: transform` in a stylesheet on every card, or a static inline style on a long list. The browser keeps those optimizations far longer than it otherwise would, which costs memory on exactly the devices that needed help, and `will-change: transform` also creates a containing block that breaks fixed descendants. MDN calls it a last resort. Apply it only to the few elements with a measured problem, and only while they animate.

**Hover-only and drag-only interactions.** `whileHover` reveals the only path to an action, or drag is the only way to reorder or dismiss. Touch has no hover and keyboards cannot drag. Pair `whileHover` with `whileFocus` or a visible control, and give every consequential gesture a button.

### GSAP in React

**No cleanup.** `useEffect(() => { gsap.to(el, { scrollTrigger: { … } }) }, [])` with no revert. On a client-side route change the tween and ScrollTrigger stay registered against detached nodes and keep running on every scroll and refresh; returning to the page stacks a second trigger on top. If React removes the pinned element itself (for example it is the component's root), the removal can crash the navigation with React's `removeChild` "not a child of this node" error, because GSAP wrapped it in a pin-spacer React does not know about. Strict Mode's double effect creates two of everything in development, including nested pin-spacers. Use `useGSAP` from `@gsap/react` with a `scope` ref; it reverts tweens, timelines, ScrollTriggers, Draggables and SplitText created inside it on unmount.

**Handlers creating tweens outside the context.** A click handler calls `gsap.to(...)` after `useGSAP` has run. That tween is not recorded and is never reverted. Wrap handlers with `contextSafe`.

**Animating the pinned element.** `gsap.to(track, { x, scrollTrigger: { trigger: track, pin: true } })`. ScrollTrigger measures the pinned element up front, and GSAP's docs say not to animate it. Pin the section and animate a child track.

**Layout changes after ScrollTrigger measured.** Lazy images without dimensions, web fonts, data that arrives after mount, an accordion above the trigger. Start and end positions are stale, pins release early, the last cards of a horizontal gallery are unreachable. ScrollTrigger refreshes on `resize`, `load`, `DOMContentLoaded` and `visibilitychange`, and lazy images load after `load`. Reserve image dimensions, rebuild triggers when data changes (`dependencies` with `revertOnUpdate: true`), and call `ScrollTrigger.refresh()` after other layout-changing content settles (`refresh(true)` from a click handler waits for the DOM to render).

**Triggers created out of page order.** Components mount in React order, not scroll order, so a pin created after a trigger lower on the page shifts that trigger's positions. Create triggers top to bottom, or set `refreshPriority` or call `ScrollTrigger.sort()`.

**Pinning on mobile.** Mobile toolbars show and hide as the user scrolls, which resizes the viewport and refreshes every trigger mid-scroll: visible jumps and pins that start in the wrong place. Set `ScrollTrigger.config({ ignoreMobileResize: true })`, size pinned sections with `svh` rather than `vh`, and test on a real iOS Safari device. `ScrollTrigger.normalizeScroll()` moves scrolling onto the JavaScript thread; GSAP marks it experimental, so it is a last resort.

**Pins in flex or transformed containers.** If the pin's container is `display: flex`, `pinSpacing` defaults to `false` and the following content scrolls under the pinned section. A transformed ancestor breaks the `position: fixed` used while pinned. Fix the DOM; `pinReparent` moves the element to `<body>` while pinned, and GSAP warns that reparenting can be expensive and to use it only if you must.

## Decision rules

- **Which tool.** CSS transitions for hover, focus and pressed styles on a single element with no React state. Motion for state-driven UI: presence, layout, gestures, scroll-linked values. GSAP for long timelines scrubbed by scroll, pinning, and its plugins. Never let two libraries or two mechanisms own the same property on the same element.
- **Spring or tween.** A spring for anything interruptible or retargeted (sheets, drag release, layout, toggles the user can flip mid-motion). A tween with a fixed duration where arrival time matters (fades, tooltips, pressed feedback). No overshoot on destructive or spatially precise destinations.
- **Duration.** Most UI animation sits between 100 and 500 ms, longer for longer travel; the more often the user sees it, the shorter and subtler it gets. Put the scale in `MotionConfig` or a shared transitions module, not per component.
- **`layout` or explicit values.** `layout` when a React commit changes geometry you did not want to compute (reorder, expand, responsive reflow). Explicit `animate` values for a known transform. Never both on the same box.
- **Presence mode.** `sync` by default, `popLayout` when siblings should reflow at once, `wait` only for a single short slot.
- **Reduced motion.** `MotionConfig reducedMotion="user"` at the root as the baseline. `useReducedMotion` wherever the reduced design differs: parallax, scroll-linked values, loops, large travel, video. `gsap.matchMedia()` with `(prefers-reduced-motion: no-preference)` around every GSAP scene, with a CSS layout that works without the scene.
- **Bundle.** `m` plus `LazyMotion` when Motion is on a first-load route that does not need layout or drag; load `domMax` asynchronously for the routes that do.
- **Route transitions.** Enter-only by default: an opacity fade keyed on `pathname`, skipped on the app's first render. Add an exit only when the design needs one, and then with a captured `useOutlet()` element or `<Routes location={location}>`, a short opacity-only exit, focus moved from the new page's heading on mount, and Back tested on a long page.

## Review checklist

- Is every `AnimatePresence` always mounted, with the condition inside it and stable id keys on every direct child?
- Does every mutation and navigation happen in the event handler, independent of any animation callback?
- Is exiting content `inert`, and does focus move somewhere deliberate when a dialog closes or a route changes?
- Does any element have both `layout` and a geometric `animate`, or animate `width`, `height`, `top` or `left` on more than one element?
- Are `border-radius` and `box-shadow` on layout-animated elements set through `style`, and do scroll containers have `layoutScroll`?
- Are repeated components that use `layoutId` wrapped in `LayoutGroup` with a unique id?
- Is any modal, toast or pin inside a transformed or `will-change` ancestor?
- Does anything call `setState` from a scroll, pointer or motion-value change on every frame?
- Do `initial` values use relative units and CSS breakpoints rather than one-time reads of the window size?
- Is the LCP element at full opacity on the app's first render?
- Does the lockfile resolve exactly one `framer-motion` version, with every import coming from one path (`framer-motion` or `motion/react`)?
- Does every route transition with an exit render a captured `useOutlet()` element or `<Routes location={location}>` instead of a bare `<Outlet />`, and is every route wrapper keyed so query-string changes do not replay it?
- After a route transition, does focus land on the new page's heading, and does Back return to the saved scroll offset?
- Does every parallax, scroll-linked value, loop, GSAP scene and autoplay have a reduced-motion alternative?
- Can the user click, type and navigate while any animation is running?
- Are staggers clamped, loops stopped offscreen and `will-change` absent from stylesheets?
- Is every GSAP tween and ScrollTrigger created inside `useGSAP` or `contextSafe`, with the pinned element itself left unanimated?
- Are triggers created in page order and refreshed after late content, and has the pinned section been scrolled on a real phone?

## References

- [motion-patterns.md](references/motion-patterns.md): read when writing presence, dialogs, lists, layout animation, scroll-linked effects, first-load entrances or React Router route transitions with Motion.
- [gsap-react.md](references/gsap-react.md): read before writing or reviewing any GSAP or ScrollTrigger code in React.
- [timing-and-principles.md](references/timing-and-principles.md): read when choosing durations, easing and springs, or judging whether a motion earns its place.
- [audit.md](references/audit.md): read when asked to audit or debug animation across a codebase; search patterns and the manual test matrix.

Related skills: `accessibility` for dialog and focus contracts, `frontend-engineering` for routing, data fetching and bundle work outside animation, `performance-benchmarking` for measuring frame and INP regressions.

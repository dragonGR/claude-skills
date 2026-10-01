# Auditing animation in a codebase

Read this when asked to audit, review or debug animation across a React codebase rather than a single diff. It lists where to look, what each hit usually means, and how to confirm a finding before reporting it.

## 1. Inventory

- `package.json` and the lockfile: `motion`, `framer-motion` (how many resolved versions), `gsap`, `@gsap/react`, `lottie-*`, CSS animation libraries. More than one resolved `framer-motion` version is a finding on its own. Importing from `framer-motion` instead of `motion/react` is not.
- React major (affects `inert`, `ref` as a prop), and how React Router is mounted: declarative (`<BrowserRouter>`, `<Routes>`) or data mode (`createBrowserRouter`, `<RouterProvider>`), and whether `<ScrollRestoration>` is rendered.
- Where the root providers are: `MotionConfig`, `LazyMotion`, and which package each is imported from.

## 2. Searches and what they usually mean

Run these with the search tool across `src`/`app`/`components`. They are ripgrep regexes, so `|` is alternation. A hit is a lead, not a finding.

- `from ['"]framer-motion` next to `from ['"]motion/react`: possibly two resolved versions and split providers; the lockfile decides.
- `<Outlet\s*/>` or `<Routes>` without `location=` inside `AnimatePresence`: the exiting page renders the new route.
- `key=\{location\.key\}` on a page wrapper: transitions replay and pages remount on query-string changes.
- `&&\s*\(?\s*<AnimatePresence`: presence boundary inside a condition.
- `key=\{(i|index|idx)\}` near `AnimatePresence` or `exit=`: index keys on exiting items.
- `onExitComplete|onAnimationComplete`: mutations, navigation or focus tied to animation.
- `mode=["']wait`: serialized transitions in front of user work.
- `layout` together with `animate=\{\{[^}]*(width|height)`: two owners of geometry.
- `animate=\{\{[^}]*(width|height|top|left|margin)`: layout properties animated.
- `useMotionValueEvent|addEventListener\(['"]scroll` followed by `set[A-Z]`: state per frame.
- `window\.|innerWidth|innerHeight` inside `initial=`: a size captured once that goes stale on resize.
- `initial=\{\{\s*opacity:\s*0` in hero or above-the-fold components: LCP content hidden for the length of the entrance.
- `repeat:\s*Infinity|repeat:\s*-1|infinite`: loops without view or preference gating.
- `will-change` in CSS or `willChange` in styles: permanent layer promotion.
- `stagger\(|staggerChildren`: unbounded stagger over data-driven lists.
- `useReducedMotion|prefers-reduced-motion|matchMedia\(`: coverage of the preference; absence is the finding.
- `gsap\.(to|from|fromTo|timeline)` outside `useGSAP` or `contextSafe`: leaks and unreverted tweens.
- `pin:\s*true`: check what is pinned, what is animated, the container and its ancestors.
- `ScrollTrigger\.create|scrollTrigger:` in components loaded lazily: creation order.
- `layoutId=` in reusable components: global id collisions.

## 3. Confirm before reporting

For each lead, check the thing that would make it harmless:

- Index key: does the list ever remove, insert or reorder while mounted? A static, never-changing list with index keys is not a bug.
- `onExitComplete`: does it only clean up visual state (reset a flag, scroll)? Then it is fine. It is a finding when it performs a mutation, navigation or focus move.
- `mode="wait"`: is it a single slot with short durations where overlap would look broken? Then it is a reasonable choice.
- Layout property animation: one element, occasional, not during scroll? Probably acceptable; say so rather than report it.
- Window read in `initial`: does the value matter after mount (a drawer that can open after a resize)? A one-shot entrance offset that is never reused is cosmetic. In a client-rendered SPA it is never a hydration bug; only report hydration if the route is prerendered or server rendered.
- Opacity-0 entrance: is the element above the fold on the app's first render? Below the fold and `whileInView` is a much smaller issue, and an entrance that runs only on later navigations does not affect LCP.
- Route transition with `<Outlet />`: is the keyed wrapper inside `AnimatePresence`? An enter-only wrapper keyed on the path with no `AnimatePresence` keeps nothing old mounted, so `<Outlet />` is fine there.
- GSAP without `useGSAP`: is there a `gsap.context` with `revert()` in cleanup, or a `kill()` on every created trigger? Then cleanup exists.
- Reduced motion: is it handled in CSS (`@media (prefers-reduced-motion: reduce)`) for the same elements? Check before claiming it is missing.
- Mixed packages: does the lockfile resolve both to the same `framer-motion` version? If so, bundle and contexts are shared and only the inconsistent imports remain.

## 4. Manual test matrix

Run the changed interaction through each row. The failure signal is in the right column.

| Condition | How | Fails when |
| --- | --- | --- |
| Reduced motion | DevTools Rendering panel, emulate `prefers-reduced-motion: reduce`; OS setting on a phone | Parallax, loops, slides or GSAP scenes still move; state changes become invisible |
| Rapid input | Toggle, open/close and delete as fast as possible; double-click | Stuck states, duplicate requests, wrong item animates out, input ignored |
| Interrupted navigation | Navigate away during an exit or while a pinned section is active; back and forward | Mutation lost, `removeChild` error, leftover pin-spacer, wrong scroll position |
| Keyboard only | Tab, Shift+Tab, Enter, Escape through the flow | Focus on `<body>` after close, focus inside a leaving element, invisible focused elements |
| Slow device | DevTools CPU throttling, and a real mid-range Android phone | Dropped frames, delayed input response, long tasks during animation |
| Mobile scroll | Real iOS Safari and Chrome Android, scroll so toolbars collapse and expand | Pins jumping, triggers firing at the wrong place |
| First load | Hard reload with DevTools network throttling; Performance panel LCP marker | Hero invisible after the bundle renders, LCP after the entrance ends |
| Route change | Navigate by keyboard, then press Back from deep in a long page | Old page shows new content during exit, focus on `<body>`, wrong scroll offset after Back |
| Offscreen | Scroll loops out of view; switch tabs; Performance panel recording | Animation frames still produced for invisible elements |
| Render cost | React DevTools "highlight updates", Profiler during scroll | Components re-rendering every frame |

## 5. Automating the regression

Playwright can run a page with reduced motion and check that nothing moves or that a static alternative renders:

```ts
import { expect, test } from '@playwright/test'

test.use({ reducedMotion: 'reduce' })

test('gallery falls back to native horizontal scroll under reduced motion', async ({ page }) => {
  await page.goto(GALLERY_PATH)
  await expect(page.locator('.pin-spacer')).toHaveCount(0)
  await expect(page.getByRole('link', { name: LAST_CARD_LINK_NAME })).toBeAttached()
})
```

`page.emulateMedia({ reducedMotion: 'reduce' })` switches the preference mid-test when a test needs both modes. For focus, assert `document.activeElement` after closing a dialog, not only that the dialog disappeared.

## 6. Reporting

Report each confirmed finding with the file and line, the user-visible failure (what a user sees or loses), the condition that triggers it, and the fix direction from the catalogue. Drop leads that failed step 3, and say which conditions from step 4 you could not test (for example no real iOS device).

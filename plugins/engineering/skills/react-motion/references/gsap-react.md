# GSAP and ScrollTrigger in React

Read this before writing or reviewing GSAP code in React, especially anything with ScrollTrigger, pinning or scroll-scrubbed timelines.

## Setup

Register plugins once at module scope, including `useGSAP` itself:

```tsx
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { useGSAP } from '@gsap/react'

gsap.registerPlugin(ScrollTrigger, useGSAP)
```

`useGSAP` runs in a layout effect, wraps the callback in a `gsap.context()`, scopes selector strings to the `scope` ref, and reverts everything created inside it on unmount: tweens, timelines, ScrollTriggers, Draggables and SplitText. Signatures:

```ts
useGSAP(callback)                                   // empty dependency array by default
useGSAP(callback, [dep])                            // runs again when dep changes
useGSAP(callback, { scope, dependencies, revertOnUpdate })
const { context, contextSafe } = useGSAP({ scope })
```

With `dependencies`, objects are reverted only on unmount unless `revertOnUpdate: true`. When the animation's geometry depends on the data, set `revertOnUpdate: true` so the old triggers and pin-spacers go away before new ones are created. The callback may return a cleanup function for listeners it added.

## Cleanup

```tsx
// Before: leaks on route change, duplicates under Strict Mode
useEffect(() => {
  gsap.to('.card', { y: 0, scrollTrigger: { trigger: '.cards', start: CARDS_START } })
}, [])

// After: scoped selectors, reverted on unmount
const containerRef = useRef<HTMLDivElement>(null)

useGSAP(
  () => {
    gsap.to('.card', { y: 0, scrollTrigger: { trigger: '.cards', start: CARDS_START } })
  },
  { scope: containerRef },
)
```

Unscoped `'.card'` in the before version also matches cards in every other mounted component. If a codebase cannot adopt `@gsap/react`, the equivalent is `const ctx = gsap.context(() => { … }, containerRef)` in the effect and `return () => ctx.revert()`.

Tweens created later, in event handlers, need the context:

```tsx
const { contextSafe } = useGSAP({ scope: containerRef })

const handleOpen = contextSafe(() => {
  gsap.to('.panel', { height: 'auto', duration: PANEL_OPEN_S })
})
```

## Horizontal pinned gallery

The most common scroll scene, written so that it survives resize, late content, reduced motion and route changes.

```tsx
import { useRef, type ReactNode } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { useGSAP } from '@gsap/react'

gsap.registerPlugin(ScrollTrigger, useGSAP)

const MOTION_OK = '(prefers-reduced-motion: no-preference)'

export function HorizontalGallery({ children, itemCount }: { children: ReactNode; itemCount: number }) {
  const rootRef = useRef<HTMLDivElement>(null)
  const sectionRef = useRef<HTMLElement>(null)
  const trackRef = useRef<HTMLDivElement>(null)

  useGSAP(
    () => {
      const section = sectionRef.current
      const track = trackRef.current
      if (!section || !track) return

      const mm = gsap.matchMedia()
      mm.add(MOTION_OK, () => {
        const distance = () => track.scrollWidth - section.clientWidth

        gsap.to(track, {
          x: () => -distance(),
          ease: 'none',
          scrollTrigger: {
            trigger: section,
            pin: true,
            scrub: true,
            end: () => `+=${distance()}`,
            invalidateOnRefresh: true,
          },
        })
      })

      return () => mm.revert()
    },
    { scope: rootRef, dependencies: [itemCount], revertOnUpdate: true },
  )

  return (
    <div ref={rootRef}>
      <section ref={sectionRef} className="gallery">
        <div ref={trackRef} className="gallery-track">
          {children}
        </div>
      </section>
    </div>
  )
}
```

Why each piece is there:

- The pinned section sits inside a wrapper the component owns. Pinning inserts a pin-spacer element around the pinned node; if the pinned node is the component's root, the spacer lands in the parent's DOM, and an unmount without revert makes React throw `Failed to execute 'removeChild' on 'Node'`.
- The section is pinned and the track is animated. GSAP's docs say not to animate the pinned element, because ScrollTrigger pre-calculates its measurements.
- `x` and `end` are functions and `invalidateOnRefresh: true`, so every refresh (resize, load, manual) recomputes them from the current DOM. Values captured once go stale on the first resize.
- `gsap.matchMedia()` creates the scene only when the user has not asked for reduced motion and reverts it if the setting changes while the page is open. Without the scene the cards must still be reachable, so give the section a native fallback under the opposite query (for example `@media (prefers-reduced-motion: reduce) { .gallery { overflow-x: auto; scroll-snap-type: x mandatory; } }`). Do not put `overflow-x: auto` on the track itself: the tween translates the track, so a track clipped to the section's width would slide its own visible box off screen.
- `revertOnUpdate` with `itemCount` rebuilds the trigger when cards are added or removed, after React has committed the new DOM.
- Images inside cards need `width` and `height` attributes or an `aspect-ratio`, or `scrollWidth` changes after the trigger measured.

Keyboard check: Tab through the cards. A card translated off screen by the tween can receive focus while invisible. Either map focus to the scroll position that reveals it (listen for `focusin` on the track and scroll the window to the matching point between the trigger's `start` and `end`) or do not pin this content.

## Refresh after late layout changes

ScrollTrigger refreshes automatically on `resize`, `load`, `DOMContentLoaded` and `visibilitychange` (the default `autoRefreshEvents`). It does not know about:

- lazy images, which load after the `load` event;
- data fetched after mount;
- web fonts that swap in and change text height;
- accordions, tabs or "show more" above a trigger.

Prefer removing the layout shift (reserved dimensions, `font-display` choices, skeletons with the final height). When content truly changes size, call `ScrollTrigger.refresh()` after it settles. From a click handler, `ScrollTrigger.refresh(true)` waits at least one animation frame (up to about 200 ms) so the DOM has rendered before measuring. Refresh is a whole-page recalculation; call it once after a batch of changes, not per image.

## Order and priority

Triggers are calculated in creation order. React mounts components in tree order, and a lazily loaded section can mount after the sections below it. A pin created later shifts every trigger below it. Create triggers top to bottom; when that is impossible, give earlier-on-the-page triggers a higher `refreshPriority`, or call `ScrollTrigger.sort()` after creating them.

## Mobile

Mobile browser toolbars resize the viewport as the user scrolls, which triggers refreshes mid-scroll and makes pinned sections jump.

```ts
ScrollTrigger.config({ ignoreMobileResize: true })
```

With that set, vertical resizes of up to a quarter of the viewport height on touch-only devices do not trigger a refresh; start and end positions may then be slightly off, which is usually better than the jump. Size pinned sections with `svh` so their height does not depend on toolbar state. `ScrollTrigger.normalizeScroll(true)` moves scrolling onto the JavaScript thread, which stops the address bar showing and hiding on most mobile browsers and keeps pins in sync with repaints, but GSAP calls it experimental, it hands control back to the browser during multi-touch and after a pinch zoom, and it changes how scrolling feels. Use it only after the steps above failed on a real device, and re-test keyboard and assistive-technology scrolling afterwards.

## Pinning and CSS

- A transformed ancestor (any `transform`, `will-change: transform`, `filter`, or a Motion wrapper with `x`/`y`/`scale`) becomes the containing block for the `position: fixed` that pinning uses. Remove the transform from ancestors. `pinReparent: true` moves the pinned element to `<body>` while it is pinned, so selectors that depend on its ancestors stop matching; GSAP warns that reparenting can be expensive and to use it only if you must.
- If the pin's container is `display: flex`, `pinSpacing` defaults to `false`, and GSAP notes that the added padding would not push siblings in a flex or absolutely positioned parent anyway. Following content scrolls under the pinned section. Wrap the pinned section in a block-level element, or space the following content yourself.
- `content-visibility: auto` or `hidden` on or around triggers makes positions impossible to calculate.
- `anticipatePin: 1` hides the one-frame jump when a fast scroll hits a pin.

## Route changes and scroll restoration

`useGSAP` cleanup kills the old page's triggers and removes their pin-spacers when it unmounts, so the next page measures a clean document. Without it, the old triggers stay registered against detached nodes and are recalculated on every refresh, and coming back to the page adds a second set. ScrollTrigger also records scroll positions and restores them after a refresh. In a React Router app that renders `<ScrollRestoration>` (data mode), the router already owns scroll on navigation, so two systems can restore different offsets after a pin changes the page height. If Back lands in the wrong place on a page with pins, call `ScrollTrigger.clearScrollMemory()` so only the router restores, and test Back from below a pinned section.

## Review checklist

- Is every tween, timeline and ScrollTrigger created inside `useGSAP` (or a reverted `gsap.context`) with a `scope`?
- Are handlers that animate wrapped in `contextSafe`?
- Is the pinned element itself free of tweens?
- Are distance-dependent values functions with `invalidateOnRefresh: true`?
- Does content that changes size after mount either reserve its space or trigger one `refresh()`?
- Are triggers created in page order, or ordered with `refreshPriority` or `sort()`?
- Is every scene inside `gsap.matchMedia()` with a reduced-motion condition and a usable CSS fallback?
- Is `ignoreMobileResize` set and has the scene been scrolled on a real phone?
- Is there any transformed ancestor or flex container around a pin?

# Timing, easing and whether motion earns its place

Read this when choosing durations, easing or springs, when defining a project's motion tokens, or when judging whether a proposed animation should exist.

## What motion is for

An animation earns its place when it does one of four jobs: shows cause and effect (this press did that), preserves spatial continuity (the card became the detail view), directs attention to what changed, or confirms that an action worked. A sequence with none of those jobs is decoration. Decoration is the first thing to cut under reduced motion, on slow devices, and in review.

Read each motion as a state transition: action, feedback, stable result. If the result is unclear with the animation removed or with reduced motion on, the interface needs a clearer non-motion cue (text, icon, focus move), not a better animation.

## Durations

Nielsen Norman Group puts most UI animation between 100 and 500 ms: around 100 ms for simple feedback such as a toggle or checkbox, 200 to 300 ms for substantial changes such as a modal moving into view, 400 ms only for big movements, and at 500 ms it starts to feel like a drag. The more often a user sees an animation, the shorter and subtler it should be; the command palette someone opens fifty times a day gets the fastest transition in the product.

Build a small scale once and reference it everywhere:

```ts
// motion/tokens.ts
export const DURATION_S = {
  feedback: 0.1,
  surface: 0.25,
  travel: 0.4,
} as const
```

Motion takes durations in seconds; GSAP also uses seconds. CSS uses `ms` or `s` explicitly. Mixing them up is a common source of 250-second animations.

Duration also scales with distance and size: a tooltip and a full-screen sheet should not share a number. Exits can be shorter than entrances, because the user has already decided and is waiting on what comes next.

## Tween or spring

- Tween (duration plus easing) when arrival time matters and the motion is not interrupted: fades, tooltips, pressed states, progress.
- Spring when the motion can be interrupted or retargeted, or should feel physical: sheets, drag release, layout changes, toggles the user can flip mid-motion. Springs carry the current value into the new target instead of restarting.
- In Motion, `visualDuration` lets a spring be specified by roughly how long the bulk of the motion takes, which keeps springs on the same scale as tweens. `bounce` controls overshoot.
- No visible overshoot on destructive actions or on destinations that must look precise (a value landing in a chart, a row snapping into a slot).

Ease-out for elements entering or responding to the user, ease-in for elements leaving, ease-in-out for elements moving from one on-screen place to another. Linear only for continuous, scroll-scrubbed or looping motion, where easing would read as speeding up and slowing down.

## Principles as review lenses

| Principle | Use in UI | Failure to catch |
| --- | --- | --- |
| Anticipation | A slight pre-motion before a drag or sheet | Delaying an action the user already committed |
| Staging | One dominant motion points at what changed | Several surfaces moving at once, competing |
| Pose to pose | Named states (`closed`, `open`, `closing`) and transitions between them | Imperative DOM mutation with no state model behind it |
| Follow-through | Dependent details settle after the main change | Focus or interaction left behind on the leaving element |
| Slow in, slow out | Easing that matches distance and interruptibility | Bounce on precise or destructive destinations |
| Arc | Pointer-driven motion follows the user's path | Objects teleporting during a spatial interaction |
| Secondary action | A small cue confirming the main action | A second animation that hides the result |
| Timing | Short for frequent feedback, longer for orientation | One duration for every distance and priority |
| Exaggeration | Extra contrast only where subtle feedback would be missed | Large travel, flashes or scale as default emphasis |
| Solid drawing | Consistent weight and alignment across states | Scaling text or icons until they blur or jump |

## Reduced motion policy

Reduced motion means less movement, not no feedback. Keep: color and opacity changes that show state, focus rings, short fades for content appearing. Replace with a fade or a static state: parallax, zooms, large translations, page slides, spinning or bouncing decoration, autoplay video and scroll-scrubbed scenes. Remove: purely decorative loops.

Flashing content is a separate, harder limit: WCAG 2.3.1 forbids content that flashes more than three times in any one-second period unless the flash stays below the general and red flash thresholds, whatever the user's settings.

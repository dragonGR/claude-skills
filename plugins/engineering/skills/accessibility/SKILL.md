---
name: accessibility
description: Accessible web UI to WCAG 2.2 AA, covering semantics, ARIA, keyboard and focus, dialogs, comboboxes, toasts and live regions, form errors, login, contrast, motion and target size. Use when building or reviewing interactive UI, fixing keyboard or screen reader bugs, or running an accessibility audit.
license: MIT
metadata:
  author: Alex Tsanis
---

# Accessibility

Target WCAG 2.2 level AA unless the project says otherwise. Section 508 and EN 301 549 both point to WCAG level AA.

Linters already catch missing `alt` and invalid ARIA attribute names. The bugs that ship and lock people out are different: custom widgets with half an ARIA pattern, focus that falls to `<body>` after something changes, announcements that never fire or fire twice, and login flows that block paste. Wrong ARIA is worse than none, because it overrides what the browser would have exposed correctly. When you report a failure, name the success criterion and its level, and check the exemptions first: a finding that WCAG does not require is a false positive.

## Failure catalogue

Each entry: what it looks like in code, why it breaks, what to do. Criterion numbers and levels are WCAG 2.2.

### Names, roles and semantics

**Clickable div.** `<div onClick={save}>`, `<span className="link" onClick>`, `<a onClick>` with no `href`, `<a href="#" onClick={openModal}>`. No tab stop, no role, no Enter or Space (2.1.1 A, 4.1.2 A). Use `<button type="button">` for actions and `<a href>` for navigation; restyle the native element instead of avoiding it. `type="button"` matters inside a form, where the default type submits.

**`role="button"` without the rest.** `<div role="button" onClick>` announces as a button but still has no tab stop and ignores the keyboard. A faithful copy needs `tabIndex={0}`, activation on Enter keydown and on Space keyup, `preventDefault` on Space keydown so the page does not scroll, disabled handling, and `aria-pressed` or `aria-expanded` if it toggles. That list is the argument for `<button>`.

**Icon button without a name.** `<button><TrashIcon /></button>` is read as "button". Add `aria-label` or visually hidden text, and `aria-hidden="true"` on the SVG. In repeated rows, put the row's identity in the name ("Delete invoice INV-2041") so twenty "Delete" buttons are distinguishable in a screen reader's list of controls. `title` alone is a last-resort name that sighted keyboard and touch users never see.

**Visible label missing from the accessible name.** The button reads "Buy now" but has `aria-label="Add product to basket"`. A voice-control user says "click Buy now" and nothing happens (2.5.3 A). The name must contain the visible text, ideally at the start. Usually the fix is deleting the `aria-label`.

**`aria-label` on something that cannot be named.** `<div aria-label="Account summary">`, `<span aria-label>`, `<p aria-label>`. ARIA 1.2 prohibits names on `generic`, `paragraph` and similar roles, and screen readers ignore them inconsistently, so the text is lost. Put the words in content (visible or visually hidden), or use an element whose role takes a name: `<nav aria-label>`, `<section aria-labelledby>` (becomes a region landmark), `role="img"` for a graphic.

**`aria-hidden` on focusable content.** An off-canvas drawer or inactive carousel slide with `aria-hidden="true"` whose links are still in the tab order; `aria-hidden` on `#root` while a modal that renders inside it is open. Keyboard focus lands on things the screen reader says do not exist. Use `inert` (Baseline since April 2023), which removes a subtree from both the tab order and the accessibility tree, or `hidden` for content that is not shown.

**Duplicate ids.** A reusable `<EmailField>` hardcodes `id="email"` and `aria-describedby="email-error"` and is rendered twice on the page. `for` and `aria-describedby` resolve to the first match, so the second field is unlabelled and reads the first field's error. Generate ids with `useId()` (React) or the framework's equivalent and derive related ids from it.

**Nested interactive elements.** A product card wrapped in `<a>` that contains an "Add to cart" `<button>`. Invalid HTML, the link's name becomes the entire card text, and the inner button either cannot be reached or triggers both. Put the link on the card heading, stretch its hit area over the card with a positioned `::after`, and lift the other controls above it with `position: relative`.

**Positive `tabindex`.** `tabIndex={1}` and up pulls elements to the front of the tab order in document-wide order, which never matches the visual layout (2.4.3 A). Use only `0` and `-1`; fix order in the DOM.

### Keyboard and focus

**Focus indicator removed.** `*:focus { outline: none }`, `outline: 0` in a CSS reset, Tailwind `focus:outline-none` with no `focus-visible:` replacement (2.4.7 AA). Style `:focus-visible`. An author-drawn indicator needs 3:1 against adjacent colors (1.4.11 AA). Prefer `outline` with `outline-offset`: a ring drawn only with `box-shadow` disappears in forced colors mode, where `box-shadow` is forced to `none`.

**Focus lost after removing the focused element.** Deleting a row, closing a popover by unmounting it, swapping "Edit" for "Save", re-rendering a filtered list. The focused node is gone, focus falls to `<body>`, the next Tab starts at the top of the page, and the screen reader says nothing. Decide the destination before the removal: the same control on the next row, else the previous row, else the list's heading or container (`tabIndex={-1}`). Move focus after the DOM commits. See `references/focus-and-announcements.md`.

**Focus lost on client-side navigation.** A React Router navigation swaps the page but leaves focus on the link that was clicked, which is often gone (focus falls to `<body>`) or still sitting in the nav. React Router has no route announcer and moves no focus, so a screen reader user hears nothing and the next Tab starts from the old place. After every pathname change except the first load, move focus to the new page's `<h1 tabIndex={-1}>`; the screen reader reads the heading, which tells the user the page changed. Skip changes that only touch filters, sort or pagination in the query string. Render the heading in the loading state too (TanStack Query `isPending`, a Suspense fallback): if it appears only after data arrives, the focus call runs against nothing.

**Route focus that fights scroll restoration.** `heading.focus()` scrolls the heading into view by default. On Back, that throws away the offset the user is returning to. Call `focus({ preventScroll: true })` and let scroll be owned by one thing: in data mode (`createBrowserRouter`), render `<ScrollRestoration />` once in the root route; it saves offsets per history entry in `sessionStorage` and restores them on navigation, and it moves no focus. Declarative mode (`<BrowserRouter>`) has no `<ScrollRestoration>`, and a client-side push does not scroll, so the new page opens at the old page's offset with focus on a heading above the viewport. Scroll to the top yourself on non-`POP` navigations (`useNavigationType()`).

**Stale or duplicate page title.** Every route shows the same `document.title`, so the history list, tabs and bookmarks cannot tell pages apart (2.4.2 A). Give every route a unique, specific title ("Invoice INV-2041, Billing, Acme"). React 19 hoists a `<title>` rendered in a page component into `<head>`; its children must be one string (`` {`Invoice ${number}`} ``, not `Invoice {number}`, which is an array and throws), and only one may render at a time, so a layout and a page that both render one, or an exiting page in an animated transition, leave two titles with undefined results. Pick one mechanism per app (React's `<title>` or `react-helmet-async`) and set a title in the page's first render (from route params while the data loads), refining it when the data arrives, so the old page's title never sits on the new page. Focusing the `<h1>` already announces the page; do not also push the title into a live region, or it is read twice.

**Skip link that does nothing.** `.skip { display: none } .skip:focus { display: block }` never shows, because `display: none` is not focusable. A working link then jumps to `#main` and the heading lands under a 64px sticky header, or a hash router swallows `#main`. Hide the link with the visually-hidden pattern until `:focus`, target `<main id="main" tabIndex={-1}>`, and set `scroll-padding-top` on `html` from the same custom property that sizes the header. A broken skip link is not automatically a 2.4.1 failure (headings and landmarks can also satisfy it), but sighted keyboard users get nothing from landmarks, so fix it anyway.

**Focused element hidden under sticky UI.** Tabbing down a page moves focus behind a sticky header, cookie banner or chat launcher. 2.4.11 AA fails only when the focused element is entirely hidden; partial cover is the AAA criterion 2.4.12. Set `scroll-padding-top` and `scroll-padding-bottom` on the scroller to the sticky heights, or make the banner modal.

**Single-character shortcuts.** A document-level `keydown` handler maps `/`, `j`, `k`, `e` or `d` to actions. Speech-input users dictate words and fire them (2.1.4 A). Offer a way to turn them off or remap them to include a modifier, or listen only while the owning component has focus. Always ignore events whose target is an input, textarea, select or `contenteditable`.

**Action on pointer down.** `onMouseDown={deleteItem}` or `onPointerDown` commits an action, so there is no way to slide off and cancel (2.5.2 A). Commit on `click`.

### Dialogs, popovers and live updates

**Modal without modal behavior.** A fixed `<div className="modal">` over the page: Tab walks into the page behind, Escape does nothing, the screen reader reads the background, and closing drops focus on `<body>`. Use `<dialog>` opened with `showModal()`: the browser makes the rest of the page inert, closes on Escape via the `cancel` event, focuses the first focusable element or the one with `autofocus`, and `close()` returns focus to the element that was focused before. Label it with `aria-labelledby` pointing at its heading. For a destructive confirmation, initial focus goes on the least destructive button. In React, the `autoFocus` prop does not reach `showModal()`: React leaves the attribute out of the DOM and calls `focus()` at commit, while the dialog is still closed and unfocusable. Call `focus()` on the chosen element right after `showModal()`.

**`<dialog>` opened or closed the wrong way.** `<dialog open={isOpen}>` in JSX renders a non-modal dialog: no inertness, no Escape. `{isOpen && <dialog>}` unmounts the element, and the HTML removing steps skip the close algorithm, so focus is never returned. Keep the element mounted, call `showModal()` and `close()` from an effect, and sync state from the `close` event (Escape closes it without your state knowing). Template in `references/widget-patterns.md`.

**Custom select or combobox that only works with a mouse.** Typical breakage: the input has no `role="combobox"`, `aria-expanded` never changes, options are bare `<div>`s, arrow keys move DOM focus into the list (typing stops working) or move a highlight with no `aria-activedescendant` (the screen reader hears nothing), the active id points at an option that a virtualized list has unmounted, Escape does not close, and the result count is not announced. Use native `<select>` when you do not need filtering or rich option content. Otherwise implement the APG combobox exactly or use a maintained library that does, and test with a screen reader.

**Hover-only content.** Tooltips, row actions with `opacity: 0` until `:hover`, and mega menus that open on `mouseenter`. Keyboard and touch users never see them, and an invisible focused button fails 2.4.7. Show on `:focus-visible` and `:focus-within` too. Author-built hover or focus popups must be dismissible with Escape without moving the pointer, stay open while the pointer moves onto them, and persist until dismissed (1.4.13 AA). The `title` attribute is exempt from 1.4.13, but it is still unusable from a keyboard or touch screen, so never put required information only there.

**Live region inserted with its content.** `{saved && <div role="status">Saved</div>}`: the region and the text arrive in the same DOM change, and many browser and screen reader combinations announce nothing. Keep an empty region mounted from the first render and change its text. Setting the same message twice may not be re-announced; clear it, then set it on a later tick.

**Toast announced twice, or never.** Twice: `role="alert"` nested inside an `aria-live` container, two toast providers mounted (layout and page), or a toast that is also rendered into a second announcer. Never: the toaster mounts on first use (see the previous entry) or uses `aria-live="off"`. Also, a toast that disappears after a few seconds with an Undo button is unreachable by keyboard; if the toast is the only way to get that information or action, it is a time limit under 2.2.1 A. Keep actionable toasts until dismissed, or offer the action elsewhere. Toasts never take focus.

**Auto-playing carousel.** Rotation that starts on its own and runs over five seconds needs a pause control (2.2.2 A). Per the APG carousel pattern: stop rotating while the pointer hovers, stop when keyboard focus enters and do not resume until the user presses the rotation control, label the button by its action ("Stop slide rotation"), keep the slide container `aria-live="off"` while rotating and `"polite"` when stopped, and make off-screen slides `inert`.

### Forms, errors and sign-in

**Placeholder as label.** The only label vanishes on the first keystroke, and placeholder grey usually fails contrast. Placeholder text is in scope for 1.4.3; it is not exempt. Use a visible `<label>` that stays while the user types; the placeholder, if kept, shows a format example.

**Errors not linked, or color only.** A red border and no text (1.4.1 A), error text not attached to the field, no `aria-invalid`, focus left on the submit button. On failed submit, move focus to the first invalid field or to an error summary whose items link to the fields. Each field gets `aria-invalid="true"` and `aria-describedby` listing its hint and error ids. Error text says how to fix the value ("Enter a date like 31/12/2026") rather than "Invalid" (3.3.1 A, 3.3.3 AA). Validate on blur or submit; do not announce an error on every keystroke.

**Disabled submit with no explanation.** The submit stays `disabled` until the form is valid. Native disabled buttons are skipped by Tab, so a screen reader user never learns why nothing works, and no error is ever shown. Disabled controls are exempt from contrast, so this is not a contrast failure; the problem is that the errors 3.3.1 asks you to identify are never identified. Keep the button enabled and report errors on submit. If it must look disabled, use `aria-disabled="true"` (still focusable; block activation in the handler) with `aria-describedby` pointing at the reason.

**Missing `autocomplete`.** Fields for the user's own name, email, phone, address and card lack tokens (1.3.5 AA), or the login form has `autocomplete="off"`. Use `username`, `current-password`, `new-password` (sign-up and change password), `email`, `tel`, `given-name`, `family-name`, `street-address`, `postal-code`, `cc-number`, `cc-exp`. Fields about another person, such as a gift recipient, are outside 1.3.5.

**Cognitive test at login (3.3.8 AA).** `onPaste={(e) => e.preventDefault()}` on password or code fields, a six-box OTP input that accepts one character per box and breaks paste, scripts that fight password managers, "type the characters in the image" with no alternative, "enter characters 3 and 7 of your password". Allow paste and autofill everywhere in the flow, use one `<input autocomplete="one-time-code" inputmode="numeric">` for codes, and offer a non-memory method (passkey, email link). Object-recognition CAPTCHAs are an allowed exception at AA, not at 3.3.9 AAA.

**Re-asking for data in the same flow.** Billing address with no "same as shipping" option (3.3.7 A).

**Select that navigates on change.** `<select onChange={(e) => navigate(e.target.value)}>` changes context while a keyboard user is still choosing (3.2.2 A). Apply with a button, or say beforehand what happens.

### Visual presentation and motion

**Contrast checked wrongly.** Rules: text 4.5:1; large text (18pt, which is 24px, or 14pt bold, about 18.7px) 3:1; UI component boundaries, state indicators, meaningful icons and chart marks 3:1 against adjacent colors (1.4.3 and 1.4.11, AA). Placeholder text counts. Disabled controls, pure decoration and logotypes are exempt. `#9ca3af` placeholder on white is about 2.5:1 and fails. Measure text over images or gradients at its worst point, and measure translucent text as the composited color on the real surface.

**Forced colors breakage.** In Windows contrast themes (`@media (forced-colors: active)`), author colors are replaced with system colors, and `box-shadow` and non-URL `background-image` become `none`. Rings made of shadows vanish, selection shown only by background tint vanishes, and custom checkboxes whose checked state is a background color look unchecked. Add `outline: 2px solid transparent` next to shadow rings (outline color is forced to a visible system color), mark state with borders, text or icons, and use `forced-color-adjust: none` only for things like color swatches. Not a separate criterion; it is the focus and state information failing for those users.

**Reduced motion ignored.** Parallax, large slide or zoom transitions and auto-scroll with no `prefers-reduced-motion` handling. Honor it: it costs one media query and prevents vestibular symptoms. For audits, be precise: Animation from Interactions (2.3.3) is AAA. What fails AA is motion that starts on its own and runs over five seconds without a pause (2.2.2 A) and flashing more than three times a second (2.3.1 A). Write motion inside `@media (prefers-reduced-motion: no-preference)` so the default is still.

**Zoom blocked or content clipped.** `<meta name="viewport" content="maximum-scale=1, user-scalable=no">` blocks pinch zoom where honored (1.4.4 AA). Fixed-height text boxes with `overflow: hidden` clip when users raise line height to 1.5 or letter spacing to 0.12em (1.4.12 AA). Two-dimensional scrolling at a 320 CSS px wide viewport fails 1.4.10 AA, except content that needs it, such as data tables and maps.

**Small targets.** A 16px close X, icon buttons packed in a table row (2.5.8 AA). Targets need 24 by 24 CSS px, or enough space that a 24px circle centred on each does not touch another target or its circle. Links inside a sentence are exempt. 44 by 44 is 2.5.5 AAA; use it for primary touch controls but do not report its absence as an AA failure. Enlarge the hit area with padding or a positioned pseudo-element, not by scaling the icon.

**Drag-only interaction.** Sortable lists, kanban boards, slider thumbs, map pins (2.5.7 AA). Keyboard support alone does not satisfy this; there must be a single-pointer path such as "Move up" and "Move down" buttons, a "Move to column" menu, or clicking the slider track.

### Structure and non-text content

**Headings that are not headings.** A styled `<div className="h2">` used as a section title (1.3.1 A) disappears from screen reader heading navigation. Real headings used only for font size mislead the same way. A skipped level (`h2` then `h4`) is a best-practice issue (axe tags `heading-order` as best practice), not a WCAG failure by itself; report it as such.

**Data table without headers.** A `<table>` whose header row is bold `<td>`, or a CSS grid of `<div>`s styled as a table. Use `<caption>`, `<th scope="col">` and `<th scope="row">` for row labels. For sortable columns put a `<button>` in the `<th>` and `aria-sort` on the `<th>`. Do not add `role="grid"` to a static table; it promises arrow-key cell navigation you then have to build.

**Chart with no text alternative.** An SVG chart exposes hundreds of unnamed paths, or nothing, and values appear only in a hover tooltip (1.1.1 A). Give the chart `role="img"` and `aria-labelledby` pointing at a title and a one-sentence takeaway, provide the data as a real table (visible or in a disclosure), and distinguish series by more than color (1.4.1 A) with direct labels or markers.

## The announcement bug in one diff

```tsx
// Before: region and text appear together, often silent
{saveState === 'saved' && <div role="status">Changes saved</div>}

// After: region exists from the first render; only its text changes
<div role="status" aria-live="polite" className="visually-hidden">
  {saveState === 'saved' ? 'Changes saved' : ''}
</div>
```

`role="status"` already implies `aria-live="polite"`; repeating it costs nothing and helps browser and screen reader pairs that do not map the role.

## Decision rules

Where focus goes:

| Event | Focus |
| --- | --- |
| Modal opens | Inside it: first field, least destructive button for confirmations, the heading (`tabIndex={-1}`) for long content |
| Modal closes | The trigger; if the trigger is gone, the nearest stable control that led to it |
| Focused item deleted | Same control on the next item, else previous, else the list heading or container |
| Client route change | New page's `<h1 tabIndex={-1}>`, focused with `preventScroll: true` |
| Submit fails validation | First invalid field, or an error summary when there are several errors |
| Save, add to cart, results updated | Stays put; announce through a status region |

- Move focus when the user's next action is in a new place. Announce when they should keep working where they are.
- `role="status"` (polite) for results and confirmations. `role="alert"` only for errors that block the task or need action now. One region per message, never nested.
- Native element when one exists (`button`, `a href`, `select`, `dialog`, `details`/`summary` for simple disclosure, `input type="range"`). An APG pattern only when native cannot express the widget, implemented fully.
- Site navigation dropdowns are disclosure buttons, not `role="menu"`; APG reserves menus for application-style command menus with their full arrow-key model.
- Tabs activate on focus when panels render without noticeable delay; otherwise require Enter or Space (APG).
- `disabled` for controls that truly cannot be used and need no explanation. `aria-disabled="true"` plus a described reason when users need to discover the control and learn why.
- `inert` for content that is present but must not be reachable (background of a custom modal, off-screen slides). `hidden` for content that is not shown.

## Review checklist

- Does every action use `<button>` and every navigation `<a href>`, with no `onClick` on `div`, `span` or `href="#"`?
- Does every icon-only control have a name that contains its visible text, if any, and identifies the row it acts on?
- Is `aria-label` used only on elements whose role accepts a name?
- Is there no `aria-hidden="true"` on or above anything focusable?
- Are ids generated per instance so two copies of a component cannot collide?
- Does every focusable element show a `:focus-visible` indicator with 3:1 contrast that survives forced colors?
- After delete, filter, close and route change, does focus land on a defined element rather than `<body>`?
- Does every route set a unique title through a single mechanism, and does Back return to the previous scroll offset with focus on the page heading?
- Does every modal use `showModal()`, close on Escape, return focus via `close()`, and stay mounted while closing?
- Do custom comboboxes set `role`, `aria-expanded`, `aria-controls` and `aria-activedescendant`, keep DOM focus in the input, and point only at rendered options?
- Is every status region mounted empty before the first message, with exactly one region per message?
- Do actionable toasts stay until dismissed, and does anything that moves on its own for over five seconds have a pause?
- Does every field have a visible label, correct `autocomplete`, and errors tied by `aria-describedby` with `aria-invalid`?
- Can users paste into every sign-in field, including one-time codes?
- Is text contrast 4.5:1 (3:1 large) including placeholders, and are exemptions (disabled, decorative, logos) applied only where they really apply?
- Are pointer targets 24px or spaced, and does every drag interaction have a click or tap alternative?
- Is motion gated on `prefers-reduced-motion: no-preference`, with no flashing?
- Is every claimed failure tied to a WCAG 2.2 criterion and level that you checked, with no AAA or best-practice item reported as an AA failure?

## References

- [widget-patterns.md](references/widget-patterns.md): read before building or reviewing a dialog, disclosure, tabs or combobox; APG keyboard models and code.
- [focus-and-announcements.md](references/focus-and-announcements.md): read when content is removed, routes change (focus, title, scroll restoration), forms fail, or the UI needs toasts or status messages.
- [testing-and-audit.md](references/testing-and-audit.md): read when running an audit, writing automated accessibility tests, or reporting findings.

Related skills: `frontend-engineering` for React state and effects around focus, `react-motion` for animation, `test-engineering` for test structure.

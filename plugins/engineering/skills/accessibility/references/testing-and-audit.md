# Testing and auditing

Read this when writing automated accessibility tests, running an audit, or writing up findings.

## What automation can and cannot tell you

axe-core, Lighthouse and `eslint-plugin-jsx-a11y` decide things a machine can compute: missing names, invalid ARIA, contrast of plain text on flat backgrounds, duplicate ids referenced by ARIA attributes, missing table headers, `aria-hidden` on focusable elements. They cannot tell whether focus lands somewhere sensible after a delete, whether a live region actually speaks, whether alt text is accurate, whether a combobox's arrow keys work, or whether a flow makes sense by ear. A clean axe report on a page with a broken modal is common. Use automation as a floor and a regression guard, never as the verdict.

## Playwright with axe

```ts
import AxeBuilder from '@axe-core/playwright'
import { expect, test } from '@playwright/test'

const WCAG_AA_TAGS = ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa']

test('checkout has no detectable WCAG A/AA violations, including the error state', async ({ page }) => {
  await page.goto('/checkout')
  await page.getByRole('button', { name: 'Place order' }).click()
  await expect(page.getByRole('heading', { name: /problems? with your order/i })).toBeFocused()

  const results = await new AxeBuilder({ page }).withTags(WCAG_AA_TAGS).analyze()
  expect(results.violations).toEqual([])
})
```

- Scan every state, not only the first paint: open dialogs, expanded menus, validation errors, empty and loading states, dark theme. axe only sees what is in the DOM at that moment.
- `wcag22aa` covers the rules axe maps to 2.2 AA; there is no `wcag22a` tag. Add `best-practice` only if the team treats those as blocking, and report them separately from WCAG failures.
- `.exclude()` and `.disableRules()` are for a tracked, dated exception with a named owner, not for turning a red build green.
- Treat `incomplete` results (axe could not decide, typically contrast over images or gradients) as a list to check by hand.

## Keyboard and name assertions

Role-based queries double as accessibility assertions: if `getByRole('button', { name: 'Delete invoice INV-2041' })` cannot find the button, neither can a screen reader user.

```ts
test('deleting an invoice moves focus to the next row', async ({ page }) => {
  await page.goto('/invoices')
  await page.getByRole('button', { name: 'Delete invoice INV-2041' }).click()
  await expect(page.getByRole('status')).toHaveText('Invoice INV-2041 deleted')
  await expect(page.getByRole('button', { name: 'Delete invoice INV-2042' })).toBeFocused()
})

test('dialog closes on Escape and returns focus to its trigger', async ({ page }) => {
  await page.goto('/settings')
  const trigger = page.getByRole('button', { name: 'Change email' })
  await trigger.click()
  const dialog = page.getByRole('dialog', { name: 'Change email' })
  await expect(dialog).toBeVisible()
  await page.keyboard.press('Escape')
  await expect(dialog).toBeHidden()
  await expect(trigger).toBeFocused()
})
```

Useful locator assertions: `toBeFocused()`, `toHaveAccessibleName()`, `toHaveAccessibleDescription()` (for linked error text), `toHaveRole()`, `toHaveAttribute('aria-expanded', 'true')`. `toMatchAriaSnapshot()` pins the accessibility tree of a region, which catches a refactor that silently drops roles or names.

Emulate user preferences with `page.emulateMedia({ reducedMotion: 'reduce' })` and `page.emulateMedia({ forcedColors: 'active' })` to exercise those CSS branches. Emulation proves the branch runs; it does not replace a look at a real Windows contrast theme.

These tests do not prove a screen reader speaks the live region. They prove the region exists with the right role and text, which is the part code controls.

## Manual audit procedure

Run it per user flow (sign in, search, checkout, settings), not per page.

1. Keyboard only. Unplug the mouse. Tab through the whole flow. At each step: is focus visible, is the order logical, can every control be operated, does Escape close what it should, does focus land somewhere sensible after every change (delete, filter, close, route change, submit error), does anything trap focus?
2. Screen reader. VoiceOver with Safari on macOS and iOS, NVDA with Firefox or Chrome on Windows, TalkBack with Chrome on Android. Complete the main task with the screen on. Listen for: names on every control, state changes (expanded, selected, invalid), announcements for async results, and silence where there should be speech.
3. Zoom and reflow. 200% browser zoom, then a 320 CSS px wide viewport (1280px at 400%). No two-dimensional scrolling for text content, nothing clipped or overlapped.
4. Text spacing. Apply line height 1.5, paragraph spacing 2em, letter spacing 0.12em, word spacing 0.16em with a user stylesheet or bookmarklet. Nothing clipped.
5. Preferences. Reduced motion on: large motion stops. Forced colors on: focus, selection, checkbox and toggle states still visible.
6. Pointer. Targets at least 24px or spaced; every drag has a click or tap path; nothing commits on pointer down.
7. Sign-in. Paste works in every field, password managers fill, one-time codes paste into one field.
8. Automated scan of every state visited above.

## Before reporting a failure

Check each candidate against these; a finding that fails one is not a WCAG AA failure.

- Is the criterion A or AA? These are AAA: 2.2.6 Timeouts, 2.3.3 Animation from Interactions, 2.4.12 Focus Not Obscured (Enhanced), 2.4.13 Focus Appearance, 2.5.5 Target Size (Enhanced), 3.3.9 Accessible Authentication (Enhanced), 1.4.6 Contrast (Enhanced).
- Does an exemption apply? Disabled controls, decoration and logotypes have no contrast requirement. Browser-drawn `title` tooltips are outside 1.4.13. Inline links in a sentence are outside 2.5.8. Fields about someone other than the user are outside 1.3.5. Object-recognition CAPTCHAs pass 3.3.8. Default, unmodified browser focus rings are outside 1.4.11.
- Is it a best practice rather than a criterion? A skipped heading level, a page with no `h1`, no `main` landmark, and content outside landmarks (axe `heading-order`, `page-has-heading-one`, `landmark-one-main`, `region`) are tagged best practice, not WCAG failures. Report them as recommendations.
- 4.1.1 Parsing is removed in 2.2. Duplicate ids matter only when they break a name, description or relationship; report them under 1.3.1 or 4.1.2 with the concrete breakage.
- Is it reachable? A missing name on an element that is `hidden` or inert is not a finding.
- Did you confirm it by hand? An axe violation is usually real; an axe "needs review" item and a Lighthouse warning are leads until checked.

## Writing a finding

- Criterion and level: "2.4.3 Focus Order (A)".
- Location: route, component, and selector or source line.
- Who is blocked and how: "Keyboard and screen reader users lose their place after deleting a row; focus returns to the top of the page."
- Reproduction: steps and the assistive technology and browser used.
- Fix: the concrete change ("After a successful delete, focus the next row's Delete button, or the list heading if none remain").
- Severity by user impact: blocks the task, makes it much harder, or is an annoyance. A modal that traps a keyboard user or a checkout button with no name outranks any number of contrast nits.

## WCAG 2.2 criteria this skill cites

| Criterion | Level |
| --- | --- |
| 1.1.1 Non-text Content | A |
| 1.3.1 Info and Relationships | A |
| 1.3.5 Identify Input Purpose | AA |
| 1.4.1 Use of Color | A |
| 1.4.3 Contrast (Minimum) | AA |
| 1.4.4 Resize Text | AA |
| 1.4.10 Reflow | AA |
| 1.4.11 Non-text Contrast | AA |
| 1.4.12 Text Spacing | AA |
| 1.4.13 Content on Hover or Focus | AA |
| 2.1.1 Keyboard | A |
| 2.1.2 No Keyboard Trap | A |
| 2.1.4 Character Key Shortcuts | A |
| 2.2.1 Timing Adjustable | A |
| 2.2.2 Pause, Stop, Hide | A |
| 2.3.1 Three Flashes or Below Threshold | A |
| 2.4.1 Bypass Blocks | A |
| 2.4.2 Page Titled | A |
| 2.4.3 Focus Order | A |
| 2.4.7 Focus Visible | AA |
| 2.4.11 Focus Not Obscured (Minimum) | AA |
| 2.5.2 Pointer Cancellation | A |
| 2.5.3 Label in Name | A |
| 2.5.7 Dragging Movements | AA |
| 2.5.8 Target Size (Minimum) | AA |
| 3.1.1 Language of Page | A |
| 3.2.2 On Input | A |
| 3.3.1 Error Identification | A |
| 3.3.2 Labels or Instructions | A |
| 3.3.3 Error Suggestion | AA |
| 3.3.7 Redundant Entry | A |
| 3.3.8 Accessible Authentication (Minimum) | AA |
| 4.1.2 Name, Role, Value | A |
| 4.1.3 Status Messages | AA |

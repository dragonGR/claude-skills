# Focus management and announcements

Read this when content is removed or replaced, a React Router route changes (focus, title, scroll restoration), a form submission fails, or the UI needs toasts or status messages.

Two questions for every state change: where is keyboard focus now, and what did a screen reader user hear? If the answers are "on `<body>`" and "nothing", it is a bug.

## Focus after deleting an item

The delete button that had focus is gone after the row is removed. Decide the next target before the removal and move focus after React commits the new list.

```tsx
import { useEffect, useId, useRef, useState } from 'react'
import { useAnnounce } from './announcer'

type Invoice = { id: string; number: string }
type DeleteResult = { ok: true } | { ok: false; message: string }
type InvoiceListProps = {
  invoices: Invoice[]
  deleteInvoice: (id: string) => Promise<DeleteResult>
}
type FocusTarget = { kind: 'row'; id: string } | { kind: 'heading' }

export function InvoiceList({ invoices, deleteInvoice }: InvoiceListProps) {
  const headingId = useId()
  const headingRef = useRef<HTMLHeadingElement>(null)
  const deleteButtons = useRef(new Map<string, HTMLButtonElement>())
  const [focusTarget, setFocusTarget] = useState<FocusTarget | null>(null)
  const announce = useAnnounce()

  async function handleDelete(invoice: Invoice, index: number) {
    const neighbour = invoices[index + 1] ?? invoices[index - 1]
    const result = await deleteInvoice(invoice.id)
    if (!result.ok) {
      announce(`Could not delete invoice ${invoice.number}. ${result.message}`, 'assertive')
      return
    }
    announce(`Invoice ${invoice.number} deleted`)
    setFocusTarget(neighbour ? { kind: 'row', id: neighbour.id } : { kind: 'heading' })
  }

  useEffect(() => {
    if (!focusTarget) return
    const element =
      focusTarget.kind === 'heading' ? headingRef.current : deleteButtons.current.get(focusTarget.id)
    element?.focus()
    setFocusTarget(null)
  }, [focusTarget, invoices])

  return (
    <section aria-labelledby={headingId}>
      <h2 id={headingId} ref={headingRef} tabIndex={-1}>
        Invoices
      </h2>
      <ul>
        {invoices.map((invoice, index) => (
          <li key={invoice.id}>
            {invoice.number}
            <button
              type="button"
              ref={(node) => {
                if (node) deleteButtons.current.set(invoice.id, node)
                else deleteButtons.current.delete(invoice.id)
              }}
              onClick={() => handleDelete(invoice, index)}
            >
              Delete invoice {invoice.number}
            </button>
          </li>
        ))}
      </ul>
    </section>
  )
}
```

Points that matter: focus moves only after the server confirms, a failed delete leaves focus where it was and says why, and focusing the neighbor's delete button keeps the user in the same column of work. The same rule applies to "Remove" in a cart, closing a tag chip, and dismissing a notification.

Replacing a whole list (new filter, next page) is similar: if the focused control was inside the list, focus the list heading or the results count, and announce the new count through the status region.

## Focus, title and scroll on client-side navigation

A full page load resets focus and scroll, and the screen reader announces the new document. A React Router navigation does none of that. React Router (7 and 8) has no route announcer and moves no focus, and it only manages scroll when `<ScrollRestoration>` is rendered, which exists in data mode (`createBrowserRouter`) and not in declarative mode (`<BrowserRouter>`). On every pathname change the app owns three things: where focus goes, what the title says, and where the page is scrolled. The examples import from `react-router`; v8 removed the `react-router-dom` package.

### Focus

Move focus to the new page's heading when the page mounts after a client navigation:

```tsx
import { useEffect, useRef, type ReactNode } from 'react'

let initialPageRendered = false

export function PageHeading({ children }: { children: ReactNode }) {
  const ref = useRef<HTMLHeadingElement>(null)

  useEffect(() => {
    // The first page of a full load keeps the browser's own focus handling
    if (!initialPageRendered) {
      initialPageRendered = true
      return
    }
    // Scroll belongs to ScrollRestoration or the reset below; focus must not move it
    ref.current?.focus({ preventScroll: true })
  }, [])

  return (
    <h1 ref={ref} tabIndex={-1} className="page-heading">
      {children}
    </h1>
  )
}
```

Focusing the heading makes the screen reader read it, which is the announcement. Do not also send the title to a live region; the user hears the page name twice.

The effect runs on mount, so the timing is right in every setup: after loaders resolve in data mode, after a lazy route's chunk arrives, after an exit animation under `AnimatePresence mode="wait"` (an effect on `location` in the layout would fire before the new page exists), and in a `viewTransition` navigation, where React Router waits for the new page's commit before the transition animates. Three conditions make it work:

- The page renders `PageHeading` in its loading and error states as well. A page that shows a spinner while TanStack Query is pending and renders its `<h1>` only with data has nothing to focus at mount, and focus stays on the removed link, which means `<body>`.
- Changes that only touch search params (filters, sort, pagination) do not remount the page, so focus stays on the control the user just used. That is the behavior you want.
- React Router keeps the same component mounted when only a path param changes (`/invoices/1` to `/invoices/2`), so the effect does not run again. Key the page element on the pathname (`<InvoicePage key={pathname} />`, or a keyed wrapper around `<Outlet />`) when a param change is a new page to the user.

React Strict Mode runs effects twice in development, so the heading may take focus on the first load in dev only. A heading focused this way is not interactive, so hiding its outline with `.page-heading:focus { outline: none }` is acceptable. Do not do this for anything a user can operate.

### Title

Give every route a unique, specific title ("Invoice INV-2041, Billing, Acme"); it is what the history list, tabs and bookmarks show (2.4.2 A). With React 19, render `<title>` from the page component and React places it in `<head>`:

```tsx
<title>{`Invoice ${invoiceNumber}, Billing, ${APP_NAME}`}</title>
```

The children must be a single string. `<title>Invoice {invoiceNumber}</title>` passes an array; a client render leaves the title empty with no error, so the bug is silent. Only one `<title>` may render at a time; if a layout and a page both render one, or an animated route exit keeps the old page mounted, both end up in `<head>` and React documents the result as undefined. Use one mechanism per app: React's `<title>`, or `react-helmet-async` where a site already uses it for other head tags, not both. Render a title in the page's loading state (from the route param) so the previous page's title never sits on the new page while data loads.

### Scroll

Data mode: render `<ScrollRestoration />` once, in the root route's component. On a new navigation it scrolls to the top; on Back and Forward it restores the offset saved for that history entry (keyed by `location.key` in `sessionStorage` by default; `getKey` changes that). `<Link preventScrollReset>` keeps the offset for in-page changes such as `?tab=billing`. It never touches focus, which is why the heading above uses `preventScroll: true`: without it, focusing a heading near the top would undo the offset ScrollRestoration just restored on Back.

```tsx
import { Outlet, ScrollRestoration } from 'react-router'

export function RootRoute() {
  return (
    <>
      <SkipLink />
      <SiteHeader />
      <main id="main" tabIndex={-1}>
        <Outlet />
      </main>
      <ScrollRestoration />
    </>
  )
}
```

Declarative mode has no `<ScrollRestoration>`, and a client-side push does not scroll, so the new page opens at the old page's offset with focus on a heading above the viewport. Reset scroll on pushes and replaces yourself, and leave Back and Forward to the browser:

```tsx
import { useLayoutEffect } from 'react'
import { useLocation, useNavigationType } from 'react-router'

function hashTarget(hash: string): HTMLElement | null {
  if (!hash) return null
  try {
    return document.getElementById(decodeURIComponent(hash.slice(1)))
  } catch (error: unknown) {
    // The fragment comes from the URL; a malformed escape must not crash the app
    if (error instanceof URIError) return null
    throw error
  }
}

export function ScrollReset() {
  const { pathname, hash } = useLocation()
  const navigationType = useNavigationType()

  useLayoutEffect(() => {
    if (navigationType === 'POP') return
    const target = hashTarget(hash)
    if (target) target.scrollIntoView()
    else window.scrollTo(0, 0)
  }, [pathname, hash, navigationType])

  return null
}
```

Render it once inside the router, above the routes. The browser's own restoration on Back can land short when the page renders its content after the history change (data still loading); if Back position matters, check it on a slow connection, and consider data mode with `<ScrollRestoration>`.

## A single announcer

Screen readers announce changes to live regions that already exist. Mount one polite and one assertive region at the app root, and send messages through them instead of rendering ad hoc regions in components.

```tsx
import { createContext, useCallback, useContext, useEffect, useRef, useState, type ReactNode } from 'react'

type Politeness = 'polite' | 'assertive'
type Announce = (message: string, politeness?: Politeness) => void

const AnnounceContext = createContext<Announce | null>(null)

export function AnnouncerProvider({ children }: { children: ReactNode }) {
  const [polite, setPolite] = useState('')
  const [assertive, setAssertive] = useState('')
  // One timer per region, so an assertive message cannot cancel a pending polite one
  const timers = useRef<Record<Politeness, number | undefined>>({ polite: undefined, assertive: undefined })

  const announce = useCallback<Announce>((message, politeness = 'polite') => {
    const setMessage = politeness === 'assertive' ? setAssertive : setPolite
    setMessage('')
    window.clearTimeout(timers.current[politeness])
    // Set the text in a later task so a repeated identical message is still a change
    timers.current[politeness] = window.setTimeout(() => setMessage(message))
  }, [])

  useEffect(() => {
    const pending = timers.current
    return () => {
      window.clearTimeout(pending.polite)
      window.clearTimeout(pending.assertive)
    }
  }, [])

  return (
    <AnnounceContext.Provider value={announce}>
      {children}
      <div role="status" aria-live="polite" className="visually-hidden">
        {polite}
      </div>
      <div role="alert" className="visually-hidden">
        {assertive}
      </div>
    </AnnounceContext.Provider>
  )
}

export function useAnnounce(): Announce {
  const announce = useContext(AnnounceContext)
  if (!announce) throw new Error('useAnnounce must be used inside AnnouncerProvider')
  return announce
}
```

```css
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
  border: 0;
}
```

`display: none`, `visibility: hidden` and the `hidden` attribute remove content from the accessibility tree; use the class above when text must be heard and not seen.

While a modal `<dialog>` is open, everything outside it is inert, including the root announcer. A dialog that needs to announce (a save inside the dialog) renders its own status region inside the dialog.

Use polite for confirmations, result counts and progress. Use assertive only for failures that stop the task or need action now; it interrupts whatever the screen reader is saying.

## Toasts

- One toaster mounted at the app root for the whole session. Toast text goes into a region that already exists; the toast's visual card can mount and unmount freely as long as the announced text goes through the persistent region.
- Exactly one live region per message. A `role="alert"` card inside an `aria-live` container, or a toast library region plus your own announcer call for the same message, reads it twice.
- Toasts never receive focus on appear. They interrupt nothing for keyboard users.
- A toast with an action (Undo, Retry, View) stays until dismissed, or its action is also available somewhere the user can reach at their own pace. If an auto-dismissing toast is the only place a piece of information appears, its timeout is a time limit under 2.2.1 A.
- Pause any auto-dismiss timer while the toast is hovered or has focus inside it, and give it a close button with a name.
- In Strict Mode development, effects that call `toast()` run twice. A doubled announcement in dev may be that, not a production bug; confirm in a production build before reporting it.

## Form errors

Every field: visible label, hint and error connected by id, `aria-invalid` only when invalid.

```tsx
import { useId } from 'react'
import { ErrorIcon } from './icons'

type TextFieldProps = {
  label: string
  name: string
  type?: 'text' | 'email' | 'tel' | 'password'
  autoComplete: string
  hint?: string
  error?: string
}

export function TextField({ label, name, type = 'text', autoComplete, hint, error }: TextFieldProps) {
  const id = useId()
  const hintId = `${id}-hint`
  const errorId = `${id}-error`
  const describedBy = [hint ? hintId : null, error ? errorId : null].filter((value) => value !== null).join(' ')

  return (
    <div className="field">
      <label htmlFor={id}>{label}</label>
      {hint ? <p id={hintId}>{hint}</p> : null}
      <input
        id={id}
        name={name}
        type={type}
        autoComplete={autoComplete}
        aria-invalid={error ? true : undefined}
        aria-describedby={describedBy || undefined}
      />
      {error ? (
        <p id={errorId} className="field-error">
          <ErrorIcon aria-hidden="true" />
          {error}
        </p>
      ) : null}
    </div>
  )
}
```

The icon plus text keeps the error from being color only. Error copy says how to fix it: "Enter a date in the format 31/12/2026", not "Invalid date".

On a failed submit:

- One error: focus that field. Its label, invalid state and error are read together.
- Several errors: render a summary at the top of the form with a heading (`tabIndex={-1}`) and a list of links, one per error, and focus the heading. Each link's click handler calls `.focus()` on its input; an in-page link alone scrolls to the field but does not reliably focus it.
- Server-side errors map back to the same fields and follow the same path. A generic "Something went wrong" banner that is not focused or announced is invisible to a screen reader user who is still on the submit button.

Do not validate on every keystroke with a live region; it reads an error before the user has finished typing. Validate on blur for format problems and on submit for everything.

## Sticky headers, skip links and focus visibility

```css
:root {
  --site-header-height: 4rem;
}

.site-header {
  position: sticky;
  top: 0;
  block-size: var(--site-header-height);
}

html {
  scroll-padding-top: var(--site-header-height);
}

.skip-link:not(:focus) {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}
```

```html
<a class="skip-link" href="#main">Skip to main content</a>
<header class="site-header">…</header>
<main id="main" tabindex="-1">…</main>
```

`scroll-padding-top` keeps both keyboard-focused elements and fragment targets clear of the header, so the skip link target and every control Tab reaches stay visible (2.4.11 AA). The header height lives in one custom property; if the header height changes at a breakpoint, change the property there. Apply the same treatment with `scroll-padding-bottom` for sticky footers and cookie bars.

If the app uses a hash router, `#main` is a route. Handle the skip link with a click handler that focuses `main` and prevents the default navigation.

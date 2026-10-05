# Widget patterns: dialog, disclosure, tabs, combobox

Read this before building or reviewing one of these widgets. Keyboard models follow the WAI-ARIA Authoring Practices Guide (APG); the code is React with TypeScript, and the same structure applies in any framework.

A widget is done when a keyboard user can operate it with the keys below and a screen reader user hears its role, name, state and the currently active item. Test both; a correct-looking attribute list proves nothing.

## Modal dialog

Keyboard (APG):

| Key | Behavior |
| --- | --- |
| Tab / Shift+Tab | Move through focusable elements inside the dialog, wrapping at the ends |
| Escape | Close the dialog |

Roles and properties: `role="dialog"` (implicit on `<dialog>`), `aria-modal="true"` for custom implementations, a name via `aria-labelledby` pointing at the visible title (or `aria-label`). `aria-describedby` only for short plain text; APG advises against it when the content is complex.

Initial focus (APG): the first focusable element by default; the least destructive button in a destructive confirmation; a static element such as the title with `tabIndex={-1}` when the content is long or starts with text the user must read. On close, focus returns to the element that opened the dialog, unless it no longer exists or the workflow points elsewhere (for example, the first cell of a row the dialog just added).

Use native `<dialog>` with `showModal()`. The browser puts it in the top layer, makes the rest of the document inert, fires `cancel` and closes on Escape, honors the `autofocus` attribute, and `close()` returns focus to the previously focused element. React's `autoFocus` prop is not that attribute: React omits it from the DOM and calls `focus()` at commit, before an effect opens the dialog, so the call lands on a hidden element and does nothing. To pick the initial element in React, focus it right after `showModal()`.

The two ways React code breaks this:

```tsx
// Non-modal: page behind stays reachable, Escape does nothing
<dialog open={isOpen}>…</dialog>

// Unmounting skips the close algorithm, so focus is not returned
{isOpen && <dialog ref={ref}>…</dialog>}
```

Keep the element mounted and drive it with the methods:

```tsx
import { useEffect, useId, useRef, type ReactNode } from 'react'

type ModalProps = {
  open: boolean
  onClose: () => void
  title: string
  children: ReactNode
}

export function Modal({ open, onClose, title, children }: ModalProps) {
  const ref = useRef<HTMLDialogElement>(null)
  const titleId = useId()

  useEffect(() => {
    const dialog = ref.current
    if (!dialog) return
    if (open && !dialog.open) dialog.showModal()
    if (!open && dialog.open) dialog.close()
  }, [open])

  // Escape and form method="dialog" close the element without going through props
  return (
    <dialog ref={ref} aria-labelledby={titleId} onClose={onClose}>
      <h2 id={titleId}>{title}</h2>
      {children}
    </dialog>
  )
}
```

The parent owns `open`; the `close` event keeps it in sync when the browser closes the dialog itself. Do not render `<Modal>` conditionally or inside `AnimatePresence`; animate it in CSS with `@starting-style` and `transition-behavior: allow-discrete` (the `react-motion` skill has the pattern).

Edge cases to check:

- The trigger disappears while the dialog is open (a menu item whose menu closed, a row the dialog deleted). Focusing a detached node does nothing, so focus ends on `<body>`. After close, if the saved trigger is no longer connected (`!trigger.isConnected`), focus the menu button, the list heading or the next row instead.
- A dialog that opens another dialog: each `showModal()` stacks in the top layer and each `close()` returns focus to what was focused before it opened. Close them in order.
- Toasts and live regions outside an open modal dialog are inert while it is open. Put the status region for the dialog's own messages inside the dialog.

When `<dialog>` is not an option (a design-system constraint you cannot change), the custom version needs all of: `role="dialog"`, `aria-modal="true"`, a name, `inert` on every sibling subtree of the dialog's container while open, Tab and Shift+Tab wrapping, Escape to close, initial focus as above, and focus returned on close. Missing any one is a bug. `aria-hidden` on the background does not stop Tab; `inert` does.

## Disclosure (show/hide)

Keyboard (APG): Enter and Space toggle the content when the button has focus.

Roles and properties: the control is a button with `aria-expanded="true"` or `"false"`; `aria-controls` pointing at the content is optional. The attribute goes on the button, never on the panel.

```tsx
const [expanded, setExpanded] = useState(false)
const panelId = useId()

<button type="button" aria-expanded={expanded} aria-controls={panelId} onClick={() => setExpanded((value) => !value)}>
  Shipping details
</button>
<div id={panelId} hidden={!expanded}>
  …
</div>
```

For a plain show/hide section with no custom behavior, `<details>` and `<summary>` give you this with no script.

Site navigation with dropdowns is a set of disclosure buttons. APG's navigation example avoids `role="menu"` because site navigation does not provide the functionality assistive technologies expect from a menu, and does not need its keyboard model. A `role="menu"` nav promises the menu keyboard model (arrow keys between items, focus moved into the menu on open), and a nav built from links rarely implements it.

## Tabs

Keyboard (APG, horizontal tabs):

| Key | Behavior |
| --- | --- |
| Tab | Into the tablist: focus lands on the active tab. From the tablist: to the next focusable element, usually the panel |
| Right Arrow / Left Arrow | Next / previous tab, wrapping at the ends |
| Home / End | First / last tab (optional in APG, cheap to add) |
| Enter / Space | Activate the focused tab when activation is manual |

For `aria-orientation="vertical"` tablists, Down and Up replace Right and Left.

Roles and properties: `role="tablist"` with a name, `role="tab"` on each tab with `aria-selected` and `aria-controls`, `role="tabpanel"` on each panel with `aria-labelledby` pointing at its tab. Only the selected tab has `tabIndex={0}`; the rest have `-1` (roving tabindex), so the tablist is one Tab stop. Give the panel `tabIndex={0}` when it does not start with a focusable element, so Tab reaches its content. The example below gives every panel `tabIndex={0}`; drop it for panels that start with a focusable element.

APG recommends activating a tab when it receives focus, as long as its panel shows without noticeable latency. If a panel fetches data on selection, use manual activation: arrows move focus, Enter or Space selects.

```tsx
import { useId, useRef, useState, type KeyboardEvent, type ReactNode } from 'react'

type TabItem = { key: string; label: string; content: ReactNode }

export function Tabs({ label, items }: { label: string; items: TabItem[] }) {
  const [selected, setSelected] = useState(0)
  const tabRefs = useRef<Array<HTMLButtonElement | null>>([])
  const baseId = useId()
  const tabId = (index: number) => `${baseId}-tab-${index}`
  const panelId = (index: number) => `${baseId}-panel-${index}`

  function moveTo(index: number) {
    setSelected(index)
    tabRefs.current[index]?.focus()
  }

  function onKeyDown(event: KeyboardEvent<HTMLDivElement>) {
    const last = items.length - 1
    switch (event.key) {
      case 'ArrowRight':
        moveTo(selected === last ? 0 : selected + 1)
        break
      case 'ArrowLeft':
        moveTo(selected === 0 ? last : selected - 1)
        break
      case 'Home':
        moveTo(0)
        break
      case 'End':
        moveTo(last)
        break
      default:
        return
    }
    event.preventDefault()
  }

  return (
    <>
      <div role="tablist" aria-label={label} onKeyDown={onKeyDown}>
        {items.map((item, index) => (
          <button
            key={item.key}
            ref={(node) => {
              tabRefs.current[index] = node
            }}
            type="button"
            role="tab"
            id={tabId(index)}
            aria-selected={index === selected}
            aria-controls={panelId(index)}
            tabIndex={index === selected ? 0 : -1}
            onClick={() => setSelected(index)}
          >
            {item.label}
          </button>
        ))}
      </div>
      {items.map((item, index) => (
        <div
          key={item.key}
          role="tabpanel"
          id={panelId(index)}
          aria-labelledby={tabId(index)}
          tabIndex={0}
          hidden={index !== selected}
        >
          {item.content}
        </div>
      ))}
    </>
  )
}
```

All panels stay in the DOM with `hidden` so every `aria-controls` points at an element that exists. If panels are expensive, render the content lazily but keep the empty panel element.

Common breakage: every tab is a Tab stop (no roving tabindex), `aria-selected` stays on the first tab, tabs are links that navigate (then it is navigation, not tabs; drop the tab roles), or the tablist has no name.

## Combobox with listbox popup

Use native `<select>` unless you need typing to filter or rich option content. A custom combobox is the widget teams most often ship broken.

Keyboard (APG) with focus in the input:

| Key | Behavior |
| --- | --- |
| Down Arrow | Opens the popup if closed and moves visual focus into it (first option) |
| Up Arrow | Optional: opens and moves to the last option |
| Alt+Down Arrow | Optional: opens the popup without moving visual focus |
| Escape | Closes the popup if open |
| Enter | Accepts the active option and closes the popup |
| Printable characters | Type into the input (editable combobox) |

With visual focus in the listbox (DOM focus is still in the input):

| Key | Behavior |
| --- | --- |
| Down / Up Arrow | Next / previous option |
| Enter | Accept the option and close |
| Escape | Close and return visual focus to the input |
| Left / Right Arrow | Return visual focus to the input and move the text cursor (editable combobox) |
| Home / End | Optional: move to the first or last option, or return to the input and move the cursor |
| Printable characters, Backspace | Edit the input text |

Roles and properties: `role="combobox"` on the input, `aria-expanded` reflecting whether the popup is shown, `aria-controls` pointing at the popup, `aria-autocomplete` set to `none`, `list` or `both`, and `aria-activedescendant` set to the id of the active option while one is active. The popup has `role="listbox"`; items have `role="option"` and `aria-selected="true"` on the active one.

DOM focus stays on the input the whole time. `aria-activedescendant` tells the screen reader which option to announce. Moving real focus into the list is the most common bug: typing stops working and the combobox role is lost.

```tsx
import { useId, useState, type KeyboardEvent } from 'react'

type Option = { id: string; label: string }

type ComboboxProps = {
  label: string
  options: Option[]
  onSelect: (option: Option) => void
}

export function Combobox({ label, options, onSelect }: ComboboxProps) {
  const baseId = useId()
  const inputId = `${baseId}-input`
  const listId = `${baseId}-list`
  const optionId = (index: number) => `${baseId}-option-${index}`

  const [query, setQuery] = useState('')
  const [open, setOpen] = useState(false)
  const [active, setActive] = useState(-1)

  const needle = query.trim().toLowerCase()
  const matches = options.filter((option) => option.label.toLowerCase().includes(needle))
  const expanded = open && matches.length > 0

  function commit(option: Option) {
    setQuery(option.label)
    setOpen(false)
    setActive(-1)
    onSelect(option)
  }

  function onKeyDown(event: KeyboardEvent<HTMLInputElement>) {
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault()
        setOpen(true)
        setActive((index) => Math.min(index + 1, matches.length - 1))
        break
      case 'ArrowUp':
        event.preventDefault()
        setOpen(true)
        setActive((index) => (index <= 0 ? matches.length - 1 : index - 1))
        break
      case 'Enter': {
        const option = expanded ? matches[active] : undefined
        if (option) {
          event.preventDefault()
          commit(option)
        }
        break
      }
      case 'Escape':
        if (expanded) {
          event.preventDefault()
          setOpen(false)
          setActive(-1)
        }
        break
    }
  }

  return (
    <div className="combobox">
      <label htmlFor={inputId}>{label}</label>
      <input
        id={inputId}
        type="text"
        role="combobox"
        aria-autocomplete="list"
        aria-expanded={expanded}
        aria-controls={listId}
        aria-activedescendant={expanded && active >= 0 ? optionId(active) : undefined}
        value={query}
        onChange={(event) => {
          setQuery(event.target.value)
          setOpen(true)
          setActive(-1)
        }}
        onKeyDown={onKeyDown}
        onBlur={() => {
          setOpen(false)
          setActive(-1)
        }}
      />
      <ul id={listId} role="listbox" aria-label={label} hidden={!expanded}>
        {matches.map((option, index) => (
          <li
            key={option.id}
            id={optionId(index)}
            role="option"
            aria-selected={index === active}
            // Keeps focus in the input so onBlur does not close the list before click fires
            onMouseDown={(event) => event.preventDefault()}
            onClick={() => commit(option)}
          >
            {option.label}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

The listbox stays mounted (hidden when closed) so `aria-controls` always resolves. Style the active option from `aria-selected` so the visual highlight and the announced option cannot drift apart.

Failure modes to check in review:

- `aria-activedescendant` points at an id that is not in the DOM. Virtualized lists unmount off-screen options; scroll the active option into the rendered window before setting the id, or do not virtualize short lists.
- `aria-expanded` is `true` while the list is empty or hidden.
- Option ids are reused across two comboboxes on the page (hardcoded prefixes instead of `useId`).
- The number of results is never announced. A polite status region outside the combobox ("7 results") helps; debounce it so it does not speak on every keystroke.
- Selection happens on hover or on `mousedown`.
- The popup is portaled to `<body>` and the combobox sits inside an open modal `<dialog>`: the portal is outside the dialog, so it is inert and unclickable. Render the popup inside the dialog; if it must escape `overflow` clipping, make it a `popover` element that stays a DOM descendant of the dialog, which puts it in the top layer without moving it out.
- Async options: while loading, keep `aria-expanded="false"` or show a "Loading" option state; do not leave the active id pointing at an option from the previous result set.

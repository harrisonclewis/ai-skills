# Design system

A restrained, neutral look (one grey scale, e.g. zinc), light and dark modes following the system setting, one font, one icon set. Behaviour comes from one headless component library; looks come from utility classes. Tailwind is used as the illustration below; any utility-class system with default tokens follows the same rules.

## Styling rules

- **Utility classes in markup, with the default tokens:** default palette, spacing scale, mobile-first breakpoints (`sm:`, `lg:`), `dark:` variants. **NEVER per-component CSS files, CSS-in-JS, or custom colour tokens.** The only global CSS sets the font and element defaults. Why: one vocabulary everyone already knows, no specificity fights, no dead CSS, and a reviewer can read the look from the markup.
- **Complete class strings.** The build scans source for class names; it can't see interpolations.

  ```
  // WRONG: bg-${color}-500 never reaches the stylesheet
  // RIGHT:
  const colors = { zinc: "bg-zinc-600 text-white", red: "bg-red-600 text-white" }
  class={colors[color]}
  ```

- **No inline `style` for what a class can do.** Popups size and place themselves with the headless library's CSS variables (`--anchor-width`, `--available-height`, `--transform-origin`).
- **`className` is for placement and spacing only** (`mt-4`, `w-full`, `col-span-2`). Components merge it with their own classes (`cx(className, own)`). Never pass a utility that competes with the component's own look (`<Button className="bg-red-500">`): add a variant instead.

## State via data attributes

Interactive state is exposed as data attributes and styled with attribute variants, not by computing class names in script:

| Attribute | Means |
| --- | --- |
| `data-hover`, `data-active`, `data-focus` | pointer over, pressed, keyboard focus (`data-focus` = focus-visible) |
| `data-disabled`, `data-invalid`, `data-checked`, `data-selected` | control state |
| `data-highlighted` | the menu item/option under the pointer or arrow keys |
| `data-open` / `data-popup-open` | an overlay or its trigger is open |
| `data-starting-style`, `data-ending-style` | an overlay entering / leaving (kept mounted until the transition ends) |

```
class="data-hover:bg-zinc-950/5 data-focus:outline-2 data-disabled:opacity-50"
```

Why: the headless library already knows the state; attributes keep that knowledge in one place, work in dev tools, and make tests able to assert state. Plain links and buttons get `data-hover/active/focus` from a small shared helper so they match library parts.

## Components

- **One headless library for behaviour** (focus trapping, keyboard, overlays, positioning, ARIA): e.g. Base UI or React Aria (React), Reka UI (Vue), Bits UI or Melt (Svelte), Kobalte (Solid). **NEVER a second component library beside it.** A new interactive control starts from the library part that has its behaviour.
- **Pages use our wrappers**, never the headless library directly (enforce with a lint import rule): the wrapper is where styling and our conventions live.
- **Reuse before adding:** extend an existing primitive rather than writing new markup. Where page-only vs shared components live is `project-structure`.
- **Alternatives are a `variant` prop** (`<Notice variant="error|success|info|warning">`). Never `tone`, never a separate component per look (`ErrorBox`, `SuccessNotice`, `WarningBanner`) for the same thing. (A prop that is the *value* being shown, such as a monitoring `level`, keeps its own name.)
- **A binding wrapper is not a variant.** `ErrorNotice` takes an action's `error` and renders `<Notice variant="error">` from it: nothing when there's no message, the request ID as its reference, and re-keyed on each failure so a repeated identical error is announced again. Why: the same three-line conditional (check message, set key, pick variant) was repeated on every form, and each copy forgot the re-key. It adds no look of its own; it only binds data to the one `Notice`.
- Component files are kebab-case and export a family (`dialog.<ext>`: `Dialog`, `DialogTitle`, `DialogDescription`, `DialogBody`, `DialogActions`). Keep props stable once used.
- **Icons** come from one set at fixed sizes (16/20 px solid), marked `data-slot="icon"` so the component around them sizes and colours them.

Expected primitives: Button (`variant`, `href`, `loading`, `disabled`, `disabledReason`), Link, Badge (`color`, `onRemove`), Notice (`variant`, `onRetry`, `onDismiss`), ErrorNotice (binds an action's `error` to `Notice variant="error"`), Reference (request ID), EmptyState, ListSkeleton, Toast, Field/Label/Description/ErrorMessage, Input (`blurred`, read-only lock), InputGroup with at most two trailing actions, Checkbox, RadioGroup (2 to 4 visible options; longer → Listbox), Listbox, Combobox, Dialog, ConfirmationDialog, Dropdown (⋯ menu), Tooltip, Table, PaginatedTable, Pagination, SearchField, SettingsSection (`variant="danger"`), Heading, Text, CopyButton, Reveal, Disclosure (instead of raw `<details>`), Stat, charts.

## Never colour alone

Every colour that carries meaning comes with a word or an icon:

- Status badges are words ("Active", "Expired", "Not verified"), colour is a second cue.
- Levels are icon + word + colour ("Warning" in amber with a warning icon).
- Invalid fields have an error message, not just a red border.
- Chart series have a legend, gaps between segments and a table (`accessibility.md`).

Why: ~8% of men can't separate red from green, and high-contrast/forced-colours modes strip colour.

Colour conventions: green = active/in use; red = destructive or critical; amber = warning; blue = informational and the selected option; a distinct colour (purple) for things apart from the app (an "Admin" badge). Red, amber and green as **levels** never double as chart series colours.

## Z-index scale

Overlays portal into `<body>`, so stacking is by z-index alone. Use exactly:

| Layer | z-index |
| --- | --- |
| page content, sticky headers | < 50 |
| dialogs and their backdrops | 50 |
| menus, listboxes, combobox popups | 60 |
| tooltips | 70 |
| toasts | 80 |

So a listbox opened inside a dialog sits above it, a tooltip above both, a toast above everything. **NEVER `z-[9999]`.** A dialog over a dialog is the library's nested dialog: each layer renders its own backdrop (the dim compounds), the parent recedes slightly, and Escape closes only the top one.

## Overlays

- Overlays animate in and out in CSS on `data-starting-style` / `data-ending-style` (or `@starting-style` for one mounted already open). Short fades; no JavaScript animation needed.
- **Open overlays lock page scroll** without shifting layout (the library's lock hides the scrollbar and compensates its width).
- **Focus returns to the opener** when an overlay closes (see `forms-and-actions.md#dialogs`).
- While a modal overlay is open, the rest of the page is hidden from assistive tech (`inert`/`aria-hidden`), which the library does.
- A listbox opens over its button with the selected option aligned to it (off for touch), at least as wide as the button.

## Motion

- **Subtle and functional:** a new page fades in and lifts 6 px over 200 ms; tab content slides 12 px from the side of the chosen tab; a hover/focus background slides between items in a group (sidebar, tabs, menu); a current-page indicator slides between items.
- **CSS first** (`transition`, `@starting-style`), leaving no transform behind on the element (a leftover `transform` creates a containing block that breaks fixed/sticky children).
- **Respect `prefers-reduced-motion`:** `motion-reduce:transition-none` on plain transitions; the animation library's "reduced motion = user" setting for JS animation (things still move to their place, without sliding). Browsers without `@starting-style` show the page at once.
- Nothing animates on arrival or when only the tenant changes; key page transitions by pathname, not by search params.
- Judge timing on a production build, not the dev server (`dev-double-render.md`).
- A sliding element measured in page coordinates inside a `position: fixed` pane needs that pane marked as its own scroll root, or it slides in from a scroll's distance away.

## Toasts

- Stack at the bottom right (across the bottom on phones), newest in front, up to 5, fanning out on hover/focus.
- Hide after 5 s, **paused while hovered or focused**; `duration: Infinity` keeps one until dismissed. Each has a dismiss button. No swipe-to-dismiss.
- Same icon and role as the matching notice variant: success → `role="status"`, error → `role="alert"`. (If the library makes each toast a `dialog`, override the container to `presentation`; a toast is not a dialog.)
- The toaster is mounted by the signed-in layout only; signing out unmounts it and its toasts. Calling `toast()` with no toaster mounted shows nothing and **never throws**, so a shared action used on a toaster-less page can't crash it (those pages show inline notices instead, `mutations.md`). What goes in a toast: `mutations.md#every-success-is-a-toast`.

## Persisted UI preferences

A per-device preference (sidebar hidden, a table's density) is stored in `localStorage` under a namespaced key (`app:sidebar=hidden`), **only when it differs from the default**.

- **Read it during render** (the initial state's initialiser), so the first paint is already right. **NEVER read it in an effect after mount** and flip the UI: that paints the default, then flashes and plays the transition.
- This is only correct where the page renders in the browser. For server-rendered or prerendered pages, use a cookie the server reads, or the server's HTML and the client's first render disagree (hydration mismatch).
- Wrap storage access in `try/catch`: blocked storage and private windows throw. Without storage, use the default and keep the control working.
- Never store user data, tokens or anything sensitive there.

## Where you are

- A page in the sidebar is its current item (`aria-current="page"`, a sliding indicator).
- A page reached through a menu marks the menu's button as selected (grey background), and inside the menu that item is `aria-current="page"` and focused when the menu opens.
- The current tenant keeps a check mark in the switcher.

## Mobile and desktop checks

Look at every UI change at **phone width (~375 px)** and **desktop width (≥ 1280 px)**, light and dark, before calling it done:

- [ ] Actions bottom right; dialog actions stacked full width on phones with the primary on top.
- [ ] Page-header actions wrap below the title on phones; danger zone buttons drop below their text.
- [ ] Tables scroll horizontally **inside** their container; the page never scrolls sideways. Visually hidden tables sit in a visually hidden box (a table is never narrower than its text and would widen the page).
- [ ] Page numbers hidden on phones; Previous/Next remain.
- [ ] Touch targets ≥ 44 × 44 px on touch screens (an invisible touch-target span extends small buttons).
- [ ] The mobile navigation is a slide-over that closes on navigation; the tenant switcher is reachable in one tap.
- [ ] Nothing depends on hover (tooltips are shortcuts; touch has no hover).
- [ ] Toasts span the bottom on phones and don't cover the primary button for long.
- [ ] Long names truncate with an ellipsis and a full title, never overflow.

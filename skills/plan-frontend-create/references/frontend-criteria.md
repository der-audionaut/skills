# Frontend criteria catalogue

The five parts below are a sieve, not a mandatory exercise: a component gets the states, breakpoints and accessibility measures its function needs, not every one listed here. What the description and the codebase do not demand stays out — and what you cannot decide from them is an open question, not an invention. Where the project already has a convention, component, library or tool, the plan uses it and introduces nothing new; the parts below decide what the project leaves open. Anchors and the place of each item in the plan come from `frontend-template.md`; this file only decides what goes in.

---

## Part A — Component split and props design

A component earns its own `K` when it carries its own responsibility, and what `### Vorhandene Bausteine` found is reused before anything is built anew. Extending an existing component beats a near-copy of it.

**Levels**
- **UI building block** — no domain knowledge, no data fetching; buttons, fields, dialogs, tables. Usually exists already.
- **Feature component** — knows the domain, composes building blocks, receives data or fetches it through the project's data layer.
- **Page or route** — wires feature components to the route, URL state and layout.

Dependencies point from page to feature to building block, never back.

**Props rules**
- Naming, casing and event naming follow the project — `onChange` or `@change` or `(change)`, whatever the existing components do.
- Required props have no default; optional props have a default stated in the props table.
- Composition over configuration: content goes in through children or slots, not through a growing set of boolean switches. Two booleans that exclude each other are one enum prop.
- Controlled or uncontrolled is decided per value and stated; a component that is both carries the uncontrolled default and the controlled override explicitly.
- Native attributes the component renders — `id`, `aria-*`, `class`, `disabled`, input attributes — are passed through to the native element in the project's usual way.
- Data that a component only hands down through more than two levels belongs in context, a store or the URL — whichever the project already uses.
- Types come from the project's existing domain and API types; the props table names them, it does not redefine them.

**Not planned**
- An abstraction for a single use — a generic component with one caller is a feature component with a longer name.

---

## Part B — States and edge cases

Every new or extended component plans the states it can be in and how it shows each — a happy path without them is not production-ready.

**States**
- **Loading** — skeleton where the layout is known, spinner where it is not; shown only after a short delay so fast responses do not flicker; the layout does not jump when the data arrives.
- **Empty** — its own message with the next sensible action, distinguished from "nothing found for this filter".
- **Error** — understandable message without technical detail, a way to retry, input already entered stays.
- **Partial data** — missing optional fields have a stated representation instead of `undefined` or an empty cell.
- **Disabled and read-only** — visible and announced; the reason is available where the user cannot infer it.
- **Optimistic update** — only where the project already does it; then with rollback and message on failure.

**Edge cases**
- Long texts, long words, longer translations (German runs up to about a third longer than English) and missing translations: truncation or wrapping is decided, never overflow.
- Large lists: from the realistic volume onward — from fixtures, API limits or the description — pagination or virtualization.
- Slow and interrupted network: a request that is superseded is aborted or its result ignored.
- Double submit: the action is locked while it runs.
- Stale data: what happens when the data changes while the form is open.
- Missing permission: the element is hidden or disabled according to the project's pattern, and the server still checks.

**Not planned**
- A loading state for data that is synchronously available.
- An empty state for a list that by its domain is never empty.
- Edge cases of data the component never receives.

---

## Part C — Responsive design

Breakpoints come from the project — tokens, theme or CSS framework configuration. Inventing new ones is a finding, not a design. Layout is written mobile-first where the project does so.

**Rules**
- Content reflows at a width of 320 CSS pixels without horizontal scrolling, except for content that needs two dimensions, such as data tables or maps (WCAG 1.4.10 Reflow).
- Text can be enlarged to 200 % without loss of content or function (WCAG 1.4.4 Resize Text).
- Interactive targets are at least 24 × 24 CSS pixels or have enough spacing (WCAG 2.5.8 Target Size (Minimum)).
- Operation works by touch and by pointer: no function that depends on hover alone (WCAG 2.1.1 Keyboard), no dragging without a single-pointer alternative (WCAG 2.5.7 Dragging Movements).
- Tables on narrow widths: scroll container with visible cue, or a card layout — decided per table, not left to chance.

**Not planned**
- A breakpoint per component.
- A separate mobile component where the layout can reflow.

---

## Part D — Accessibility

The standard is the one under `### Maßstab Barrierefreiheit`. The list below names, by WCAG 2.2 number, the criteria a component plan touches most often; it is not the whole standard. Anything above the standard's level is planned as a recommendation, not as an acceptance criterion.

**Rules**
- Semantic HTML before ARIA: a `button` is a button, not a `div` with a role. ARIA only where no native element exists, and then complete — name, role, value and state (WCAG 4.1.2 Name, Role, Value; 1.3.1 Info and Relationships).
- Everything is operable by keyboard, without a keyboard trap (WCAG 2.1.1 Keyboard; 2.1.2 No Keyboard Trap).
- Focus order follows the visual and logical order (WCAG 2.4.3 Focus Order).
- Focus is visible and not hidden by sticky headers, banners or overlays (WCAG 2.4.7 Focus Visible; 2.4.11 Focus Not Obscured (Minimum)).
- Focus management: a dialog moves focus into itself, keeps it there and returns it to the trigger on close; after a route change, focus goes to the new heading or main region according to the project's pattern.
- Every form field has a visible label and, where needed, an instruction; the accessible name contains the visible label (WCAG 3.3.2 Labels or Instructions; 2.5.3 Label in Name).
- Errors are identified in text, linked to the field and, where possible, come with a suggestion (WCAG 3.3.1 Error Identification; 3.3.3 Error Suggestion).
- Asynchronous changes — loading finished, saved, error, result count — are announced through a live region or status message without moving focus (WCAG 4.1.3 Status Messages).
- Text contrast at least 4.5:1, large text 3:1; component boundaries and focus indicators at least 3:1 (WCAG 1.4.3 Contrast (Minimum); 1.4.11 Non-text Contrast).
- Information is not conveyed by color alone (WCAG 1.4.1 Use of Color).
- Images and icons carry an alternative text or are marked as decorative; icon-only buttons have an accessible name (WCAG 1.1.1 Non-text Content).
- Content that appears on hover or focus can be dismissed, can be hovered and stays until hover or focus leaves it or it is dismissed (WCAG 1.4.13 Content on Hover or Focus).
- Motion: animations that start automatically can be paused (WCAG 2.2.2 Pause, Stop, Hide); `prefers-reduced-motion` is respected where the project animates (WCAG 2.3.3 Animation from Interactions, level AAA).

**Not planned**
- Your own focus trap or live region where the project's dialog or toast component already provides one.

---

## Part E — Verification

Each step states how its result is checked, at the level the project already uses. A tool the project does not have is not introduced by the plan; then a manual check path takes its place.

**Levels**
- **Component tests** — one test per listed state and per interaction the props table promises, with the project's test library; queries by role and accessible name where the library supports it, so the test checks accessibility on the side.
- **Integration or end-to-end tests** — for the flow across components, if the project has them: loading → data → action → message.
- **Automated accessibility check** — axe or a comparable tool, if the project has one; it does not replace the manual path.
- **Manual path** — keyboard pass through the whole flow (Tab, Shift+Tab, Enter, Space, Escape, arrow keys where the pattern requires them), screen-reader sample with the reader the project names or one common pairing, the widths and zoom level from Part C, plus a tablet and the desktop width.

**Not planned**
- Snapshot tests as the only proof of a state — they show that something changed, not that it is right.

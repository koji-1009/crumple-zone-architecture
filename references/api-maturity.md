# Browser API Maturity Assessments

Individual assessments applying the maturity framework from architecture.md section 5.4. Each entry records what CRZ does with the API and why, plus the event that would change that decision.

This is a decision register, not a compatibility table. Current support data belongs to MDN and Baseline, which maintain it; duplicating it here produces a document that is wrong without anyone noticing. Three kinds of fact are kept: the month by which every engine had shipped an API, which is what a support floor is compared against; engine counts at the resolution the decision needs; and a specific gap that is itself the basis of a decision, which stays because the entry's revisit event names the change that would retire it. Excluded are per-engine version numbers, incidental statements about what is missing today, and predictions about what vendors will do. Dates derived by arithmetic are not tabulated either — the reach rule is stated once in architecture.md section 5.4.

A recorded month says every engine had shipped by then. It does not promise the feature stayed settled: a specification can be revised afterwards and earlier implementations reclassified, which is what the Specification Stability axis is for. CSS anchor positioning is the entry where that happened.

Last reviewed: 2026-08

## Summary

| API | CRZ Relevance | Decision | Basis | Revisit |
| --- | --- | --- | --- | --- |
| Drag and Drop | High | Avoid; Isolate where the interaction is a hard requirement | Structural design flaw — the specification is built on mouse events | Rebuilt on Pointer Events |
| `<select>` styling | Very High | Isolate, as progressive enhancement | One engine has shipped | Every engine ships |
| Date/Time inputs | Very High | Use with constraints; Isolate beyond basic date entry | Picker UI cannot be styled, format is locale-bound, no timezone | Styling hooks extend to date inputs, or Temporal reaches every engine |
| `<datalist>` | High | Avoid for combobox; tolerate for trivial hints | Filtering and rendering deliberately underspecified | Spec defines filtering and styling hooks |
| `<input type="month/week">` | Moderate-High | Avoid — decompose into `<select>` | Two engines provide no picker, degrading to a bare text field | Those engines ship pickers |
| Clipboard (async) | Moderate | Direct delegation | Sound on all four axes | -- |
| File System Access | Low-Moderate | Avoid | One engine, and not on a standards track | Second engine plus standards track |
| `<dialog>` | Very High | Direct delegation | Sound on all four axes | -- |
| Invoker Commands (`command` / `commandfor`) | Very High | Default trigger; one project-level fallback below the floor | Sound; where absent, the control is inert | Reach only — project support floor |
| Popover API | High | Default for non-modal overlays; CSS guard below the floor | Sound; where absent, content renders inline | Reach only — project support floor |
| `dialog closedby` | Moderate | Enhancement only — never the sole close path | Has not shipped in every engine | Every engine ships |
| `<details name>` / `::details-content` | High | Direct delegation | Sound; where absent, panels open together and expansion is instant | Reach only — project support floor |
| `field-sizing: content` | Moderate | Progressive enhancement | Sound; where absent, the field keeps its fixed size | Reach only — project support floor |
| CSS anchor positioning | Moderate | Use the settled subset with a static fallback position; isolate anything beyond it | Parts of the feature are still converging, and the module is not Baseline as a whole | The feature reaches Baseline as a whole |
| Speculation Rules | Moderate | Optional enhancement, never load-bearing | One engine only | Second engine ships |
| Interest invokers (`interestfor`) | Moderate | Avoid | One engine, with a WebKit objection filed; hover-only affordance | Cross-vendor consensus |
| CloseWatcher | Low | Avoid — `<dialog>` and popover cover the need | Not in every engine, and no CRZ use case that they do not serve | Every engine ships |

Revisit names the event that would change the decision. "Reach only" means no such event is expected: the API is settled and shipped everywhere, and what remains is whether the project's support floor reaches versions that predate it. That comparison belongs to the project, not to this document — see architecture.md section 5.4. A project serving current browsers adopts a "Reach only" API as it stands.

---

## 1. HTML Drag and Drop API

**Failure pattern**: Structural design flaw

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Fail | Mouse-event-based specification. Most mobile browsers do not implement it. Touch event polyfills simulate rather than implement the API |
| Composability | Fail | Requires `preventDefault()` on `dragover` to enable dropping — non-obvious event model inversion. `dataTransfer` uses MIME-type interface designed for OS clipboard, not DOM interactions. Drag image customization is severely limited |
| Failure Mode Transparency | Fail | On mobile, silently does nothing. No error, no fallback — `draggable` attribute is ignored |
| Specification Stability | Pass | Stable in WHATWG Living Standard since HTML5 |

### CRZ Strategy: Avoid (preferred) or Isolate

- **Sortable lists in admin panels**: Provide explicit reorder controls (move up/down buttons, position number input). More accessible, works everywhere
- **Kanban-style boards**: If drag-and-drop is a hard requirement, isolate behind a single island component using `@dnd-kit` (Pointer Events internally). The island exposes `onCardMove(cardId, fromColumn, toColumn, position)` — no DnD API leaks beyond this boundary
- **File upload drop zones**: The one legitimate use case. `drop` event with `dataTransfer.files` works reliably on desktop. Wrap in an island that provides `<input type="file">` as the primary interaction, with drop zone as progressive enhancement

**Revisit condition**: A future spec revision rebuilding DnD on Pointer Events with mobile-first semantics.

---

## 2. `<select>` Styling (`appearance: base-select`)

**Failure pattern**: Implementation lag

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Fail | Shipped in Chrome/Chromium. Safari and Firefox have positive vendor positions but have not shipped |
| Composability | Pass (conditional) | When supported, integrates cleanly with CSS, supports rich HTML content in `<option>`, uses standard `::picker()` pseudo-elements |
| Failure Mode Transparency | Pass | Non-supporting browsers render a classic `<select>` — functional but unstyled |
| Specification Stability | Marginal | WHATWG Stage 2. Naming has changed multiple times (`<selectmenu>` -> `<selectlist>` -> `appearance: base-select`). Current API surface appears settled |

### CRZ Strategy: Isolate with progressive enhancement

- Use `appearance: base-select` today, but only for visual customization that degrades gracefully
- Do NOT use JavaScript-based select replacements (Select2, React Select, Headless UI Listbox) in new CRZ projects
- If rich content in options is a hard requirement and cross-browser consistency is mandatory today: a single `<RichSelect>` island that feature-detects support and falls back to native `<select>`

**Revisit condition**: every engine ships it.

---

## 3. Date/Time Input Types

**Failure pattern**: Implementation lag (partial)

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Marginal | All major browsers support the type, but UX differs significantly. Safari displays current date as ghost value when empty |
| Composability | Fail | Picker UI cannot be styled via CSS. Shadow DOM structure differs across browsers. Displayed format follows browser locale and cannot be overridden. No timezone support |
| Failure Mode Transparency | Pass | Falls back to text input. Invalid dates trigger `:invalid` pseudo-class |
| Specification Stability | Pass | Stable in WHATWG Living Standard |

### CRZ Strategy: Use with constraints, Isolate when UX requirements exceed native capability

- **Basic date entry** (birthdate, start date, deadline): Use native `<input type="date">` directly. UX inconsistencies are cosmetic, not behavioral
- **Advanced requirements** (date range, timezone-aware, custom calendar): Isolate behind a dedicated island. Consider Temporal API as it reaches Baseline
- **Never** build a custom date picker from scratch. Accessibility, keyboard, and locale handling is enormous. Use a library wrapped behind a project-owned interface
- For Safari empty-value display: CSS workaround or document as known cosmetic issue. Do not switch to `type="text"`

**Revisit condition**: `appearance: base-select`-style customization extends to date inputs. Temporal API reaches Baseline.

---

## 4. Clipboard API (Async)

**Failure pattern**: None — trustworthy

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Pass | `navigator.clipboard.writeText()` works across all major browsers. `read()` has stricter permissions on Firefox |
| Composability | Pass | Promise-based, integrates with async/await |
| Failure Mode Transparency | Pass | Throws clear errors on permission denial. Feature detection is straightforward |
| Specification Stability | Pass | W3C Working Draft, but in every engine with no redesign pending — the write path has been stable for years |

### CRZ Strategy: Direct delegation

- Use `navigator.clipboard.writeText()` directly for copy operations
- For paste requiring `read()`, check permissions and provide fallback (manual paste instruction)
- No crumple zone needed

---

## 5. File System Access API

**Failure pattern**: Implementation lag (severe — single vendor)

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Fail | Chrome/Chromium only. Firefox and Safari do not implement and have expressed concerns |
| Composability | Pass (where supported) | Promise-based, integrates with streams |
| Failure Mode Transparency | Pass | Feature detection is clean |
| Specification Stability | Fail | WICG proposal. Not on W3C standards track. No cross-vendor consensus |

### CRZ Strategy: Avoid

- Single-vendor APIs violate the cross-platform parity requirement
- File import: `<input type="file">`
- File export: Blob URL + download attribute
- Bulk operations: server-side via BFF

**Revisit condition**: Second engine implementation and formal standards track.

---

## 6. `<datalist>` Element

**Failure pattern**: Underspecification

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Fail | Filtering algorithm differs: Chrome uses prefix match, Firefox uses substring match. `label` attribute rendering varies. When options are dynamically filtered by API, browsers re-apply their own filtering on top, silently hiding valid options |
| Composability | Fail | Cannot be styled. Font size ignores page zoom. Cannot display rich content. Interaction with `autocomplete` attribute is inconsistent |
| Failure Mode Transparency | Pass | Falls back to plain text input — functional, just without suggestions |
| Specification Stability | Marginal | In WHATWG Living Standard, but rendering and filtering behavior deliberately underspecified |

### CRZ Strategy: Avoid for anything beyond trivial suggestions; Isolate for combobox patterns

- **Simple, short suggestion lists** (5-15 static options, no API backing, no label/value distinction): acceptable as progressive enhancement
- **API-backed autocomplete**: Do not use `<datalist>`. Browser's opaque re-filtering of already-filtered results makes behavior unpredictable. Build a combobox island using ARIA pattern or Downshift
- **Label/value distinction** (display "Tokyo, Japan" but submit "TYO"): Do not use `<datalist>`. Cross-browser rendering inconsistency
- `<datalist>` occupies a dangerous middle ground — it looks like it should work for combobox use cases, but its underspecified behavior means you cannot predict what the user sees

**Revisit condition**: WHATWG spec amended to define filtering behavior, label rendering, and styling hooks. Also if `appearance: base-select`-style customization extends to datalist.

---

## 7. `<input type="month">` (and `type="week"`)

**Failure pattern**: Implementation lag

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Fail | Firefox and Safari desktop do not provide a picker widget — fall back to plain text input |
| Composability | Fail | Where picker exists it cannot be styled. Fallback text input has no format guidance |
| Failure Mode Transparency | Marginal | Degrades to `<input type="text">` with no affordance — technically transparent, experientially opaque |
| Specification Stability | Pass | Stable in WHATWG Living Standard. Problem is implementation, not specification |

### CRZ Strategy: Avoid — decompose into trustworthy primitives

- **Option A**: Two native `<select>` elements (year + month). Universal, accessible, zero JS
- **Option B**: `<input type="date">` with guidance that only month matters. Value truncated server-side
- **Option C** (if rich UI required): Island with custom month picker exposing `onMonthSelect(year, month)`
- Same analysis applies to `<input type="week">`

**Revisit condition**: Firefox and Safari ship native month/week picker widgets.

---

## 8. `<dialog>` Element

**Failure pattern**: None — trustworthy

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Pass | In every engine since 2022-03. `showModal()` gives top-layer placement, focus trapping, inertness of the rest of the page, and Esc handling consistently across engines |
| Composability | Pass | `::backdrop`, `:modal`, `<form method="dialog">` closing with `returnValue`, `close` and `cancel` events. Entry/exit animation composes with `transition-behavior: allow-discrete` and `@starting-style` |
| Failure Mode Transparency | Pass | Where the element is unknown, its content renders inline as ordinary flow content — visible and unstyled, not silently missing |
| Specification Stability | Pass | WHATWG Living Standard |

### CRZ Strategy: Direct delegation

- The modal is an HTML-layer construct. A component holding an `isOpen` boolean to reproduce modality is that state moved from layer 1 to layer 4 for nothing
- Close paths that need no script: `<form method="dialog">` with a submit button, and `command="close"` / `command="request-close"` (section 10)
- `requestClose()` (in every engine since 2025-05) fires `cancel` before `close`, so an unsaved-changes guard is a `cancel` listener rather than a wrapper around every close path
- `closedby` (light dismiss) has not shipped in every engine. Treat it as an enhancement: with support, a click outside dismisses; without it, the dialog stays open and the explicit close control still works. Never make it the only way out
- Dialog content that needs validation, dynamic fields, or multi-step flow is still an island — the island owns the content, not the open/close mechanics

**Revisit condition**: none for the element itself. `closedby` stops being enhancement-only when every engine ships it.

---

## 9. Popover API

**Failure pattern**: None in the API. Below the support floor, absence degrades to visible inline content

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Pass | In every engine since 2025-01. Top-layer placement, light dismiss, and Esc handling are consistent; the page behind stays interactive by definition (popovers are never modal) |
| Composability | Pass | `popovertarget` / `popovertargetaction` on a plain `<button>`, `:popover-open`, `::backdrop`, `beforetoggle` / `toggle` events. Composes with anchor positioning for placement |
| Failure Mode Transparency | Marginal | Where the `popover` attribute is unknown, the element is not hidden — its content renders inline as ordinary flow content. The failure is visible, not silent, but it is a layout break rather than a graceful absence |
| Specification Stability | Pass | WHATWG Living Standard. Shipped in every engine as of 2025-01, with no redesign pending |

### CRZ Strategy: Default for non-modal overlays, with a CSS fallback

- Menus, toasts, hint panels, and disclosure overlays are popovers. A `div` toggled by a class reimplements top layer, stacking, and dismissal in the least reliable layer, and gets all three subtly wrong
- Where the support floor reaches versions older than 2025-01, guard the unsupported case in CSS rather than script: `@supports not (selector(:popover-open)) { [popover] { display: none } }` keeps the content out of the flow, leaving the trigger inert rather than the page broken
- `popover="auto"` gives light dismiss and one-at-a-time behavior; `manual` opts out of both. `hint` drives hover-triggered affordances, which have no touch equivalent — the reason to avoid it is the interaction, not its support
- Popovers are never modal. Anything that must block interaction with the page is a `<dialog>` opened with `show-modal`

**Revisit condition**: none. What remains is reach — compare 2025-01 against the project's support floor.

---

## 10. Invoker Commands (`command` / `commandfor`)

**Failure pattern**: None in the API. Below the support floor, absence degrades to an inert control

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Pass | In every engine since 2025-12. Behavior is defined by the button's activation behavior, so keyboard, touch, and assistive technology paths are identical to any `<button>` |
| Composability | Pass | Plain `<button>` plus two attributes. Built-in commands (`show-modal`, `close`, `request-close`, `show-popover`, `hide-popover`, `toggle-popover`) map one-to-one onto existing methods; `value` sets `returnValue` on close. Custom commands (`--` prefix) dispatch a `CommandEvent` on the target with `source` and `command`, using the standard event model |
| Failure Mode Transparency | Marginal | Where unsupported, the button is inert: correctly focusable, correctly labelled, and does nothing. No visual signal. Detection is one check — `'command' in HTMLButtonElement.prototype` |
| Specification Stability | Pass | WHATWG Living Standard. Shipped in every engine as of 2025-12, with no redesign pending. The command surface is an extension point — further built-in commands are under discussion, which adds to it rather than redesigning it |

### CRZ Strategy: Default trigger, with a project-level fallback script

- Use `command` / `commandfor` as the standard way to open and close dialogs and popovers. It is the only declarative way to reach `showModal()`
- The support floor decides whether a fallback is needed, not the calendar. A project serving current desktop browsers takes the API as-is. Where the floor reaches versions older than 2025-12, add one feature-detected fallback script at the layout level — a single project-wide crumple zone covering every invoker, deleted when the floor clears. Never wire fallbacks per component
- IDs are the binding, so a component rendered N times needs N unique IDs. Derive them from props (``id={`confirm-${item.id}`}``), the same constraint that governs a component's `<script>` running once for N instances
- Scope: built-in commands need no JavaScript at all. Custom `--` commands still require a listener, so they carry the same "inert until the listener attaches" gap as any click handler — the gain there is the declarative binding and the removal of trigger-side `querySelector`, not the removal of the gap
- Test consequence: a built-in command is active at parse time, so an E2E click on it needs no wait-for-hydration condition. This is a property of the HTML layer, not of the testing tool

**Revisit condition**: none. What remains is reach — compare 2025-12 against the project's support floor. Further built-in commands extend the surface without changing this assessment.

---

## 11. Declarative Disclosure: `<details name>` and `::details-content`

**Failure pattern**: None — trustworthy

### Maturity Assessment

| Axis | Rating | Detail |
| --- | --- | --- |
| Cross-Platform Parity | Pass | `<details>` in every engine for years; `name` (exclusive accordion) since 2024-09; `::details-content` since 2025-09 |
| Composability | Pass | `name` groups panels without script. `::details-content` with `transition-behavior: allow-discrete` animates the expansion; `toggle` event exposes state changes |
| Failure Mode Transparency | Pass | Without `name` support, multiple panels open at once. Without `::details-content`, expansion is instant. Both are experience degradation with the content still reachable |
| Specification Stability | Pass | WHATWG Living Standard |

### CRZ Strategy: Direct delegation

- Accordions, disclosure widgets, and FAQ lists are HTML-layer constructs. An island tracking which panel is open reproduces browser behavior in the least reliable layer
- `hidden="until-found"` is an enhancement, not a delivery mechanism. Where supported, in-page search reveals the collapsed content and fires `beforematch`; one engine has not shipped it, and there the content stays hidden and unfindable. Never let it be the only route to content the user must be able to reach
- The limit is the same as always: content that must be fetched or validated on expand is an island

**Revisit condition**: none for `name` and `::details-content`. `hidden="until-found"` stops being enhancement-only when every engine ships it.

---

## 12. Additional 2026 Assessments

| API | Failing axis | Assessment | CRZ Strategy |
| --- | --- | --- | --- |
| `field-sizing: content` | None — in every engine since 2026-06 | Auto-growing inputs and textareas without a script. Where absent, the field keeps its fixed size — cosmetic | Progressive enhancement. Pair with `min-width` / `max-width` |
| CSS anchor positioning | Specification Stability | Positions an overlay against its trigger without a measurement loop. `anchor-name`, `anchor()`, `@position-try`, `position-try-fallbacks` and `position-area` reached every engine in 2026-01, but the module is not Baseline as a whole: `position-anchor` has been through initial-value changes that reclassified earlier implementations, and `position-visibility` has no implementation | Use the settled subset, and declare a static fallback position first so browsers below the floor place the element somewhere usable. Treat the unsettled parts as an island concern. Do not adopt a JS positioning library to close the gap |
| Speculation Rules | Cross-Platform Parity (single engine) | Prefetch and prerender for MPA navigation, declared as a `<script type="speculationrules">` block. Absent support means ordinary navigation | Optional enhancement only. Exclude state-changing URLs (sign-out, cart, language switch); handle `Sec-Purpose: prefetch` server-side. Never let perceived speed depend on it |
| Interest invokers (`interestfor`) | Cross-Platform Parity | Declarative hover/focus/long-press triggers for popovers. Shipped in one engine, with a WebKit objection filed against it | Avoid. Use click-activated `command` / `popovertarget` instead — a hover-only affordance is a touch-platform problem regardless of API support |
| CloseWatcher | Cross-Platform Parity | Unifies Esc, Android back, and gesture dismissal for custom UI. Not in every engine | Avoid. `<dialog>` and popover already receive close requests; needing CloseWatcher usually means a custom overlay that should have been one of them |

**Revisit conditions** for these five are the ones in the Summary.

---

## References

- [Fifty problems with standard web APIs in 2025](https://zerotrickpony.com/articles/browser-bugs/)
- [DragDropTouch polyfill](https://github.com/drag-drop-touch-js/dragdroptouch) — documents the mouse-event basis of the DnD spec
- [Chrome Blog: A customizable select](https://developer.chrome.com/blog/a-customizable-select)
- [Open UI: Customizable Select Element Explainer](https://open-ui.org/components/customizableselect/)
- [MDN: `<input type="date">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/date)
- [WHATWG Issue #9986: `<datalist>` behavior inconsistencies](https://github.com/whatwg/html/issues/9986)
- [MDN browser-compat-data Issue #25723: datalist meta-issue](https://github.com/mdn/browser-compat-data/issues/25723)
- [MDN: `<input type="month">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/month)
- [MDN: Invoker Commands API](https://developer.mozilla.org/en-US/docs/Web/API/Invoker_Commands_API)
- [MDN: Popover API](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API)
- [MDN: `<dialog>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/dialog)
- [MDN: Exclusive accordions using the HTML details element](https://developer.mozilla.org/en-US/blog/html-details-exclusive-accordions/)
- [MDN: Speculation Rules API](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API)
- [Chrome Platform Status: Interest Invokers](https://chromestatus.com/feature/4530756656562176)

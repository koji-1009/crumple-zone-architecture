# Browser API Exceptions

Section 5.4 of architecture.md sets the adoption rule: at Baseline Newly Available, adoption is a judgment against the project's support floor; at Widely Available, reach stops being a question. Most APIs need nothing beyond that rule, and nothing about them is recorded here.

This file holds the exceptions — the APIs where the rule gives the wrong answer, and what to do instead. Drag and Drop is Widely Available and must still be avoided. `<datalist>` passes feature detection and then behaves differently per engine. An entry earns its place by naming a behavior the rule cannot predict, together with the alternative to reach for. Support data belongs to MDN and Baseline, which maintain it; a status line appears here only where the decision turns on it.

Last reviewed: 2026-08

## Summary

| API | Why the rule gives the wrong answer | Decision | Revisit |
| --- | --- | --- | --- |
| Drag and Drop | Widely Available, but the specification is built on mouse events | Avoid; Isolate where the interaction is a hard requirement | Rebuilt on Pointer Events |
| `<select>` styling | Below the floor everywhere, and every JS alternative is worse than waiting | Isolate, as progressive enhancement | Every engine ships it enabled |
| Date/Time inputs | In every engine, but the picker cannot be styled and the format is locale-bound | Use with constraints; Isolate beyond basic date entry | Styling hooks extend to date inputs, or Temporal reaches every engine |
| File System Access | Single vendor and off the standards track, so the rule never starts | Avoid — `<input type="file">`, blob download, server-side | Second engine plus standards track |
| `<datalist>` | In every engine, but filtering and rendering are deliberately underspecified | Avoid for combobox; tolerate for trivial hints | Spec defines filtering and styling hooks |
| `<input type="month/week">` | Specified everywhere, but two engines degrade it to a bare text field | Avoid — decompose into `<select>` | Those engines ship pickers |
| CSS anchor positioning | Shipped in every engine while the module is still being revised | Use the settled subset and declare `position-anchor` explicitly | The module settles |
| Speculation Rules | Below the floor, and prerender is single-engine | Optional enhancement, never load-bearing | Second engine ships |
| Interest invokers (`interestfor`) | The interaction is hover-only, so support is not what decides it | Avoid | -- |
| CloseWatcher | Below the floor, and `<dialog>` and popover already cover the need | Avoid | -- |

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
| Cross-Platform Parity | Fail | One engine ships it enabled. A second has it behind preferences, and the third has it in a preview release |
| Composability | Pass (conditional) | When supported, integrates cleanly with CSS, supports rich HTML content in `<option>`, uses standard `::picker()` pseudo-elements |
| Failure Mode Transparency | Pass | Non-supporting browsers render a classic `<select>` — functional but unstyled |
| Specification Stability | Marginal | WHATWG Stage 2. Naming has changed multiple times (`<selectmenu>` -> `<selectlist>` -> `appearance: base-select`). Current API surface appears settled |

### CRZ Strategy: Isolate with progressive enhancement

- Use `appearance: base-select` today, but only for visual customization that degrades gracefully
- Do NOT use JavaScript-based select replacements (Select2, React Select, Headless UI Listbox) in new CRZ projects
- If rich content in options is a hard requirement and cross-browser consistency is mandatory today: a single `<RichSelect>` island that feature-detects support and falls back to native `<select>`

**Revisit condition**: every engine ships it enabled.

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

**Revisit condition**: `appearance: base-select`-style customization extends to date inputs. Temporal API reaches every engine.

---

## 4. File System Access API

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

## 5. `<datalist>` Element

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

## 6. `<input type="month">` (and `type="week"`)

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

## 7. Shorter Exceptions

| API | Why the rule gives the wrong answer | CRZ Strategy |
| --- | --- | --- |
| CSS anchor positioning | Every engine shipped the core in 2026-01, so the rule reads it as settled, but the module is still being revised — `position-anchor` went through three initial values before `normal`, so each engine's earlier releases behave differently from its current one | Use the settled subset. Declare `position-anchor` explicitly rather than relying on its initial value, and declare a static fallback position so browsers below the floor place the element somewhere usable. Do not adopt a JS positioning library to close the gap |
| Speculation Rules | Prefetch and prerender for MPA navigation, declared as a `<script type="speculationrules">` block. Below the floor, and prerender is single-engine | Optional enhancement only. Exclude state-changing URLs (sign-out, cart, language switch); handle `Sec-Purpose: prefetch` server-side. Never let perceived speed depend on it |
| Interest invokers (`interestfor`) | Declarative hover, focus, and long-press triggers for popovers. Support is not the deciding factor — a hover-only affordance has no touch equivalent | Avoid. Use click-activated `command` / `popovertarget` |
| CloseWatcher | Unifies Esc, Android back, and gesture dismissal for custom UI, which the rule would eventually admit | Avoid. `<dialog>` and popover already receive close requests; needing CloseWatcher usually means a custom overlay that should have been one of them |

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
- [MDN: Speculation Rules API](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API)

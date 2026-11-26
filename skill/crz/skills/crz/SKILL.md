---
name: crz
description: Implement Astro applications with Crumple Zone Architecture — trust the browser, design for failure modes, minimize client state. Use when writing or modifying Astro code (pages, components, islands, Actions, middleware, state placement, project structure).
---

# Astro Crumple Zone Implementation

Build healthy Astro applications with Crumple Zone Architecture. Trust the browser, design for failure modes, minimize framework dependency.

Prerequisite: Astro 7+. An adapter is required once anything renders on demand (a page, an Action, a Server Island).

## Rendering Mode

Decide per page, by one test: does the output depend on the request?

* Yes — it reads cookies, the session, headers, the query string, or a POST body, is gated by auth, or shows data that must be current on every request → on-demand. Frontmatter and middleware run per request
* No → prerendered. Frontmatter and middleware run once, at build. There is no request: cookies, session, headers, and the query string are empty, and `Astro.redirect` degrades to a client redirect

Never prerender a request-dependent page, and never rebuild the request-dependence on the client (an island or script that reads the query string or fetches per-user data). A `server:defer` component is tested on its own: it always renders on demand, so a prerendered page may hold one.

A shared cache (CDN) may hold a response only when its output depends on nothing in the request but the URL: a prerendered page, or an on-demand page that reads no cookie, session, or header. Per-user parts of the page move into a Server Island (see Server Islands). Every on-demand response is `Cache-Control: private, no-store` by default (see the middleware); a page that passes this test opts into sharing with `Astro.response.headers.set("Cache-Control", ...)`.

This is the origin's half of the contract. It holds only when the CDN honors the origin's `Cache-Control` (no TTL rule that caches past `private` or `no-store`), keys on the full URL including the query string, and forwards to on-demand routes the cookies they read. Configuring the CDN to meet that is infrastructure, outside this skill.

The same test governs `Astro.cache.set()`. A cache hit is served before middleware runs, so never call it on an auth-gated route or a per-user island.

In this document "server" means wherever frontmatter runs — the build for a prerendered page, the request for an on-demand one. Everything that names cookies, sessions, POST, PRG, form Actions, auth, or query params applies to on-demand pages only.

`output` only sets the default. A project with any request-dependent page sets `output: 'server'` and marks every page that does not depend on the request with `export const prerender = true`. The default then sits on the correct side: no request-dependent page can be built ahead of time by omission. A project with none leaves the default (`'static'`), and still needs the adapter if it has a Server Island or an Action.

## Priorities

When concerns conflict, choose in this order:

1. Correctness > Experience
   * Data integrity and security are not traded for smoother interactions
2. Recoverability > Continuity
   * Ability to reconstruct state from canonical sources over never losing state
3. Visibility > Abstraction
   * Layer boundaries visible in code structure over boundaries hidden by framework
4. Simplicity > Richness
   * Fewer layers and less code over more features and interactivity

## Reliability Layers

1. HTML / Browser APIs
   * Never breaks. Use as foundation. Semantic correctness (`<button>`, not `<div onclick>`) is the precondition
   * Declarative invocation belongs here — `command` / `commandfor`, `popovertarget`, `<details>`, `<form method="dialog">`. Built-in commands are active as soon as the markup is parsed: no listener to attach, no hydration to wait for, no click lost in between
2. CSS
   * Breaks visually only. No functional impact. Interactions achievable with `:hover`, `:focus`, `::after`, or `:target` belong here — not in an island
3. Stateless island (props only)
   * Safe if inputs are correct. Guarantee inputs server-side
4. Stateful island (local state)
   * Highest risk. Minimize this layer

Rule: always ask "what happens if this breaks?" and implement in the lowest-numbered layer that accomplishes the task.

HTML output notes (Astro 7): unclosed tags are build errors. Invalid nesting (e.g., `<div>` inside `<p>`) is passed through as-is — the browser's error recovery restructures the DOM, so the rendered DOM diverges from the source. Author structurally valid HTML; the compiler does not correct it. Whitespace follows JSX rules by default (`compressHTML: 'jsx'`): whitespace within a single line is preserved (a space between two inline elements on the same line survives), while whitespace and line breaks around elements are removed. The one trap: inline elements separated by a line break lose their spacing — keep spaced inline elements on the same line, or write the space explicitly.

Islands in layers 3-4 should be wrapped in each framework's error boundary mechanism. If the island crashes, display a fallback UI instead of a blank space. This structurally enforces the isolation guarantee — the rest of the page remains intact.

Server-side errors (on-demand pages): provide `pages/500.astro` for unhandled exceptions. In `---` frontmatter, catch expected errors (DB failures, external API errors) and render an error state in HTML. Action handlers throw `ActionError` with appropriate codes — the calling island or form receives the error and displays feedback.

Partial failure (on-demand pages): when frontmatter fetches from multiple sources, catch each independently. Display successful results and show a warning banner for failed sources. Do not fail the entire page because one source is unavailable — this applies "Recoverability > Continuity" at the data level.

Build-time failure (prerendered pages): never bake a source's failure into the output — no warning banner or error state in prerendered HTML. It would be served to every visitor until the next build, including after the source recovers, and no reload clears it. What the build does when a source fails is a deployment decision, not this skill's. A source whose failure the user must see when viewing is read on demand (the page, or a `server:defer` component).

## Component Decisions

### Page Structure

Pages combine semantic HTML for structure with components at concern boundaries:

```astro
<Layout>
  <section>
    <h1>{title}</h1>
    <ItemFilter />
    <ItemTable items={items} />
  </section>
  <aside>
    <RecentActivity server:defer />
  </aside>
</Layout>
```

The page file holds frontmatter (data fetching, PRG handling), layout, the semantic HTML skeleton (landmarks, headings), and component composition — including conditional composition. Data display (lists, tables, cards, field groups) lives in extracted components. A page-level `<script>` is allowed only for behavior that spans multiple sections (see Script Behavior).

Extraction is the default, not the exception. When building a page, actively look for splittable components — any section you can name is a component. Signals, strongest first:

* It carries `<script>` behavior — repeated wiring is the strongest boundary marker (rule in Script Behavior)
* It fetches its own data (enables partial failure pattern)
* It has independent failure modes (limits blast radius)

Within a single page, extraction needs no reuse justification — decomposing `.astro` is free at runtime (compiles to HTML; props and slots are the only interface). Reuse across 2+ pages is the design-component context: promote to `shared/components/` (project-wide) or the feature's `components/` (feature-wide). Extraction bounds what an edit can touch — not runtime errors: an uncaught frontmatter error in any component still fails the whole render — the response on demand, the build when prerendered (runtime containment comes from the partial failure pattern).

Inline HTML remains only for the skeleton and nameless structural glue (grid frames, dividers). Smell: an `.astro` file over ~100 lines — look for a boundary.

Guards — split wide, not deep:

* Compose one level per page (page → sections) by default. A deeper level needs its own nameable concern — never a pass-through component that only forwards props
* Do not extract what you cannot name — a component without a nameable concern is fragmentation, not decomposition

Use Astro (.astro) by default:

* Data display (tables, cards, lists)
* Navigation, layout, conditional rendering

Use islands only when local state is required:

* Forms with validation or dynamic fields
* Dialogs with internal state (validation, dynamic fields, multi-step) — the island owns the content, never the open/close
* Real-time calculations (price, filtering)
* Browser-native APIs (Geolocation, Web Speech)

Submit-only forms use `<form>` + `FormData` — no island needed.

Client directives:

* `client:idle` — default for most islands
* `client:load` — elements the user interacts with before scrolling (search box, primary navigation toggle)
* `client:visible` — for below-the-fold content

Server Islands (`server:defer`):

Defer server rendering of auxiliary components. The page loads with fallback content; the component renders asynchronously via a separate server request. The component itself always renders on demand, also inside a prerendered page.

When to use frontmatter (default):

* Page's main content — block the render to guarantee delivery
* Auxiliary content needed immediately — use partial failure pattern on error
* Content whose absence would mislead the user

When to use `server:defer`:

* Expensive computation where the user expects loading time (analytics charts, aggregated reports)
* Personalized content on an otherwise cacheable or prerendered page (user profile widget on a public page)

Fallback design: if the Server Island fails, the fallback remains silently — no error UI, no retry. Design the fallback as content meaningful on its own, not just a loading indicator.

A Server Island is its own entry point, requested separately from the page, and anyone holding its URL can request it. Middleware lets it through without the auth redirect (see Security Boundary). An island that renders non-public data checks `Astro.locals.user`, and authorization for its props, itself, and renders the signed-out state otherwise — a sign-in link in an app bar, or nothing. Encrypted props are tamper-proof, not access control.

Per-user parts of a layout — an app bar showing the signed-in user — are one `.astro` component that reads `Astro.locals.user`, and the caller decides how it renders. The layout for auth-gated pages renders it directly: those pages are on-demand and private. The layout for prerendered or CDN-cacheable pages renders it with `server:defer`, so the page HTML depends on no user and only the island request is per-user. A form in such a component posts to an explicit on-demand URL (`<form method="POST" action="/logout">`, handled by `pages/logout.astro` with PRG), never to the current page, which may be a static file.

Island responses fall under the same private default as every on-demand response (see the middleware): the island URL carries encrypted props, not the cookie, so a shared cache would serve one user's island to another. An island whose output depends on nothing but its props opts into sharing by setting its own `Cache-Control`.

Decision test:

1. Can markup alone do it (open or close a dialog or popover, disclosure, submit)?
   * Yes → `command` / `commandfor`, `popovertarget`, `<details>`, `<form method="dialog">`. Layer 1. No script, no island
2. Does it need DOM manipulation without state (scroll-to-top, clipboard copy)?
   * Yes → `<script>` tag. Layer 1 (Browser API). No island needed
3. Does it declare local state? (useState / ref() / $state)
   * No → Astro component
   * Yes → Island. Group values that change together (e.g., form fields) into a single state object. Keep independently changing values as separate declarations. If the island's state serves more than one user interaction, split each interaction into its own island

Island verification — before writing an island, confirm each hook is necessary:

* useState holding server data → pass as props from frontmatter. No island needed
* useState for dialog/popover open state → `<dialog>` or `[popover]` driven by `command` / `commandfor`. No island needed
* useState for which accordion panel is open → `<details name="group">`. No island needed
* useState for tooltip/hover display → HTML `title` attribute or CSS `:hover`. No island needed
* useState for scroll/carousel position → CSS `overflow-x: auto`. No island needed
* useState for filter/sort selection → URL query params + server-side filtering. No island needed
* useEffect fetching data → move to frontmatter. Never fetch in islands
* useEffect syncing URL → URL is canonical. Use `<a>` or `window.location.assign()`
* useEffect for DOM manipulation without state → use `<script>` tag

If all state values are replaceable, the island is unnecessary — rewrite as `.astro` component or `<script>`.

For the fuller decision model including navigation-first evaluation, see architecture.md section 5.1.

### Declarative Invocation

Before reaching for `<script>` or an island, check whether markup already carries the behavior. `command` / `commandfor` binds a `<button>` to a target element by ID, and the browser handles activation, focus, top layer, and Esc.

```astro
---
const { item } = Astro.props;
const dialogId = `confirm-${item.id}`;
---
<button type="button" command="show-modal" commandfor={dialogId}>Delete</button>

<dialog id={dialogId}>
  <p>Delete {item.name}?</p>
  <form method="dialog">
    <button value="cancel">Cancel</button>
  </form>
  <form method="POST">
    <input type="hidden" name="id" value={item.id} />
    <button type="submit">Delete</button>
  </form>
</dialog>
```

Built-in commands: `show-modal`, `close`, `request-close` (fires a preventable `cancel` first), `show-popover`, `hide-popover`, `toggle-popover`. `value` on a closing button sets the dialog's `returnValue`.

Rules:

* Always write `type="button"` on a command button. Inside a `<form>` it is required: with the default type the browser returns from the button's activation behavior before it ever reads `commandfor`, so the command never runs and nothing is reported. Outside a form it costs nothing and keeps one rule
* Failures are silent by design. A `commandfor` naming no element, a command value that is neither built-in nor `--`-prefixed, and a dialog command aimed at something that is not a `<dialog>` all do nothing — no error, no event. IDs are derived from props here, so a bad interpolation fails this way
* The command event is cancelable and does not bubble. `preventDefault()` on it cancels the built-in action, and `preventDefault()` on the button's `click` stops the command event from firing at all. Not bubbling means document-level delegation receives nothing — wire each target
* That cancelation covers command buttons only. Esc, `<form method="dialog">`, and `close()` fire no command event, so a guard built on it lets those paths through. To hold every dismissal, close with `command="request-close"` and listen for the dialog's own `cancel` event, which Esc raises too — and keep `command="close"` and `<form method="dialog">` out of a dialog that needs guarding, since both close without raising it
* Do not open a dialog from a script when a button opens it. `dialog.showModal()` behind a click listener is the SPA-era form: it rebuilds in JS what the button already declares, and it stays inert until that listener attaches. Write `command="show-modal"`. The script form belongs in the project-level fallback below the support floor, and the fact that such a fallback is possible is not a reason to write one by hand
* Openings that no user action triggers are the exception — an Action failing, a session about to expire, a server-sent event. There is no button to declare, so the island that owns that outcome calls `showModal()`. This stays an exception because the trigger is a program event, not an interaction
* Dialog open/close is not state. An island holding `isOpen` moves a layer-1 construct into layer 4. The island owns dialog content only when that content needs validation, dynamic fields, or multi-step flow
* IDs are the binding. A component rendered N times needs N unique IDs — derive them from props, the same constraint as a component `<script>` running once for N instances
* Modal overlays use `<dialog>` + `show-modal`. Non-modal ones (menus, user-triggered toasts, hint panels) use `popover` + `popovertarget` or `command="toggle-popover"`. Accordions and disclosure use `<details>` / `<details name>`. Never build any of these from a `div` plus class toggling
* `closedby="any"` (light dismiss) is not Baseline. Add it as an enhancement and always keep an explicit close control
* Behavior with no built-in command is a custom command, not a click listener on the button. The name starts with `--`, and the browser dispatches a `CommandEvent` on the target
* The support floor decides the fallback, not the calendar. `command` / `commandfor` shipped in every engine in 2025-12, so a project serving current browsers takes it as-is. Where the floor reaches older versions, add one feature-detected fallback script at the layout level (`'command' in HTMLButtonElement.prototype`) — one project-wide crumple zone, never per-component wiring, deleted when the floor clears

Custom commands:

```astro
---
const { token } = Astro.props;
const fieldId = `token-${token.id}`;
---
<button type="button" command="--copy" commandfor={fieldId}>Copy</button>
<output id={fieldId} data-token-field>{token.value}</output>

<script>
  document.querySelectorAll("[data-token-field]").forEach((field) => {
    field.addEventListener("command", (event) => {
      if (event.command === "--copy") {
        navigator.clipboard.writeText(field.textContent ?? "");
      }
    });
  });
</script>
```

* The listener goes on the target, so the script belongs to the target's component — not the button's. This is the same rule as behavior living with the markup it drives
* `event.command` distinguishes several commands on one target, replacing a set of separate click listeners. `event.source` identifies which button was pressed, so multiple triggers for one target need no extra wiring
* The listener is still ordinary JS, so this variant carries the same "inert until the script runs" gap as any listener. The gain is the binding and the wiring, not the removal of the gap

### Script Behavior

`<script>` handles layer-1 behavior that no markup declares: scroll, clipboard, focus moves, measurement. Opening and closing dialogs and popovers is not in this set — that is Declarative Invocation above. Keep each script next to the markup it drives:

* One component, one concern, one `<script>`. A page script wiring two unrelated widgets is the signal to split — extract each widget's markup together with its script into a dedicated component
* Repeated script behavior is a component-boundary detector. The same wiring appearing twice — across sections or pages — marks a component to extract, owning both the markup and the script
* Cross-cutting behavior that reads fields across sections is the parent's concern. Keep it at the parent — a page-level `<script>` is the legitimate home for section-spanning behavior. Pushing it into a child forces the child to query ancestor DOM
* The goal is locality of source, not DOM sandboxing. Astro `<script>` is module-scoped, and document-wide `querySelector` is acceptable. Do not add wrapper elements or `data-scope` attributes just to narrow queries
* A component's `<script>` runs once per page even when the component renders N times. Wire with `querySelectorAll` or `data-*` lookups and per-element listeners — never assume a single instance
* Pass server values to a script through `data-*` attributes on the element it drives and read them from its `dataset` — the value stays declared in markup, each instance carries its own, and the script stays bundled
* The moment a behavior needs local state, it becomes an island (see decision test) — split the markup, not the script

For the reasoning behind granularity and behavior locality, see architecture.md section 5.5.

## Pages and Data

Execution boundary: the `---` block runs on the server. The HTML below it is the output. No JavaScript reaches the browser unless explicitly added via `<script>` or an island. Do not define client-side functions in frontmatter.

* Fetch data (DB, API, CMS) in `.astro` frontmatter — per request or at build, per Rendering Mode. Never fetch in islands
* Static configuration may be imported directly in islands — no need to serialize through props. A module is static configuration if it requires no server-side execution (no I/O, no async, no secrets). Examples: `as const` objects, Zod schemas, enum definitions, display constants
* Pass only rendering data as props. Never pass auth tokens, API keys, or session data
* Use `Astro.locals` (set by middleware) for auth/role data

```astro
---
const role = Astro.locals.role;
if (!role) return Astro.redirect("/login");
const items = await getItems();
---
<Layout>
  <ItemTable items={items} />
  <EditForm client:idle initialData={items[0]} />
</Layout>
```

## State Placement

Canonical sources (same value on every read):

| State                                                      | Where                                           | Why                                                                                    |
| ---------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------- |
| Auth / session                                             | HttpOnly Cookie                                 | Server-readable, hidden from JS                                                        |
| Current page                                               | URL path                                        | Browser-managed                                                                        |
| View state (filters, pagination, sort, search, active tab) | URL query params                                | Shareable, bookmarkable, reproducible — with or without authentication                 |
| In-progress data                                           | sessionStorage                                  | Per-tab, survives navigation                                                           |
| User preferences                                           | Server-side DB / HttpOnly Cookie / localStorage | DB if backend exists; cookie if BFF-only and server reads; localStorage if client-only |

URL vs cookie is not about authentication — it is about state type. View state (what you are looking at) always goes in the URL, even in authenticated apps. Identity state (who you are) goes in cookies. Preference state (how you want things) goes in DB or cookies. When URL params and cookie defaults overlap, URL params take precedence.

Transient state (lost on reload):

* Form input where the UI updates before submission (validation errors, dynamic fields, live preview) → component local state (controlled component)

A controlled component moves value from HTML to framework — higher to lower reliability. If the UI does not update before submission, use `<form>` + `FormData`.

Never use:

* Global state libraries (Redux, Zustand, Jotai, Pinia, Vuex, Svelte stores)
* Cross-island state sharing (Context, provide/inject)
* localStorage for auth/session
* `history.pushState()` / `history.replaceState()` in islands — the URL is a canonical source. To update query params (filters, pagination), use `<a>` or `window.location.assign()` with the new query string

## Mutation Pattern

Server handles correctness, client provides feedback as a crumple zone.

Client-initiated mutations that accept user input requiring validation must use Astro Actions (`astro:actions`). Actions provide type-safe server functions with built-in Zod validation — the caller gets compile-time type errors if the contract is violated. Mutations without user input (logout, session clear) use `<form method="POST">` with PRG — no Action needed. Navigation without data change uses `<a>` — not an Action.

The page a mutation posts to is on-demand — a form without `action`, or with a form Action, posts to the current page. Do not move a mutation into a client island's Action call to keep its page prerendered.

```typescript
// src/actions/index.ts
import { defineAction, ActionError } from "astro:actions";
import { z } from "astro/zod";

export const server = {
  createItem: defineAction({
    accept: "form",
    input: z.object({ name: z.string().min(1) }),
    handler: async (input, context) => {
      const user = context.locals.user;
      if (!user) throw new ActionError({ code: "UNAUTHORIZED" });
      return await saveItem(input, user);
    },
  }),
};
```

Choose the lowest layer that meets the requirement:

1. `<form method="POST">` with PRG — pure HTML. No Action, no JS. Use when the form carries no user-authored content (logout, deletion by ID, re-processing a previous result). POST processes the request, stores results in `Astro.session` if needed, and redirects. This avoids the browser's "resubmit form?" warning on reload
2. `<form action={actions.createItem}>` — HTML + Action. Still zero JS. Use when POST accepts user input that requires type-safe Zod validation and structured error handling
3. Island calls `actions.createItem()` — JS required. Use only when the mutation requires JS to modify the DOM (disable button, show spinner, display error, update list without reload)

PRG pattern (layer 1):

```astro
---
// pages/items/[id].astro
import { setFlash } from "../../flash";
const { id } = Astro.params;
if (Astro.request.method === "POST") {
  await deleteItem(id, Astro.locals.user); // data/ checks existence and ownership
  setFlash(Astro.cookies, "item-deleted");
  return Astro.redirect("/items");
}
const item = await getItem(id);
---
<form method="POST">
  <button>Delete</button>
</form>
```

The form has no fields: the target is in the URL, and the session cookie identifies the user. Anything the POST does read from the body is validated before use, like any other entry point.

Action definition:

* `accept: "form"` — for `<form>` submissions (layer 2). Prefer this
* `accept: "json"` — for programmatic calls from islands (layer 3). Use when the island constructs the payload without a form element

Client feedback (crumple zone):

1. Submission starts → disable button, show progress
2. Success → reconstruct from canonical sources:
   * `window.location.reload()` — redisplay the same page
   * `window.location.assign(url)` — move to another page
   * To confirm the outcome on the page the user lands on, use a post-navigation message (below)
3. Failure → re-enable button, show error

If feedback breaks, the action still completes or fails correctly on the server.

Post-navigation messages (toast, snackbar):

The typical case: the user saves, lands on the saved item's page, and sees "Saved." once. The server side of the mutation writes a message key into a one-shot cookie, the navigation carries it, middleware reads it on the next page request and deletes it once a page has rendered, and the layout renders the message. One module owns the keys and the cookie:

```typescript
// src/flash.ts
import type { AstroCookies } from "astro";

const FLASH_MESSAGES = {
  "item-saved": "Saved.",
  "item-deleted": "Deleted.",
} as const;

export type FlashKey = keyof typeof FLASH_MESSAGES;

// __Host-: the browser accepts the cookie only with Secure, Path=/ and no Domain
const COOKIE = "__Host-flash";
const SCOPE = { path: "/", secure: true } as const;

export function setFlash(cookies: AstroCookies, key: FlashKey) {
  cookies.set(COOKIE, key, { ...SCOPE, httpOnly: true, sameSite: "lax", maxAge: 60 });
}

export function readFlash(cookies: AstroCookies) {
  return cookies.get(COOKIE)?.value;
}

export function clearFlash(cookies: AstroCookies) {
  cookies.delete(COOKIE, SCOPE);
}

export function flashMessage(key: string | undefined) {
  return key && Object.hasOwn(FLASH_MESSAGES, key)
    ? FLASH_MESSAGES[key as FlashKey]
    : undefined;
}
```

The Action handler sets the key once the outcome is decided, and returns what the caller needs to navigate:

```typescript
// src/actions/index.ts — createItem, with `import { setFlash } from "../flash"`
handler: async (input, context) => {
  const user = context.locals.user;
  if (!user) throw new ActionError({ code: "UNAUTHORIZED" });
  const item = await saveItem(input, user);
  setFlash(context.cookies, "item-saved");
  return { id: item.id };
},
```

The caller navigates on success. A form Action (layer 2) renders the page in the POST response, so the page redirects; an island (layer 3) assigns:

```astro
---
// pages/items/new.astro
import { actions } from "astro:actions";
const result = Astro.getActionResult(actions.createItem);
if (result && !result.error) return Astro.redirect(`/items/${result.data.id}`);
---
<form method="POST" action={actions.createItem}>
  <input name="name" required />
  {result?.error && <p role="alert">Could not save. Check the name and try again.</p>}
  <button>Save</button>
</form>
```

```typescript
// island wrapper
const { data, error } = await actions.createItem(formData);
if (error) return showError(error);
window.location.assign(`/items/${data.id}`);
```

```typescript
// middleware.ts — in `auth`, after the auth redirect, replacing the final `return next()`
// (imports readFlash and clearFlash from "./flash")
const headers = context.request.headers;
const dest = headers.get("sec-fetch-dest");
const isPageRequest =
  context.request.method === "GET" &&
  (dest === null || dest === "document") &&
  !headers.has("sec-purpose");
const flash = isPageRequest ? readFlash(context.cookies) : undefined;
context.locals.flash = flash;

const response = await next();
const redirected = response.status >= 300 && response.status < 400;
if (flash && !redirected) {
  clearFlash(context.cookies);
  // This response carries one user's message: keep it out of shared caches.
  response.headers.set("Cache-Control", "private, no-store");
}
return response;
```

```astro
---
// shared/layout/Layout.astro
import { flashMessage } from "../../flash";
const message = flashMessage(Astro.locals.flash);
---
{message && <p role="status" class="flash">{message}</p>}
```

A message the user must acknowledge — a failure reported after a redirect, a warning — takes the dismissible variant. The popover renders in the top layer, above every stacking context, and its close button is declarative:

```astro
{message && (
  <div id="flash" popover="manual" role="status">
    <p>{message}</p>
    <button type="button" popovertarget="flash" popovertargetaction="hide">Close</button>
  </div>
)}

<script>
  document.getElementById("flash")?.showPopover();
</script>
```

* A popover stays hidden until shown, and no attribute opens one on load, so the variant costs one line of layout script — the same exception as other openings no user action triggers. If the script does not run, only the message is lost
* `popover="manual"`, not `auto`: an auto popover closes when the user opens a menu or any other auto popover, and on any outside click

* An Action that calls `setFlash` is a navigate-on-success Action, the same way an endpoint that redirects is. Its callers follow the navigation it implies — a redirect (layer 2) or `assign()` / `reload()` (layer 3). A caller that stays on the page shows the outcome in place and calls an Action that sets no flash; calling a navigate-on-success Action without navigating breaks its contract, and the message surfaces on a later, unrelated page
* The page a navigate-on-success Action lands on is on-demand and never served from a shared cache. A prerendered or CDN-cached response never reaches the middleware, so the cookie waits and surfaces on the next page that does. The response that renders the message is forced to `private, no-store`, even on a page that opted into sharing, so a CDN never stores one user's message
* The cookie carries a key, never the message text. `FlashKey` makes a setter naming a missing key a type error. The reader checks with `Object.hasOwn`, so a tampered cookie can only pick a message the app already has (a plain lookup would also match inherited keys such as `constructor`), and the rendered text is never user-controlled
* The `__Host-` prefix keeps a sibling subdomain from planting the cookie. The browser also rejects a deletion that lacks `Secure` or `Path=/`, so set and clear share one scope inside `flash.ts` — a mismatched clear leaves the message showing on every page until it expires. WebKit does not accept a `__Host-` cookie on `http://localhost`, so a dev server on plain HTTP loses the message in Safari; serve dev over HTTPS when testing it there
* Delete in middleware, after `next()`, and only when the response is not a redirect. A redirect — the auth redirect, or `Astro.redirect` in a page — passes the cookie on to the page that finally renders. Cookies set after `next()` still reach the response headers. Declare `flash` on `App.Locals`
* Read on page requests only. An island's Action call, a Server Island fetch, or a speculative prefetch (`Sec-Purpose`) would otherwise take the message before the page that should show it. A missing `Sec-Fetch-Dest` header counts as a page request; in browsers below the floor that omit it, a prefetch can take the message
* The default message is server HTML. Placement and fade-out are CSS. No island, no client state, no script — a `role="status"` paragraph is visible as parsed, which a popover is not
* `Astro.session` carries result data across a PRG redirect; the flash cookie carries only the outcome message, and needs no session driver
* If the cookie is lost, only the message is missing — the outcome was decided before the navigation. The cookie belongs to the browser, not the tab, so the message can also appear in another tab's next page, and a back navigation can show it again from cache
* An outcome shown without a navigation (a form Action's validation error, an island's Action returning an error) is displayed in place, not through the cookie. A failure that still ends in a navigation (a layer-1 POST that redirects either way) sets its own key the same way

View/Wrapper separation — when an island calls Actions, separate View from integration:

* View (`react/ItemFormView.tsx`): pure React component. Receives data and callbacks as props. No `astro:actions` import
* Wrapper (`ItemForm.tsx`): imports `astro:actions`, manages state, passes props to View

The wrapper is the crumple zone between UI and server actions. View changes and action logic changes stay independent. Split when the View has enough complexity that framework integration (astro:actions, state management) should not live alongside rendering logic. A single-action button does not need separation.

`pages/api/` is only for external consumers (webhook receivers, SSE/streaming) — not for mutations initiated by the client. Client-initiated mutations — including file uploads — use Actions (`accept: "form"` handles FormData with files) with processing logic in `features/*/data/`. Mutations without user input (logout, session clear) use PRG.

## Security Boundary

Server-side code — frontmatter, Action handlers, endpoints, middleware — is the trust boundary between browser and backend. Where pages render on demand, the Astro server is a BFF.

Required for all applications:

* HTTPS in production
* All cookies: `HttpOnly` + `SameSite=Lax` + `Secure`
* External API calls only on the server — `.astro` frontmatter, Action handlers, or API routes, through `features/*/data/`
* Validate all Action and API route inputs with Zod

When authentication is required:

* Middleware checks auth on every page request. API routes and Server Islands are entry points that check `Astro.locals.user` themselves
* Every auth-gated page is on-demand — a prerendered page is built without a request and never passes this check

Forbidden:

* Islands fetching external APIs directly
* Secrets in `PUBLIC_` env vars
* Auth tokens or API keys as island props
* Auth data via `document.cookie`

Browser-native APIs (Geolocation, Chrome AI, Web Speech) belong to the HTML/Browser API layer. Browser choice is the user's responsibility.

Middleware:

```typescript
import { defineMiddleware, sequence } from "astro:middleware";

// On-demand responses are private unless the page or island opted into sharing.
const cachePolicy = defineMiddleware(async (context, next) => {
  const response = await next();
  if (!context.isPrerendered && !response.headers.has("Cache-Control")) {
    response.headers.set("Cache-Control", "private, no-store");
  }
  return response;
});

const auth = defineMiddleware(async (context, next) => {
  // Prerendered pages are built without a request: no cookies, no headers, no auth.
  // An auth-gated page marked prerendered fails the build instead of shipping its content.
  if (context.isPrerendered) {
    if (!isPublicPath(context.url.pathname)) {
      throw new Error(`Auth-gated page is prerendered: ${context.url.pathname}`);
    }
    return next();
  }

  const session = context.cookies.get("session")?.value;
  context.locals.user = session ? await resolveUser(session) : null;

  // API routes and Server Islands are entry points that check context.locals.user
  // themselves. No redirect: an API route returns an HTTP status; the browser follows a
  // redirect from an island request and swaps the login page into the island.
  // ("/_server-islands/[name]" is Astro's internal route for island requests.)
  if (context.url.pathname.startsWith("/api/")) return next();
  if (context.routePattern === "/_server-islands/[name]") return next();

  if (!context.locals.user && !isPublicPath(context.url.pathname)) {
    return context.redirect("/login");
  }

  return next();
});

export const onRequest = sequence(cachePolicy, auth);
```

Keep each early return only when the project has that kind of route: the `isPrerendered` guard with prerendered pages, the `/api/` branch with API routes, the island branch with Server Islands.

`isPublicPath` is an allowlist: it covers every prerendered path — including `404` and feeds — public on-demand pages such as `/login`, and the framework routes a signed-out visitor uses (`/_image`, `/_actions/{name}` for an Action called without a session). Any other page is gated, so a forgotten one fails closed.
Advanced routing (`src/fetch.ts`):

* `src/fetch.ts` is a reserved filename. Creating it replaces Astro's default request pipeline entirely — the exported `fetch()` must call the `astro()` handler (or compose `astro/fetch` handlers), otherwise middleware, Actions, and page rendering never run and every route breaks
* Do not create this file by default. Use it only for cross-cutting infrastructure that middleware cannot express (structured logging around pipeline phases, tracing, Hono interop), and always delegate to `astro()`
* Auth and authorization stay in middleware. Do not move them into `fetch.ts` — one visible security boundary, not two

## Navigation

Do not add `<ClientRouter />` — it converts the MPA into an SPA at runtime, reintroducing the client-side routing layer this architecture minimizes.

## Project Structure

`middleware.ts`, `flash.ts`, `actions/`, and `pages/api/` exist only when the project has on-demand rendering.

```
src/
  middleware.ts
  flash.ts                  — post-navigation message keys and the one-shot cookie (setFlash / readFlash / clearFlash / flashMessage)
  actions/
    index.ts                — Action definitions (handlers call into features/*/data/)
  pages/
    api/                    — endpoints for external consumers only (webhook receivers, SSE/streaming)
  features/
    {feature}/
      types.ts
      data/                 — data I/O, server-only (backend boundary: swap internals without changing callers). API fetch, file processing, DB queries, storage writes. Only frontmatter, Action handlers, and API routes call into data/. Islands must not import from this directory
      components/           — .astro (display) + island wrappers (Action integration)
        react/              — pure React views (no Astro imports, no framework integration)
  shared/
    layout/                 — AdminLayout, UserLayout
    components/             — project-specific shared components (Pagination, Badge). Built from ui/ primitives
    lib/                    — generic utilities only (date formatting, string helpers). Domain-specific logic belongs in features/
    types/                  — shared type definitions
  components/ui/            — third-party UI primitives (shadcn/ui, etc.). If not using a UI library, this directory is unnecessary — put primitives in shared/components/
```

## Test Strategy

Test effort follows the reliability layers:

1. Layer 1-2 (HTML/CSS, rendered output)
   * E2E tests (Playwright). Verify that pages return correct content and structure
2. Server (Actions / data access)
   * Unit tests. Guarantee server-side correctness — validation, authorization, data mutations
3. Islands (layers 3-4)
   * Storybook for human visual/interaction review. Whether feedback is appropriate is a human judgment
   * Automated tests (Playwright, component tests) raise confidence but do not determine correctness — they verify mechanics (button disables, error shows), not UX quality

The implementation layer determines test stability. A control the browser drives (`command` / `commandfor`, `<form>`, `<a>`) is active as soon as the markup is parsed, so an E2E click on it needs no wait condition. A control a listener drives is inert until its script executes — an island's until hydration completes — so the test must wait for evidence that the listener exists, or it is flaky. A click that needs a retry or an arbitrary wait is a finding about the implementation layer, not about the test.

Test directory mirrors source structure:

```
tests/
  fixtures/                     — real API response samples
  features/
    {feature}/
      data/
        {module}.test.ts
  shared/
```

Test shared infrastructure (fetch wrappers, caching, rate limiting) in shared test files. Consumer tests should not re-test shared behavior — they test their own logic assuming the shared layer works.

The server is the source of correctness. Test the server thoroughly; verify islands with human eyes.

## Post-Application Review

After applying CRZ principles, review every change against these checks before finalizing:

1. Blast radius check
   * For each new layer or pattern introduced, answer: "What happens if this breaks?" If the answer is "nothing, because the layer below already handles it," the layer is redundant.
2. Canonical source honesty
   * Is the declared canonical source genuinely authoritative, or is it a derived cache being treated as one? A sessionStorage copy of server data is a cache, not a canonical source. Name the actual authority and acknowledge stopgaps as stopgaps.
3. Dead code check
   * Are all exported functions called? Unused initializers, sync functions, or cache hydration calls indicate an over-designed layer.
4. Simplicity check
   * Did the change add a layer, abstraction, or intermediate state? Is that layer actually needed, or does a simpler mechanism (frontmatter props, direct DOM update, existing browser API) already solve the problem? Remove any layer that exists only to satisfy a principle rather than to solve a real problem.
5. Component decomposition check
   * Pages hold frontmatter, layout, semantic skeleton, and composition — data display lives in components; a page-level `<script>` only for section-spanning behavior. Every nameable section is extracted; inline HTML remains for the skeleton and nameless glue. The same markup on 2+ pages → promote to `shared/components/` (design-component context). Repeated script wiring marks a missed component boundary. An `.astro` file over ~100 lines needs a boundary search. No pass-through components that only forward props.
6. Island necessity check
   * For each island, list every `useState` call. Can each value be a server prop, URL query param, HTML attribute, CSS rule, declarative invoker (`command` / `commandfor`, `popovertarget`, `<details>`), or `<script>` DOM call? If yes for all values, the island should be an `.astro` component. An `isOpen` boolean is answered by the invoker wherever a button opens the dialog, which is the usual case.
7. Document consistency check
   * Does the change add, remove, or rename a feature, route, or component? If yes, verify that CLAUDE.md, PROJECT.md, and any other project documentation reflect the current state. Deleted features must be removed from documentation.

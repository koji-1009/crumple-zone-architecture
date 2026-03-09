# Crumple Zone Architecture

An architecture for building healthy web applications by trusting the browser. Design from failure modes — each layer is expected to fail independently; design works backwards from the blast radius.

Many capabilities that frameworks once reimplemented are now available as modern browser standards. Embrace this evolution of the browser. Minimize reimplementation by frameworks, build on MPA, and place client components only where interactive behavior is required.

## Background

The primary cause of unmaintainable frontends is the accumulation of client state complexity. In SPAs, the entire application becomes a risk zone holding client state, and over time dependencies and state become entangled. This architecture pushes state toward the server and canonical sources, structurally limiting what remains on the client.

Keep rewriting the frontend a realistic option. In systems where a backend API exists, the server can stay thin as a BFF — meaning frontend technology choices and replacements do not affect the backend. This structure is especially effective when serving web, mobile, and desktop applications from the same backend API.

## Priorities

When concerns conflict, choose in this order.

1. Correctness > Experience
   * Data integrity and security are not traded for smoother interactions
2. Recoverability > Continuity
   * The ability to reconstruct state from canonical sources is preferred over never losing state
3. Visibility > Abstraction
   * Layer boundaries visible in code structure are preferred over boundaries hidden by the framework
4. Simplicity > Richness
   * Fewer layers and less code are preferred over more features and interactivity

### Correctness and Experience Are Compatible

Correctness and experience were once a trade-off, but browser capabilities have closed much of that gap.

`ViewTransition` introduced smooth page transitions. `<form>` + `FormData` provides form handling integrated with autofill and accessibility. `sessionStorage` enables UI state persistence across pages.

Delegating to the browser rather than reproducing in a framework makes correctness and experience compatible.

## 1. Reliability Layers

| Layer                     | Impact on Failure                     | Design Stance                  |
| ------------------------- | ------------------------------------- | ------------------------------ |
| HTML / Browser APIs       | None. Behaves per spec                | Trust as foundation            |
| CSS                       | Layout breaks. Functionality intact   | Contain to visual impact       |
| Stateless UI (props only) | Display gaps if input is wrong        | Guarantee inputs on the server |
| Stateful UI (local state) | Feature outage. May become inoperable | Minimize this layer            |

The reliability of the HTML layer depends on semantic correctness. A `<button>` provides keyboard interaction, focus management, and screen reader support from the browser; a `<div onclick>` provides none of these. Using the appropriate HTML elements is the precondition for this layer to serve as a trustworthy foundation.

Browser-native APIs (Geolocation, Web Speech, etc.) also belong to this layer. Browser choice is the user's responsibility, outside the application provider's scope.

Design criterion: ask "what happens when this element breaks?" and push implementation toward layers with smaller blast radius.

## 2. Declarative State

Do not manage screen state in framework memory. Reconstruct it from canonical sources that return the same value on every read.

Canonical sources (idempotent reads):

* URL (path and query parameters)
  * Browser-managed
* Cookies (HttpOnly)
  * Browser-managed. Also readable by the server
* sessionStorage
  * Browser-managed. Scoped per tab. Holds the UI state that SPA frameworks would keep in component memory
* localStorage
  * Browser-managed. Shared across all tabs, persistent. For user preferences (theme, etc.)
  * The `storage` event provides automatic cross-tab synchronization
* Server data
  * Derived by the server using the above as keys

Transient state (not a canonical source):

* UI framework local state. Lost on reload

Minimize what enters local state. For form input, there is a choice of which layer holds the state:

| Requirement                          | Implementation             | Where state lives                                                             |
| ------------------------------------ | -------------------------- | ----------------------------------------------------------------------------- |
| Submit values only                   | `<form>` + `FormData`      | HTML layer. The `<input>` element holds the value itself. No framework needed |
| Validation or dynamic display needed | Controlled component       | Framework layer. Value managed in local state                                 |
| Preserve input across navigation     | Evacuate to sessionStorage | Browser-managed canonical source                                              |

A controlled component moves value from the HTML layer to the framework layer — from higher to lower reliability. Do this only when there is a clear reason such as validation or dynamic display.

Consequences:

* Screens are deterministically constructed from canonical sources
* State changes update a canonical source and trigger a reload. A reload is not a failure; it is the reconstruction mechanism

## 3. Security Boundary

Treat application code running on the client as an untrusted zone. The server functions as a BFF that returns HTML.

Baseline for all applications:

* HTTPS in production
* All cookies: `HttpOnly` + `SameSite=Lax` + `Secure`
* Secrets and external API calls stay on the server

```
Browser (untrusted)
  Application code holds no auth credentials
  Does not call external service APIs
  Can only call the server's API routes
──── HTTPS + HttpOnly + SameSite Cookie ────
Server (trust boundary / BFF)
  Cookie → session or preferences
  Backend API calls
  Secrets live here only
────────────────────────────────
Backend APIs / DB
```

Under this structure, even if XSS occurs:

* HttpOnly cookies cannot be read by JS
* SameSite cookies are not sent to foreign origins
* API keys reside on the server and do not leak
* UI components never received auth credentials in the first place

When authentication is required, middleware checks auth on every request and redirects unauthenticated users. Public applications without auth still benefit from the baseline above.

## 4. Experience Layer

Page transitions and interaction feedback are designed as crumple zones. If they break, functionality continues; the impact is contained to experience degradation.

`ViewTransition` — apply to MPA page switches for SPA-equivalent transitions:

* Persistent structures (header, sidebar) do not animate
* Only the content area fades
* Avoid size interpolation (slide, morph). Bitmap scaling causes text stretching
* In unsupported browsers, falls back to standard MPA navigation

Interaction feedback — if it breaks, the operation still completes:

| Moment           | Feedback                            | If it breaks                                     |
| ---------------- | ----------------------------------- | ------------------------------------------------ |
| Submitting       | Button disabled, progress indicator | Risk of double submission, but data is sent      |
| After success    | Message displayed after reload      | No message, but data is saved                    |
| After failure    | Error message displayed             | User unaware of failure, but data is unchanged   |
| Input validation | Pre-submission validation display   | Server-side validation acts as the final defense |

Feedback has a dual structure — client for experience, server for correctness. If the client side breaks, the server side guarantees functional integrity.

## 5. Decision Flows

### 5.1 Choosing the Implementation Layer

```
Does user interaction change the display?
├─ No → Server-rendered (HTML)
└─ Yes → Can a page navigation solve it?
          ├─ Yes → Navigate via link (<a>)
          └─ No → Does it need local state?
                   ├─ No → <script> (DOM manipulation only: dialog.showModal(), scroll, clipboard)
                   └─ Yes → Client component
                             Minimize local state;
                             extract stateless children
```

### 5.2 Where to Place State

```
Can it live in a canonical source? (idempotent reads)
├─ Server needs to read it       → Cookie (HttpOnly)
├─ Should be shareable/bookmarkable → URL query params
├─ Per-tab persistence is enough  → sessionStorage
├─ Cross-tab, persistent, client-only → localStorage
└─ None of the above (temporary until submission)
                                  → Component local state (transient)
```

## 6. Premises

1. Trust the browser and minimize framework dependency. The scope of MPA continues to expand as browsers gain native capabilities
2. Design for failure modes. Begin with fault containment, not feature addition
3. The server is the authority on state. The client is a display layer; a reload is the reconstruction mechanism from canonical sources
4. Make security boundaries visible. Use explicit API routes, not implicit RPCs

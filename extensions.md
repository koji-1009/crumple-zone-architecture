# Crumple Zone Architecture — Extensions

Clarifies the scope of the design principles and provides patterns for extending beyond that scope.

## 1. Scope of the Design Principles

### Achievable with MPA + Islands

Applications where the server returns HTML per request and island components are placed only where interactivity is needed.

* Content sites, blogs, documentation
* Admin panels, dashboards
* E-commerce (product listings, cart, checkout)
* Business applications (CRUD, approval workflows)
* Social media feeds (timelines, likes, comments)

### Requires SPA

Applications that require continuous bidirectional sync between client and server, or rendering beyond the browser's HTML pipeline.

* Real-time collaborative editing (Google Docs: Operational Transform / CRDT)
* Vector/raster drawing tools (Figma: WebGL/Canvas engine)
* In-browser IDEs (VS Code Web: virtual filesystem, terminal)

These require a different architecture, not an extension of MPA.

### On the Boundary (addressable with extension patterns)

Applications that are MPA at their core but require partial real-time capability.

* Messaging (real-time chat message receipt)
* Notification systems (unread badges, push notifications)
* Live dashboards (auto-updating metrics)
* Real-time comment feeds

## 2. Extension Patterns

### 2.1 Stream Inside an Island (small-scale real-time)

Contain real-time updates within a single island component, receiving data via Server-Sent Events (SSE).

```
Astro API route ──SSE──→ Browser EventSource (native API)
                              ↓
                         Island component receives and updates display
```

* Connection management is handled by EventSource (browser-native)
* Display updates use local state within the island component
* The rest of the page remains server-rendered
* On disconnect, EventSource auto-reconnects (browser spec)

Use cases: notification badges, chat widgets, status displays

### 2.2 Sync Layer as Canonical Source (large-scale real-time)

Introduce a library that encapsulates network, local cache, and automatic synchronization (e.g., Firebase Firestore), and treat it as a canonical source.

```
Firestore server ←auto-sync→ Firestore SDK local cache
                                    ↓
                              Application code reads
                              from cache (idempotent)
```

Qualification as a canonical source:

* Reads from local cache are idempotent (same value on every read)
* Synchronization is managed by the library, not application code
* Readable offline from cache

Relationship to design principles:

| Canonical source | Manager     | Sync mechanism    |
| ---------------- | ----------- | ----------------- |
| URL              | Browser     | User action       |
| Cookie           | Browser     | Server Set-Cookie |
| sessionStorage   | Browser     | Application code  |
| localStorage     | Browser     | `storage` event   |
| Sync layer cache | Library SDK | SDK auto-sync     |

All share the same structure: idempotent reads, synchronization handled by a separate mechanism.

### 2.3 Reactive sessionStorage

sessionStorage persists UI state across pages as a canonical source, but the browser does not fire change events for modifications within the same tab.

To detect changes from an island, dispatch a `CustomEvent` on write and listen for it on the receiving side.

```typescript
// Writer side (island or any script)
sessionStorage.setItem('likeState', JSON.stringify(state));
window.dispatchEvent(new CustomEvent('session-update', { detail: { key: 'likeState' } }));

// Receiving island
useEffect(() => {
  const handler = (e: CustomEvent) => {
    if (e.detail.key === 'likeState') {
      setState(JSON.parse(sessionStorage.getItem('likeState') ?? 'null'));
    }
  };
  window.addEventListener('session-update', handler as EventListener);
  return () => window.removeEventListener('session-update', handler as EventListener);
}, []);
```

* sessionStorage remains the canonical source. The CustomEvent is only a notification mechanism.
* If the island breaks, the sessionStorage value is intact. The next page load reconstructs from the canonical source
* Limit to non-sensitive UI state (like-button states, tab open/close, etc.)

Use cases: immediate cross-page UI state propagation

### 2.4 In-place Editing

A pattern common in admin data grids where clicking a cell opens it for direct editing. The island holds only the "which cell is being edited" state; on confirmation, it calls an Action. The table display remains an Astro component — the island intervenes only during editing.

```
Astro component (renders the full table)
  ↓ delegates click to island
Island (manages editing cell state)
  ↓ on confirm
Astro Action (updates on server, guarantees correctness)
  ↓ on success
Reload reconstructs the full table
```

* Separating display from edit-state management keeps the island's responsibility minimal
* If the island breaks, the table remains visible

### 2.5 Map Component

External libraries such as Leaflet require client-side initialization, making them a legitimate island use case. Location and marker data are fetched in the frontmatter and passed as props.

```astro
---
const locations = await getLocations(); // fetched server-side
---
<MapView client:visible locations={locations} />
```

* Data fetching completes on the server; the island handles rendering only
* `client:visible` defers JS loading until the map enters the viewport
* If the map fails to render, the rest of the page remains intact

### 2.6 Optimistic Updates

Update the UI immediately while the server processes the mutation. On reload, server data overwrites — the server remains the authority on state.

Two error-handling strategies depending on the consequence of failure:

**Fire-and-forget** — for operations where failure has no meaningful consequence (likes, emoji reactions). Uses the reactive sessionStorage mechanism from 2.3 — write to sessionStorage and notify via CustomEvent, then send the Action in the background. Any discrepancy resolves on reload.

```
Click
  → write to sessionStorage (immediate display)
  → notify island via CustomEvent
  → send Action (background)
Reload
  → server data overwrites as canonical source
```

**Revert on error** — for operations where the user expects persistence (wishlist, cart, bookmark, follow). Update local state optimistically and revert if the server returns an error.

```
Click
  → update local state (immediate feedback)
  → send Action
  → on error: revert to pre-mutation state
Reload
  → server data overwrites as canonical source
```

Both patterns share the same canonical authority model. The difference is only in how the client handles the gap between optimistic display and server confirmation.

* Fire-and-forget
  * Limit to operations where failure has no meaningful consequence
* Revert on error
  * The operation must be one the server can definitively accept or reject
  * Do not apply to operations with irreversible side effects (payments, external API calls)

### 2.7 Cache Layer

Cache layers (CDN, Service Worker, browser cache) are read-side optimizations. They sit between the canonical source (server) and the display, serving previously fetched data to reduce latency or provide continuity during network interruptions.

```
Server (canonical source)
  → Cache layer (read-side optimization)
    → Browser renders page
```

* The cache holds the last known correct state — not a diverged client state
* Stale or missing cache falls through to the server
* Reload reconstructs from the canonical source, overwriting any cached value
* Cache strategy (stale-while-revalidate, network-first, etc.) is a performance decision, not an architectural one

Cache layers are crumple zones by nature: if they break or become stale, the server corrects on the next reachable request. No state authority moves to the client.

In SSR environments, client-side caching is often redundant for server-readable canonical sources (cookies, databases). Every navigation reconstructs state from the canonical source via SSR. A client-side cache is justified when: (a) the canonical source is client-only (localStorage), (b) network latency must be hidden (offline-first), or (c) client-side navigation bypasses SSR (e.g., ViewTransition with ClientRouter).

Example: Firestore uses IndexedDB as a local cache, with the SDK managing synchronization. As a cache layer (2.7), it provides offline reads and reduced latency. As a sync layer (2.2), bidirectional synchronization elevates the cache to a canonical source — reads become idempotent regardless of connectivity. The two patterns share the same infrastructure but differ in architectural role.

### 2.8 Security Considerations

sessionStorage and sync layer caches, unlike HttpOnly cookies, are readable and writable by XSS. Combined with the security boundary from the design principles:

| Canonical source | XSS resistance                   | What to store                    |
| ---------------- | -------------------------------- | -------------------------------- |
| HttpOnly Cookie  | Unreadable by JS                 | Auth credentials, sessions       |
| URL query params | Readable (assumed non-sensitive) | Filters, pagination              |
| sessionStorage   | Readable/writable                | In-progress data (non-sensitive) |
| localStorage     | Readable/writable                | User preferences (non-sensitive) |
| Sync layer cache | Readable/writable                | Display data (non-sensitive)     |

Auth credentials always go in HttpOnly cookies. This does not change with extension patterns.

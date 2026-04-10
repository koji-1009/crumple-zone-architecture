# ClientRouter Exit Strategy

> Status: Active
> Created: 2026-03
> Scope: Astro `<ClientRouter />` -> browser-native cross-document view transitions

## Context

Astro's `<ClientRouter />` (formerly `<ViewTransitions />`) was introduced in mid-2023 when no browser supported cross-document view transitions. It intercepts navigation, fetches the next page, swaps the DOM, and optionally animates the transition — effectively converting an MPA into an SPA at runtime.

In CRZ, the ClientRouter is classified as a **crumple zone**: a layer that absorbs browser API gaps and is designed to be removed when the platform catches up.

## Approach: Fix Upstream, Then Remove

CRZ does not passively wait for the platform to mature. Where the ClientRouter has known defects, fix them in Astro directly. Each fix reduces the gap between ClientRouter's current behavior and what the browser can handle natively. When the remaining gap is small enough, removal becomes a minor migration rather than a rewrite.

## Current Role

### Already replaceable by browser-native APIs

| Responsibility | Native alternative |
| --- | --- |
| Triggering cross-document view transitions | `@view-transition` CSS at-rule |
| Named transitions (`transition:name`) | CSS `view-transition-name` property |
| Basic fade/morph animations | Browser-native animation defaults |
| Animation customization | `::view-transition-old()` / `::view-transition-new()` pseudo-elements, `view-transition-class` |

### Still requires ClientRouter (or equivalent)

| Responsibility | Why native is insufficient |
| --- | --- |
| `transition:persist` (DOM element preservation) | Full page load destroys DOM state. CRZ does not use this — each page is a fresh render |
| Script re-execution control | Native cross-document reloads all scripts |
| Fallback simulation for unsupported browsers | View Transition API cannot be polyfilled |
| Astro lifecycle events (`astro:before-swap`, etc.) | No native equivalent with identical semantics |

## Exit Conditions

Three conditions must converge before ClientRouter can be fully removed:

### 1. `transition:persist` becomes unnecessary

The project does not rely on DOM element preservation across navigations, OR browser APIs provide an alternative mechanism.

**Current assessment**: Most CRZ target applications (content sites, admin panels, CRUD tools) do not need this. If architecture already treats each page load as a fresh render (which CRZ encourages), this condition is met today.

### 2. Navigation API reaches baseline

The Navigation API is supported in all major browsers, including Safari.

**Current assessment**: Chrome and Firefox ship it. Safari does not yet. Track via [web-platform-dx/web-features](https://github.com/web-platform-dx/web-features).

### 3. Fallback is no longer a concern

The project accepts that browsers without View Transition API support get standard page navigation, OR the API reaches Baseline Widely Available.

**Current assessment**: Global support exceeds 85%. Firefox ships same-document transitions; cross-document is in active development.

## Migration Phases

| Phase | Trigger | Action |
| --- | --- | --- |
| **Current** | Now | Classify ClientRouter as crumple zone. Fix upstream defects. Minimize coupling to Astro lifecycle events. Avoid `transition:persist`. Write view transition CSS using standard properties alongside Astro directives |
| **Preparation** | Exit conditions 1-2 met | Audit `transition:persist` usage. Migrate shared state to URL/cookies/server session. Test `@view-transition` CSS on feature branches |
| **Migration** | All exit conditions met | Remove `<ClientRouter />`. Replace with `@view-transition` at-rule. Replace `transition:name` with CSS `view-transition-name`. Replace Astro lifecycle listeners with `pagereveal` / `DOMContentLoaded` / `load` |
| **Post-migration** | Migration complete | Delete crumple zone classification. Full browser-native cross-document view transitions |

## Design Guidelines for CRZ Projects Today

To keep the exit path clean:

1. **Do not depend on `transition:persist`**. Treat each page as a full render. Persist state in URL, cookie, or server session — not in the DOM

2. **Do not depend on Astro lifecycle events for business logic**. Use `astro:before-swap`, `astro:after-swap`, `astro:page-load` only for progressive enhancement (loading indicators, analytics). Site must function without them

3. **Avoid Astro-specific view transition directives**. CRZ does not use `transition:name` or `transition:animate` (see architecture.md Section 4). If a project does use them, write equivalent `view-transition-name` in CSS so that only the Astro directives need deletion at migration time

4. **Do not rely on script de-duplication**. ClientRouter prevents re-execution of scripts already in the DOM. Native cross-document transitions reload all scripts. Write idempotent scripts or guard against double-initialization

5. **Treat ClientRouter as a crumple zone, not infrastructure**. It appears in exactly one place (global layout `<head>`), and no component imports from `astro:transitions/client` for anything other than type annotations

## Relationship to CRZ Core Principles

This exit strategy applies CRZ's first premise: **trust the browser, calibrated to maturity**. The ClientRouter was necessary when the platform had a gap. As the gap closes, retaining it violates the premise — it becomes personally-owned scope that the browser should own.

CRZ practitioners actively contribute to closing the gap. Fixing Astro's ClientRouter defects upstream accelerates the exit timeline for all CRZ projects — each merged PR makes the crumple zone thinner.

## References

- [Astro docs: View Transitions](https://docs.astro.build/en/guides/view-transitions/)
- [Bag of Tricks: ClientRouter vs @view-transition comparison](https://events-3bg.pages.dev/jotter/feature-comparison/)
- [Bag of Tricks: Migration path](https://events-3bg.pages.dev/jotter/migrate/)
- [Astro roadmap discussion #770: Completion of the View Transition API](https://github.com/withastro/roadmap/discussions/770)
- [Astro docs issue #10902: ClientRouter positioning](https://github.com/withastro/docs/issues/10902)

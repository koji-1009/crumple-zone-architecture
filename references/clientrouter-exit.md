# ClientRouter Departure

> Status: Closed — CRZ does not use `<ClientRouter />` (departure declared 2026-07)
> Created: 2026-03
> Updated: 2026-07
> Scope: Astro `<ClientRouter />` -> browser-native cross-document view transitions

## Context

Astro's `<ClientRouter />` (formerly `<ViewTransitions />`) was introduced in mid-2023 when no browser supported cross-document view transitions. It intercepts navigation, fetches the next page, swaps the DOM, and optionally animates the transition — effectively converting an MPA into an SPA at runtime. In CRZ it was classified as a crumple zone: a layer that absorbs a browser API gap, designed to be removed when the platform catches up.

The platform has caught up. The `@view-transition` CSS at-rule triggers cross-document transitions (Chromium 126+, Safari 18.2+), `view-transition-name` and the `::view-transition-*` pseudo-elements cover naming and animation customization, and the Navigation API reached Baseline Newly Available in 2026-01. The exit conditions this document set in 2026-03 are met, with one accepted trade-off: Firefox ships same-document transitions only, so Firefox users get standard page navigation — experience degradation, not functional failure, the same crumple-zone contract the experience layer applies everywhere (architecture.md section 4). Astro's documentation points the same way: using `<ClientRouter />` "will increasingly become unnecessary" as browser APIs evolve.

## Policy

CRZ does not use the ClientRouter. The `@view-transition` CSS at-rule is the architecture's page-transition mechanism — this is the default in skill/crz.md and architecture.md section 4. The crumple zone starts at zero thickness.

Adopting the ClientRouter requires a concrete, named requirement that only it satisfies: fallback simulation for unsupported browsers, `transition:persist` (DOM preservation across navigations), script re-execution control, or Astro lifecycle events. CRZ removes these needs structurally — each page is a fresh render, scripts are idempotent, state lives in URL, cookie, or server session. Absent such a requirement, the ClientRouter is personally-owned scope that the browser now owns, and carrying it violates the first premise: trust the browser, calibrated to maturity.

CRZ practitioners contributed upstream fixes to the ClientRouter while it was still the necessary crumple zone. That work closed part of the gap this departure stands on.

## References

- [Astro docs: View Transitions](https://docs.astro.build/en/guides/view-transitions/)
- [Bag of Tricks: ClientRouter vs @view-transition comparison](https://events-3bg.pages.dev/jotter/feature-comparison/)
- [Astro roadmap discussion #770: Completion of the View Transition API](https://github.com/withastro/roadmap/discussions/770)
- [web-platform-dx/web-features](https://github.com/web-platform-dx/web-features) — Firefox / Baseline tracking

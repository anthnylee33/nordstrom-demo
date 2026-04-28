# ADR-0001 — Strategic decision: rewrite the frontend in Next.js, accepting the loss of upstream Discourse compatibility

**Status:** Proposed (Devin draft) — **CTO sign-off required before any other ADR is approved**
**Date:** 2026-04-28
**Decision owner:** Engineering leadership + Product
**Approver:** CTO

## Context

`anthnylee33/nordstrom-demo` is a downstream fork of [Discourse](https://github.com/discourse/discourse). Discourse is an Ember.js + Ruby on Rails application, ~14 years old, with a mature plugin/theme ecosystem (100+ official plugins, thousands of community plugins) and an upstream development pace of ~8–15 commits/day.

The team has chosen to migrate the frontend from Ember.js to React 19 + Next.js 15 (App Router), using **Strategy B with Strategy C-style backend reuse**: a greenfield Next.js application consuming the Discourse REST API, deployed on a parallel subdomain, cutting over per-feature once parity is achieved. The Rails backend is reused as-is.

Per the migration plan attached to the original engagement, this is **a rewrite, not a migration.** Concrete scope as measured against this branch:

- 7,008 frontend source files / 182,420 LOC to rewrite
- 706 components, 158 routes, 53 services, 56 models, 85 helpers, 15 modifiers in the core app
- 1,720 additional frontend files across 42 plugins
- 661 QUnit tests + the UI half of 1,457 RSpec system specs to re-author
- ~6,000–18,000 engineer-hours of component porting alone, before integration

## The load-bearing decision

**Migrating the frontend to React/Next.js means walking away permanently from the upstream Discourse plugin and theme ecosystem.** This decision is not technically reversible cheaply once the new repo exists and the team is staffed against it.

Specifically:

1. **No more upstream merges.** Discourse upstream ships ~8–15 commits/day; ~80% of those touch Ember frontend code. Once we replace the Ember frontend, every upstream commit that touches `app/assets/javascripts/` or `frontend/` is an active conflict, not a merge. We give up access to upstream security backports, performance work, and feature work for the frontend layer indefinitely.
2. **No more upstream plugins.** Every existing Discourse plugin (~100 official + thousands of community) ships its UI as Ember code. The plugins still load on the Rails backend, but their UI surface is gone. Operators cannot install new plugins from the existing ecosystem and have them work end-to-end.
3. **No more upstream themes.** Themes are Ember + Handlebars + SCSS. The theme system is one of Discourse's largest customer-facing features; we are giving it up at v1 and would have to design a replacement in v2 if we want comparable customizability.
4. **The fork becomes a hard fork.** Operationally, we are spinning up a new product that started from a snapshot of Discourse. The Rails backend stays Discourse-compatible, but the user-facing application is a new product with its own roadmap, its own bugs, and its own security surface.
5. **Operator migration cost.** Any third-party Discourse instance that was running our fork as a drop-in replacement must now also reckon with a new frontend, new assets, new plugin compatibility list, and new operational tooling.

## The alternative we are rejecting

Stay on Ember. The Ember 6.10 → 7.0.0-beta.1 upgrade we shipped on `demo/ember-upgrade` (PR #1, PR #2) demonstrates that the upstream Ember path is viable and currently green. Continuing on this path would:

- preserve upstream merges
- preserve plugin/theme ecosystem
- avoid the multi-quarter rewrite cost
- cost roughly 2–3 engineer-quarters of follow-up work to keep up with upstream + finish the deprecation cleanup flagged in PR #2

We are choosing not to do this. The product justification for the rewrite must be **more valuable than the multi-team multi-quarter cost + permanent loss of ecosystem compatibility.** That justification is owned by Product, not Engineering — and it must exist in writing before we proceed.

## Decision

We will execute the rewrite (Strategy B with Strategy C-style backend reuse). We accept the loss of upstream compatibility and the loss of plugin/theme ecosystem in exchange for the product outcomes Product is targeting (assumed: a modernized, customized forum/community product, not a generic Discourse install).

## Consequences

### Positive

- Modern React 19 + Next.js 15 application from day one — Server Components, streaming, edge-deployable, modern DevX.
- Frontend bundle size goes down significantly compared to a 7.0.0-beta.1 Ember bundle for end users (Server Components do most of the work server-side).
- SEO posture improves materially under Server Components vs the current Rails-rendered-then-Ember-hydrated approach.
- The team gains full control over the frontend architecture — we are no longer constrained by upstream Discourse's design choices.

### Negative

- 9–18 months to v1 cutover for the recommended team size (1 lead + 3–4 senior frontend + 1 backend + QA + SRE + PM + designer).
- We must maintain two products in parallel (Ember + Next.js) for the duration. Bug fixes during this period have to be doubled.
- Permanent loss of upstream ecosystem compatibility, as detailed above.
- Plugin compatibility shim is *not* in v1 scope (per ADR-0002). We must decide a per-plugin disposition (port / drop / defer) — see [`inventory/plugins.md`](../inventory/plugins.md).
- The Discourse admin app is staying on Ember for v1 (proxied behind `/admin/...`). We are *not* eliminating Ember from the codebase at v1; we are just eliminating it from the user-facing surface.

### Required pre-conditions

Before this ADR is signed off, the following must be true:

1. **Written product justification** signed by VP Product or equivalent that articulates the business outcome the rewrite is unblocking. "Modernization" alone is not sufficient — there must be a specific user-visible product outcome.
2. **CFO sign-off on multi-quarter cost.** A v1 with the recommended staffing is roughly $2.5–4M of engineering cost over 12 months at typical fully-loaded rates.
3. **Plugin disposition list** reviewed and approved by Product. Specifically the rows in `inventory/plugins.md` flagged "Port (v1) or Drop" — Product needs to make those calls based on production usage data.
4. **Communication plan** for any third-party operators of this fork — they need 6+ months of advance notice before plugin compatibility breaks.

If any of these four pre-conditions are missing, **do not approve this ADR.** Instead, return to the Ember upgrade path (`demo/ember-upgrade`) and revisit the rewrite when the product justification is clearer.

## Devin's recommendation

If your goal is a modernized Discourse fork: **do not approve this ADR.** Stay on Ember (the work we already shipped on `demo/ember-upgrade` is the right answer). The cost of this rewrite is much higher than its benefits when the goal is "Discourse but newer."

If your goal is a new community/forum product that happens to seed from Discourse data: **approve this ADR**, and treat the work in this Phase 0 directory as the foundation of a new product team, not as a maintenance investment in the existing fork. Staff and time-box accordingly.

This ADR is the only place in Phase 0 where the right answer might be "do not proceed." Every other ADR assumes this one is approved. Make this decision deliberately.

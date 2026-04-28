# ADR-0002 — v1 scope: core forum only, admin and chat deferred to v2

**Status:** Proposed (Devin-recommended; Q2 was skipped during Phase 0 scoping)
**Date:** 2026-04-28
**Decision owner:** Product + Engineering lead
**Approver:** VP Product
**Depends on:** ADR-0001 approved

## Context

The Ember Discourse application has three roughly-independent product surfaces:

1. **Core forum** — browse, read, post. Topics, posts, categories, tags, search, profile, login, signup, notifications.
2. **Admin** — site settings (~1,500 of them), users, customize → themes, logs, badges, plugins panel. Used by staff only.
3. **Chat** — separate real-time UI inside the same app, with its own router and components.
4. **Plugin-owned UI** — features each plugin contributes (reactions, voting, AI composer, calendar, etc.) — see `inventory/plugins.md`.

For the v1 cutover (going from "Ember is the frontend" to "Next.js is the frontend") we have to decide which of these are in scope.

## Decision

**v1 scope is core forum only.** Admin, chat, and plugin-owned UI are **deferred to v2.** During v1 cutover:

- Routes under `/admin/*`, `/chat/*`, and plugin-owned route prefixes are served by the **legacy Ember app** behind a path-level proxy (Cloudflare Worker, Nginx, or similar).
- All other routes are served by the **new Next.js app**.
- A small set of core-forum-required plugins (`hcaptcha`, `local-dates`, `presence`, plus per-product-decision a subset of `reactions`/`solved`/`post-voting`/`topic-voting`/`checklist`) is ported to React for v1 — see `inventory/plugins.md`.

## Why this scope

### Why exclude admin

The admin surface is the most plugin-coupled and the most schema-driven part of Discourse. It contains:

- Roughly 1,500 site settings, schema-driven UI (every setting renders via a generic `setting-row` component dispatching on type)
- The customize → themes UI, which is essentially an in-browser IDE for Ember themes
- Plugin admin panels — each plugin can register its own admin tab; ~20 plugins do this in our codebase
- Reports / dashboard, which embeds dynamic chart components

A "real" admin port is ~3–6 months of additional work. Worse, the schema-driven setting UI means we'd need to design a plugin-extension API for admin in React **before** porting any plugin admin UI, which is a large architectural commitment. **Defer this entirely.** Keep the Ember admin app alive behind `/admin/*` for as long as it takes to ship v1; revisit in v2.

### Why exclude chat

Discourse Chat is a substantial product on its own:

- Its own router, drawer UI, and modal stack
- Real-time channels, threads, message editing, reactions
- Mobile-specific layout and gestures
- Hundreds of components and tests

It is not part of the core forum reading/posting flow. Migrating it costs roughly as much as migrating the core forum. **Defer this to v2** and proxy `/chat/*` to the legacy Ember app at v1 cutover. Many forums don't use chat at all; if ours doesn't, drop it permanently in v2.

### Why exclude plugin-owned UI by default

There are 42 plugins with frontend code in this repo. Porting them all to React is multi-quarter work on its own. The only ones that are required at v1 are the ones whose UI sits inside the **core forum content rendering path**: those that render UI inside cooked posts (`local-dates`, `reactions`, `solved` indicators, `post-voting` widgets, etc.) or inside the topic page chrome (`presence` indicators).

Per `inventory/plugins.md`, the v1-required plugin set is at most 7 plugins, and which ones depends on per-forum usage data. **Product owns that list** — engineering will not unilaterally pick which plugins to port.

### Why include core forum

This is the only surface that affects every user, every visit, every page load. Migrating it is the entire reason this rewrite exists. If we don't include it, we've shipped a worse Discourse, not a better one.

## v1 acceptance criteria

The v1 Next.js app is "ready for cutover" when:

- Every route in [`inventory/routes.md`](../inventory/routes.md) marked "Phase 2" or "Phase 3" is implemented
- Every REST endpoint in [`inventory/v1-json-endpoints.txt`](../inventory/v1-json-endpoints.txt) that is reachable from a v1 route is consumed
- All 16 MessageBus channels relevant to v1 are subscribed and tested
- Authentication parity with the Ember app (see ADR-0004): login, signup, password reset, 2FA, passkey, OAuth providers
- Composer parity (see Phase 3 in the migration plan): markdown, preview, draft autosave, oneboxes, image upload, mentions, emoji
- Mobile responsive layout
- a11y: WCAG 2.1 AA on all v1 routes
- Performance: Lighthouse mobile-emulated score ≥ Ember baseline + 10 points (we expect a meaningful improvement from Server Components)
- SEO: `<title>`, OG, JSON-LD, canonical URLs all present and matching Ember
- Test parity: every QUnit test for a v1 route has been re-authored as a Vitest or Playwright test (≥ same coverage)
- Path-level proxy is configured and `/admin/*`, `/chat/*`, plugin-owned paths still work (i.e., user can still hit the Ember admin)
- All 7 high-severity advisories from the CISO summary that block v1 (per CISO triage of `pnpm audit` baseline) are mitigated

## Out of scope for v1 — deferred to v2 or later

- Admin app (`/admin/*` proxied to Ember)
- Chat app (`/chat/*` proxied to Ember)
- Theme system (no themable UI in v1; design system is fixed per ADR-0006)
- Plugin extension API for third-party plugins (none of our plugins are third-party)
- Multilingual / i18n beyond what Rails serves (the Ember app uses `discourse-i18n`; the React app inherits the same translation files via `/translations/<locale>.json`)
- Multi-site / multi-tenant features (Discourse `Multisite`)

## Out of scope permanently (drop)

- Anything from a plugin classified "Drop" in `inventory/plugins.md`
- IE11 compatibility (Ember already gave it up)
- Browser-extension integrations (Discourse Hub, etc.)

## Open question for Product

**Which of the "Port or Drop" plugins in `inventory/plugins.md` are required for v1?** Engineering needs production usage data (e.g., what % of posts use reactions, post-voting, etc.) before staffing can be finalized. Recommended path: pull a 30-day analytics window before approving this ADR.

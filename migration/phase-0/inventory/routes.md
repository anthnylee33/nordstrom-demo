# Route inventory — Ember `app-route-map.js` → Next.js App Router

**Source:** `frontend/discourse/app/routes/app-route-map.js` (158 `this.route(...)` calls)
**Target:** Next.js 15 App Router file-system routes under `app/`

## v1 scope (must ship for cutover)

These are the routes a logged-in user touches in the read + post flow of a typical forum visit. Everything below is required for v1 parity.

| Ember route | URL pattern | Next.js target | Phase | Notes |
|---|---|---|---|---|
| `discovery.index` | `/` | `app/(forum)/page.tsx` | 2 | Server Component, fetches `/latest.json` |
| `discovery.latest` | `/latest` | `app/(forum)/latest/page.tsx` | 2 | |
| `discovery.top` | `/top` | `app/(forum)/top/page.tsx` | 2 | |
| `discovery.unread` | `/unread` | `app/(forum)/unread/page.tsx` | 2 | requires session |
| `discovery.new` | `/new` | `app/(forum)/new/page.tsx` | 2 | requires session |
| `discovery.hot` | `/hot` | `app/(forum)/hot/page.tsx` | 2 | |
| `discovery.categories` | `/categories` | `app/(forum)/categories/page.tsx` | 2 | |
| `discovery.category` | `/c/*category_slug_path_with_id` | `app/(forum)/c/[...slug]/page.tsx` | 2 | catch-all segment |
| `discovery.categoryNone` | `/c/*slug/none` | `app/(forum)/c/[...slug]/none/page.tsx` | 2 | "no subcategory" filter |
| `topic` | `/t/:slug/:id` | `app/(forum)/t/[slug]/[id]/page.tsx` | 2→3 | read in Phase 2; reply/edit/delete in Phase 3 |
| `topicBySlugOrId` | `/t/:slug_or_id` | `app/(forum)/t/[slug_or_id]/page.tsx` | 2 | redirector |
| `tag.show` | `/tag/:tag_slug/:tag_id` | `app/(forum)/tag/[slug]/[id]/page.tsx` | 2 | |
| `tags.index` | `/tags` | `app/(forum)/tags/page.tsx` | 2 | |
| `users` | `/u` | `app/(forum)/u/page.tsx` | 2 | directory |
| `user.summary` | `/u/:username/summary` | `app/(forum)/u/[username]/summary/page.tsx` | 2 | |
| `user.userActivity.*` | `/u/:username/activity/...` | `app/(forum)/u/[username]/activity/[[...filter]]/page.tsx` | 2 | optional catch-all |
| `user.userNotifications.*` | `/u/:username/notifications/...` | `app/(forum)/u/[username]/notifications/[[...filter]]/page.tsx` | 3 | interactive |
| `user.preferences.*` | `/u/:username/preferences/...` | `app/(forum)/u/[username]/preferences/[[...tab]]/page.tsx` | 3 | writable |
| `badges.show` | `/badges/:id/:slug` | `app/(forum)/badges/[id]/[slug]/page.tsx` | 2 | |
| `full-page-search` | `/search` | `app/(forum)/search/page.tsx` | 2 | |
| `signup` | `/signup` | `app/(auth)/signup/page.tsx` | 3 | proxies form to Rails |
| `login` | `/login` | `app/(auth)/login/page.tsx` | 3 | |
| `forgot-password` | `/password-reset` | `app/(auth)/password-reset/page.tsx` | 3 | |
| `email-login` | `/session/email-login/:token` | `app/(auth)/session/email-login/[token]/page.tsx` | 3 | |
| `password-reset` | `/u/password-reset/:token` | `app/(auth)/u/password-reset/[token]/page.tsx` | 3 | |
| `activate-account` | `/u/activate-account/:token` | `app/(auth)/u/activate-account/[token]/page.tsx` | 3 | |
| `confirm-new-email` | `/u/confirm-new-email/:token` | `app/(auth)/u/confirm-new-email/[token]/page.tsx` | 3 | |
| `confirm-old-email` | `/u/confirm-old-email/:token` | `app/(auth)/u/confirm-old-email/[token]/page.tsx` | 3 | |
| `groups.index` | `/g` | `app/(forum)/g/page.tsx` | 2 | |
| `group.activity.*` | `/g/:name/activity/...` | `app/(forum)/g/[name]/activity/[[...tab]]/page.tsx` | 2 | |
| `group.members` | `/g/:name/members` | `app/(forum)/g/[name]/members/page.tsx` | 2 | |
| `about` | `/about` | `app/(forum)/about/page.tsx` | 2 | |
| `faq`, `tos`, `privacy`, `guidelines`, `conduct`, `rules` | `/faq` etc | `app/(forum)/[policy]/page.tsx` | 2 | static-ish, can be SSG |
| `new-topic`, `new-message`, `new-invite` | `/new-topic` etc | redirect to topic with composer open | 3 | |
| `exception`, `exception-unknown` | `/exception`, `/404` | `app/error.tsx`, `app/not-found.tsx` | 1 | App Router built-ins |

**v1 route count:** ~50 leaf routes (some collapse via dynamic segments).

## Deferred to v2 (admin)

Everything under `/admin/...` in the route map. Strategy: **keep the Ember admin app alive** behind a path-level proxy (Cloudflare Worker or Nginx) that routes `/admin/*` to the legacy Ember bundle while everything else hits the Next.js app. This is the only practical way to deliver v1 in 12 months.

| Ember route | URL pattern | v2 target | Notes |
|---|---|---|---|
| `adminDashboard` | `/admin` | proxy to legacy | dashboard, reports, plugins panel — all Ember |
| `adminUsers` | `/admin/users/*` | proxy to legacy | |
| `adminSiteSettings` | `/admin/site_settings/*` | proxy to legacy | ~1,500 settings, schema-driven |
| `adminCustomize.*` | `/admin/customize/*` | proxy to legacy | themes, components |
| `adminLogs.*` | `/admin/logs/*` | proxy to legacy | |
| `adminApiKeys` | `/admin/api/*` | proxy to legacy | |
| `adminBadges.*` | `/admin/badges/*` | proxy to legacy | |
| `wizard` | `/wizard` | proxy to legacy | first-run setup |

## Deferred to v2 (chat + plugin-owned routes)

The chat plugin (Discourse Chat) is a substantial app on its own — it has its own router, drawer UI, real-time channels, and a separate test suite. Treat chat as its own migration epic *after* v1.

| Ember route | URL pattern | v2 target |
|---|---|---|
| `chat.*` | `/chat/*` | proxy to legacy in v1; rewrite in v2 |
| `discourse-ai.*` | `/admin/plugins/discourse-ai/*` | proxy (admin-side) |
| Plugin-specific routes (40+) | `/plugins/<plugin>/*` | proxy in v1; case-by-case rewrite in v2 |

## Routes we will not implement (deprecate)

- Legacy `top` filter sub-routes (e.g. `/top/all`, `/top/yearly`) — keep as 301 redirects to `/top` with query params.
- Legacy `tag` route under `/tag/:tag_name` (no slug/id) — keep as 301 redirect.
- IE11-targeted routes — none in this codebase, but worth noting nothing in the new app should rely on `<noscript>` fallbacks for the same reason.

## Notes on App Router translation

- **`(forum)` route group** wraps the main app; provides shared `layout.tsx` (header, sidebar, footer), and is parallel to `(auth)` which has a stripped-down layout.
- **Catch-all segments** (`[...slug]`) replace Ember's `*category_slug_path_with_id` glob.
- **Optional catch-all** (`[[...filter]]`) handles routes where the leaf segment is optional (e.g. `/u/:username/activity` and `/u/:username/activity/topics`).
- **Server Components by default**. Mark client-interactive components with `"use client"` only at the leaf — header/sidebar shell stays server-rendered.
- **Streaming + Suspense** for the topic page: render the topic header immediately, stream posts in chunks. This is a strict win over the current Ember + Rails-server-rendered first-paint architecture for SEO and Time-to-First-Byte.
- **Parallel routes** (`@notifications`, `@composer`) for the persistent UI surfaces that overlay the main route — notifications panel, composer drawer, modal stack.

## Open questions

1. Should the v1 cutover preserve every legacy URL exactly (so external links keep working), or are 301 redirects acceptable for some patterns (e.g. `/t/:slug_or_id` → canonical `/t/:slug/:id`)? Recommendation: preserve exactly, redirects are a regression risk for SEO.
2. The `discovery` route uses runtime-derived periods + filters (`Site.currentProp("periods")`, `Site.currentProp("filters")`) to register routes dynamically. App Router is static — we have to enumerate `top`, `latest`, `new`, `unread`, `hot`, `categories` explicitly. Confirm this list is complete (currently sourced from `frontend/discourse/app/lib/discovery-filter.js`).
3. Some routes have route-level `model()` hooks that mutate global state (e.g., `application.applyTransition()`). These need explicit equivalents in the loader pattern; this is captured in ADR-0006.

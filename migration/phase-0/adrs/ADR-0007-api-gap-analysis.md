# ADR-0007 — API gap analysis: Rails-side endpoints to add or modify

**Status:** Proposed
**Date:** 2026-04-28
**Decision owner:** Backend lead
**Depends on:** ADR-0001, ADR-0002, ADR-0005

## Context

The Ember app and the Rails backend are co-developed in this repo. Many "API endpoints" the Ember app calls are not part of Discourse's [public REST API](https://docs.discourse.org/) — they are private endpoints designed for the Ember app's specific needs (e.g., compound responses that merge what would otherwise be 3 calls; endpoints that respond differently based on `Accept: application/json` vs `text/html`).

The Next.js app needs to consume these endpoints. Three failure modes:

1. **Endpoint exists, response shape is fine** — no Rails work needed. Most endpoints fall here.
2. **Endpoint exists, response shape is too coupled to Ember internals** — Rails needs minor refactoring (e.g., adding fields, separating concerns) to make the response reusable.
3. **Endpoint doesn't exist** — the Ember app does the work client-side that the Next.js app would prefer to push server-side. Rails needs new endpoints.

This ADR enumerates the Rails-side work for v1.

## Method

A static scan of `frontend/discourse/app/` for `ajax(...)` and `fetch(...)` calls produced 504 unique `.json` endpoint paths (`inventory/all-json-endpoints.txt`) and 306 non-`.json` paths. Filtering out admin and plugin-specific paths leaves 313 v1-relevant endpoints (`inventory/v1-json-endpoints.txt`). I then walked the v1 list looking for failure modes 2 and 3.

## Findings — endpoints that need Rails-side work for v1

### Category 1 — Add SSR-friendly response variant

The Ember app calls these endpoints with `Accept: application/json` and gets back a payload optimized for hydration. The Next.js Server Components want the same payload but **without the bits that assume Ember-side hydration** (e.g., embedded route URLs, embedded `data-ember-action` strings).

| Endpoint | Issue | Rails work |
|---|---|---|
| `GET /t/:id.json` | Includes `actions_summary`, `cooked` post HTML, `details.suggested_topics`. Generally fine. **But** `cooked` HTML contains `data-ember-action` attributes for some plugins. | Strip `data-ember-action` server-side when `Accept: application/json` and `User-Agent` is not Ember. ~1 day. |
| `GET /c/:slug/:id.json` | Includes `topic_list` with full topic objects. Fine. | None. |
| `GET /categories.json` | Includes both `category_list.categories` and `topic_list`. The Ember app deduplicates client-side; the Next.js app would prefer separate endpoints. | Optionally split into `/categories.json` (just categories) and `/categories/topic_list.json` (topics by category). ~3 days. Optional optimization, not v1-blocking. |
| `GET /u/:username.json` | Heavy payload — includes user, badges, top categories, custom fields, recent posts, summary stats. Can be 50KB+. | Add a `?fields=` query parameter to fetch only requested fields. ~2 days. Performance optimization for the profile route. |
| `GET /search.json` | Returns posts, topics, users, categories, tags, groups in one response. Fine for a search-results page. | None. |

### Category 2 — Cookie-only auth → token-or-cookie auth

The Ember app authenticates via cookie. The Next.js app's Server Components also authenticate via cookie (per ADR-0004), but the **proxied calls from Vercel/Cloudflare to Rails** need to forward the cookie correctly. This is a cookie-domain concern, not a Rails-code concern, but worth flagging:

| Endpoint | Issue | Rails work |
|---|---|---|
| (any authed endpoint) | `_t` cookie must be readable by both subdomains. | Verify `Rails.application.config.session_store domain: :all` is in effect; verify Discourse's `cors_origins` setting includes the Next.js subdomain. ~1 day SRE/config. |

### Category 3 — Endpoints that return HTML, not JSON, in some flows

| Endpoint | Issue | Rails work |
|---|---|---|
| `POST /session.json` | On success, returns JSON. On 2FA-required, returns JSON. **On legacy redirect flows, sometimes returns 302 to a Rails-rendered page.** | Force JSON response for any request with `Accept: application/json`. Already mostly the case; verify edge cases (passkey, SSO providers). ~2 days. |
| `POST /uploads.json` | Returns `307` redirect to S3 in some configurations; the Ember app handles this. | None — Next.js Server Action follows the redirect transparently. |
| `GET /session/csrf.json` | Already JSON. | None. |

### Category 4 — Endpoints that don't exist (Rails would need to add them)

| Endpoint | Why we want it | Rails work |
|---|---|---|
| `GET /t/:id/posts.json?after=:id&limit=20` | The Ember app fetches posts incrementally as the user scrolls. The current API returns *all* posts in `GET /t/:id.json` and the Ember app virtualizes client-side. For SSR streaming, we want explicit pagination. | Add a `posts.json` sub-resource on topic. ~1 week. |
| `GET /site/lite.json` | The Ember app calls `/site.json` (~150KB) at boot. Server Components want a smaller "just the public bits" response that's safe to cache widely. | Add a stripped-down `/site/lite.json` response. ~3 days. |
| `POST /next/server-event` (synthetic) | When Server Actions need to invalidate Server Component caches, they call back to Next.js's `revalidateTag`. **No Rails work** — this is internal to Next.js. | None. |
| `GET /translations/:locale.json` | Already exists in Discourse for the Ember app. Confirm the Next.js subdomain is allowed. | Verify `cors_origins`. ~1h. |
| `GET /draft.json?username=...&draft_key=...` | Already exists; just needs better doc + stable response shape for v1. | Doc + spec. ~2 days. |

### Category 5 — MessageBus

The Discourse `MessageBus` server is part of the Rails app. The Next.js app subscribes via the [`message-bus-client`](https://www.npmjs.com/package/message-bus-client) npm package. **No Rails work**, except verifying the `MESSAGEBUS_BASE_URL` site setting includes the Next.js subdomain in CORS allowlist.

### Category 6 — Server Actions → Rails write proxy

For each Server Action in the Next.js app that performs a write, we proxy to a Rails endpoint with the user's cookie + CSRF token forwarded. **No Rails work for the endpoints themselves** (they already exist for the Ember app), but we want to confirm:

- Rails accepts `application/json` request bodies on every write endpoint (some endpoints historically only accepted `application/x-www-form-urlencoded`).
- Rails rate limiting (`RackAttack`) covers requests originating from Vercel/Cloudflare egress IPs (otherwise we'd see "all requests from one IP" → unintended throttling).

### Category 7 — Plugin endpoints

For plugins ported in v1 (`hcaptcha`, `local-dates`, `presence`, plus the Product-decided subset of `reactions`/`solved`/`post-voting`/etc.):

| Plugin | Endpoint surface | Rails work |
|---|---|---|
| `discourse-hcaptcha` | None (server validates captcha as part of `/session.json`) | None |
| `discourse-local-dates` | None (cook-time HTML rendering) | None |
| `discourse-presence` | MessageBus channels `/discourse-presence/...` | Verify CORS for Next.js |
| `discourse-reactions` | `POST /discourse-reactions/posts/:post_id/custom-reactions/:reaction/toggle` | None |
| `discourse-solved` | `PUT /solution/accept`, `PUT /solution/unaccept` | None |
| `discourse-post-voting` | `POST /post_voting/vote`, `DELETE /post_voting/vote` | None |

## Summary of Rails-side work for v1

| Bucket | Endpoints | Rough effort |
|---|---|---|
| Category 1 (response shape) | 5 | ~10 days |
| Category 2 (cookie/CORS config) | n/a | ~1 day |
| Category 3 (HTML→JSON forcing) | 1 | ~2 days |
| Category 4 (new endpoints) | 5 | ~3 weeks |
| Category 5 (MessageBus CORS) | n/a | ~1h |
| Category 6 (write-proxy verify) | n/a | ~3 days (audit) |
| Category 7 (plugin endpoints) | varies by plugin disposition | per-plugin |

**Total Rails work for v1: ~5–6 engineer-weeks**, plus per-plugin work depending on the Product decision in ADR-0002.

## Tickets to open (after this ADR is approved)

1. `[Backend] Strip Ember-specific attributes from cooked HTML when Accept: application/json` — owner: Backend lead
2. `[Backend] Add /site/lite.json` — owner: Backend lead
3. `[Backend] Add /t/:id/posts.json paginated sub-resource` — owner: Backend lead
4. `[Backend] Force JSON response on /session.json edge cases (passkey, SSO)` — owner: Backend lead
5. `[Backend] Audit write endpoints for application/json body acceptance` — owner: Backend lead
6. `[SRE] Verify session cookie domain scope and CORS allowlist for next.forum.example.com` — owner: SRE
7. `[SRE] Add Cloudflare Worker for path-level proxy (Next.js / legacy Ember split)` — owner: SRE
8. `[Backend] Open RFC for OpenAPI codegen story (v2 work)` — owner: Backend lead

## Open issues

- **MiniRacer / Asset-Processor relationship.** The Discourse asset-processor uses MiniRacer (V8 in Ruby) to transpile JS at boot. None of that is reachable from the Next.js app, but it ships as part of Rails. Don't accidentally remove it during the Ember sunset in Phase 5.
- **Unicorn/Pitchfork tuning.** The Rails app may see traffic-pattern shifts when Server Components do the heavy reading (more concentrated bursts). Plan a load-test pass before v1 cutover.

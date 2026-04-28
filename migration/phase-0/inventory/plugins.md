# Plugin inventory — disposition for v1

**Source:** `plugins/*/` directories (43 plugins, 42 with frontend code, ~1,720 frontend files total)

## Definitions

- **Drop** — the plugin's user-facing behavior is not required for v1. The plugin is removed from the Next.js app. The Rails side may continue to load the plugin if it has server-side value (background jobs, API), but the React frontend has no UI for it.
- **Port (v1)** — the plugin's user-facing behavior is required for v1. We will reimplement the client-side surface in React/Next.js. Server-side stays as-is.
- **Defer (v2)** — the plugin is required eventually but not at v1 cutover. The legacy Ember admin shell or a Worker-level path proxy keeps it alive in the interim.
- **Server-only** — the plugin has no client-facing surface that matters; the React app does not need to do anything for it.

## Disposition matrix

| Plugin | Frontend size | v1 disposition | Rationale |
|---|---|---|---|
| `automation` | small | Defer (v2) | Admin-only UI |
| `chat` | **very large** (own SPA) | Defer (v2) | Massive scope on its own; treat as a separate epic |
| `checklist` | small | Drop or Port | Renders interactive checkboxes inside cooked posts. Post-cooking happens server-side; the JS just wires up onclick → API. **If posts in your forum use checklists, port.** |
| `discourse-adplugin` | medium | Drop | Ad-injection logic; reconsider for v2 monetization |
| `discourse-affiliate` | small | Drop | URL rewriting; can move server-side |
| `discourse-ai` | medium-large | Defer (v2) | Admin config + composer UI; defer composer-AI to v2, keep admin proxied |
| `discourse-apple-auth`, `-microsoft-auth`, `-login-with-amazon`, `-lti`, `-oauth2-basic`, `-openid-connect` | small each | Server-only | Auth-button rendering is on the login page; covered by ADR-0004. Server-side OAuth flows stay untouched |
| `discourse-assign` | medium | Defer (v2) | Staff-only feature; not in v1 scope |
| `discourse-cakeday` | small | Drop | Cosmetic; can revive in v2 |
| `discourse-calendar` | medium | Drop or Port | Has both content-rendering (in posts, via cooking) and standalone routes (`/upcoming-events`). Server-side cooking handles most of it; standalone routes can defer |
| `discourse-chat-integration` | tiny (admin only) | Defer (v2) | Admin-only UI |
| `discourse-data-explorer` | medium (admin) | Defer (v2) | Admin-only |
| `discourse-details` | tiny | Server-only | Just a `<details>` HTML rewriter at cook time |
| `discourse-gamification` | medium | Drop or Port | Leaderboard route (`/leaderboard`) — port if business cares |
| `discourse-github` | small | Server-only | Onebox-style rendering, server-side |
| `discourse-graphviz` | small | Server-only | Cook-time SVG rendering |
| `discourse-hcaptcha` | small | Port (v1) | Login/signup captcha — required for v1 auth flows |
| `discourse-lazy-videos` | small | Server-only | Cook-time YouTube/Vimeo lazy embeds |
| `discourse-local-dates` | medium | Port (v1) | Renders timezone-aware dates inside posts; widely used. Port as a small `<LocalDate>` Server Component + client hydration |
| `discourse-math` | medium | Server-only | KaTeX/MathJax cook-time rendering |
| `discourse-narrative-bot` | small | Defer (v2) | New-user onboarding; can re-enable in v2 |
| `discourse-patreon` | small | Drop | Niche; revive only if needed |
| `discourse-policy` | medium | Defer (v2) | Posts that require user acknowledgement; specialized |
| `discourse-post-voting` | medium | Drop or Port | Stack-Overflow-style voting per post. **Port if your forum uses this voting model.** |
| `discourse-presence` | medium | Port (v1) | "X is typing..." indicators in topic + composer. Required for v1 if you keep that UX |
| `discourse-reactions` | large | Drop or Port | Emoji reactions on posts. **Port if your forum uses reactions.** Otherwise keep just the standard "like." |
| `discourse-rewind` | medium | Drop | Year-in-review feature |
| `discourse-rss-polling` | tiny | Server-only | Background job, no UI |
| `discourse-solved` | small | Port (v1) | "Mark as solution" — common Q&A feature; required if your forum uses it |
| `discourse-subscriptions` | medium | Defer (v2) | Stripe-based; re-evaluate for v2 monetization |
| `discourse-templates` | small | Defer (v2) | Composer reply templates; admin-side mostly |
| `discourse-topic-voting` | medium | Drop or Port | Topic-level upvoting; port if used |
| `discourse-user-notes` | small | Defer (v2) | Staff-only |
| `discourse-zendesk-plugin` | small | Defer (v2) | Admin-only integration |
| `footnote` | small | Server-only | Cook-time markdown footnote rendering |

## Categorization summary

- **Port for v1:** `discourse-hcaptcha`, `discourse-local-dates`, `discourse-presence`, plus *any* of [`checklist`, `discourse-post-voting`, `discourse-reactions`, `discourse-solved`, `discourse-topic-voting`] that are actively used by the forum's topic content. **The decision on which of those to port is a product call**, not a technical one — they're each ~3–10 engineer-weeks.
- **Defer to v2:** all admin-only plugins, `chat`, `discourse-ai`, `discourse-assign`, `discourse-data-explorer`, `discourse-policy`, `discourse-narrative-bot`, `discourse-templates`, `discourse-user-notes`, `automation`, `discourse-zendesk-plugin`, `discourse-chat-integration`, `discourse-subscriptions`. These ride on the legacy Ember admin/chat shell behind a path proxy until v2.
- **Drop entirely (or pending business case):** `discourse-adplugin`, `discourse-affiliate`, `discourse-cakeday`, `discourse-calendar`, `discourse-gamification`, `discourse-patreon`, `discourse-rewind`. Most of these can come back in v2 if there's a business case.
- **Server-only (zero React work):** all the cook-time rendering plugins (`discourse-details`, `discourse-github`, `discourse-graphviz`, `discourse-lazy-videos`, `discourse-math`, `footnote`, `discourse-rss-polling`) plus the OAuth-provider plugins. The Rails backend keeps cooking posts to HTML; the React app just renders the HTML. **No work required.**

## Action required from the business

The following plugins each have a binary "port for v1 or drop" decision that depends on whether the forum's content actually uses them. **Before Phase 1 starts, run a quick analytics query on production to determine usage.**

| Plugin | Question |
|---|---|
| `checklist` | What % of posts contain `[ ]` / `[x]` markdown checkboxes? |
| `discourse-post-voting` | How many topics use the "post voting" tag/category? |
| `discourse-reactions` | What % of posts have at least one reaction? |
| `discourse-solved` | What % of topics have an accepted-answer post? |
| `discourse-topic-voting` | How many topics live in voting-enabled categories? |
| `discourse-calendar` | How many posts contain `[date]` / event markup? |
| `discourse-gamification` | How many users visit `/leaderboard` per week? |
| `discourse-local-dates` | What % of posts contain `[date]` markdown? |

If usage is < 1% of relevant content for any of these, drop. Otherwise, port.

## Files

The raw inventory of plugin frontend files (1,720 across 42 plugins) is in the repo at `plugins/*/assets/javascripts/` and `plugins/*/test/`. The raw counts per-plugin are not committed here because they are easily re-derivable; if needed, run:

```
find plugins -maxdepth 4 -type f \( -name "*.js" -o -name "*.gjs" -o -name "*.hbs" \) | awk -F/ '{print $2}' | sort | uniq -c | sort -rn
```

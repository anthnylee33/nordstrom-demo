# ADR-0005 — Data layer and real-time strategy

**Status:** Proposed
**Date:** 2026-04-28
**Decision owner:** Engineering lead
**Depends on:** ADR-0001, ADR-0003

## Context

The Ember app uses:

- **Ember Data** as a client-side ORM/store (~56 models, ~50 adapters under `frontend/discourse/app/models/`). Records are loaded via REST adapters, normalized into a global store, and observed by templates via auto-tracking.
- **`ajax(...)`** (a thin wrapper over `jQuery.ajax`) for one-off REST calls. ~504 unique `.json` endpoints + 306 non-`.json` endpoints — see `inventory/all-json-endpoints.txt`.
- **MessageBus** as a long-poll-based pub/sub for real-time updates. The Ember app subscribes to 16 channels (`inventory/messagebus-channels.txt`) for things like new posts in a topic, notification badge updates, sidebar refreshes, etc.

We need to replace all three in the Next.js app.

## Decision — data layer

**Server Components by default, React Query for client interactivity, no Ember-Data port.**

### Server Components for read-only forum surfaces

Phase 2 routes (latest, top, categories, topic view, profile, search, tags) are read-mostly. They become Server Components that call the Discourse REST API directly via `fetch`:

```ts
// app/(forum)/t/[slug]/[id]/page.tsx
export default async function TopicPage({ params }: { params: { slug: string; id: string } }) {
  const topic = await getTopic(params.id);
  return <TopicView topic={topic} />;
}

async function getTopic(id: string): Promise<Topic> {
  const res = await fetch(`${process.env.DISCOURSE_API_BASE_URL}/t/${id}.json`, {
    headers: forwardSessionHeaders(),
    next: { revalidate: 60, tags: [`topic:${id}`] },
  });
  if (!res.ok) notFound();
  return res.json();
}
```

Cache layer is Next.js's built-in fetch cache, keyed by URL + headers, with `revalidateTag` for invalidation. Topic page is tagged `topic:<id>` so MessageBus events on `/topic/<id>` can `revalidateTag("topic:<id>")` server-side and re-stream the page.

### React Query for client interactivity

Phase 3 surfaces (composer, draft autosave, notifications dropdown, "new posts" indicator, presence, likes/bookmarks) need a client-side cache that responds to user interaction faster than Server Component re-fetches. Use `@tanstack/react-query` v5:

- Query keys mirror Discourse model identifiers: `["topic", id]`, `["user", username]`, `["notifications", userId]`.
- Mutations are Server Actions in Next.js (typed with Zod) that proxy to Rails and invalidate the relevant query keys.
- Optimistic updates for low-stakes interactions (like, bookmark, react). Pessimistic confirmation for high-stakes (delete, flag, edit-other-user).

### Hand-written types, codegen later

For v1, hand-write TypeScript types in `src/lib/api/types.ts` for the ~50 models. This is faster than building a codegen pipeline that introspects Rails serializers.

For v2, evaluate options for typed API contracts:
- **`rails_event_store`-style typed events** — overkill
- **`graphql-ruby` + `graphql-codegen`** — would require introducing GraphQL on Rails (~3 months)
- **OpenAPI spec generated from Rails serializers + `openapi-typescript`** — most pragmatic; ~4 weeks

### What we're not doing

- **Not porting Ember Data.** Ember Data's identity-map / unloadAll / peekRecord semantics do not map cleanly to React Query. Most call sites in the Ember app that lean on Ember Data's identity map are doing it as a workaround for Ember template re-render rules; React doesn't need that workaround.
- **Not standing up an API gateway.** The Next.js app talks directly to Rails. Adding a gateway (Kong, Apollo Server, etc.) is an extra hop and an extra failure mode for v1.
- **Not introducing a new state-management library.** No Redux, Zustand, Jotai, MobX, etc. for v1. Server Components + React Query + a few Context providers cover the surface. If we hit a wall in Phase 3 we re-evaluate.

## Decision — real-time

**Keep MessageBus. Wrap it in a thin Next.js adapter; do not migrate to a different real-time technology.**

### Why keep MessageBus

- It's battle-tested in production for a decade. Replacing it for the sake of replacing it is a v2-or-later concern.
- The npm package [`message-bus-client`](https://www.npmjs.com/package/message-bus-client) is the official client. It works in any browser environment, including Next.js.
- The 16 channels we care about are well-understood; reproducing the same semantics in Pusher/Ably/SSE means re-implementing the server side, which we explicitly chose not to do (Strategy C-style backend reuse).

### Adapter shape

```ts
// src/lib/realtime/messageBus.ts
"use client";
import MessageBus from "message-bus-client";

MessageBus.baseUrl = process.env.NEXT_PUBLIC_DISCOURSE_API_BASE_URL!;
MessageBus.start();

export function useMessageBusSubscription<T>(
  channel: string,
  handler: (payload: T) => void,
) {
  useEffect(() => {
    const sub = MessageBus.subscribe(channel, handler as (data: unknown) => void);
    return () => MessageBus.unsubscribe(channel, sub);
  }, [channel, handler]);
}
```

### Server-Component invalidation pattern

When a MessageBus event indicates that a Server-Component-rendered surface has stale data (e.g., `/topic/<id>` event signaling new posts), the client receives the event, then the Server Component is re-fetched **only if the user is currently viewing that surface**. Two implementations:

- **Naive:** the client receives the event and `router.refresh()`s. App Router re-streams the current route.
- **Targeted:** the client receives the event, calls a Next.js Server Action that calls `revalidateTag("topic:<id>")`, then `router.refresh()`s. Avoids re-fetching unrelated routes.

For v1, ship the naive pattern. Optimize to targeted only if it shows up in profiling.

### Channels to subscribe in v1

From `inventory/messagebus-channels.txt`:

| Channel | Phase | Action |
|---|---|---|
| `/categories` | 2 | Refresh sidebar/category tree |
| `/site/banner` | 2 | Show admin-set site banner |
| `/site/read-only` | 1 | Show read-only mode notice |
| `/global/asset-version` | 1 | Show "new version available, refresh" prompt |
| `/refresh-sidebar-sections` | 2 | Re-render sidebar |
| `/refresh_client` | 1 | Force-reload (rare; admin tool) |
| `/u/${username}` | 3 | User-specific notifications + presence |
| `/client_settings` | 2 | Live site-setting updates |

The other 8 channels in the inventory are admin/staff/plugin-specific and are handled by the proxied legacy Ember app.

## Caching policy

| Surface | Cache | TTL | Invalidation |
|---|---|---|---|
| `/site.json` | Next.js fetch cache + `unstable_cache` | 1 hour | `revalidateTag("site")` on admin change |
| `/categories.json` | Next.js fetch cache | 5 min | MessageBus `/categories` |
| `/t/<id>.json` | Next.js fetch cache, tagged | 60s for SEO crawlers, 0 for logged-in users | MessageBus topic-channel events |
| `/c/<slug>.json` | Next.js fetch cache, tagged | 60s | MessageBus `/categories` |
| `/u/<username>.json` | Next.js fetch cache, tagged | 5 min | User edit MessageBus event |
| `/notifications.json` | React Query, 30s stale time | — | MessageBus `/u/<username>` |
| `/search.json?q=...` | React Query | 0 (always re-fetch) | — |

**Logged-in users get 0-second TTL on personalizable content** (read-state markers, unread-count badges, draft indicators). Logged-out users get the full SEO-friendly cache. The cache key includes `Authorization`-equivalent headers so the bucket is correctly partitioned.

## Open issues

- **Long-poll vs WebSocket.** MessageBus is HTTP long-poll by default. For very large deployments, Discourse has a `MessageBus::Backends::Redis` config that supports WebSocket. Decide before v1 whether to upgrade; not blocking.
- **Read-state synchronization.** Discourse tracks read-state per user per topic per post. The Ember app maintains this on the client and POSTs back to `/topics/timings`. This needs careful porting in Phase 3 — get it wrong and "unread" badges become unreliable.
- **Composer drafts.** Drafts are auto-saved every few seconds via `/drafts.json`. React Query mutations + a debounce are the natural fit. Watch out for race conditions on rapid input.

# Service inventory — Ember services → React replacements

**Source:** `frontend/discourse/app/services/` (53 services)

In Ember, a *service* is a singleton injected via `@service` into routes/components. The closest React equivalents are: a Context provider for app-wide state, a custom hook for component-local state, or a server-side singleton living in `app/lib/` for build/runtime utilities. Some Ember services don't need a React equivalent at all — they're better expressed as Server Components reading directly from cookies/headers.

## Mapping table

| Ember service | React replacement | Phase | Notes |
|---|---|---|---|
| `app-events.js` | EventEmitter singleton in `lib/app-events.ts` | 1 | Decoupled pub/sub; keep API mostly identical for porting plugins later |
| `composer.js` | `composer/ComposerProvider` Context + `useComposer` hook | 3 | The biggest single piece of v1 work |
| `current-user`, `session` | `lib/auth/getSession()` Server Component helper + `SessionProvider` Context | 1 | Source of truth is Rails; client cache for UI hints |
| `site` | `lib/site/getSiteConfig()` cached `fetch('/site.json')` | 1 | Cached at request scope via `unstable_cache` |
| `site-settings` | `lib/site/getSiteSettings()` (subset of `/site.json`) | 1 | Read-only on client; admin writes go through `/admin/...` (proxied to Ember in v1) |
| `store` (Ember Data store) | `@tanstack/react-query` `QueryClient` + REST adapters in `lib/api/` | 1 | Replace Ember Data, do not port. Schemas in `lib/types/`. |
| `message-bus` | `lib/realtime/messageBus.ts` thin adapter over the existing JS lib | 1 | The Discourse `MessageBus` JS client (`message-bus-client` npm package) works fine in Next.js |
| `notifications` | `useNotifications()` hook backed by React Query + MessageBus | 3 | Real-time |
| `presence` | `usePresence(channel)` hook backed by MessageBus | 3 | "X is typing..." |
| `desktop-notifications` | `useDesktopNotifications()` hook | 3 | Just wraps the Notification API |
| `network-connectivity` | `useNetworkConnectivity()` hook over `navigator.onLine` | 1 | |
| `keyboard-shortcuts` | `useKeyboardShortcuts()` hook + global `<KeyboardShortcutsProvider>` | 2 | |
| `screen-track` | Server Action `trackTopicView` invoked from client | 2 | Posts read-state to Rails |
| `header`, `footer`, `sidebar-state`, `breadcrumbs`, `navigation-menu` | each becomes a Context provider in the corresponding layout component | 2 | UI shell state |
| `modal` | `useModal()` hook + `<ModalProvider>`; render via `<Slot>` in root layout | 2 | Replace Ember modal stack with `react-aria-components` Dialog or `radix-ui` Dialog |
| `loading-slider` | Built-in Next.js `loading.tsx` files + a thin streaming bar | 2 | Native to App Router |
| `route-history`, `route-scroll-manager`, `scroll-direction`, `scroll-manager`, `scroll-state` | Mostly handled by the App Router + `next/navigation` `useRouter` and a single `useScrollRestoration()` hook | 1 | App Router has built-in scroll restoration; only need a small custom layer for "scroll to post N" deep-linking |
| `search`, `search-preferences-manager` | `useSearch()` hook + Server Component for the `/search` page | 2 | |
| `bookmark-api` | server-side `lib/api/bookmarks.ts` + Server Action wrappers | 3 | |
| `emoji-store` | `lib/emoji/store.ts` build-time generated lookup table | 1 | Static data — generate at build time, not runtime |
| `discovery` | not needed — discovery state lives in URL (`searchParams`) and Server Components re-fetch | 2 | Eliminated by SSR |
| `pm-topic-tracking-state`, `more-topics-tabs` | hooks consuming MessageBus + React Query cache invalidations | 3 | |
| `key-value-store` | `lib/storage/kv.ts` thin wrapper around `localStorage` w/ SSR safety | 1 | |
| `session-store` | `lib/storage/session.ts` thin wrapper around `sessionStorage` | 1 | |
| `nested-view-cache` | not needed — App Router caches Server Component renders, React Query caches client data | — | Eliminated |
| `map-cache` | `lib/cache/lru.ts` if needed; most usages eliminated by React Query | — | Audit per call site |
| `restricted-routing` | Next.js middleware (`middleware.ts`) checking `/login-required` paths | 2 | |
| `client-error-handler`, `deprecation-warning-handler` | Next.js `error.tsx` + `react-error-boundary` + Sentry SDK | 1 | New-tech stack, not a port |
| `logs-notice` | admin-only — defer to v2 (proxied admin) | v2 | |
| `media-optimization-worker` | Web Worker module under `app/workers/` | 3 | Image upload optimization; only needed when composer ships |
| `interface-color`, `element-classes` | CSS variables + `useTheme()` hook | 2 | See ADR-0006 |
| `a11y` | hooks + `react-aria-components` | 1 | Use a11y library instead of custom service |
| `capabilities` | `lib/capabilities.ts` server-side read of `request.headers` | 1 | UA detection |
| `language-name-lookup` | static JSON imported at build time | 1 | |
| `restricted-routing` | Next.js middleware | 2 | |
| `user-status`, `user-tips` | lightweight Context providers | 2 | |
| `presence` | MessageBus-backed hook | 3 | |
| `document-title` | `next/head` / `metadata` exports per route | 1 | Native to App Router |
| `admin-custom-user-fields`, `admin-post-menu-buttons`, `admin-topic-menu-buttons` | not needed — admin proxied to Ember in v1 | v2 | |
| `category-type-chooser`, `group-automatic-members-dialog` | local component state, not services | 2 | |
| `blocks` | n/a — internal Discourse API for content rendering; covered by Server Component cooked-HTML rendering | 2 | |

## Themes

- **Roughly 40% of services** become **React Context providers** (anything app-wide and stateful).
- **Roughly 30%** become **plain library modules** (`lib/...`) — utility functions and singletons.
- **Roughly 20%** become **custom hooks** (anything that's per-component-instance + reactive).
- **Roughly 10% disappear entirely** because their concern is handled natively by the App Router, React Query, or React 19 Server Components.

## Open issues

- **Plugin-injected services** — Discourse plugins inject services via `withPluginApi`. The Next.js app needs an explicit plugin-extension API; for v1, since most plugins are deferred, just stub the hook points and document the contract for v2.
- **`store` (Ember Data) port** — Ember Data has `unloadAll`, `peekRecord`, etc. The closest React Query equivalents are `queryClient.removeQueries` + `queryClient.getQueryData`. Some call sites will need rewrites, not 1:1 translations. Audit during Phase 2 per route.

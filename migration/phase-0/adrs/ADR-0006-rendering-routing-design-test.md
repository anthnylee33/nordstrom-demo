# ADR-0006 — Rendering, routing, design system, and test strategy (combined)

**Status:** Proposed
**Date:** 2026-04-28
**Decision owner:** Engineering lead + Design
**Depends on:** ADR-0001, ADR-0003

This ADR bundles four decisions that are tightly coupled. They could be split into 4 ADRs but a coordinated proposal is more useful for review.

## 1. Rendering — Next.js 15 App Router with Server Components by default

### Decision

- **Framework:** Next.js 15 App Router. Stay on the App Router, do not use the Pages Router.
- **React version:** React 19 (current stable when Next.js 15 shipped). Server Components mandatory.
- **Default rendering mode:** Server Component for every route. Mark `"use client"` only at the leaf of an interactive component.
- **Streaming:** opt into `<Suspense>` boundaries on the topic page to stream posts in chunks. First post + topic header should hit the wire under 200ms TTFB.
- **Static generation:** SSG for the static-policy pages (`/about`, `/faq`, `/tos`, `/privacy`, `/guidelines`, `/conduct`, `/rules`). Everything else is dynamic (cached, but server-rendered per request).
- **Edge vs Node:** start with Node runtime (default). Move specific routes to the Edge runtime later only if proven faster *and* compatible with the auth middleware (some OAuth flows need Node-only crypto).

### Why App Router

- Server Components are the v1 win — they're the entire reason this rewrite is justifiable from a performance/SEO angle.
- File-system routing maps cleanly to Discourse's URL space (see `inventory/routes.md`).
- Streaming + Suspense + parallel routes are the right primitives for a topic page (header → posts → comments → related topics).
- The Pages Router is in maintenance mode; starting on it would mean a second migration in 12 months.

### Why not the alternatives

- **Remix** — viable but smaller ecosystem; the user requested Next.js explicitly.
- **Astro** — great for static-mostly sites; not the right shape for a forum.
- **TanStack Start** — too new for v1.
- **Pages Router** — see above.

## 2. Routing — file-system structure

### Decision — top-level structure

```
nordstrom-demo-web/
├── src/
│   ├── app/
│   │   ├── (forum)/                 ← main app, full layout
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              ← /
│   │   │   ├── latest/page.tsx
│   │   │   ├── top/page.tsx
│   │   │   ├── unread/page.tsx
│   │   │   ├── new/page.tsx
│   │   │   ├── categories/page.tsx
│   │   │   ├── c/[...slug]/page.tsx
│   │   │   ├── t/[slug]/[id]/page.tsx
│   │   │   ├── tag/[slug]/[id]/page.tsx
│   │   │   ├── tags/page.tsx
│   │   │   ├── search/page.tsx
│   │   │   ├── u/[username]/...
│   │   │   ├── g/[name]/...
│   │   │   ├── badges/[id]/[slug]/page.tsx
│   │   │   ├── about/page.tsx
│   │   │   └── (policy)/{faq,tos,privacy,guidelines,conduct,rules}/page.tsx
│   │   ├── (auth)/                   ← stripped layout
│   │   │   ├── layout.tsx
│   │   │   ├── login/page.tsx
│   │   │   ├── signup/page.tsx
│   │   │   ├── password-reset/page.tsx
│   │   │   ├── session/email-login/[token]/page.tsx
│   │   │   └── u/{password-reset,activate-account,confirm-new-email,confirm-old-email}/[token]/page.tsx
│   │   ├── error.tsx
│   │   ├── not-found.tsx
│   │   └── api/
│   │       └── (Server Actions live colocated; explicit /api/ routes only for webhook endpoints)
│   ├── components/
│   │   ├── topic/
│   │   ├── post/
│   │   ├── composer/
│   │   ├── header/
│   │   ├── sidebar/
│   │   ├── ui/                       ← design system primitives
│   │   └── icons/
│   ├── lib/
│   │   ├── api/                      ← Discourse REST client + types
│   │   ├── auth/                     ← session helpers
│   │   ├── realtime/                 ← MessageBus adapter
│   │   ├── cache/
│   │   ├── markdown/                 ← cooked-HTML rendering helpers
│   │   ├── i18n/                     ← translation loader
│   │   └── utils/
│   ├── hooks/
│   ├── workers/                      ← Web Workers (image optimization, etc.)
│   ├── styles/
│   └── middleware.ts
├── public/
├── tests/
│   └── e2e/                          ← Playwright
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── eslint.config.js
├── vitest.config.ts
├── playwright.config.ts
└── package.json
```

### Decision — proxy / cutover layer

- **Cloudflare Worker** in front of `forum.example.com` that path-routes:
  - `/admin/*` → legacy Ember origin
  - `/chat/*` → legacy Ember origin
  - plugin-owned paths (per ADR-0002) → legacy Ember origin
  - everything else → Next.js origin (Vercel)
- The Worker also handles the `_t` cookie scope (see ADR-0004) so session works on both apps.
- **Health checks:** the Worker has a `/.cutover-status` endpoint that reports which paths route where. Used by SREs during cutover.

## 3. Design system — Tailwind 4 + CSS variables + headless component primitives

### Decision

- **CSS:** **Tailwind 4** (utility-first). Matches the team's existing skill set in `anthnylee33/sushi`. CSS variables (`--color-primary`, `--font-body`, etc.) drive theming.
- **Component primitives:** **`react-aria-components`** for accessibility-first headless primitives (Dialog, Menu, ComboBox, Toast). Style with Tailwind. Avoid `shadcn/ui` because we want a more opinionated a11y story for a public-facing forum.
- **Icons:** import from `lucide-react` (matches what most React-shop UI libs use).
- **Typography & spacing tokens:** CSS variables defined in `src/styles/tokens.css`; consumed by Tailwind via `@theme`.
- **No theming system in v1.** The design system is fixed. Users do not get to pick light/dark, and they do not get to install themes. v2 reintroduces light/dark via `prefers-color-scheme` + a manual override; v3+ reconsiders user-installed themes.

### Why this stack

- Tailwind 4 is fast (Oxide engine, Rolldown bundler — same stack used in `anthnylee33/sushi`); matches existing org expertise.
- `react-aria-components` ships with WCAG-compliant primitives by default. Accessibility for a public forum is a load-bearing requirement.
- CSS variables let us add light/dark and theming later without rewriting components.
- This stack is well-trodden: many production React apps use it; hires know it.

### Why not the alternatives

- `shadcn/ui` — copy-pasted code with weaker a11y defaults; great for SaaS dashboards, less great for a public forum
- CSS Modules — fine, but slower to iterate than Tailwind for a team
- Vanilla Extract — overkill for v1
- MUI / Chakra — heavier, less control over visual identity, harder to escape later

## 4. Test strategy

### Decision

- **Unit:** **Vitest** + **React Testing Library**. Matches Discourse's QUnit philosophy (test rendered output, not internals). Fast, ESM-native.
- **Component:** **Storybook** (or **Ladle** if we want lighter weight) for visual development; Vitest snapshots for behavior.
- **E2E:** **Playwright** with a dedicated test Discourse instance. Run against a Docker-Compose Postgres + Redis + Rails + Next.js stack.
- **Visual regression:** Playwright screenshots + Percy or Chromatic for diff review.
- **Lighthouse CI** on every PR preview deploy. Performance + a11y + SEO scores must not regress.
- **Type check:** `tsc --noEmit` in CI; **strict** mode mandatory.
- **Lint:** ESLint flat config with `@typescript-eslint`, `react`, `react-hooks`, `next/core-web-vitals`, `jsx-a11y`. Prettier as formatter.

### CI matrix

| Job | Runs on | Required |
|---|---|---|
| `lint` | every PR | yes |
| `typecheck` | every PR | yes |
| `unit` (Vitest) | every PR | yes |
| `build` | every PR | yes |
| `e2e` (Playwright) | every PR (smoke), nightly (full suite) | smoke yes, full nightly |
| `lighthouse` | every PR preview | warn but not block |
| `security audit` (`pnpm audit --audit-level=high`) | every PR | yes |
| `dependency review` | every PR | warn |

### Coverage targets

- v1 Phase 2 routes: **≥80% line coverage** on Server Components + supporting `lib/`
- v1 Phase 3 interactivity: **≥70% line coverage** (composer is genuinely hard to unit-test, lean on Playwright)
- Critical paths (login, signup, post-create, password-reset): **100% Playwright coverage** before v1 cutover

### Test data

- A Discourse seed fixture (one site, 5 categories, 50 topics, 200 posts, 10 users) lives in `tests/fixtures/`. Loaded into the test Rails instance via a rake task.
- Playwright test users have known passwords (only on the test instance, never on production).

## Open issues

- **Internationalization story for v1.** The Ember app uses `discourse-i18n`. The Next.js app should consume the same translation YAML files via `/translations/<locale>.json`. For v1, ship `en` only; queue the multi-locale work for v2.
- **Real device testing.** Playwright headless covers desktop and mobile-emulated; real-device coverage (BrowserStack) should ride a quarterly cadence, not per-PR.
- **Accessibility audit.** Every new route needs an `axe-core` Playwright check. Failing accessibility checks should block CI.

# Phase 0 — Ember → Next.js Migration

**Status:** Draft, awaiting human sign-off
**Branch:** this PR
**Owner:** TBD (engineering leadership to assign tech lead)
**Strategy:** Greenfield Next.js 15 App Router consuming Discourse REST API + Rails backend reuse (Strategy B with Strategy C-style backend reuse, per [the migration plan](../../ember-to-react-nextjs-migration-plan.md))

This directory is the Phase 0 deliverable from the migration plan. **Nothing in here is implementation code** — Phase 0 is the inventory + decision pack that the team needs to sign off on before any Next.js code is written.

## What's in here

```
migration/phase-0/
├── README.md                           ← you are here
├── inventory/                          ← raw + summarized inventories of the existing Ember surface
│   ├── all-json-endpoints.txt          ← 504 unique `.json` endpoints called from Ember
│   ├── v1-json-endpoints.txt           ← 313 of those that fall in the v1 scope (no admin, no plugin features)
│   ├── all-non-json-endpoints.txt      ← 306 non-`.json` endpoints (form posts, redirects, etc.)
│   ├── messagebus-channels.txt         ← 16 MessageBus channels the Ember app subscribes to
│   ├── routes.md                       ← annotated route map → Next.js App Router targets
│   ├── plugins.md                      ← plugin-by-plugin assessment, with v1 disposition
│   └── services.md                     ← service-by-service mapping → React replacements
├── adrs/
│   ├── ADR-0001-strategic-decision.md  ← THE load-bearing decision: walk away from Discourse upstream
│   ├── ADR-0002-v1-scope.md            ← what's in v1 vs deferred to v2+
│   ├── ADR-0003-repo-and-monorepo.md   ← new repo `nordstrom-demo-web` vs in-monorepo
│   ├── ADR-0004-auth-model.md          ← cookie-shared session vs OIDC token exchange
│   ├── ADR-0005-data-and-realtime.md   ← Server Components + React Query + MessageBus adapter
│   ├── ADR-0006-rendering-routing-design-test.md  ← combined: SSR strategy, App Router mapping, Tailwind 4, Vitest+Playwright
│   └── ADR-0007-api-gap-analysis.md    ← Rails-side endpoints we'll need to add or modify
└── staffing.md                         ← team shape, timeline, gates, exit criteria for Phase 0 → 1 transition
```

## Numbers at a glance

These numbers come from a static scan of `frontend/`, `plugins/`, and `themes/` on `demo/ember-upgrade`. Use them to calibrate scope arguments, not for engineering estimates.

| Surface | Count | Where |
|---|---|---|
| Frontend source files | 7,008 | `frontend/`, `plugins/`, `themes/` |
| Frontend LOC | 182,420 | same |
| Routes (Ember) | 158 | `frontend/discourse/app/routes/app-route-map.js` |
| Components (Ember) | 706 (669 `.gjs` + 37 class `.js`) | `frontend/discourse/app/components/` |
| Services | 53 | `frontend/discourse/app/services/` |
| Models | 56 | `frontend/discourse/app/models/` |
| Helpers | 85 | `frontend/discourse/app/helpers/` |
| Modifiers | 15 | `frontend/discourse/app/modifiers/` |
| REST endpoints called | 504 unique `.json` + 306 other | scan of `ajax(...)` + `fetch(...)` |
| MessageBus channels | 16 | `messageBus.subscribe(...)` |
| Plugins (with frontend code) | 42 of 43 | `plugins/*/` |
| Plugin frontend files | 1,720 | `plugins/**/*.{js,gjs,hbs}` |
| Theme frontend files | 23 | `themes/**/*.{js,gjs,hbs}` |
| QUnit tests | 661 | `frontend/**/tests/` |
| RSpec tests | 1,457 | `spec/` |

## Sign-off matrix

These ADRs need named owners + signatures before Phase 1 starts. **Do not start Phase 1 with any row marked "open."**

| ADR | Owner role | Approver | Status |
|---|---|---|---|
| 0001 — Strategic decision (walk away from upstream) | Eng leadership + Product | CTO | Open |
| 0002 — v1 scope | Product + Eng lead | VP Product | Open |
| 0003 — Repo & monorepo strategy | Eng lead | DevEx lead | Open |
| 0004 — Auth model | Eng lead + Security | CISO | Open |
| 0005 — Data layer + real-time | Eng lead | — | Open |
| 0006 — Rendering, routing, design, test | Eng lead + Design | — | Open |
| 0007 — API gap analysis (Rails work) | Backend lead | — | Open |

## How Phase 0 ends

Phase 0 is "done" when:

- All 7 ADRs are approved (ADR-0001 by the CTO or equivalent, ADR-0004 by the CISO, the rest by the eng lead).
- The plugin disposition list in [`inventory/plugins.md`](inventory/plugins.md) has been reviewed and each plugin has been classified (drop / port / defer).
- The Rails-side API gap list in [`adrs/ADR-0007-api-gap-analysis.md`](adrs/ADR-0007-api-gap-analysis.md) has Rails team owners assigned and tickets opened.
- Phase 1 staffing in [`staffing.md`](staffing.md) is committed (i.e., real people, not roles).

After all four are true, open the Phase 1 bootstrap PR.

## Reminders

- This is a **rewrite, not a migration.** No automated codemod path. Per-component effort is 1–4h simple, 1–3 days complex. With ~706 components in core + 1,720 in plugins, **6,000–18,000 engineer-hours of porting work alone** before integration.
- This **walks away from the upstream Discourse plugin & theme ecosystem.** That decision is in ADR-0001 and is not reversible cheaply.
- This fork has **no GitHub Actions runners enabled.** That has to be fixed regardless of which migration path is chosen — see ADR-0006.
- The Ember 7.0.0-beta.1 upgrade we already shipped (PR #1, PR #2 against `demo/ember-upgrade`) is the **insurance policy** for this migration: if Phase 0 review concludes the rewrite is too expensive or strategically wrong, the Ember branch is in a good state to keep using.

## Outstanding open questions for human review

These are flagged inline in the ADRs but listed here for visibility:

1. **ADR-0001:** Is leadership signing off on permanently leaving the upstream Discourse ecosystem in writing?
2. **ADR-0002:** Is "core forum only for v1" acceptable, or does the business need admin/chat/AI plugin parity at cutover?
3. **ADR-0003:** New repo `nordstrom-demo-web` vs `apps/web-next/` workspace — Devin recommends the new repo; awaiting confirmation.
4. **ADR-0004:** Cookie-shared session vs OIDC — Devin recommends cookie-shared for v1; CISO approval still required.
5. **ADR-0007:** ~30 Rails-side endpoints will need to be added or expanded before v1 ships. Is the Rails team scoped + funded for that work?

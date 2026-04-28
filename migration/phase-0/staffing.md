# Phase 0 → Phase 1 staffing & timeline

**Status:** Proposed
**Date:** 2026-04-28
**Owner:** Engineering leadership
**Depends on:** ADR-0001 approved (otherwise this is moot)

## Phase 0 sign-off team (this PR)

Phase 0 is review work, not implementation. The reviewers needed:

| Role | Responsibility | Time commitment |
|---|---|---|
| CTO (or VP Eng) | ADR-0001 strategic decision sign-off | 1–2 days (read + meet) |
| VP Product | ADR-0002 v1 scope, plugin disposition list | 1 week (analytics + decision) |
| Eng lead (TBA) | All ADRs technical review, sign owners on tickets | 2 weeks |
| DevEx lead | ADR-0003 repo placement | 1 day |
| CISO | ADR-0004 auth model, CORS/cookie scope | 3 days |
| Backend lead | ADR-0007 API gap analysis, Rails ticket creation | 1 week |
| Designer (TBA) | ADR-0006 design system — confirm Tailwind 4 + react-aria + token strategy | 3 days |

**Phase 0 review window:** 2 weeks from PR open.

## Phase 1 team (the bootstrap, after Phase 0 sign-off)

Phase 1 is "stand up the new repo, the auth bridge, and one route end-to-end." Per the migration plan, 4–6 weeks of work for a small team:

| Role | Count | When |
|---|---|---|
| Tech lead | 1 | full-time, all 6 weeks |
| Senior frontend engineer | 2 | full-time, all 6 weeks |
| Backend engineer | 1 | half-time, all 6 weeks (Rails-side categories 1, 4, 7 from ADR-0007) |
| SRE / DevOps | 1 | quarter-time, weeks 1, 5, 6 (Cloudflare Worker, Vercel project, secrets) |
| QA engineer | 1 | quarter-time, weeks 4–6 (Playwright bootstrapping) |
| Product manager | 1 | quarter-time |
| Designer | 1 | half-time, weeks 1–3 (design system tokens) |

## Phase 2+ team (after Phase 1)

Per the migration plan:

- 1 tech lead (full-time)
- 3–4 senior frontend engineers (full-time)
- 1 backend engineer (continues part-time from Phase 1)
- 1 QA / test-automation engineer (continues from Phase 1)
- 1 SRE (part-time, ramps up at Phase 5 cutover)
- 1 product manager (part-time)
- 1 designer (part-time)

Less than this and the timeline doubles.

## Hiring asks

If the team isn't already on staff:

- 4 senior React/Next.js engineers (lead + 3) — typical fully-loaded comp $180–250K each, hiring lead time 3–4 months in current market
- 1 senior backend engineer with Rails experience — should already be on staff if the Rails Discourse fork is being maintained
- 1 SRE/DevOps with Cloudflare Worker experience
- 1 senior product designer comfortable with design systems

If the team is being assembled from scratch, **Phase 1 should not start before the team is staffed.** Trying to "start with what we have" with this scope creates a 24-month death-march.

## Timeline

Per ADR-0001 / migration plan, with the team listed above:

| Milestone | Calendar | Cumulative |
|---|---|---|
| Phase 0 review + sign-off | 2 weeks | 2 weeks |
| Phase 1 (bootstrap) | 4–6 weeks | 6–8 weeks |
| Phase 2 (read-only forum parity) | 3–6 months | 4.5–7.5 months |
| Phase 3 (interactivity + composer) | 3–6 months | 7.5–13.5 months |
| Phase 4 (admin port) | 3–6 months (or **defer to v2**) | 10.5+ months |
| Phase 5 (cutover + Ember removal) | 1–3 months | 11.5–13.5 months |

**v1 (excluding admin) on the recommended team: 9–12 months.** Add 6+ months if admin is in v1 scope. Add 3+ months if hiring is incomplete at Phase 1 start.

## Gates

Each phase has a binary "ship / don't ship" gate. **Do not collapse these.**

| Gate | Criteria | Fail action |
|---|---|---|
| Phase 0 → 1 | All ADRs signed, plugin disposition list reviewed, Rails team funded | Halt; revisit ADR-0001 |
| Phase 1 → 2 | Bootstrap PR merged, auth bridge works end-to-end on a staging Rails instance, one route renders correctly, CI green | Pause; do not start Phase 2 against a broken auth bridge |
| Phase 2 → 3 | All Phase 2 routes implemented, Lighthouse ≥ baseline + 10, axe-core 0 violations, beta-flag user can read everything | Pause; do not start Phase 3 against incomplete Phase 2 |
| Phase 3 → 4 | All Phase 3 interactivity working, composer parity, real-time updates working, beta-flag user can do everything | Pause; do not start Phase 4 against incomplete Phase 3 |
| Phase 4 → 5 | (if in scope) admin parity OR explicit deferral with proxy in place | Defer to v2 if not done |
| Phase 5 (cutover) | 100% rollout with fallback flag, 30 days clean, no rollback events | Roll back; debrief |

## Risks (high → low)

1. **Hiring slips.** The market for senior React/Next.js engineers is competitive. If hiring takes 6 months, the timeline doubles. Mitigation: contract hires for Phase 1, full-time for Phase 2+.
2. **API gap analysis underestimated.** Per ADR-0007, ~5–6 weeks of Rails work for v1. If that's actually 12 weeks, Phase 2 slips. Mitigation: Backend lead reviews `inventory/all-json-endpoints.txt` and `inventory/all-non-json-endpoints.txt` line-by-line in Phase 0.
3. **Plugin disposition disagreement.** If Product wants more plugins ported than Engineering can fit in v1, the timeline slips. Mitigation: get the disposition list signed before Phase 1 starts, with a one-line rationale per plugin.
4. **Auth model change.** If CISO blocks cookie-shared and forces OIDC into v1, that's +4–6 weeks. Mitigation: get CISO sign-off on ADR-0004 in Phase 0 review, not later.
5. **Composer port (Phase 3) takes longer than estimated.** Composer is the single hardest component in Discourse. Realistic estimate is 6–10 engineer-weeks for two engineers. If it slips, push to v1.5 with a "minimum viable composer" (no oneboxes, no draft autosave) for cutover and ship the rest after. Mitigation: prototype the composer early in Phase 3, not late.
6. **Real-time integration regressions.** MessageBus + React Query + Server Component invalidation is unfamiliar territory. Mitigation: prototype topic-page real-time updates as a Phase 1 stretch goal.
7. **Performance regressions on logged-in users.** Server Components are great for SEO but logged-in user pages can't be cached server-side as aggressively. Mitigation: profile early, lean on React Query + edge cache for personalized fragments.
8. **Cutover-day operational issues.** The Cloudflare Worker, the cookie-domain config, the `MESSAGEBUS_BASE_URL` CORS — all have to be perfect on cutover day. Mitigation: run a full cutover rehearsal on a staging environment with synthetic load.

## What I (Devin) can ship in this engagement

In the current Devin session: this Phase 0 PR.

If you spin up a follow-up Devin session after Phase 0 is signed off: the Phase 1 bootstrap PR (new repo, Next.js 15 + Tailwind 4 + Vitest + Playwright + Lighthouse-CI scaffold, auth middleware skeleton, one read-only route working against a mocked Discourse API). That's a full 4–6 weeks of human-team work compressed into a starter PR for a real human team to take from there.

I cannot ship Phase 2, 3, 4, or 5. Those are multi-engineer-quarter efforts that are not session-shaped.

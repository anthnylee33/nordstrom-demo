# ADR-0003 — Repository placement: new repo `anthnylee33/nordstrom-demo-web`

**Status:** Proposed (Devin-recommended; Q3 was skipped during Phase 0 scoping)
**Date:** 2026-04-28
**Decision owner:** Engineering lead
**Approver:** DevEx lead
**Depends on:** ADR-0001 approved

## Context

The Next.js app needs a home. Three options were considered:

1. **A new repo** — `anthnylee33/nordstrom-demo-web` (or similar). Cleanest separation; the existing Discourse fork stays frozen on the Ember side.
2. **A new workspace in this monorepo** — `apps/web-next/` alongside `frontend/discourse/`, `frontend/pretty-text/`, etc. Single source of truth, but pnpm workspaces and CI become more complex.
3. **Inside `frontend/`** — alongside the Ember workspaces. Tightest coupling. Easy to accidentally couple Next.js code to Ember internals; hardest path to delete Ember when v1 cuts over.

## Decision

**Option 1: a new repo.** Initial name: `anthnylee33/nordstrom-demo-web`.

The new repo is what gets staffed, what CI runs against, what Vercel/Cloudflare deploys, and what the product team treats as "the new app." The existing `nordstrom-demo` repo remains the home of the Rails backend + Ember legacy frontend until the cutover, then has its `frontend/` directories deleted.

## Why a new repo

- **Clean operational separation.** The Next.js app's CI, deploy, lint, type-check, and test pipelines are independent of the Rails app's CI (which runs RSpec, Sidekiq tests, and a much heavier image). Trying to share a single CI surface produces 30+ minute build times for a 5-minute Next.js change.
- **Smaller cognitive surface for new hires.** A senior frontend engineer joining the project does not need to clone, set up, and understand Discourse + Postgres + Redis + Ember + Sidekiq before writing their first React component.
- **Independent deploy lifecycle.** Vercel/Cloudflare-style deploys are atomic per-commit. A Rails backend change should not trigger a Next.js redeploy and vice versa.
- **Independent versioning.** The Next.js app will ship semver-versioned packages for its design system, its API client, and its types. The Rails app does not.
- **Easier to open-source the new app or close-source it independently** if that ever becomes a strategic decision.

## Why not the alternatives

### Why not `apps/web-next/` workspace in this monorepo

- pnpm workspace + Next.js 15 + the existing 54-package workspace surface is a non-trivial integration. Next.js's bundler (Turbopack) and pnpm's symlink strategy already require care for transitive dependency hoisting (we hit this in PR #1 with `@glimmer/env`).
- CI: the existing repo has no GitHub Actions runners enabled. Setting them up to run only on changed paths (`apps/web-next/**`) adds infrastructure complexity that's not needed in a separate repo.
- The Rails app's developer setup (Postgres, Redis, Ruby 3.4, Sidekiq, Imagemagick, etc.) is irrelevant to Next.js development. Forcing every frontend engineer to have it installed creates friction.
- The argument for keeping it in-monorepo is "atomic cross-cutting changes" — but the API contract between Next.js and Rails should be **versioned**, not synchronously deployed. If atomic deploy is required, you have a brittle architecture.

### Why not `frontend/`

- Highest coupling risk. A junior engineer can `import` from `frontend/discourse/app/...` and get away with it during v1, then we discover the coupling at v2 deletion time.
- pnpm hoisting + Ember deps + Next.js deps in the same lockfile is a recipe for the kind of `@glimmer/env` resolution issue we hit in PR #1, multiplied by the much larger Next.js + React surface.
- Hardest to delete cleanly when v1 cuts over.

## Repo bootstrap (when ADR is approved)

```bash
gh repo create anthnylee33/nordstrom-demo-web \
  --description "Next.js 15 frontend for the Nordstrom demo Discourse fork (v1 cutover target)" \
  --private \
  --add-readme false

# Bootstrap with create-next-app
pnpm dlx create-next-app@latest nordstrom-demo-web \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --turbo \
  --use-pnpm \
  --import-alias "@/*"

cd nordstrom-demo-web
git remote add origin https://github.com/anthnylee33/nordstrom-demo-web.git
git branch -M main
git push -u origin main
```

Phase 1 PR (the bootstrap) lives in the new repo. Phase 0 (this PR) lives in the old repo because it's documenting decisions about migrating away from the old repo.

## Cross-repo coupling

Two pieces of cross-repo coupling are unavoidable:

1. **API client types.** The Next.js app needs TypeScript types for the Discourse REST API responses. Three options:
   - generate types from the Rails app's serializers (preferred; build-time codegen)
   - hand-write types in `nordstrom-demo-web/src/lib/api/types.ts` (acceptable for v1; lower fidelity)
   - publish a `@discourse/api-types` package from the Rails app (ideal long-term; not v1)

   **Decision: hand-write for v1**, with an issue tracking the codegen story for v2.

2. **Translation strings.** Both apps need the same `client.en.yml` translation strings. The Next.js app fetches them at build/runtime via `/translations/<locale>.json` from the Rails app — a tiny endpoint that already exists in Discourse. No duplication.

## Repo settings (when created)

- Default branch: `main`
- Branch protection: required reviews, required CI green
- CODEOWNERS: yet-to-be-named tech lead + DevEx lead for `package.json`, `pnpm-lock.yaml`, `next.config.ts`, `.github/`
- Required CI: lint, typecheck, unit tests, build, Lighthouse-CI on PR previews
- Deployment: Vercel preview per PR, production deploy on main
- Secrets: `DISCOURSE_API_BASE_URL`, `DISCOURSE_SESSION_COOKIE_NAME` (default `_t`), CSRF/HMAC secrets per ADR-0004

## Open question

If your organization has a strong preference for monorepos for cross-cutting reasons (org-wide tooling, shared CI infrastructure, etc.), the in-monorepo `apps/web-next/` option is workable. **Override this ADR before Phase 1 starts** rather than after.

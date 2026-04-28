# ADR-0004 — Authentication model: cookie-shared session with Rails for v1

**Status:** Proposed (Devin-recommended; Q4 was skipped during Phase 0 scoping)
**Date:** 2026-04-28
**Decision owner:** Engineering lead + Security
**Approver:** CISO
**Depends on:** ADR-0001, ADR-0003

## Context

The Next.js app (new repo, new subdomain) needs to authenticate users. The Rails app already has a working authentication system: cookie-based session (`_t` cookie), CSRF tokens, 2FA, passkey/WebAuthn, OAuth providers (Apple, Microsoft, Google, Discord, GitHub, etc., via the `discourse-*-auth` plugins), email-login tokens, and rate limiting.

Three options:

1. **Cookie-shared session with Rails.** Set the `_t` cookie at the parent domain (`.example.com`); both `forum.example.com` (Rails + Ember) and `next.forum.example.com` (Next.js) read it. Next.js validates the session by calling `GET /session/current.json` on Rails. Rails remains the auth source of truth; Next.js holds no session of its own.
2. **OIDC / token exchange.** Next.js exchanges a code for a JWT or opaque token, holds its own session, refreshes against Rails. Cleanest separation.
3. **Defer until cutover.** Ship a staff-only feature flag, defer the public auth model decision.

## Decision

**Option 1: cookie-shared session with Rails for v1.** Reconsider OIDC for v2.

## Why cookie-shared for v1

### It actually works in practice

- Discourse already issues a `_t` cookie scoped to its domain; if the Next.js app is deployed at a subdomain (`next.forum.example.com`), setting cookies at the parent domain (`.forum.example.com`) is a one-line Rails configuration change (`Rails.application.config.session_store :cookie_store, key: '_t', domain: :all` — already set in Discourse).
- Next.js middleware reads the cookie, calls `GET /session/current.json` on Rails with `Cookie: _t=...` and `Accept: application/json`, gets back the current user as JSON, and populates `request.user` for Server Components via `headers()`.
- Logout: hit `DELETE /session/<username>` on Rails; Rails invalidates the cookie; Next.js sees no cookie on next request → user is logged out.

### It's the smallest auth-system surface

- We do not run our own session store, our own JWT signer, our own refresh-token rotation, our own 2FA enforcement, or our own passkey ceremony. Rails owns all of that and has been hardened for a decade.
- We avoid the entire class of "implementing auth wrong" CVEs (token leakage, JWT alg-confusion, session-fixation, race conditions in refresh).

### It defers the right decision

- The OIDC / token-exchange story is genuinely better long-term — it lets the Next.js app outlive the Rails backend, supports machine-to-machine flows, and aligns with industry conventions. **But it is not a v1-blocking improvement.** Shipping cookie-shared first lets us deliver v1 in 9–18 months; switching to OIDC in v2 is a 4–6 week project on top of a working v1, not 4–6 weeks of v1 critical-path work.

## Implementation sketch

### Cookie scope

```
# config/initializers/session_store.rb (Rails — adjust if needed)
Rails.application.config.session_store :cookie_store,
  key: "_t",
  domain: ".forum.example.com",   # parent domain
  same_site: :lax,
  secure: Rails.env.production?,
  httponly: true
```

The `domain: :all` setting that Discourse ships with already produces this behavior in production; verify on a staging deploy before cutover.

### Next.js middleware

```ts
// nordstrom-demo-web/src/middleware.ts
import { NextRequest, NextResponse } from "next/server";

const PUBLIC_PATHS = ["/login", "/signup", "/about", "/categories", "/c/", "/t/", "/u/", "/tag/"];

export async function middleware(req: NextRequest) {
  const cookie = req.cookies.get("_t")?.value;
  if (!cookie) {
    // unauthenticated — pass through; Server Components decide whether to redirect
    return NextResponse.next();
  }

  // Lightly validate by hitting Rails. Cache for 60s in middleware.
  const res = await fetch(`${process.env.DISCOURSE_API_BASE_URL}/session/current.json`, {
    headers: { Cookie: `_t=${cookie}`, Accept: "application/json" },
    next: { revalidate: 60, tags: [`session:${cookie.slice(0, 16)}`] },
  });

  if (res.status === 200) {
    const user = await res.json();
    const headers = new Headers(req.headers);
    headers.set("x-discourse-user", JSON.stringify(user.current_user));
    return NextResponse.next({ request: { headers } });
  }

  // Cookie present but invalid — delete it
  const next = NextResponse.next();
  next.cookies.delete("_t");
  return next;
}

export const config = { matcher: ["/((?!api|_next|.*\\..*).*)"] };
```

Server Components read `headers().get("x-discourse-user")` to know who's logged in.

### CSRF

- Read CSRF token from the Rails-rendered `<meta name="csrf-token">` on the legacy app, OR call `GET /session/csrf.json` from Next.js to get a token.
- Forward the token on every write request as `X-CSRF-Token: <token>`. This is what the Ember app already does.
- Server Actions in Next.js fetch the token at request time and include it on the proxied call to Rails.

### Login + signup pages

- Render the login form in Next.js. Submit to a Server Action that proxies to `POST /session.json` on Rails.
- 2FA: Rails returns `{ second_factor_required: true, ... }`. Render the 2FA form, submit to `POST /session.json` with the second-factor code. (This is exactly what the Ember app does — there's no client-side crypto for 2FA.)
- Passkey/WebAuthn: more involved. The browser does the WebAuthn ceremony; Next.js relays the assertion to Rails at `POST /u/${username}/security_key.json`. Plan ~1–2 weeks for v1 passkey support; consider deferring to v2 if the user base is small.
- OAuth: render provider buttons that link to `/auth/<provider>` on Rails (already implemented). The OAuth dance happens on Rails; the user ends back at Rails-issued cookie. The Next.js app does nothing special; it just sees a `_t` cookie on the next request.

### Logout

```ts
// Server Action
"use server";
export async function logout() {
  const username = headers().get("x-discourse-user-username");
  await fetch(`${process.env.DISCOURSE_API_BASE_URL}/session/${username}`, {
    method: "DELETE",
    headers: { Cookie: cookies().toString(), "X-CSRF-Token": await getCsrfToken() },
  });
  cookies().delete("_t");
  redirect("/");
}
```

## Security considerations (CISO must review)

1. **Cookie domain scope.** `.forum.example.com` makes the `_t` cookie readable by both `forum.example.com` and `next.forum.example.com`. **Any subdomain takeover anywhere under `.forum.example.com` becomes a session-stealing primitive.** CISO must confirm DNS hygiene before this ADR is approved.
2. **CSRF.** Both apps must enforce CSRF on state-changing requests. The Next.js Server Actions must propagate CSRF tokens; the Rails app already enforces them. Verify with a Burp/ZAP scan at staging.
3. **CORS.** The Next.js → Rails calls are server-to-server (Server Components on Vercel/Cloudflare reaching the Rails origin). They are not subject to browser CORS. **However**, any client-side `fetch(...)` from the Next.js app to Rails *is* subject to CORS — the Discourse `cors_origins` site setting must be updated to allow the Next.js subdomain. Verify before v1 cutover.
4. **Cookie attributes.** `HttpOnly`, `Secure`, `SameSite=Lax` (not `Strict` — `Strict` breaks OAuth redirect flows).
5. **Session cache TTL.** The middleware caches `/session/current.json` results for 60s. Logout actions must explicitly invalidate the cache (`revalidateTag("session:...")`). A 60s window of "logged out user still appears logged in" is the trade-off; CISO must accept this. Setting TTL to 0 doubles Rails request load.
6. **Double-submit cookie pattern** if a CSRF cookie + header check is used, the cookie must be `SameSite=Strict` on the CSRF cookie itself but `Lax` on the session cookie; this is what Discourse already does.
7. **Rate limiting.** Login attempts from the Next.js app must hit the same Rails rate-limiter as the Ember app. Verify the Rails `RackAttack` config covers requests from the Next.js IP range (Vercel/Cloudflare egress).

## v2 path: OIDC

When v1 is stable and we want to decouple Next.js from Rails for things like:

- letting Next.js outlive the Rails app (replace the backend gradually)
- adding M2M (machine-to-machine) flows for partners
- aligning with org-wide SSO

… the migration is:

1. Stand up an OIDC provider in front of Rails (Discourse already supports being an OAuth2 provider via `discourse-oauth2-provider`-like plugins, or Keycloak in front of it).
2. Next.js gets its own session, exchanges code for token, refreshes against the OIDC provider.
3. Cookie-shared mode is removed.

Plan ~4–6 weeks for the v2 OIDC migration. Until then, accept the cookie-sharing pattern.

## Open question for CISO

If the organization's security baseline forbids cookie-sharing across subdomains (some orgs do; it's a defensible position given the subdomain-takeover risk), **OIDC must be in v1, not v2.** That adds ~4–6 weeks to the v1 timeline. Confirm before approving this ADR.

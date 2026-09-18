# 02 · Architecture

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

This document reconstructs the current-state origin architecture **from source** (commit `18e02022631c1c9dba1b6e5ac734ae27b54632cb`), names the trust boundaries and where secrets/sessions/object-level authorization live, and proposes the future-state layout behind Cloudflare.

The machine-readable source of truth is `audit/run-1/architecture.md` (full trace pointers) and `audit/run-1/coverage-ledger.json`.

---

## 1. Current-state origin (from source)

```

                    Untrusted Internet
                     storefront visitors / bots / webhooks
        ┌────────────────────────────────────────────────────┐
        │                                                    │
        │  PUBLIC ROUTES (route workspaces, no session gate) │
        │   GET/POST  /graphql              storefront GraphQL│
        │   POST      /cart/* , /api/orders REST checkout     │
        │   POST      /api/user/tokens       admin JWT mint   │
        │   POST      /api/user/token/refresh admin refresh   │
        │   POST      /api/customer/tokens  customer JWT mint │
        │   GET/POST  /api/stripe/paymentIntents  Stripe mint │
        │   POST      /api/paypal/orders    PayPal create     │
        │   GET       /images/*             image optimizer    │
        │   GET       /static/* , /media/*  static assets     │
        └──────────────────────┬─────────────────────────────┘
                               │
        ┌──────────────────────┴─────────────────────────────┐
        │  SESSION MIDDLEWARE  (express-session, 2 flavors)  │
        │   adminSessionMiddleware  -> 'asid' signed cookie   │
        │   storeFrontSessionMiddleware -> 'sid' signed cookie│
        │   API/graphql routes EXEMPTED (stateless design)    │
        └──────────────────────┬─────────────────────────────┘
                               │
        ┌──────────────────────┴─────────────────────────────┐
        │  ADMIN ROUTE GATE  (per-request getCurrentUser)    │
        │   /admin/graphql, /oms/*, catalog/evershop admin    │
        │   identity from 'asid' cookie OR admin JWT bearer   │
        │   single role "admin"; NO RBAC                      │
        └──────────────────────┬─────────────────────────────┘
                               │
        ┌──────────────────────┴─────────────────────────────┐
        │  INTERNAL / DB                                      │
        │   PostgreSQL 15 (postgres-query-builder, BLOBs)     │
        │   connect-pg-simple session store                   │
        │   local FS image/upload cache under <root>          │
        │   rate-limit MemoryStore (per-process)              │
        └────────────────────────────────────────────────────┘
```

## 2. Trust boundaries

| # | Boundary | Contains | Highest trust | Notes |
|---|---|---|---|---|
| B1 | Internet → Public routes | storefront GraphQL, REST checkout, JWT mint, payment mint, image optimizer, static | **untrusted** | Middleware classifies rate tiers here |
| B2 | Session boundary | SSR page requests carrying `asid`/`sid` signed cookies | untrusted ambient identity | Session provides identity for SSR only; API routes exempted |
| B3 | Admin gate | `/admin/*`, `/oms/*`, operator APIs | semi-trusted (single admin role) | Gate = `getCurrentUser` only; no RBAC; vulnerable to session-cookie forgery if secret known |
| B4 | Webhook boundary | Stripe/PayPal callbacks | **untrusted but signature-checked** | Stripe uses `constructEvent(rawBody, sig)` correctly |
| B5 | Internal/DB | Postgres, session store, BLOB/media cache | trusted | Compose publishes `0.0.0.0:5432` (lead `deployment.compose.*`) |

## 3. Internet-exposed vs operator vs callback routes

**Internet-exposed (no auth gate):** `/api/graphql` (public mutation-capable surface), `/api/carts/*`, `/api/orders`, `/api/stripe/paymentIntents`, `/api/paypal/orders`, `/api/user/tokens`, `/api/user/token/refresh`, `/api/customer/tokens`, `/api/customer/reset-password`, `/images/*`, `/static/*`, `/media/*`.

**Operator-only (admin gate):** `/admin` SPA, `/admin/graphql`, `/oms/*` (capture/refund/cancel), catalog/admin routes, `/api/files`, `/api/images` (admin-gated uploads).

**Callbacks (signature-checked):** `/api/stripe/stripeWebHook` (via `[bodyJson]webhook`), PayPal webhook, and any shipment/tax feed callbacks.

**Critical subtlety:** the admin gate is enforced per-route by a *gate middleware*, but the **session cookie is the only ambient identity** for the operator, and the API routes (including checkout and payment mint) bypass session middleware *entirely* — object-level ownership on those surfaces is therefore absent by design (leads 1–4).

## 4. Where secrets, sessions, and object-level authorization live

- **Secrets:** `configExample.text` pins the cookie secret; `getCookieSecret()` falls back to `'keyboard cat'`; `docker-compose.yml` pins `postgres/postgres`; no `.env`/provisioner emits a strong default (leads `evershop-auth-cookie-secret-static-default`, `deployment.compose.*`).
- **Sessions:** connect-pg-simple in Postgres; signed cookies `asid` (admin 24h) / `sid` (storefront); `saveUninitialized` default true in code, `false` in `configExample.text`; no rotation on login, no destroy on logout (lead `evershop-auth-session-lifecycle-no-invalidation`).
- **Object-level authorization:** essentially absent on the consumer surfaces. Cart/order/payment-intent/PayPal accesses are keyed by client-supplied uuids with no ownership predicate; the only control is uuid secrecy (122-bit). The OMS admin surfaces rely on the route-level admin gate, which has a concurrency race on capture/refund and an asymmetric restock on cancel.

## 5. Proposed future-state: users → Cloudflare → origin

```

   Users (storefront + operators + partners)
        │  HTTPS, validated certs
        ▼
        ┌─────────────────────────────────────────────────────┐
        │  CLOUDFLARE EDGE                                     │
        │   DNS proxied (origin hidden) · TLS Full (strict)   │
        │   WAF custom rules · managed WAF · rate limiting     │
        │   Bot Management + Turnstile on pay/search/login     │
        │   Zero Trust Access (SSO) on admin + operator APIs   │
        │   API Shield / schema validation (optional)          │
        │   Cache Rules (conservative on checkout/login)       │
        │   Workers: webhook signature check + origin allowlist│
        │   DDoS protection · Logpush / security analytics     │
        └─────────────────────┬───────────────────────────────┘
                    www         api          admin       hooks
              ┌─────────────┬─────────────┬───────────┬────────────┐
              ▼             ▼             ▼           ▼
   storefront SSR     store/cart      operator APIs   Stripe /
   public pages       checkout API    /admin, /oms    PayPal & feed
   (cacheable)        + JWT mint      (Access SSO)   webhooks (sig-checked)
              │             │             │           │
              └─────────────┴─────────────┴───────────┘
                                    │
                                    ▼
                        ORIGIN (allowlisted to Cloudflare IPs)
                          Node/Express + Postgres
                          loopback-bound DB, generated secrets
```

### Why the split

- **`admin` separated** → Zero Trust Access can require SSO/MFA before any operator request reaches the origin gate, converting the origin's single-role model into something acceptable at launch.
- **`api` separated** → can apply the strictest rate limits (JWT-mint, payment mint) and Turnstile, without touching cacheable `www`.
- **`hooks` separated** → edge workers can verify webhook signatures/expected domains before the origin even parses the body, and the origin allowlist can be locked to Cloudflare.

## 6. What the origin still must fix (Cloudflare cannot)

> Object-level authorization is outside the edge's reach. Cloudflare can rate-limit and gate *who may reach the endpoint*; it cannot know *which cart/order a caller owns*. The missing ownership predicates (leads 1–4), the webhook settlement-state guard (lead 4), and the OMS race/restock bugs are origin work regardless of edge quality.
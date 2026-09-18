# Harbor & Heath Evershop — Source Architecture

> **Context:** Stand-in repository for "Harbor & Heath Outdoor Co." is Evershop v2.2.1 ([`evershopcommerce/evershop`](https://github.com/evershopcommerce/evershop)), clone commit `18e0202`. All analysis is source-local, read-only; no runtime execution.

---

## 1. Stack

| Layer | Technology | Notes |
|---|---|---|
| **Runtime** | Node.js 18+ (Express) | Process-scoped in-memory rate-limit store |
| **Database** | PostgreSQL 15 (via `postgres-query-builder`) | Connection-pool in `config.json` |
| **API** | GraphQL (server-side) + REST (Express routes) | GraphQL schema split: storefront (`*.graphql`) + admin (`*.admin.graphql`) |
| **Frontend** | React SSR (`@evershop/evershop-uv-react`) + Vite build | Admin SPA (`admin/pages`), storefront SSR pages (`frontStore`) |
| **Auth** | Cookie-based sessions (`express-session` / `connect-pg-simple`) + JWT (`jsonwebtoken`) | JWTs for admin/customer token endpoints; cookie session for SSR |
| **Storage** | Local filesystem (Postgres-managed BLOBs + `fs.writeFile`) | Upload cache under `BUILDPATH` |
| **Build** | npm workspaces; `postinstall` runs webpack/Vite | `dist/` compiles all SSR bundles |

---

## 2. Module Layout

```
packages/evershop/src/
├── bin/lib/            # Express bootstrap, middleware wiring, rate-limit init
├── lib/                # Shared libs: secureFetch, getConfig, jwt, router
├── modules/            # Domain slices (ecommerce verticals):
│   ├── auth/           # Login, session, cookie config, getCurrentUser gate
│   ├── catalog/        # Product/collection/category pages + GraphQL
│   ├── checkout/       # Cart, checkout flow, placeOrder REST
│   ├── cms/            # Media uploads, image optimizer, pageBuilder
│   ├── customer/       # Account, address, password-reset, order tracking
│   ├── graphql/        # Schema composition, middleware, SSR query builder
│   ├── om/             # Order lifecycle (cancel/ship/invoice status manager)
│   ├── pageBuilder/    # Draft previews (changeset token)
│   ├── setting/        # Site config, tax rates, shipping rates
│   ├── stripe/         # Stripe webhook + createPaymentIntent
│   └── paypal/         # PayPal webhook + createOrder
└── modules/base/       # Rate-limiter, session config, static asset middleware
```

---

## 3. Trust Boundaries

```
          ┌──────────────────────────────────────────────────┐
          │            Untrusted Internet                    │
          │  (storefront visitors, payment providers,        │
          │   webhooks, bot/crawler traffic)                 │
          └──────────────┬───────────────────────────────────┘
                         │
        ┌────────────────┴──────────────────────────────────┐
        │  PUBLIC ROUTES                                    │
        │  GET/POST /graphql          (storefront GraphQL)  │
        │  POST   /cart/*             (REST cart/checkout)  │
        │  POST   /user/tokens        (admin JWT mint)      │
        │  POST   /user/token/refresh (admin JWT refresh)   │
        │  POST   /customer/tokens    (customer JWT mint)   │
        │  POST   /customer/reset-password                  │
        │  GET    /images/*           (image optimizer)     │
        │  GET    /static/*           (public assets)       │
        └────────────────┬──────────────────────────────────┘
                         │
        ┌────────────────┴──────────────────────────────────┐
        │  ROUTE-LEVEL AUTH (session/cookie gate)           │
        │  All /cart/*, /checkout/*, /customer/* pages      │
        │  getCartByUUID() — NO ownership binding           │
        └────────────────┬──────────────────────────────────┘
                         │
        ┌────────────────┴──────────────────────────────────┐
        │  SESSION-ON-PAGES-ONLY middleware (protects SSR)   │
        │  Exemptions: /images/*, /static/*, /adminStatic/* │
        └────────────────┬──────────────────────────────────┘
                         │
        ┌────────────────┴──────────────────────────────────┐
        │  ADMIN ROUTE GATE  (per-request getCurrentUser)   │
        │  /admin/graphql, /oms/*, /catalog/*admin/*         │
        │  Single role ("admin"); no RBAC                   │
        └────────────────┬──────────────────────────────────┘
                         │
        ┌────────────────┴──────────────────────────────────┐
        │  INTERNAL / DB                                   │
        │  Postgres connection pool, BLOB store             │
        └───────────────────────────────────────────────────┘
```

---

## 4. Request Lifecycle (Simplified)

```
GET /api/cart/my-cart
  → Express router (scanForRoutes.js mounts public API)
  → cartREST → getCartByUUID(uuid)          ← unscoped query
  → returns { id, status, items, addresses }  ← full object
  → rate-limit check (MemoryStore, 120/60s tier)
  → session cookie set (secure: false; saveUninitialized: true)
```

```
POST /api/graphql (storefront)
  → express.json + cookieParser (secret: 'keyboard cat')
  → buildResolvers(false) merges storefront resolvers
  → Query.order(uuid) → getOrdersBaseQuery() ← no auth filter
  → Query.cart(id) → getCartByUUID()          ← no ownership check
  → validateSchema(schema, document)           ← wrong API, query not validated
```

```
POST /stripe/api/stripeWebHook/[bodyJson]webhook
  → rawBody capture
  → stripe.webhooks.constructEvent(rawBody, sig)   ← strict signature check
  → returns 200 OK (payment intent events handled)
```

---

## 5. Authentication & Session Model

| Mechanism | Scope | Issuance | Storage |
|---|---|---|---|
| **Cookie session** | SSR page loads | `express-session` / `connect-pg-simple` | PostgreSQL `session` table; signed cookie |
| **JWT (admin)** | `/admin/graphql`, `/oms/*` | POST `/api/user/tokens` (public endpoint) | Client-provided `Authorization: Bearer` |
| **JWT (customer)** | `/customer/*` pages | POST `/api/customer/tokens` (public endpoint) | Client-provided `Authorization: Bearer` |

**Hardening gaps identified at source level:**
- `cookie.secure = false` explicitly in production (`addDefaultMiddlewareFuncs.ts:87`)
- `SameSite` not set
- `saveUninitialized` defaults to `true`
- Cookie secret defaults to `'keyboard cat'` when `AUTH_SECRET` unset (`getCookieSecret.ts:3`)

---

## 6. Rate Limiting

Classified by `rateLimit.ts:classifyRequest`:

| Tier | Limit | Window | Applied to |
|---|---|---|---|
| `auth` | 8 requests | 15 minutes | Known auth paths only |
| `api` | 120 requests | 60 seconds | Other `/api/*` |
| `page` | 300 requests | 60 seconds | Non-API routes |

**Gap:** JWT-mint paths (`/api/user/tokens`, `/api/user/token/refresh`, `/api/customer/tokens`) are not in `AUTH_POST_PATHS`, so they receive the 120/60s tier — brute-forceable bcrypt credentials under that allowance.

---

## 7. Key Source-Verified Suspected Leads

| # | Suspected Lead | Source Location | Status |
|---|---|---|---|
| 1 | **Cart IDOR** — `getCartByUUID(uuid)` unscoped | `checkout/services/getCartByUUID.ts:9` | Parent-verified at source |
| 2 | **Order IDOR** — `Query.order(uuid)` no auth filter; shared schema | `oms/graphql/types/Order/Order.resolvers.js:11` | Parent-verified |
| 3 | **Checkout data-disclosure + victim-cart disable** — `createOrder` takes `cart_id` from body, returns full order with addresses | `checkout/api/createOrder/placeOrder.js:14` | Parent-verified |
| 4 | **JWT brute-force tier gap** — public token mint endpoints on 120/60s tier | `auth/api/getUserToken/route.json` + `rateLimit.ts:16` | Parent-verified |
| 5 | **Cookie hardening** — `secure: false`, no SameSite, `keyboard cat` default | `addDefaultMiddlewareFuncs.ts:85` + `getCookieSecret.ts:3` | Parent-verified |
| 6 | **Image optimizer DoS** — unbounded `w` param, unbounded cache writes | `cms/pages/frontStore/images/images.ts` + `imageProcessor.ts` | Parent-verified |
| 7 | **GraphQL validate misuse** — `validateSchema(schema, document)` wrong API; queries never validated; no depth/alias limits | `graphqlMiddleware.js` | Parent-verified |
| 8 | **Admin gate is route-level only** — single admin role, no RBAC | `auth/api/global/[getCurrentUser]auth.ts:17` | Recon-verified |

---

## 8. Constraints (Sandbox)

No OS-enforced sandbox was available during this run. All runtime-dependent verification (payload crafting, HTTP behavior testing, browser rendering, dependency audit) is recorded as **needs_validation** and must not be marked confirmed without that execution step.

---

*Document capped at source-level reconnaissance. Findings requiring runtime confirmation are flagged `needs_validation` in the final report.*

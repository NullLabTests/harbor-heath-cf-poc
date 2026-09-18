# 05 · Edge Design (Week One Reference Architecture)

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

This is the week-one edge reference architecture for cutover readiness. It is a *security posture* design: what the edge must enforce on day one so the origin's verified defects are contained while their fixes land.

---

## 1. Hostname plan

| Hostname | Purpose | Route policy |
|---|---|---|
| `www.harbor-heath.example` | Storefront SSR + cacheable public pages | DNS proxied; TLS Full (strict); Cache Rules (public, conservative); WAF managed; Bot Management on interactive endpoints |
| `api.harbor-heath.example` | Cart/checkout REST, GraphQL, JWT mint, payment mint | DNS proxied; TLS Full (strict); **no caching** (or Cache Rules off); turnstile + rate limits; API Shield schema validation (optional, prioritized after lead 16) |
| `admin.harbor-heath.example` | Operator SPA, `/admin`, `/oms`, admin GraphQL | DNS proxied; TLS Full (strict); **Zero Trust Access (SSO)**; no cache; Bot Management |
| `hooks.harbor-heath.example` | Stripe/PayPal/feed webhooks | DNS proxied; TLS Full (strict); Workers webhook signature check; allowlist of expected `User-Agent`/signature schemes; no cache |
| `media.harbor-heath.example` | `/media/*`, `/assets/*`, `/images/*` | DNS proxied; TLS Full (strict); Workers: active-content block + `/images` param whitelist; Cache Rules (long TTL, immutable-safe); rate limiting |

Rationale: splitting `www` (cacheable) from `api`/`admin` (sensitive) lets cache policy and rate limits differ per surface without a single blunt configuration. Splitting `hooks` lets signature verification and event filtering happen on a laptop-unfriendly lot.

## 2. Admin only through Cloudflare Access

- **Policy:** `admin.*` requires SSO (IdP of record) with MFA; no bypass for private IPs or office ranges — operators share IPs, so identity (not IP) is the signal.
- **Why:** the origin's admin gate is a single-role, route-level check (architecture.md §7.8). Zero Trust Access adds an identity + MFA boundary *before* the origin ever sees the request — the cheapest, effectively-best antidote to leads 5–6 (concurrency races), 8 (cookie forgery), 11 (token outlives disable) for the *operator* surface: an Access-policy owner can pull an operator instantly regardless of token TTL.
- **Caveat:** Access does not protect the *shopper* JWT surface (id 11). That part is origin fix.

## 3. Checkout and login as sensitive; conservative cache

- **Checkout:** set `Cache-Control` strictly; **Cache Rules**: `*` for `api.*`= Bypass. Turnstile on `/api/carts/*`, `/api/orders`, `/api/stripe/paymentIntents`, `/api/paypal/orders`, and any password/address mutate endpoint. Rate limits: auth-tier-ish budgets (per client IP from the *edge-computed* IP, not spoofable XFF).
- **Login / password reset:** Turnstile + the strictest rate rules (mirroring the intended origin `auth` tier for the mid-tier blinds: `/api/user/tokens`, `/api/customer/tokens`, `token/refresh`, `*/reset-password`). Because the origin currently rate-limits these at 120–300/min (lead 9), the edge is the only enforcement available at week one.
- **Cache boundary:** never cache pages containing account/session state; never cache `api`; never cache `?changeset=` preview paths (lead 24).

## 4. Webhook authenticity

- **Workers on `hooks.*`:** (1) verify the provider signature over the raw body *again* (defense-in-depth on top of the origin's correct `constructEvent(rawBody, sig)`); (2) filter event types against an allowlist; (3) enforce expected origin (Stripe → Stripe's published IPs/UA, PayPal → their signing scheme); (4) only then proxy to the origin's hook route.
- **Why again?** The origin already verifies correctly. Edge re-verification is about containing *future* origin bugs (e.g., a handler that skips verification) and giving security a single choke point for "only signed, expected events reach the order-state machine" (id 4).
- Set `hooks.*` origin lock separate from `www` so a storefront compromise cannot replay into hooks (least-privilege route separation).

## 5. Origin allowlist to Cloudflare

- Firewall/security-group the origin to **Cloudflare-proxied IP ranges only**; TLS Full (strict) so the edge presents a verifiable cert to the origin.
- Remove any direct internet listener: docker-compose currently exposes `0.0.0.0:3000` (and would expose `5432`); the DB must be loopback-bound and un-published (lead 22).
- Do not expose SSH/admin/jump hosts directly; out-of-band access via approved channel only.
- Workers "origin lock checks": on sensitive hostnames, a Worker can verify CF-Connecting-IP and connection attributes before forwarding — plus the network allowlist above.

## 6. What WAF cannot fix (scope honesty)

- **Object-level authorization (ids 1–4):** the edge cannot know which cart/order a caller owns. WAF can rate limit and attest the caller; it cannot add an ownership predicate.
- **Business logic / state machine (ids 4-6, 21):** payment-state downgrades, capture/refund races, asymmetric restock — no edge rule can serialize DB state.
- **Stored trust bugs (ids 8, 17, 23, 24):** a known cookie secret, stored active content, cached draft bytes — edge has good stop-gaps (headers, caching, active-content blocking) but the durable fix is origin.
- **Provider semantics (id 7):** PayPal idempotency and Stripe event delivery are external; the edge cannot decide them.

## 7. Failure modes and rollback

| Failure | Detection | Rollback / containment |
|---|---|---|
| False-positive WAF/rate rule blocks checkouts at peak | Traffic drop on `api.*` + error-rate alerting (Logpush → analytics) | Rule off-switch with one-click; keep Turnstile as softer signal; staged rule rollout in shadow/observe mode first |
| Origin bug under edge rules (still exploitable) | Synthetic checks + lead-1–4 fixture reruns after deploy | Keep Access SSO + origin allowlist + Workers filters in place regardless; feature-flag checkouts is not possible → pre-verified fixes only, no live experiments (ids 1–4 are source-proven; land them with tests) |
| Access SSO outage locks operators out of admin | Operator cannot reach `admin.*` (or standing denial) | Break-glass: separate non-public break-glass path via Access policy exceptions is a *security* decision — pre-authorize 2 operators with time-boxed exception, `ma=2` signed |
| Workers webhook buffer fails / event burst | hooks error-rate + queue depth | Webhook delivery retry on provider side (Stripe/PayPal retry) — do not relax the signature check to "get traffic through" |
| Edge-side origin allowlist mis-deployed (origin unreachable) | 5xx spike at cutover | Manage allowlist as code (terraform), test non-prod zone first; keep a pre-announced fast-rollback state of the previous CF record set |
| Baseline load path (images derivative cache) fills origin disk | Disk/volume alerts on the images store (id 18) | Edge `/images` param whitelist (Workers) as mandatory day-one; origin eviction follow-up |

Cutover principle: **the origin behind Cloudflare is the same artifact that was audited at commit `18e0202`.** Nothing is "tuned live" as a substitute for the origin fixes. Edge rules are the containment; origin fixes are the cure.
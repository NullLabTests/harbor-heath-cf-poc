# 06 · Week-One Plan

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

Day-by-day cutover-readiness plan. Each item names an owner (Cloudflare admin, origin engineering, security, payments). Contingent items reference the audit records (ids from `engagement/04`).

**Assumption:** the frozen origin artifact is commit `18e0202` (or the approved fork with the blocking fixes from §Launch blockers applied and re-validated).

---

## Monday — DNS, TLS, and origin lock

| # | Task | Owner | Exit criteria |
|---|---|---|---|
| 1 | Create the zone, add `www/api/admin/hooks/media` hostname records as **proxied** DNS | Cloudflare admin | Records resolve via Cloudflare; origin IP never returned |
| 2 | Enable TLS **Full (strict)** for all hostnames; upload/verify the origin cert | Cloudflare admin + origin eng | Handshake test per hostname succeeds; no mixed-content warnings |
| 3 | Harden origin ingress: firewall/security-group allows **Cloudflare proxied IP ranges only**; HTTPS-only | Origin eng (+ security) | External probe of origin IP: no listener reachable (ids 10, 22) |
| 4 | Pre-cutover blast-radius: DB loopback-bound, 5432 un-published; rotate `postgres/postgres` to generated secret (id 22) | Origin eng (ops) | No DB listener beyond localhost; old default credentials invalid |
| 5 | Provision and rotate **real `session.cookieSecret`**; set explicit cookie flags (secure, httpOnly, SameSite=Strict) (id 8 + hardening) | Origin eng | Cookie set-cookie shows explicit flags; forged `keyboard cat` cookies rejected |

## Tuesday — Access on admin + operator APIs

| # | Task | Owner | Exit criteria |
|---|---|---|---|
| 6 | Enable **Zero Trust Access** on `admin.harbor-heath.example` (+ `/admin/*`, `/oms/*` operator API paths): SSO + MFA for all operators | Cloudflare admin + security | Anonymous/API-only callers to admin get 403 at edge; operators log in via IdP only |
| 7 | Define the break-glass exception procedure (2 pre-authorized operators, time-boxed, audited) (edge-design §7) | Security | Procedure documented and rehearsed |
| 8 | Autoscale note: confirm Access connector reachability from operator network; no IP allowlist for admins (identity not IP) | Cloudflare admin | Operators on shared/office IPs unaffected by Access policy |

## Wednesday — WAF baseline + sensitive-endpoint rules

| # | Task | Owner | Exit criteria |
|---|---|---|---|
| 9 | Managed WAF on; **custom rules** (shadow→observe→block) for the sensitive paths: JWT-mint (`/api/user/tokens`, `/api/customer/tokens`, `*token/refresh`), password reset, `/admin/*` fallback | Cloudflare admin + security | Rules in observe first; log review confirms expected traffic; set to block (ids 9, 15) |
| 10 | Strict **rate limiting** on `api.*` (auth-tier-ish budgets per edge-computed client IP); cover locale-prefixed path aliases (id 9 countermeasure) | Cloudflare admin | Synthetic test: burst past limit → 429 at edge regardless of origin tier |
| 11 | **Workers header normalization** on `api.*`, `hooks.*`, `www.*`: strip/overwrite client `X-Forwarded-For` from the edge-computed client IP; inject `Referrer-Policy: no-referrer` globally (ids 10, 15) | Cloudflare admin + security | Origin sees only edge-canonic XFF; response headers include Referrer-Policy |
| 12 | **Turnstile** on checkout/login/reset surfaces (`/api/carts/*`, `/api/orders`, payment-intent mint, PayPal create, reset, login) (ids 1–3, 7, 9) | Cloudflare admin + security | Interactive challenges appear on forms; API-only abuse blunted |

## Thursday — Bot on pay and search + cache exclusions

| # | Task | Owner | Exit criteria |
|---|---|---|---|
| 13 | **Bot Management** on checkout + login/reset + gift-card/store-pickup endpoints; keep search readable (manageable bot traffic) | Cloudflare admin + security | High-scoring automation challenged on checkout; search not broken |
| 14 | **Cache Rules** (conservative): `www.*` cacheable pages (long TTL, cache-busting on stock), all `api.*` bypass, **no cache for account/session state**, preview `?changeset=` paths bypass + not indexed (id 24) | Cloudflare admin + origin eng | Assert: checkout/login responses never cached; draft page never served cached |
| 15 | **Workers media security** on `media.*`: block active content (`.svg`/`.html`-typed) from pickup, canonicalize `/images` params (whitelist w/h/q/f), Session-consistent origin (ids 17–19) | Cloudflare admin + security | Script-capable upload served → blocked at edge; `/images` accepts only allowlisted params |
| 16 | **API Shield schema validation** on `/graphql` (interim depth/complexity/alias caps) targeting id 16; review report before enabling strict blocking | Cloudflare admin + security | GraphQL abuse sampled/blocked; storefront contract unaffected |

## Friday — Webhook path, logging, origin lock, exception process

| # | Task | Owner | Exit criteria |
|---|---|---|---|
| 17 | **Webhook Workers** on `hooks.*`: signature re-verification (Stripe constructEvent equivalent + provider allowlist), event-type allowlist, forward only to hooks-origin route (id 4) | Cloudflare admin + security + payments | Signed, allowlisted events flow; unsigned/unknown rejected at edge |
| 18 | Confirm provider **webhook endpoints are registered against `hooks.*`** test URLs; test-mode event replay (Stripe) & PayPal webhook simulator | Payments + origin eng | Test events processed; delivery retries OK; origin lock intact |
| 19 | **Logpush / security analytics**: capture edge events for `admin.*`, `api.*`, `hooks.*`; dashboards for 429/403 spikes and webhook accept/reject counts | Cloudflare admin | SIEM/analytics has week-one baseline dashboards |
| 20 | **Exception process**: documented runbook for (a) WAF rule off-switch, (b) Access break-glass, (c) rate-limit tuning, (d) webhook-filter exceptions — every exception logged with expiry (edge-design §7) | Security | Runbook reviewed by security lead |
| 21 | **Cutover rehearsal**: dry-run the DNS cutover + origin allowlist flip on the non-prod zone | Cloudflare admin + origin eng | Same exact steps rehearsed; rollback path verified |

## After week-one (backlog, owned)

- Object-level authorization predicates on cart/order/payment intent mint / PayPal create (ids 1–3) — **origin eng**
- Webhook order-state guard (still-pending gate; cancelability routing; rollback events) (id 4) — **origin eng**
- Capture/refund atomic gates + adapter idempotency (ids 5–6) — **origin eng** (+ payments for provider idempotency confirmation)
- JWT access status re-check + refresh rotation/revocation (ids 11–12) — **origin eng**
- Session rotate/destroy + login lookup wildcard fix + reset POST-only code (ids 13–15) — **origin eng**
- Image pipeline: derivative-key allowlist + eviction, SVG pixel cap + fetch size cap, resolved-IP SSRF check (ids 18–20) — **origin eng**
- Upload content sniffing + sanitized anchor href (ids 17, 23) — **origin eng**
- cancelOrder restock symmetry + dead-knob wiring (id 21) — **origin eng**
- Origin rate-tier fix (mint paths → auth tier) and `TRUST_PROXY_HOPS` = real depth (ids 9–10) — **origin eng**
- Re-run of the audit fixtures on the fixed commit (validation plans in `NEEDS-VALIDATION.md`) — security + origin eng
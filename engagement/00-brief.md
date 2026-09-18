# 00 · Engagement Brief

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Period:** 2026 · **Audit run:** `harbor-heath-run-1`

---

## 1. Customer profile

Harbor & Heath Outdoor Co. (fictional) is an EU omnichannel retailer: a direct-to-consumer website plus approximately 40 physical stores, selling apparel, footwear, and outdoor equipment. Traffic peaks around holiday and weather events. The business depends on card payments, account login, store pickup, gift cards, and partner inventory feeds.

The company is replatforming a brittle monolith onto a self-hosted Node commerce stack consisting of:

- a public storefront,
- an operator `/admin` (plus operator APIs),
- store/catalog/checkout APIs, and
- payment/shipping/tax webhooks.

## 2. Business risk

- **Revenue at stake:** checkout and payment flows are the core money path. A defect here is a direct revenue and trust risk.
- **Regulatory:** payment card traffic (PCI DSS scope), EU consumer data. Account/PII exposure has direct regulatory exposure.
- **Operational:** a small security team cannot babysit scanner output; engineering will not stop feature work for unprioritized noise.
- **Reputation:** outdoor/weather event peaks concentrate checkout pressure — scale events interact with rate-limit and cache defects.

## 3. Go-live constraint

Go-live behind Cloudflare is roughly **eight weeks out**. Two engineering expectations frame this audit:

1. "We will tune rules after launch" is **not acceptable** for `/admin` and checkout. These surfaces must be materially ready in week one.
2. Cloudflare terminates TLS, hides the origin, and provides WAF, rate limiting, bot management, and cache. This POC must therefore separate **origin defects** (Cloudflare cannot fix) from **edge-enforceable controls** (week-one actions) and **unproven leads** (do not block on them; validate them).

## 4. In-scope systems

The origin twin of the commerce engine (public codebase at commit `18e02022631c1c9dba1b6e5ac734ae27b54632cb`):

- Web/protocol/auth: sessions, cookies, CSRF, password reset, admin default routes, JWT issuance
- Storefront and operator GraphQL + REST APIs, object-level authorization
- Cart/checkout/order lifecycle, Stripe and PayPal webhooks and admin callbacks
- Account management, store pickup, cache/personalized content, SSR/HTML injection on product/CMS content
- File/image upload and the image-processing pipeline
- Secrets in config, CI, Docker, `.env` examples; compose deployment posture
- Rate limiting, CORS, host header, TLS-termination assumptions
- Data isolation/lifecycle, resource exhaustion, webhooks/RPC, client-side controls as they affect the origin

## 5. Out-of-scope systems

- Kernel, OS, and memory-safety of unrelated native runtimes (Node/Postgres are the runtime; no in-repo safe-language native parsers handle attacker bytes)
- AI/LLM systems (no such trust boundary exists in the target)
- Desktop/mobile/local IPC surfaces (none exposed)
- Post-launch Cloudflare-specific configuration tuning and live traffic tuning
- The Cloudflare account/control-plane configuration itself (this POC covers the *origin*)

## 6. Rules of engagement

- **Source-local only.** All analysis is against the checked-out origin source. No attacking of any public Evershop or Harbor & Heath infrastructure.
- **No origin code changes.** The target worktree was kept read-only and clean (`worktree_dirty: false`).
- **No remote contacting.** No deployed endpoints, no external/shared services, no live credentials. Webhook and provider behaviors that require external observation are recorded as `needs_validation` with an owner-observed plan, never fetched.
- **No invented runtime proof.** This environment lacks an OS-enforced sandbox; the skill's universal rule is to record runtime-dependent results as unverifiable rather than fabricate them.
- **Artifacts:** skill artifacts live under `audit/run-1/`; the customer pack under `engagement/`. No files were written into the origin worktree.

## 7. Success criteria for this POC

1. **Clarity:** the customer can answer — *origin defects vs edge-enforceable vs unproven* — for every record. No `needs_validation` row is presented as a vulnerability, and no row lacks an exact unresolved fact.
2. **Source grounding:** every finding and lead points at real repository files and lines.
3. **Actionable week one:** the controls in `engagement/04` and `engagement/06` are implementable with allowed Cloudflare products and named owners.
4. **Reproducible:** validators (`validate-findings.cjs`, `validate-coverage-ledger.cjs`) pass; the run log (`engagement/08`) lets a second run be additive rather than starting over.
5. **Honest:** confirmed = 0 is reported as a *capability limit* (no sandbox), not as a clean bill of health.
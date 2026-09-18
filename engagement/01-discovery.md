# 01 · Discovery

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

This document records the questions a solutions engineer would ask the customer, the assumptions we made because this engagement ran against a **public architectural twin** of the origin, and the identity/payment/admin/webhook/cache assumptions that are explicitly *assumptions* until the customer confirms them.

---

## 1. Discovery questions an SE would have asked

### Identity & accounts
1. What is the real admin population — headcount, shared vs individual accounts, MFA status today?
2. Are storefront customers separate from the ~40 physical-store staff accounts, and how is store pickup identity handled?
3. What password rules and account-lockout behavior does the business expect (EU retailers commonly need self-service reset + no indefinite lockout)?
4. Who owns password-reset email delivery, and is there an off-origin email-sending service?

### Payments
5. Which payment providers are contractually final — Stripe, PayPal, both? Who holds the merchant account keys?
6. Are any gift cards, store credit, or loyalty balances stored in the origin and subject to authorization checks?
7. Who can void/capture/refund, and does the business need segregation (operator ≠ refund approver) or is single-admin fine for launch?

### Admin & operators
8. What device population uses `/admin` (managed devices, BYOD)? Is SSO/MFA acceptable for all operator access, or is there a legacy integration that needs plain login?
9. Which operator APIs are consumed by store systems (POS feeds, inventory partners) vs humans?

### Webhooks & third parties
10. Do partners integrate via webhooks-in (feeds) or are all integrations outbound polling? Who registers webhook endpoints?
11. Which external parties need signed callbacks (shipping, tax) beyond Stripe/PayPal?

### Caching & content
12. What is the fastest-moving storefront content (product pages, CMS/landing, stock badges) — what is cacheable, what must never be cached?
13. Is there any draft/preview workflow (page builder) whose content could leak wholesale?

### Deployment
14. Where does the origin actually run (cloud provider, region, IP range) so the Cloudflare allowlist can be scoped?
15. Who can change `TRUST_PROXY_HOPS`, DB credentials, and cookie secret, and is there a secrets pipeline already?

---

## 2. Assumptions we made because this is a twin repo

| Topic | Assumption | Why it matters |
|---|---|---|
| **Origin identity** | The Evershop v2.2.1 codebase at commit `18e0202` is a faithful architectural twin of the real Harbor & Heath origin. | Every trace/line reference is accurate for this commit. If the real origin diverges, re-run the validation steps on the real repo. |
| **Deployment** | Go-live is "stockself Node stack behind Cloudflare, origin not otherwise exposed." | Rate-limit and XFF leads (`rate-limit.*`) hinge on whether a single overwriting proxy fronts the app — a deployment fact, recorded as such. |
| **Admin model** | The single `"admin"` role in source is the real operator model at launch. | Cloudflare Access SSO on `/admin` is recommended precisely because origin RBAC is minimal. |
| **Payment processing** | Stripe (main) and PayPal (secondary) with signature-verified webhooks. | Webhook leads depend on provider event-delivery semantics (idempotency of `PayPal-Request-Id`, Stripe delivered-event ordering), which no source can settle. |
| **DB access** | Postgres runs where compose renders it (published 5432). | The `postgres/postgres` default-credential lead is unfixable-by-source and must be addressed operatively. |
| **No edge in front of origin in the twin** | For hardening review we evaluate the origin *as if directly exposed*; the report separately recommends the week-one edge. | This forces XFF/trust-proxy and cache leads to be stated as deployment questions, not assumptions. |
| **Mail** | Reset emails are delivered by a real provider with possible link-prefetch. | The reset-token-in-URL lead's crux is browser/mail prefetch Referer behavior, which is deployment-observable only. |

---

## 3. Explicit assumptions by surface

### Identity
- **Assumption:** accounts are email+password with bcrypt-hashed storage (source-confirmed) and JSON JWT issuance on the public mint endpoints. 
- **Assumption (needs owner check):** account disable/deletion semantics are expected to revoke access immediately; the source shows JWT access tokens outlive it (lead `evershop-auth-jwt-access-no-account-recheck`).

### Payment
- **Assumption:** attacker-funded money-in to another user's pending order is rare but possible; the real exposure depends on provider event delivery and merchant key handling (leads `stripe.paymentIntent.mint.*`, `stripe.webhook.canceledEvent.*`, `paypal.createOrder.*`).

### Admin
- **Assumption:** every operator is an "admin"; there is no RBAC beyond the single role (source-confirmed). CSRF on the admin cookie is **not** exploitable cross-site (rejected record) but cookie hardening is still advised.

### Webhooks
- **Assumption:** webhook endpoints verify signatures correctly in source (Stripe: `constructEvent` on raw body — verified correct pattern). Delivered-event ordering is the open question.

### Cache
- **Assumption:** storefront pages are cacheable, checkout/login are sensitive; the page-builder preview-canonical drift lead (`evershop:pageBuilder:ssr:preview-cache-control-canonical-drift`) is a *shared/edge cache* question, not an origin cache question (source has no in-repo storefront HTML cache).
# 07 · Talk Track

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

---

## The 10-minute verbal briefing

**1. Problem (1 min).** We go live behind Cloudflare in ~8 weeks. The two things nobody wants to happen: security says "we'll tune rules after launch" for `/admin` and checkout, and engineering gets an unprioritized scanner dump that halts features. This POC was built to answer exactly four questions: what must be fixed in origin, what Cloudflare can enforce in week one, what remains unproven, and what the week-one control plane looks like.

**2. Method (2 min).** We ran a structured source-grounded audit against the origin codebase twin (commit `18e0202`): 27 coverage cells each hunted by an isolated agent, every candidate recorded with a real file:line trace, then independent verifiers tried to refute the highest-value records. Two machine checkers validated the output (both passed). Crucially, this environment had **no execution sandbox**, so the skill's rules *forbid* us from claiming runtime "confirmed" findings — everything runtime-dependent stays "needs_validation" with the exact unresolved fact. That is a discipline, not a gap: nothing is over-claimed.

**3. What we proved (3 min).** Verdict counts: 0 confirmed, 24 needs_validation, 1 rejected.
- The **object-level authorization gap in the commerce core** — carts, orders, and payment-intent minting bound only by a client-supplied uuid, no ownership check. A 122-bit uuid hides it day-to-day, but the control is absent, and the edge cannot synthesize it.
- The **Stripe webhook can downgrade a settled order** (no still-pending gate, no refund/restock/events).
- Admin **JWT-mint endpoints run under a ~15× weaker rate tier**, and per-IP buckets can be spoofed via client X-Forwarded-For when trust-proxy is mis-set.
- **Secrets/defaults**: cookie secret falls back to the public constant `keyboard cat`; compose publishes Postgres with `postgres/postgres`.
- An independent verifier **rejected** the admin-cookie CSRF claim — the browser Lax default blocks cross-site POSTs — so it is a hardening note, not a finding.

**4. What we did not prove (1 min).** Any claim needing a runtime result: anonymous PII reads, uuid-leak reachability, provider idempotency (PayPal), Stripe event-delivery ordering, deploy-time XFF and caching facts. Each of the 24 leads carries the exact blocker and the fixture that resolves it — none is labeled a production vulnerability.

**5. Recommended controls (2 min).**
- **Origin (blocking, before cutover):** ownership predicates on cart/order/payment objects; webhook settlement guard; atomic capture/refund; real cookie secret; DB not published with default credentials.
- **Edge (week one):** Cloudflare Access SSO on `/admin` and operator APIs; strict rate limits on credential + checkout endpoints with edge-computed IP; Workers header normalization (XFF + Referrer-Policy); Turnstile on checkout/login/reset; hooks Workers for webhook signature + event allowlist; conservative cache rules; media Workers blocking script-capable content and bounding `/images`.

**6. The ask (1 min).** Approve the week-one plan (engagement 06): three owners — Cloudflare admin, origin engineering, security — with the blocking origin items (ids 1–4, 8, 22) resourced before cutover, and the fixture re-runs on the fixed commit to close out the 24 leads.

---

## The 3-minute version (for an ecommerce director)

We audited the code that will run your store. Bottom line: the **wiring is basically sound** — sessions, JWT, webhook signatures, bcrypt are all used — but three things need doing before launch.

**First, your checkout core has an authorization gap.** Carts, orders, and payment requests are found only by a "ticket number" the customer carries, with no real "whose is this?" check. A 122-bit ticket number hides it in practice, but we shouldn't launch a store depending on a secret ticket number instead of a proper check. That is origin engineering work the Cloudflare rules **cannot** do — the edge can't know who owns an order.

**Second, your payment webhook could flip a paid order back to "cancelled"** if a late or duplicate cancellation event arrives — with no refund, no restock, and no notice. Again: a code fix in the origin.

**Third, the admin area and login need their guard up now.** We can put operator login behind Cloudflare's SSO with MFA in week one, and add proper rate-limiting against password guessing — those are edge changes we can ship immediately — but our origin review also found the admin sessions use a fallback "keyboard cat" secret on default installs and the database rides with a default password unless someone fixes it. Those are quick, cheap, do-it-now items.

One honest caveat: we did **not** run exploit-demonstration tests this pass — the environment had no sandbox, so nothing is "confirmed busted", everything is "verified in code, needs a smoke test in staging". I'd rather we fix and then test the top items than ship and tune later.

**So: yes to cutover, with three origin fixes, one secrets/patching hour, and the Cloudflare week-one plan — and a smoke test on the fixed code before we go dark.**
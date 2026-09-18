# Harbor & Heath Outdoor Co. — Origin Readiness Audit Report

**Run:** `harbor-heath-run-1` · **Report date:** 2026-09-18
**Target:** Evershop v2.2.1 (`packages/evershop` workspace) — public architectural twin of the Harbor & Heath commerce origin
**Source ref:** `18e02022631c1c9dba1b6e5ac734ae27b54632cb` (branch `dev`, worktree clean)
**Repo:** `/home/illy/workspaces/harbor-heath-origin/evershop`

---

## 1. Run Profile, Scope, and Execution Boundary

| Item | Value |
|---|---|
| Profile | `standard` |
| Budget | none set (all 27 ledger units assigned) |
| Execution policy | `sandboxed-source-and-local-only` |
| Sandbox available | **No** — no OS-enforced sandbox (no external-network isolation, sanitized env, resource limits, or descriptor-based promotion tooling) |
| Agent capacity available | `general` and `explore` sub-agents |
| Prior-run use | none (`prior_run_paths` empty) |
| Coverage waves | 1 (27/27 units assigned) |
| Hunters | 8 (`hunter-authn`, `hunter-checkout-rest`, `hunter-cms`, `hunter-customer-admin`, `hunter-graphql`, `hunter-infra`, `hunter-oms-admin`, `hunter-payments`, `hunter-storefront-content`) |
| Recon agents | 5 (`recon-1a` … `recon-1e`) |
| Independent verifiers (Phase 5, this session) | 6 (`verifier-checkout-idor`, `verifier-order-idor`, `verifier-stripe-mint`, `verifier-stripe-webhook`, `verifier-admin-csrf`, `verifier-cookie-secret`) |
| Scope paths | `packages/evershop/src`, `packages/postgres-query-builder`, `packages/create-evershop-app`, `Dockerfile`, `docker-compose.yml`, `docker`, `package.json`, `package-lock.json` |

Because no OS-enforced sandbox is available, **no target-controlled builds, tests, browsers, dynamic HTTP probes, or exploit payloads were executed**. Per the skill's no-sandbox rule, any candidate whose decisive fact requires a runtime result is recorded as **`needs_validation`** with the exact unresolved fact — never promoted to `confirmed`. This run is a **source-grounded partial pass**: it proves structure and code-path defects deterministically from source, and it leaves runtime confirmation to bounded fixtures and owner-observed staging checks.

**Verification depth (client-approved):** fresh independent verifiers were dispatched on the six highest-value records — 5 retained leads (cart IDOR, order IDOR, Stripe payment-intent mint, Stripe webhook downgrade, cookie-secret) plus the CSRF lead (which it rejected). The remaining `needs_validation` records carry exact unresolved blockers and bounded validation plans rather than confirmed claims; none is presented as a demonstrated vulnerability. The run is recorded complete with this disclosed verification scope.

**Deferred / out-of-scope coverage** (recorded, not hunted): kernel/privileged-interface and memory-safety of unrelated native runtimes, AI/LLM threat models (no such boundary in target), and desktop/mobile/local IPC. Cookie transport hardening was folded into the session/cookie unit rather than treated as a standalone class. A separate coverage-critic snapshot was not recorded during the interrupted handoff, so this report makes **no clean-coverage claim**.

---

## 2. Security Posture Summary

The origin is a mainstream Node commerce stack that ships with several **defense-in-depth defaults missing** and **object-level authorization widows across the checkout/payment core** — exactly the surfaces a retailer exposes to the internet eight weeks before go-live.

The strongest source-proven defect cluster is **broken object-level authorization on cart/order/payment objects**:

- Public storefront cart and GraphQL APIs bind a cart **only by client-supplied `uuid`** with no ownership predicate (`getCartByUUID` → `getCart`, predicates `uuid=? AND status=true`), and the API routes skip session middleware entirely ("Restful api should be stateless", `addDefaultMiddlewareFuncs.ts:224-225`).
- The public `Order(uuid)` GraphQL query returns order PII (customer email/name/IP address, shipping/billing addresses) with no ownership check, because `*.admin.graphql` is the only schema split and storefront `buildResolvers(false)` accepts the shared OMS Order type.
- Public Stripe and PayPal endpoints mint payment intents / repoint integration order ids by a client-supplied order uuid, and the Stripe webhook can downgrade a **settled** order to `canceled` without refund, restock, or events.

An **unexpected real mitigant** is present: cart/order uuids are `gen_random_uuid` (122-bit), so exploitation presupposes a uuid-leak channel (shared checkout links, referrers, logs, admin grids). That changed reachability — not the missing control. The webhook canceled-sink additionally requires the payment account secret key to cancel a live intent, which bounds the attacker-reachable surface (but the ordering-race downgrade on an authorized-but-capturing intent with delayed/redelivered cancellation remains).

Around the core sit **credential-surface and secret-handling gaps**: public admin/customer JWT-mint endpoints run bcrypt under the permissive `api` rate tier (120/60s, ~15× the `auth` tier of 8/15min — and 300/60s via a locale-prefixed spelling), the cookie/session secret falls back to the public constant `keyboard cat` on default and example-config installs, passwords-reset tokens ride in URLs, ILIKE login lookup leaves `_` wildcard-ambiguous, and compose ships PostgreSQL with `postgres/postgres` published on the network. The image pipeline has unbounded derivative-cache cardinality, unbounded SVG/external-fetch amplification, and an SSRF allowlist that checks hostname strings only.

CSRF on the admin session cookie, by contrast, was **refuted as a cross-site vulnerability** by independent verification (browser SameSite=Lax default withholds the cookie on the POST-only sinks) and is retained only as a hardening note.

**Bottom line:** no confirmed runtime-exploited finding (none can exist in a no-sandbox run), 24 source-proven, precisely-blocked leads that a fixtures instance or owner-observed staging can resolve, and 1 rejected lead. None of the 24 carries a severity label; the customer must decide which to validate first against week-one edge controls.

---

## 3. Confirmed Findings

**None.** The skill requires a complete source trace **and** a bounded observed result for `confirmed`; this environment has no OS-enforced sandbox, so no runtime result was produced and no record is confirmed. This is the explicit, disclosed limitation of the run, not an assertion that the origin is clean.

---

## 4. NEEDS VALIDATION — Lead Table

24 source-proven leads. Each is a prioritized lead, **not a confirmed vulnerability**; none carries a severity. Rows are ordered by checkout/payment/auth criticality.

| # | Fingerprint | Title | Boundary | Exact blocker | Local next step | Owner-observed deployment check |
|---|---|---|---|---|---|---|
| 1 | `checkout:getCartByUUID:unscoped` | Public cart/order APIs + storefront GraphQL bind carts only by client-supplied uuid (object-level authz gap) | checkout / storefront API + /graphql | Whether an unrelated HTTP client can reach a victim cart via a known uuid (needs runtime); uuid-leak channel existence | Anonymous cart-B PII read + item mutation + order placement vs sessionless control in a throwaway instance | Reproduce on staging from a second host with no session cookie; test `allowGuestCheckout` variant |
| 2 | `oms.graphql.order.query.uuid.unscoped` | Public storefront `Order(uuid)` returns any order's PII with no ownership check | storefront /graphql, OMS Order type | Anonymous PII read can't be executed end-to-end here; uuid v4 not brute-forceable | Add order B + C; anonymous `order(uuid:B){customerEmail …}` fixture | Anonymous /graphql order(uuid) on a real test order; inventory uuid-leak channels |
| 3 | `stripe.paymentIntent.mint.unbound.orderUuidLookup` | Public payment-intent mint bound only to client order uuid | checkout / Stripe | Live PG+Stripe path not exercisable; webhook sink may need merchant secret; victim pay = real money | Double-stub Stripe/pool; call handler body `{order_id: <ungranted uuid>}` → returns `client_secret` | Stripe test keys: mint from unauthenticated context; merchant-cancel intent → webhook flips order |
| 4 | `stripe.webhook.canceledEvent.downgrades.settledOrder` | Late/non-authoritative `payment_intent.canceled` can downgrade a settled order (no still-pending gate, no refund/restock/events) | stripe webhook / OMS payment state machine | Genuine delayed cancellation of an already-settled intent is a Stripe-delivery runtime fact | constructEvent-signed canceled fixture on stripe_captured/authorized/refunded rows vs payment_failed | Test account: auto-capture, then slow/redeliver a pre-existing cancellation; watch for bare downgrade |
| 5 | `oms.captureOrder.state-race` | Concurrent admin capture requests both pass `isCapturable` and invoke gateway settle | OMS admin / gateway adapter | Double gateway settle is provider behavior (Stripe rejects, PayPal mints) and needs concurrent runtime | Barrier + recording mock, parallel `captureOrder(uuid)` → 2 gateway calls + 2 capture rows | Provider dashboard scan for duplicate captures after double-click |
| 6 | `oms.refundOrder.amount-race` | Concurrent refunds pass remaining-amount gate and over-refund | OMS admin / gateway adapter | Provider refund semantics + concurrent runtime | Recording refund mock; 2×50% parallel refunds → sum>100% or ceiling holds | Provider dashboard: refunded total > captured total after concurrent refund |
| 7 | `paypal.createOrder.integrationOrderId.unbound.repoint` | Public PayPal create can repoint a pending order's `integration_order_id` to a new PayPal order (fixed `PayPal-Request-Id=order.uuid`) | checkout / PayPal webhook | PayPal real idempotency (replay vs new id) is external; wrapper window is ca. 6h TTL | Mock refusing idempotency: second POST changes `integration_order_id`, first approve URL dead | Real PayPal sandbox: buyer A then buyer B POST same order; record new-vs-replayed id |
| 8 | `evershop-auth-cookie-secret-static-default` | Cookie/session secret falls back to public `keyboard cat`; no provisioning path sets one | auth / session + cookie parser | Forged signed cookie needs a valid session id; off-origin cookie-injection channel unproven | Unit: fixture config → returns `keyboard cat`; cookieParser accepts it; no provisioner emits a secret | Read live `system.session.cookieSecret`; probe cookie-intel boundary reachability |
| 9 | `rate-limit.credential-endpoints.missing-auth-tier.jwt-mint` | Public JWT-mint endpoints run bcrypt under `api` tier (120/60s; 300/60s via locale spelling), not `auth` (8/15min) | auth / rate limiter | Offline 429 differential not executable here; edge throttling is a deployment fact | 9 wrong-password mints (no 429) vs 9th admin-login (429); 121st mint → first 429 | Confirm no WAF rule narrows these POSTs; decide acceptable per-IP guess budget |
| 10 | `rate-limit.ip-accounting.spoofable-xff.trust-proxy-default-1` | With `TRUST_PROXY_HOPS` unset, client XFF selects the rate-limit bucket key | auth / rate limiter / compose | Whether edge strips XFF is deployment topology; deterministic-but-executed proof needs bounded local run | Boot pinned stack, distinct `X-Forwarded-For` per request → no 429; repeat with `TRUST_PROXY_HOPS=0` | Confirm production XFF normalization and `TRUST_PROXY_HOPS` == real proxy depth |
| 11 | `evershop-auth-jwt-access-no-account-recheck` | Access tokens outlive account disable/password change (no DB status re-check on JWT path) | auth / JWT | Live 200-vs-401 observation needs runtime PG+HTTP | Static bonded checks complete; remaining step is staging-only | Stage: mint token, set `status=0`/change password, confirm bearer 200 until expiry |
| 12 | `evershop-auth-jwt-refresh-no-rotation` | Refresh tokens never rotate/revoke; survive password change | auth / JWT | Replay-before-after-password-change needs running app + scratch PG | Sandbox app: mint, change password, replay old refresh → new access token both times | Stage replay of a captured refresh token before/after reset |
| 13 | `evershop-auth-login-ilike-wildcard-underscore` | Login ILIKE escapes `%` only; `_` wildcard can bind a session to the wrong account | auth / login | Which row `.load() LIMIT 0,1` returns needs live PG with underscore-pair rows | Disposable PG: `a_b@x.com`/`acb@x.com` same password → POST login binds to `acb@x.com` | Query deployed tables for underscore-neighbor email pairs |
| 14 | `evershop-auth-session-lifecycle-no-invalidation` | Session id never rotated on login, never destroyed on logout; raw sid echoed; `saveUninitialized` default true | auth / sessions | Fixation requires running app vs connect-pg-simple store | Source trace complete; bounded local: anonymous GET obtains sid, login route not session-exempt | Confirm deployed `saveUninitialized`; HTTP trace of fixation + logout-reuse |
| 15 | `evershop-auth-reset-token-in-url` | Password-reset token travels/renders as URL query; no Referrer-Policy anywhere | auth / password reset | Real Referer emission is browser/prefetch + mail-client behavior | Static surface trace complete (reset.ts:27 → ResetPasswordPage.tsx:147); document default Referrer-Policy | Open emailed link in browser; click external link; check Referer + access logs for `?token=` |
| 16 | `graphql.storefront.unboundedQueryAmplification` | Storefront GraphQL executes unbounded depth/alias/fan-out, no validation, 100kb JSON only | storefront /graphql | Per-query growth curve needs sandbox measurement; edge depth/complexity caps unknown | Fixture schema, alias 10/100/500/1000, Category depth 10/20/40, fan-out 3/4/5 under caps | Confirm whether edge/WAF caps GraphQL depth/complexity/aliases in front of /graphql |
| 17 | `cms.getMulter:mimeAllowlist:clientDeclaredMimetypeOnly` | Upload allowlist trusts client-declared Content-Type; script-capable SVG can be stored under image types | cms / upload | Multer/busboy mimetype provenance + serve-static SVG Content-Type need sandbox/browser fixture | Bounded multipart POST: part `Content-Type: image/png`, bytes `<svg><script>` names `evil.svg` → passes; serve via static.ts | Confirm /api/files,/api/images admin-gated; /assets/* anonymous; no edge content strip |
| 18 | `cms.imageProcessor:derivativeCache:unboundedCardinalityNoEviction` | Public /images writes unbounded, never-evicted derivative files keyed by attacker `(w,h,q,f)` | cms / images DoS | Derivative byte-growth + dry-key regions need pinned sharp fixture; edge cap is deployment fact | Existing unit harness + scratch `CACHE_DIR`; N distinct tuples → N retained files, no cleanup | Watch `<root>/.evershop/images` size/count; confirm edge caps /images |
| 19 | `cms.imageProcessor:representationAmplification:svgUnboundedRasterPlusUnboundedFetch` | SVG raster caps disabled; unbounded external fetch buffered whole | cms / images DoS | Peak RSS / fetched-size amplification needs bounded fixture; hostile-SVG presence + IMAGE_ALLOWED_HOSTS are deployment facts | Sandbox: 20000×20000@300dpi SVG → measure RSS vs capped raster; loopback blob fetch size | Confirm presence/absence of SVG in operator storage + IMAGE_ALLOWED_HOSTS hosts |
| 20 | `cms.secureFetch:ssrfAllowlist:hostnameWithoutResolvedIpCheck` | Image-proxy SSRF allowlist checks hostname strings only (rebinding / internal reuse) | cms / SSRF | DNS rebind between check and connect needs loopback DNS/listener fixture; entries operator-controlled is deployment fact | `safe.test` → public IP first, 127.0.0.1 second; observe connection; high-port variant | List actual IMAGE_ALLOWED_HOSTS + provider storage hosts; verify no internal/shadowed resolution |
| 21 | `oms.cancelOrder.restock-asymmetric` | Cancel unconditionally restocks full ordered qty ignoring `manage_stock`; shipped-cancel over-restocks; `reStockAfterCancellation` is a dead knob | OMS / inventory | Stateful cancel+restock rollup needs scratch PG; shipped-cancel path needs cancelable payment flow | Place-order→cancel fixture for manage_stock false/true; shipped/partially_delivered cancel qty inflation | Confirm no extension reads `reStockAfterCancellation`; OMS balances after cancel-after-shipment |
| 22 | `deployment.compose.postgres-default-creds.public-port` | docker-compose: postgres/postgres, port 5432 published on 0.0.0.0 | deployment / config | Host exposure + deployed-credential state are environmental facts | Offline compose: start db, `psql -h database -U postgres -W` pw `postgres` succeeds; 0.0.0.0:5432 | ss/netstat bind + firewall/security-group ingress; rotate to generated secret, bind loopback |
| 23 | `evershop:catalog:editor:image:link:unsanitized-href` | Admin rich-text image blocks render unsanitized `link` as anchor href (potential stored XSS) | cms / catalog editor | `@evershop/editorjs-image` not vendored (link field reachability) + no sandbox for render/exec | Install editorjs-image offline; render fixture `{type:'image',data:{link:'javascript:…'}}` through Editor.tsx | Admin-author link stored and served to shoppers; confirm reach + execution |
| 24 | `evershop:pageBuilder:ssr:preview-cache-control-canonical-drift` | Draft/scheduled SSR pages lack Cache-Control/noindex; canonical replay risk at edge cache | pageBuilder / SSR / CDN | Edge cache key behavior (query-string strip, Set-Cookie handling) is deployment fact | Fixture store: compare live vs out-of-window vs `?changeset=` headers; route through query-stripping cache | Preview responses `private,no-store`+noindex; expired-out-of-window → 404 no-cache |

---

## 5. Rejected Record

`base-admin.session-cookie-csrf` — Admin session cookie without SameSite/CSRF token is **not** exploitable cross-site.

Independent Phase 5 verification (`verifier-admin-csrf`) refuted the claim from source: the code sets only `{ maxAge }` on the session cookie (plus `secure=false` in production) and never sets SameSite — so browsers apply the **default SameSite=Lax**, which does **not** attach the cookie to cross-site POST requests. Every claimed state-changing sink (`POST /api/orders/:id/capture`, `POST /api/files/*`, multipart upload) is POST-only and therefore unreachable cross-site regardless of the missing CSRF token or Origin checks. The record is retained (rather than a duplicate `needs_validation`) so future runs do not repeat the unsupported claim.

The underlying conditions — `secure=false` in production, and no explicit SameSite — remain **hardening notes** (see §6), not findings.

---

## 6. Hardening Notes and Positive Source Patterns

### Hardening notes (defense-in-depth; not vulnerabilities)
- **Session cookie flags:** `cookie.secure=false` is forced in production and SameSite is never set. Make `secure`/`httpOnly`/`SameSite=Strict` explicit, and consider requiring HTTPS-only origin delivery post-cutover.
- **CSRF for future GET/state surfaces:** the Lax default neuters the current POST-only sinks, but any future GET state-change would re-open the class; set SameSite=Strict now.
- **Admin RBAC:** the admin gate is a single `"admin"` role check, route-level only. Cloudflare Access SSO on /admin is the cheapest compensating control on the critical path.
- **Referrer policy:** no `Referrer-Policy` header exists anywhere; pair it with the reset-token fix.
- **Trusted-proxy posture:** `TRUST_PROXY_HOPS` has no default hardening; confirm it equals the real proxy depth behind Cloudflare.
- **Container hardening:** compose publishes the DB; bind to loopback and rotate the default credentials regardless of post-validation outcome.

### Positive source patterns
- **Stripe webhook signature verification is strict** — `stripe.webhooks.constructEvent(rawBody, sig)` over rawBody; this is the correct pattern and should be mirrored at edge (Workers signature check) as defense-in-depth.
- **Raw-body capture is used for signature checks** (not the parsed JSON body).
- **JWT credentials are stored hashed (bcrypt)**; rate-tier is the only weakening on the mint path.
- **Server-side PII lives in the GraphQL schema intentionally**, and the `.admin.graphql` / `.admin.resolvers.*` split is the right mechanism — the defect is its **enforcement**, not the modeling.
- **File-upload paths are admin-gated and object storage is Postgres-BLOB-backed**; the image processor runs server-side with pinning-friendly dependencies.
- **Order lifecycle uses a status registry with a cancelability gate** (`isCancelable`), which the webhook bypasses — a contained, fixable spot.

---

## 7. Coverage Summary

| Status | Count |
|---|---|
| `covered` | 5 |
| `candidate` | 22 |
| `blocked` | 0 |
| `deferred` | 0 |
| **Total units** | **27** (all wave 1) |

- 27/27 ledger units were assigned across 9 conjuncted hunters; every `candidate` unit carries ≥1 non-empty `result_fingerprints`.
- **Verifier linkage added in this session:** `local_checks` from `verifier-checkout-idor`, `verifier-order-idor`, `verifier-stripe-mint`, `verifier-stripe-webhook`, `verifier-admin-csrf`, and `verifier-cookie-secret` were recorded on the linked units (aggregate `reviewed_paths` = unit-level `reviewed_paths`; `method:"source"`, `artifact:null`).
- **Excluded attack classes** (recorded with reasons on the affected units): memory-safety/native-runtime and kernel surfaces, AI/LLM boundaries, desktop/mobile/local IPC, cookie transport as a standalone class.
- **Critic result:** the separate coverage-critic snapshot was not recorded before the interrupted handoff; accordingly this report makes **no clean-coverage claim**.

### Validator results (final)
- `node .agents/skills/security-audit/validate-findings.cjs audit/run-1/findings.json` → **PASS: 25 findings valid**
- `node .agents/skills/security-audit/validate-coverage-ledger.cjs audit/run-1/coverage-ledger.json` → **PASS: 27 coverage units valid**

### Verdict counts
| Verdict | Count |
|---|---|
| `confirmed` | 0 |
| `needs_validation` | 24 |
| `rejected` | 1 |
| **Total** | **25** |
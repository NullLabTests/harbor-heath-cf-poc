# Needs Validation — Prioritized Leads

**Run:** `harbor-heath-run-1` · **Source ref:** `18e02022631c1c9dba1b6e5ac734ae27b54632cb`

Every record below is a **prioritized lead, not a confirmed vulnerability**. None carries a severity (unproven leads have no severity). Each row states the source-proven trace, the **exact unresolved fact**, and the bounded local fixture and owner-observed deployment check that resolve it.

Runtime execution was not permitted in this environment (no OS-enforced sandbox), so **no lead below was demonstrated at runtime**. Do not treat these as live-test guidance; the fixture steps run only in a disposable sandbox (local) or on the customer's own staging data (deployment).

---

## 1. `checkout:getCartByUUID:unscoped` — Public cart/order APIs and storefront GraphQL bind carts only by client-supplied uuid (object-level authz gap)

**Boundary:** storefront REST (`/api/carts`, `/api/orders`) + storefront `/graphql`

**Source trace:**
- `packages/evershop/src/modules/checkout/services/getCartByUUID.ts:9` — thin passthrough of caller `uuid` into `getCart()`.
- `packages/evershop/src/modules/checkout/services/cart/Cart.js:185-186` — `getCart()` predicates are only `uuid = ?` and `status = true`.
- `packages/evershop/src/modules/checkout/api/addCartItem/addItemToCart.js` — REST cart mutation keyed by `:cart_id`.
- `packages/evershop/src/modules/checkout/api/createOrder/placeOrder.js:14` — order placement takes `cart_id` from the body.
- `packages/evershop/src/bin/lib/addDefaultMiddlewareFuncs.ts:224-225` — API/GraphQL routes skip the session middleware ("Restful api should be stateless"), so no server-side ownership binding exists to enforce.
- `packages/evershop/src/modules/checkout/services/orderValidator.ts:117` — guestCheckout gate hardcodes the auth requirement; it does not bind ownership.
- Cart uuid is `gen_random_uuid` (122-bit): independent mitigation affecting reachability, not the missing control.

**Blocker:** whether an anonymous/unrelated HTTP client can reach a victim cart's data/operations using only a known uuid; and whether a uuid-leak channel (shared checkout links, client JSON/urls, referrers, logs, support screenshots) actually exists on a live deployment. Both are decisive runtime facts not observable from source or a permitted local run.

**Local resolution:** throwaway instance at this commit (Postgres migrations up). Session X creates cart A with contact info; unrelated session Y (no cookies) (1) `GET/POST /api/graphql { cart(id:A){ customerEmail … } }` and assert A's PII returns while `myCart` is null; (2) `POST /api/carts/A/items` mutates A; (3) `POST /api/orders { cart_id:A }` creates an order row and flips A `status=false`; (4) scoped siblings (`/api/carts/me/items`, `myCart`) must NOT reach A. Read-only assertions first; mutations only on throwaway fixtures; tear down after.

**Deployment resolution:** owner replicates the fixture on staging (NODE_ENV=production) from a separate host/IP with no storefront session cookie and confirms cart-A read/mutation/order placement (incl. `allowGuestCheckout=false` variant on the gate). Operator fixtures only.

---

## 2. `oms.graphql.order.query.uuid.unscoped` — Broken object-level authz: public storefront `Order(uuid)` returns any order's PII

**Boundary:** storefront `/graphql` (route `access: public`)

**Source trace:**
- `packages/evershop/src/modules/oms/graphql/types/Order/Order.resolvers.js:11` — `Query.order(uuid)` is the only order resolver; resolves via `getOrdersBaseQuery()`.
- `packages/evershop/src/modules/oms/services/getOrdersBaseQuery.js` — bare `select().from('order')`, no `customer_id` predicate.
- `packages/evershop/src/modules/graphql/services/buildTypes.js:19` — excludes only `*.admin.graphql`; shared `Order.graphql` merges into the public storefront schema.
- `packages/evershop/src/modules/customer/pages/frontStore/orderView/index.ts:9-13` — documented guest track-order contract (not the vector used).
- PII fields exposed server-side: `customerEmail`, `customerFullName`, `userIp`, `shippingAddress`, `billingAddress` (fullName/telephone/address1).

**Blocker:** end-to-end anonymous PII read cannot be executed here (no sandbox); uuid is v4 (not brute-forceable), so reachability depends on an un-verified uuid-leak channel — a reachability nuance, not a refutation of the control defect.

**Local resolution:** dev-mode throwaway Postgres; customer A + order B. Anonymous `POST /graphql { order(uuid: <B>){ orderNumber customerEmail customerFullName userIp shippingAddress{…} } }` → assert PII returns. Positive control: second known uuid identical. Negative control: `Customer{ orders }` under a customer token still filters by `customer_id`.

**Deployment resolution:** same anonymous /graphql query against a real checkout from a test account; capture the 200 PII body; confirm no deployment-specific resolver adds an ownership predicate; inventory reachable uuid-leak channels (order emails, admin grids, referrers).

---

## 3. `stripe.paymentIntent.mint.unbound.orderUuidLookup` — Public Stripe payment-intent mint bound only to a client-supplied order uuid

**Boundary:** checkout / Stripe (`POST /api/stripe/paymentIntents`, route `access: public`)

**Source trace:**
- `packages/evershop/src/modules/stripe/api/createPaymentIntent/route.json` — public.
- `packages/evershop/src/modules/auth/api/global/[getCurrentUser]auth.ts:18` — auth middleware short-circuits on public routes.
- `packages/evershop/src/modules/stripe/api/createPaymentIntent/createPaymentIntent.ts` — loads order by client `order_id` uuid + `payment_method='stripe'` + `payment_status='pending'` only; creates a real Stripe PaymentIntent; returns `client_secret` to the caller.
- `packages/evershop/src/modules/stripe/api/stripeWebHook/[bodyJson]webhook.ts` — the same uuid is later the sole binding for order payment-state transitions. Amount is server-computed.

**Blocker:** end-to-end mint against a real Postgres row + Stripe API not exercisable; per Stripe's model a `payment_intent.canceled` delivery requires cancellation, which uses the **account secret key** — the webhook canceled-branch sink may be unreachable by an attacker holding only the returned `client_secret`. Money-in transitions fire only if the attacker actually pays the full amount from their own card. Impact shape (attacker-funded fulfillment of arbitrary pending orders; duplicate `payment_transaction` rows re-emitting `order_placed`) needs a live merchant Stripe account.

**Local resolution:** scratch DB row (`payment_method='stripe'`, `payment_status='pending'`, random uuid). Double-stub Stripe (create → fixture intent with client_secret) + pool; invoke handler with no auth middleware and body `{ cart_id:'x', order_id:'<uuid caller was never granted>' }`; assert a `client_secret` returns (missing-ownership-predicate proof) and that an unknown/purged uuid returns `INVALID_PAYLOAD`.

**Deployment resolution (Stripe test keys):** (1) create a real order, note its uuid; (2) from unauthenticated context POST `/api/stripe/paymentIntents` with it and get a `client_secret`; (3) merchant-cancel that intent in test mode and observe webhook flip to `canceled`; (4) attacker-completed payment on another user's pending order → observe `amount_capturable_updated`/`succeeded` and duplicate `order_placed` emissions.

---

## 4. `stripe.webhook.canceledEvent.downgrades.settledOrder` — Missing ordering-safe settlement guard lets a late/non-authoritative `payment_intent.canceled` downgrade a settled Stripe order

**Boundary:** stripe webhook / OMS payment state machine

**Source trace:**
- `packages/evershop/src/modules/stripe/api/stripeWebHook/[bodyJson]webhook.ts` (canceled branch) — writes `payment_status='canceled'` for any state except already-canceled, including `stripe_captured`/`stripe_refunded`; bypasses `cancelOrder`'s `isCancelable` gate (`stripe_captured` registers `isCancelable:false`).
- `packages/evershop/src/modules/oms/services/updatePaymentStatus.ts` — checks only that the status exists in the registry; no still-pending requirement (unlike the `payment_failed` branch). No void/refund/restock/shipment-rollup/`order_canceled` emission; `changePaymentStatus` hookAfter re-projects `order.status='canceled'` via `'canceled:*'`.

**Blocker:** whether a genuine Stripe-delivered `payment_intent.canceled` can reach an already-settled order — Stripe's model says a single intent that reached `succeeded` cannot later be canceled — so the downgrade needs delayed/redelivered cancellation of an earlier state or a race on an authorized-but-capturing intent. External delivery-ordering runtime fact; signed-fixture execution impossible here (no sandbox).

**Local resolution:** harness calling the webhook route with a `stripe.webhooks.generateTestHeaderString`-signed `payment_intent.canceled` against rows at `stripe_captured`/`stripe_authorized`/`stripe_refunded`/pending; assert non-pending rows flip to canceled while `payment_failed` does not, with no payment_transaction/refund/order_canceled created. Refutation check: `createPaymentIntent` on a `stripe_captured` order returns `Invalid order` (bounds multi-intent surface to the pending window).

**Deployment resolution:** test Stripe account: place an order, auto-capture to `stripe_captured`, then deliberately redeliver/retry a pre-existing `payment_intent.canceled` (or pause+resume webhook delivery to force a late cancellation racing a manual capture); observe whether the order is downgraded to canceled with no refund/restock; monitor the `order.status` reset.

---

## 5. `oms.captureOrder.state-race` — Concurrent admin capture requests can both pass `isCapturable` and invoke gateway settle

**Boundary:** OMS admin / gateway adapter

**Source trace:** `packages/evershop/src/modules/oms/api/captureOrder/captureOrder.ts` (gate read + settle write are not conditional on an atomic `UPDATE … WHERE payment_status=<authorized>`; no lock). Provider behavior: Stripe rejects second capture of an already-captured PaymentIntent; PayPal accepts multiple captures of one authorization and mints a new capture id per distinct `PayPal-Request-Id`; idempotency not enforced by core.

**Blocker:** double settlement requires concurrent requests against a live DB + gateway with a barrier; the second-settle money movement is external provider behavior.

**Local resolution (OS sandbox, docker + network isolation):** seed order `payment_status=<method>_authorized` + one authorization transaction row; capture-handler mock waits on a barrier, records each invocation, returns fresh transaction ids; two parallel `captureOrder(uuid)` calls, release barrier after both enter `factory.capture`; assert 2 gateway calls + 2 capture rows commit. Show single settle when the fix gates on conditional `UPDATE order SET payment_status=<captured> WHERE payment_status=<authorized>` (or `pg_advisory_xact_lock`).

**Deployment resolution:** provider capture logs scan for duplicate capture entries against the same authorization/PaymentIntent after near-simultaneous admin clicks on the Capture button.

---

## 6. `oms.refundOrder.amount-race` — Concurrent refund requests can all pass the remaining-amount gate and over-refund

**Boundary:** OMS admin / gateway adapter

**Source trace:** `packages/evershop/src/modules/oms/api/refundOrder/refundOrder.ts` (or adapter equivalent) — remaining-amount read is not atomic with the refund write; concurrent requests can each observe the full remaining amount. Provider semantics external (Stripe refunds per-PaymentIntent; PayPal may mint a new refund).

**Blocker:** real double-refund requires concurrent requests + a provider honoring both refunds; runtime disallowed and provider semantics not source-visible.

**Local resolution (non-destructive, sandbox):** seed one captured payment_transaction; recording refund mock returning distinct transactionIds; two parallel `refundOrder(amount=50%)` calls with a barrier so both pass the gate before either records; assert refund sum > 100% (race) or second rejected by a ceiling check (gate upheld) — either outcome discriminates the claim.

**Deployment resolution:** provider dashboard scan for any order whose cumulative refunded total exceeds captured total after a double-click/concurrent-refund scenario.

---

## 7. `paypal.createOrder.integrationOrderId.unbound.repoint` — Public PayPal create can repoint a pending order's `integration_order_id` to a new PayPal order

**Boundary:** checkout / PayPal (`POST /api/paypal/orders`, public)

**Source trace:** `packages/evershop/src/modules/paypal/api/createOrder/createOrder.ts` — loads order by uuid with no principal predicate; fixed `PayPal-Request-Id=order.uuid` (line 145) and `invoice_id=order.order_number` (line 73); `integration_order_id` write unconditional. Window bounded by `reconcileAbandonedOrders.ts:64-67` (default 6h TTL cancels pending paypal orders).

**Blocker (decisive):** whether PayPal replays the original order id under a fixed `PayPal-Request-Id` (idempotent → overwrite is a no-op) or mints a different id across statuses/time must be observed against the real `/v2/checkout/orders` API with real merchant credentials. Cross-principal knowledge of the pending uuid depends on peer-validated IDOR units.

**Local resolution (mock refusing idempotency):** seed pending paypal order O; two POSTs from different caller identities with same `order_id`; assert second returns its own `paypalOrderId`/`approveUrl`, `order.integration_order_id` changes, and the first approve URL dies. Proves handler write is unconditional (already source-visible); cannot prove the real repoint.

**Deployment resolution (real PayPal sandbox):** buyer A POSTs then buyer B POSTs the same O; record whether PayPal returns a new distinct id or replays A's id. If distinct: complete B's approval URL and verify capture recorded against O with B's id (money leg steered to attacker-controlled intent). If replay: overwrite is a no-op; re-assess.

---

## 8. `evershop-auth-cookie-secret-static-default` — Static, publicly-known default session/cookie secret (`keyboard cat`)

**Boundary:** auth / session + cookie parser

**Source trace:** `packages/evershop/src/modules/auth/services/getCookieSecret.ts` (fallback `'keyboard cat'`); `configExample.text:31` pins it as `cookieSecret`; `bin/install`/`.env.example` provision none; same value seeds `cookieParser` + both session middlewares (`addDefaultMiddlewareFuncs.ts`); sinks read `request.signedCookies['asid']`/`['sid']` and trust whatever session id survives verification. Note `express-session` is configured without `cookie.signed:true` (`addDefaultMiddlewareFuncs.ts:78-83`).

**Blockers:** forged signed cookie yields identity only if the sid matches a live connect-pg-simple row mapping to an active `admin_user`/`customer` (runtime/DB fact); no off-origin cookie-injection channel (subdomain/host-header/wildcard-cookie/CSP bypass) is demonstrated in source; live deployment may override the secret; which layer owns sid integrity needs runtime confirmation.

**Local resolution (no server/DB/contact):** unit-test `getCookieSecret` with fixture config → returns `'keyboard cat'`; cookieParser accepts a cookie signed with `'keyboard cat'` and rejects another secret (forge equivalence); statically assert no provisioner (createEverShopApp.createConfigFile, bin/install templates, `.env.example`) emits a `session.cookieSecret`; assert sess.cookie omits `signed:true`.

**Deployment resolution:** (a) read live `system.session.cookieSecret`; (b) attempt a forged signed `asid`/`sid` for an attacker-derived sid and observe the lookup; (c) probe off-origin injection channel (host-header poisoning, XSS, unencrypted link UXSS); (d) verify whether forging sid integrity actually reaches admin/storefront API.

---

## 9. `rate-limit.credential-endpoints.missing-auth-tier.jwt-mint` — Public admin/customer JWT-mint endpoints run bcrypt under the permissive `api` tier

**Boundary:** auth / rate limiter

**Source trace:** `packages/evershop/src/bin/lib/rateLimit.ts` tiers: `auth` 8/15min (AUTH_POST_PATHS), `api` 120/60s, `page` 300/60s. `/api/user/tokens`, `/api/user/token/refresh`, `/api/customer/tokens` are not in `AUTH_POST_PATHS` → `api` tier. Locale-prefixed spelling (`/fr/api/…`) lands on `page` tier (2.5×). JWT mint runs bcrypt credential checks.

**Blockers:** offline 429-differential not executable (no sandbox); whether an edge/WAF ruleset already throttles these POSTs is a deployment fact; the locale-prefixed escalation requires an enabled non-default locale.

**Local resolution (offline sandbox, commit 18e0202, dummy admin row):** 9 wrong-password mints from one IP → 200/401 on all 9 (no 429) while a 9th `/admin/user/login` → 429 (auth tier), a ~15×/window differential (225× via the api-tier count); continue to the 121st mint → first 429; with NODE_ENV=production + one enabled locale, `/fr/api/user/tokens` ×300 → no 429 until 301 (page tier). Stop at first observed 429; dummy credentials only.

**Deployment resolution:** confirm no ingress rule narrows these POSTs and the enabled-locale set; decide whether 120/min (or 300/min) sustained per-IP credential guessing is acceptable. Remediation intent: move the mint/refresh paths into `AUTH_POST_PATHS` or make auth-tier assignment route-metadata-driven; add classification unit tests.

---

## 10. `rate-limit.ip-accounting.spoofable-xff.trust-proxy-default-1` — Per-IP rate-limit buckets are attacker-influenceable when `TRUST_PROXY_HOPS` defaults to 1

**Boundary:** auth / rate limiter / compose

**Source trace:** `getTrustProxyHops.ts` returns 1 when `TRUST_PROXY_HOPS` unset; `docker-compose.yml` publishes `0.0.0.0:3000` with no proxy service → direct client = single trusted hop → client `X-Forwarded-For` is authoritative for `req.ip`; the rate-limit MemoryStore keys per-IP buckets on `req.ip`.

**Blocker:** whether production actually runs behind a single real proxy/edge that rewrites/strips XFF before the app; deterministic-but-executed proof (pinned Express 4.22.2 / express-rate-limit 8.7.0) needs a bounded local run (no sandbox).

**Local resolution (offline sandbox, pinned lockfile):** boot with `TRUST_PROXY_HOPS` unset on :3000; 10 `POST /admin/user/login` each carrying distinct `X-Forwarded-For` → assert no 429 and bucket key changed (read `RateLimit-*` headers); repeat with `TRUST_PROXY_HOPS=0` → XFF no longer alters `req.ip`.

**Deployment resolution:** confirm production XFF normalization and that `TRUST_PROXY_HOPS` == real proxy depth (0 direct, 1+ per hop behind Cloudflare's overwriting proxy); record any firewall/WAF in front of :3000.

---

## 11. `evershop-auth-jwt-access-no-account-recheck` — API access tokens are honored after account disable/deletion and password change

**Boundary:** auth / JWT

**Source trace:** `packages/evershop/src/lib/util/jwt.ts:98-131` `verifyToken` does no DB lookup and no jti/version check; `jwtUserAuth`/`jwtCustomerAuth` attach decoded claims with no `status=1` re-read; `[getCurrentUser]auth.ts:15-51` gates on uuid+roles only; refresh endpoints' `status=1` re-read (auth:30-34, customer:30-34) proves the intended inactivation the access path lacks.

**Blocker:** decisive 200-vs-401 observation needs runtime (mint → set `status=0` → authorized request within TTL).

**Local resolution:** source-bonded checks complete (no execution). Remaining runtime step is staging-only.

**Deployment resolution:** staging: mint access+refresh via `/user/tokens`; set `admin_user.status=0` (or change password) in the DB; authorized write call with the original access token stays 200 until expiry (900s admin / 1800s customer); session+cookie and refresh paths return 401; restore test rows.

---

## 12. `evershop-auth-jwt-refresh-no-rotation` — Refresh tokens have no rotation/revocation/family and survive password changes

**Boundary:** auth / JWT

**Source trace:** refresh endpoints issue new access tokens from an un-rotated refresh token; no family/jti tracking; `JWT_ADMIN_REFRESH_SECRET` global rekey is the only invalidation.

**Blocker:** replay-before/after-password-change needs the running app + scratch PG (no sandbox).

**Local resolution (scratch sandbox, never target repo/DB):** fresh secrets + disposable PG; mint refresh via `/user/tokens`; change account password; POST `/user/token/refresh` with the OLD refresh token → still returns fresh accessToken; call twice with identical refresh token → both succeed (no rotation/revocation); response contains no new refreshToken.

**Deployment resolution:** capture a refresh token to logs/debug dump; replay before and after a password reset; confirm no endpoint revokes a single token before 15h/30h expiry; confirm rekeying is the only global invalidation.

---

## 13. `evershop-auth-login-ilike-wildcard-underscore` — Login credential lookup escapes only `%`, leaving `_` as a wildcard

**Boundary:** auth / login

**Source trace:** `loginUserWithEmail`/`loginCustomerWithEmail` build `WHERE email ILIKE '<escaped email>' AND status=1` escaping `%` but not `_`; `.load()` `LIMIT 0,1` returns the first matching row under deployment row ordering.

**Blocker:** which row wins with underscore-differing email pairs (e.g. `a_b@x.com` vs `acb@x.com`) and a coinciding password is a data-state + runtime outcome.

**Local resolution (disposable PG 15 + app):** create both accounts with the same known password P; `POST /customer/login {email:'a_b@x.com', password:P}`; assert `session.customerID` resolves to `acb@x.com`. Repeat admin. Remediation intent: escape `_` and `\` (or exact `=` / proper escape-clause ILIKE) in both lookup services.

**Deployment resolution:** query deployed customer/admin tables for underscore-neighbor email pairs whose neighbor exists and shares/repeated a password.

---

## 14. `evershop-auth-session-lifecycle-no-invalidation` — Session id is never rotated on login, never destroyed on logout; raw sid returned

**Boundary:** auth / sessions

**Source trace:** `loginUserWithEmail`/`loginCustomerWithEmail` mutate the session in place (no `session.regenerate()`); `logoutUser`/`logoutCustomer` null one property then bootstrap `session.save()` same sid (no `destroy` anywhere in `packages/evershop/src`); JSON login endpoints echo `request.sessionID`; active session middleware defaults `saveUninitialized=true`; `SESSION_EXEMPT_ROUTE_IDS` excludes only images/static/adminStaticAsset (login is not exempt).

**Blocker:** fixation + post-logout sid reuse require running the app vs connect-pg-simple (no sandbox); deployed `saveUninitialized` (`configExample.text` sets `false`) and cookieSecret are owner-observable facts.

**Local resolution:** trace complete. Bounded local: fresh anonymous GET to a sessioned route obtains an sid; login route not session-exempt.

**Deployment resolution:** confirm deployed `saveUninitialized` and cookieSecret; HTTP trace showing GET creates anonymous session → client-proposed sid honored → login persists identity under that same sid (fixation) → logout leaves the sid reusable and still mapped to prior identity; validate the cookie-planting vector (XSS/forged-cookie/subdomain) via peer XSS lead.

---

## 15. `evershop-auth-reset-token-in-url` — Password-reset token travels and is rendered as a URL query parameter; no Referrer-Policy anywhere

**Boundary:** auth / password reset

**Source trace:** `POST /customers/reset-password` → `resetPassword.ts:27` → `sendResetPasswordEmail.ts:28` (48h/single-use bounds at `updatePassword.js:44,63-69`); SSR page reads `window.location.search` (`ResetPasswordPage.tsx:147`) and renders a hidden token input (line 74) posting to `/customers/password`; `grep` finds no `Referrer-Policy` header (only `static.ts:88` default headers).

**Blocker:** real Referer emission depends on browser default Referrer-Policy (modern defaults strip query strings cross-origin), mail-client prefetch/tracker behavior, and whether CDN/proxy/access logs capture `?token=` — none measurable without executing a real reset flow.

**Local resolution:** static surface trace complete; document effective browser-default Referrer-Policy conditions controlling cross-origin leakage.

**Deployment resolution:** live store with a known email: open the emailed link, (a) click an external link on the reset page and capture Referer at the external origin; (b) inspect `window.location.search`/history for the token; (c) review server/proxy/CDN logs for `?token=`; (d) observe mail-client prefetch. Remediation intent: strict global `Referrer-Policy` (no-referrer / strict-origin), POST-only one-time exchange code never in the URL, and strip token from URL after first load.

---

## 16. `graphql.storefront.unboundedQueryAmplification` — Untrusted storefront GraphQL queries parse/execute with no validation or depth/alias limits

**Boundary:** storefront `/graphql`

**Source trace:** `graphqlMiddleware.js` — `validateSchema(schema, document)` (wrong API: validates the schema against the document rather than the document against the schema), so documents are effectively unvalidated; no depth/complexity/alias caps; `body-parser` default 100kb JSON limits body size but the query shape (aliases, deep `Category` parent chains, fan-out) is unbounded; pinned `graphql 16.12.0`.

**Blocker:** the per-request growth curve (resolver/DB-statement count, CPU/wall vs selection size, alias count, Category depth) needs sandbox measurement; whether an edge/WAF/API gateway imposes operation caps in front of `/api/graphql` is a deployment fact.

**Local resolution (OS sandbox, dummy PG fixture):** build storefront schema; drive `graphqlMiddleware` at fixed points: aliases 10/100/500/1000; Category parent depth 10/20/40; children fan-out 3/4/5 × branching 3; count DB statements and wall time under strict caps; stop at first superlinear observation; confirm an introspection query resolves.

**Deployment resolution:** confirm whether any edge/WAF/gateway rule limits GraphQL depth/complexity/alias cardinality in front of `/api/graphql`; if none, unbounded execution is reachable as described.

---

## 17. `cms.getMulter:mimeAllowlist:clientDeclaredMimetypeOnly` — Upload allowlist trusts the client-declared Content-Type, letting script-capable content pass as image

**Boundary:** cms / upload (`/api/files`, `/api/images`, private; storefront `/assets/*`)

**Source trace:** multer `fileFilter` (multer ^2.2.0/busboy) bases the allowlist on the multipart part's declared `Content-Type`; filename extension not authoritative; `serve-static` (^1.15.0) serves stored files with their stored content type; script-capable `.svg` (or content served as `image/svg+xml`) executes on direct/iframe navigation (not `<img>`).

**Blocker:** multer/busboy mimetype provenance and serve-static SVG Content-Type + browser behavior need sandbox/browser fixtures (deps not vendored); reachability of `/api/files`,`/api/images` depends on admin credential (peer auth findings).

**Local resolution (OS sandbox):** unittest harness (`packages/evershop/src/modules/cms/api/fileUpload/tests/unit`) or bounded jest/multer fixture POSTing a multipart part with header `Content-Type: image/png` and bytes `<svg><script>…` named `evil.svg`; assert the fileFilter passes and storage holds the SVG bytes; serve via `static.ts` on isolated loopback; capture response Content-Type + body; confirm anonymous `GET /assets/media/…` reaches the media branch.

**Deployment resolution:** confirm admin gating of `/api/files`,`/api/images`; confirm anonymous `/assets/*` reachability at origin; confirm no edge WAF/content-inspection strips/blocks the stored `.svg` or admin-authored absolute URLs before assessing visitor-visible result.

---

## 18. `cms.imageProcessor:derivativeCache:unboundedCardinalityNoEviction` — Public `/images` writes unbounded, never-evicted derivative files keyed by attacker `(src,w,h,q,f)`

**Boundary:** cms / images (public GET `/images`, `imageProcessor.ts`, `imageProcessor.test.js` existing unit harness)

**Source trace:** derivative cache key is the client-tuple `(src,w,h,q,f)`; every miss dry-writes a retained file; no eviction/cleanup path; page-tier per-IP limiter (300/60s, MemoryStore, `req.ip` bucket) is the only in-app bound and shares the XFF weakness (lead 10).

**Blocker:** effective disk/cardinality growth + which extreme/negative w/h/q keys error vs succeed needs a pinned-sharp runtime fixture; operator-plausible retained-growth impact depends on deployed volume sharing, replica count, and whether the edge caps `/images`.

**Local resolution (strict OS sandbox, small scratch volume):** drive `/images` (or imageProcessor directly, reuse committed `media/` fixtures + scratch `BUILDPATH`) through distinct `(w,h,q,f)` tuples; assert (a) each miss returns a distinct file, (b) CACHE_DIR grows monotonically with tuple count and no call triggers cleanup, (c) which extreme/negative keys error (dry regions); measure per-file bytes to extrapolate bounded-budget growth.

**Deployment resolution:** check deployed `<root>/.evershop/images` size/count over a production window; confirm whether that volume is shared with logging/media stores; verify whether the edge (WAF/rate-limit) actually caps `/images` beyond the in-app per-IP page-tier limiter.

---

## 19. `cms.imageProcessor:representationAmplification:svgUnboundedRasterPlusUnboundedFetch` — SVG rasterization caps disabled; unbounded external fetch buffered whole

**Boundary:** cms / images (public GET `/images`, `imageProcessor.ts`, `secureFetch`, sharp ^0.33.5 / undici ^7.x)

**Source trace:** SVG branch removes the SVG-specific pixel/dimension cap; `secureFetch` buffers the full external body with no size cap; amplification = declared canvas × density (e.g. 20000×20000 @300dpi) vs default-capped raster, plus fetched-body buffering.

**Blockers:** exact memory/CPU spike for a given SVG canvas/density/body depends on pinned sharp/libvips/librsvg + undici; presence of a hostile SVG in any readable store location (default media/public/theme ship none; upload bypass is a needed precondition) and the `IMAGE_ALLOWED_HOSTS`/cloud-provider set are deployment facts.

**Local resolution (OS sandbox, low memory limit):** run the SVG branch on a 20000×20000 @300dpi hostile SVG; record peak RSS vs the same canvas under the default pixel cap and the amplification factor; fetch a bounded-but-sizable blob via secureFetch from loopback; record buffered size vs response size (confirming no cap). Stop at minimum observed amplification.

**Deployment resolution:** confirm presence/absence of (a) any SVG in operator media/public/theme storage (incl. via upload endpoint) and (b) any `IMAGE_ALLOWED_HOSTS`/cloud hosts a third party can reference; confirm the process memory supervisor/setting to weigh per-request spikes.

---

## 20. `cms.secureFetch:ssrfAllowlist:hostnameWithoutResolvedIpCheck` — Image-proxy SSRF allowlist validates hostname strings only (DNS rebinding/internal reuse)

**Boundary:** cms / SSRF (image proxy fetch)

**Source trace:** `secureFetch` allowlist (`IMAGE_ALLOWED_HOSTS` + provider-derived storage hosts) matches hostnames only; no resolved-IP check at connect time; `guardedConnector` checks hostname but connection-time resolution can differ (DNS rebinding); no port restriction.

**Blocker:** which hosts are allowlisted in the deployed store (operator-controllability, DNS shadows/history), and undici rebinding/port behavior, are deployment + runtime facts not in source.

**Local resolution (sandbox, loopback-only):** allowlist `safe.test`; first DNS query → public test IP (hostname check passes), connection-time query → 127.0.0.1; call `secureFetch('http://safe.test/x')`; observe whether the loopback listener receives a connection. Repeat the port dimension with an allowlisted name served only on an internal high port.

**Deployment resolution:** list actual `IMAGE_ALLOWED_HOSTS` + provider storage hosts; confirm each is fully operator-controlled with fixed A/CNAME records (no expired/shadowed subdomains, no custom S3/Azure/GCS endpoints into internal address space); confirm none resolves to non-public space.

---

## 21. `oms.cancelOrder.restock-asymmetric` — Order cancellation unconditionally restocks full ordered quantities, ignoring `manage_stock`

**Boundary:** OMS / inventory

**Source trace:** cancel path runs the restock UPDATE for full ordered qty regardless of `manage_stock`; shipped/partially_delivered rollup under a cancelable payment status over-restocks delivered goods; `oms.order.reStockAfterCancellation` exists but cancelOrder ignores it (dead knob).

**Blocker:** qty drift requires executing place-order trigger + cancel + restock against a PostgreSQL DB (no sandbox); the delivered-goods over-restock needs a shipped/partially_delivered row under a cancelable payment flow (bootstrap makes paid non-cancelable); availability/sales impact is runtime-evaluated at deployment.

**Local resolution (sandbox + scratch PG):** manage_stock=false and manage_stock=true products, one-item orders; place-order + `POST /api/orders/:id/cancel`; assert qty unchanged for false, restored for true; shipped/partially_delivered cancel → qty inflated by full ordered qty; set `reStockAfterCancellation=false` → still restocks (dead knob proof).

**Deployment resolution:** confirm no extension reads `reStockAfterCancellation`; confirm whether any live flow keeps payment cancelable at shipped/partially_delivered rollup; check OMS inventory after cancel-after-shipment against prior shipping records.

---

## 22. `deployment.compose.postgres-default-creds.public-port` — docker-compose ships PostgreSQL with hard-coded `postgres/postgres` and publishes it

**Boundary:** deployment / config

**Source trace:** `docker-compose.yml` — `POSTGRES_PASSWORD=postgres` (known constant), Postgres 16 service with `ports` exposing 5432; documented in architecture.md §Deployment; `Dockerfile` scope.

**Blocker:** internet/tenant reachability of 5432/3000 depends on host firewall/VM network/cloud security groups (compose doesn't control it); whether the deployed DB still runs the default credential is deployment state; default-credential auth can only be shown by booting the composed stack (no sandbox).

**Local resolution (isolated offline fixture):** render compose, start db with pinned values, from a second container on the same bridge run `psql -h database -U postgres -W` with password `postgres`; assert auth succeeds and the DB listens on 0.0.0.0:5432; tear down (non-destructive, dummy data).

**Deployment resolution:** observe whether 5432/3000 are exposed beyond loopback (ss/netstat bind state, firewall, cloud SG ingress) and whether the deployed DB still runs `postgres/postgres`. Remediation intent: generated credential via secret mount, DB port bound to loopback (or publish removed), non-mutable image tag.

---

## 23. `evershop:catalog:editor:image:link:unsanitized-href` — Storefront rich-text image blocks render an unsanitized `link` as an anchor href

**Boundary:** cms / catalog editor (admin authoring → storefront render)

**Source trace:** `components/common/form/Editor.tsx` image tool; rendered anchor href from block `data.link` un-sanitized; `@evershop/editorjs-image@^1.1.0` (declared, not vendored). Relevance for stored XSS / cross-principal script.

**Blockers:** whether editorjs-image saves/emits a `link` field (reachability through the admin UI) is not source-visible (not vendored); browser render/execution check impossible (no sandbox); whether a lower-trust authoring path exists without full admin rights is not source-visible.

**Local resolution (OSS sandbox, offline registry snapshot):** install `@evershop/editorjs-image@1.1.0`, instantiate with the same config as `Editor.tsx`; assert save/output can produce `data.link`; render fixture `{type:'image',data:{link:'javascript:alert(1)',file:{url:'…'}}}` through `Editor.tsx` and inspect emitted `<a>` href server + client output.

**Deployment resolution:** if reachability+execution confirmed, start the storefront with a fixture DB, author a product/category description whose image block carries `link:`, request the page as an anonymous shopper, record whether served HTML contains the un-sanitized `javascript:` href and whether it executes (stored cross-principal XSS).

---

## 24. `evershop:pageBuilder:ssr:preview-cache-control-canonical-drift` — Draft/scheduled SSR pages lack Cache-Control/noindex; canonical replay risk at edge cache

**Boundary:** pageBuilder / SSR + shared/edge cache

**Source trace:** draft/preview/schedule-windowed landing responses set no `Cache-Control`/`X-Robots-Tag`; canonical URL emitted regardless of draft state; `?changeset=<token>` decides draft bytes; no in-repo storefront HTML cache, so the replay step depends on an edge/shared cache.

**Blockers:** whether a deployment edge cache strips query strings for cache keys and honors/strips Set-Cookie before storing is an external deployment fact; no response-header observation possible (no sandbox).

**Local resolution (OS sandbox):** production-mode storefront vs fixture DB; record headers for (1) live landing, (2) out-of-window landing, (3) same URL + valid `?changeset=`; route token-ed and token-less requests through a query-stripping shared cache (Varnish/squid ignoring Set-Cookie/Vary); verify whether the token-less request returns draft bytes stored under the canonical key.

**Deployment resolution:** owner regression assertions: preview responses set `Cache-Control: private, no-store` + `X-Robots-Tag: noindex`; out-of-window/status=false landings respond 404 `public, no-cache`; confirm edge cache-key behavior (query-string + Set-Cookie/Vary handling) so the canonical token-less key can never host draft bytes.

---

## Order of business for the customer

1. Resolve leads 1–4 (cart/order/payment object-level authz + webhook state machine) first — they are the checkout/payment core Cloudflare cannot fix by rule alone.
2. Pair leads 9–10 (rate-limit tiers + XFF) with week-one edge rules (WAF/rate-limiter/bot) — these are the ones the edge *can* step in on immediately.
3. Lead 22 (compose DB defaults) is the cheapest operational fix and should happen before cutover regardless of validation outcome.
4. Leads 17–20 and 23 affect the image/upload surface; validate the upload-bypass precondition first, then the amplification/SSRF chain.
# 03 · Why This Method

**Engagement:** Harbor & Heath Outdoor Co. — Cloudflare origin-readiness POC
**Audit run:** `harbor-heath-run-1`

> **Method attribution:** the procedure this document explains is Cloudflare's open-sourced [`security-audit-skill`](https://github.com/cloudflare/security-audit-skill) (MIT; github.com/cloudflare/security-audit-skill), executed in FULL AUDIT MODE via the installed copy at `~/.agents/skills/security-audit`. Phase names, verdicts, schemas, and validators below are that skill's, verbatim.

This file is the customer explanation of the tool: why a generic "find vulns in this repo" prompt is the wrong input to an edge-control design, what the method actually did phase by phase in *this* run, who the agents were, and how to read the results. It is meant to be underlined by a security lead when briefing leadership.

---

## 1. Why generic SAST / "find me vulns" is the wrong input here

A generic scanner prompt produces a **flat list of suspicious things** with no notion of:

- **where the code sits on the trust boundary** (a public checkout endpoint is not the same as an admin-only service),
- **what a real attacker can actually supply** (route gating vs object-level ownership),
- **which defects the edge (Cloudflare) can neutralize** vs which are origin-only,
- **what is proven** vs what is inferred from source.

For a go-live-behind-Cloudflare decision in eight weeks, that ambiguity is poisonous: engineering gets an unprioritized list, security gets "we'll tune rules after launch", and nobody can say which defects *must* be fixed in origin.

This method inverts the shape of the output. Instead of bugs, it produces **claims organized by attack surface**, each with:

- a stable fingerprint (so the same root cause is the same record across runs),
- a source trace (entrypoint → propagation → sink) with real file:line references,
- a verdict discipline (`confirmed` / `needs_validation` / `rejected`) that literally forbids over-claiming,
- and a mapping to edge controls and residual origin work.

---

## 2. What the method is — and what it is not

- It is a **source-grounded security review process** executed by a parent orchestrator that decomposes a codebase and delegates, isolates, and reconciles independent analysis until records agree.
- It is **not** a fuzzer, not a penetration test, not a static analyzer that "knows" the framework, and not "an AI wand". Where runtime proof is required, the method demands it — and if the runtime capability is absent, it **says so** and downgrades the lead accordingly.
- It produces machine-checked artifacts (JSON ledgers and findings) that pass structural validators, so a second run is **additive**, not a restatement.

---

## 3. Phases as performed in THIS run

**Phase 1 — Reconnaissance.** A parent agent mapped the codebase: stack, module layout, trust boundaries, route classes (public / operator / callback), identity model, rate-limit tiers, secrets handling. Output: `audit/run-1/architecture.md` and a **coverage ledger** of 27 units — each unit = one (surface, boundary, subsystem, attack-class) cell, the grid the hunt will be measured against.

**Phase 2 — Coverage-led hunting.** Nine isolated hunter agents (`hunter-authn`, `hunter-checkout-rest`, `hunter-cms`, `hunter-customer-admin`, `hunter-graphql`, `hunter-infra`, `hunter-oms-admin`, `hunter-payments`, `hunter-storefront-content`) each owned disjoint ledger units. Each hunter recorded candidates as **ledger checks** with source traces and `result_fingerprints` — the machine key back to `findings.json`. All 27 units were wave-1 assigned (22 `candidate`, 5 `covered`).

**Phase 3 — Candidate validation.** Every distinct candidate is meant to go to a fresh verifier tasked to **disprove it**. In this run, candidate shaping and later record verification were merged per the client-approved verification scope (see §6), because the environment could not execute runtime steps anyway: no sandbox, no external contact.

**Phase 4 — Structured output.** The parent wrote `audit/run-1/findings.json` — 25 records, each matching exactly one of three verdict schema branches — then ran the two validators (`validate-findings.cjs`, `validate-coverage-ledger.cjs`) until both passed. The schemas are strict (`additionalProperties: false`): no stray fields, no missing required fields, no verdict that doesn't match its branch.

**Phase 5 — Independent record verification (key leads).** Six fresh verifier agents (`verifier-checkout-idor`, `verifier-order-idor`, `verifier-stripe-mint`, `verifier-stripe-webhook`, `verifier-admin-csrf`, `verifier-cookie-secret`) — none of whom hunted — independently re-read the cited source for the six highest-value records. **Result:** five leads retained as `needs_validation`; the admin-cookie CSRF lead was **rejected** (see §5). Their `local_checks` were recorded in the ledger linked to the affected units.

**Phase 6 — Reporting.** Reports (`REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`) plus this customer pack were written from the *final records*, not from hunting prose. The prose never changes a verdict, severity, blocker, or demonstrated impact.

---

## 4. Parent vs hunter vs verifier — and why the finder never confirms

| Role | Does | Does NOT |
|---|---|---|
| **Parent** | Decomposes scope, writes ledger + schema-shaped records, runs validators, reconciles fingerprints, writes reports | Marks its own candidates confirmed; claims any runtime result it did not observe |
| **Hunter** | Owns ledger units, records source traces and candidates | Confirms; verifies another hunter's claims |
| **Verifier** | Re-reads every cited source location; tries to **refute** the record; returns a schema-shaped corrected record | Confirms without independently reproducing decisive local results |

The **separation of hunter and verifier** is the method's anti-bias mechanism. A confirmation is only as good as the freshest set of eyes that *tried* to break it. In this run, the separation did its job: it killed the CSRF claim and narrowed the payment-mint claim.

---

## 5. The three verdicts — with examples from THIS run

- **`confirmed`** — a source trace *and* a bounded observed runtime result. **Count here: 0.** There is no execution sandbox in this environment, so the skill's rules forbid us from claiming any. Saying "we proved nothing at runtime" is a *capability statement*, not a judgment that the origin is clean.
- **`needs_validation`** — the source trace is real and the mechanism is source-proven, but one decisive fact is unresolved (a runtime behavior, a deployment configuration, or an external provider semantic). **Count here: 24.** Example from this run: **`checkout:getCartByUUID:unscoped`** — source proves the cart is found only by client-supplied uuid with no ownership predicate; the unresolved fact is whether a second party can actually obtain a victim cart uuid and reach it on a live deployment. The record names the exact blocker and the bounded fixture that resolves it.
- **`rejected`** — the claim was disproved during validation. **Count here: 1.** Example: **`base-admin.session-cookie-csrf`** — a hunter hypothesized that the admin session cookie's missing SameSite was a cross-site CSRF. The independent verifier showed the code sets no SameSite attribute, so browsers default to SameSite=Lax, which **does not attach the cookie to cross-site POSTs**, and every claimed sink is POST-only — so there is no cross-site exploit. The reason (and trace) is retained so no future run re-raises it, and the leftover hardening note survives as a hardening note, not a finding.

**The rule:** `needs_validation` is not a parking place for a hunch and `rejected` is not a rebuke — both are evidence-shaped states.

---

## 6. What this Codespace could and could not execute

**Could:** read source; run the validators (Node scripts); orchestrate sub-agents; produce the artifacts in this repo.

**Could not:** execute target-controlled builds, browsers, fixtures, HTTP probes, or exploit payloads. The environment has **no OS-enforced sandbox** (no external-network isolation, no sanitized empty env, no resource-limit enforcement, no descriptor-based promotion tooling). Per the method's universal execution rule, any candidate that needed a runtime result was recorded as `needs_validation` with the exact unresolved fact. Additionally, the engagement rules forbade contacting deployed endpoints or external/shared services. Per client instruction, fresh verifiers were applied to the six highest-value records; the other eighteen retained leads carry trace-verified dispositions from the earlier validation passes and exact resolution plans.

**Upgrade path:** running this same method in a sandbox-capable environment (or against a staging instance, with owner consent) can promote qualified leads toward `confirmed` using the local fixtures already written into each record's `validation_plan`.

---

## 7. What the coverage ledger is and how to read it

`audit/run-1/coverage-ledger.json` is the hunt's scoreboard:

- **27 units** = (surface × boundary × subsystem × attack-class) cells. A clean `status: "covered"` (5 units) means the unit's attack class was examined and no candidate surfaced. `status: "candidate"` (22 units) means at least one source-traced candidate was recorded, cross-referenced by `result_fingerprints`.
- Each unit lists `reviewed_paths`, `local_checks` (hunter or verifier checks with `agent_id`, `invariant`, and outcome), and `unresolved` notes.
- This run added verifier checks (`method:"source"`, `artifact:null`) to the ledger units linked to the six verified fingerprints — that is how you can see *which unit an independent reviewer re-examined*.

**Reading it:** a row `status:"candidate"` + fingerprint F means "the record F in `findings.json` came out of this unit"; the adjacent verifier `local_checks` tell you who re-checked it and what conclusion was reached.

---

## 8. What validators were run and whether they passed

Both passed on the final state of the artifacts:

```
node validate-findings.cjs        audit/run-1/findings.json        → PASS: 25 findings valid
node validate-coverage-ledger.cjs audit/run-1/coverage-ledger.json → PASS: 27 coverage units valid
```

Validator success proves **format and ledger consistency** (schema shape, fingerprint linkage, unit/check invariants). It is not a claim that the origin is secure — it is the "the paperwork is internally consistent" check that lets the next run diff cleanly.

---

## 9. How a later second run would be additive

- **Diff fingerprints:** stable fingerprints mean a second run can say "the same root cause is still present (or fixed)" without re-deriving everything.
- **Stage validation:** promoting `needs_validation` → `confirmed` uses the exact local fixtures already written into each record; a sandbox-enabled run does only the missing work.
- **Coverage ledger carries state:** `covered` units need no re-hunt unless the source changed; changed-source re-validation is scoped to the diffs.
- **The rejection stays cached:** a no-longer-claimed lead is not re-raised unless new evidence appears.

---

## 10. How to brief leadership with this output

- Say "confirmed: 0" without embarrassment — and immediately: "the limit is that this pass had no execution sandbox; it deliberately did not claim runtime proof."
- Say the shop is directionally right: a CISO should treat the 24 `needs_validation` leads as engineering backlogs with owners, starting with the checkout/payment core.
- Say the edge is part of the solution, not the whole solution: Cloudflare can enforce *who may reach an endpoint* and *how fast*, but it **cannot enforce object ownership** inside the origin.
- Do **not** read the 24 leads as "24 vulnerabilities". None has a severity. Read them as "24 precise questions we can machine-answer."
- Use `engagement/04` to filter: of the 24, some are *edge-blockable today*, some are *origin-blocking for launch*, some are *validate-then-decide*.

---

## 11. What this method does NOT replace

- **Penetration testing against production-like data** — this was source-local; no live traffic, no real data.
- **A PCI DSS assessment** — payment flows were reviewed at the code level for object-level authorization and order-state integrity, but compliance evidence (scoping, controls, ASV scans) is outside this POC.
- **Payment-processor review** — Stripe/PayPal event-delivery and idempotency semantics had to be recorded as `needs_validation`; the processor (or a test-account observation) is the authority.
- **People/process security** — no assessment of operator hygiene, secrets management culture, or incident workflow.
- **Cryptographic review of the runtime/OS** — scope boundaries are documented; unrelated native runtime memory-safety was out of scope.
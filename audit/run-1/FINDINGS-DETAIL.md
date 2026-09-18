# Findings Detail

**Run:** `harbor-heath-run-1` · **Source ref:** `18e02022631c1c9dba1b6e5ac734ae27b54632cb`

## Confirmed medium/high/critical findings

**None.**

This file normally carries the complete ordered source trace, target-neutral reproduction, observed result, conditions, containment, and source-level remediation for every `confirmed` `medium`, `high`, or `critical` record. This run produced **zero `confirmed` records**, because the execution policy of this environment is `sandboxed-source-and-local-only` and **no OS-enforced sandbox was available**: the skill permits a `confirmed` verdict only with a complete source trace **and** a bounded, observed runtime result, and no runtime result was produced.

For the record:

- Any runtime-dependent lead is recorded as `needs_validation` with the exact unresolved fact, not softened or inflated to a severitied finding.
- The rejection record `base-admin.session-cookie-csrf` (retained so future runs do not repeat the claim) is detailed in `REPORT.md` §5.

## Where the detail lives

- **Source traces and evidence for every lead:** `findings.json` (machine-readable, validated against `report-schema.json`).
- **Per-lead exact blocker + bounded local fixture + owner-observed deployment check:** `NEEDS-VALIDATION.md`.
- **Source architecture, trust boundaries, request lifecycles, rate-limit tiers:** `architecture.md`.
- **Coverage bookkeeping including verifier `local_checks`:** `coverage-ledger.json`.

Rerunning the highest-value local fixtures from `NEEDS-VALIDATION.md` on a sandbox with an OS-enforced boundary is the prescribed path to promote leads toward `confirmed`.
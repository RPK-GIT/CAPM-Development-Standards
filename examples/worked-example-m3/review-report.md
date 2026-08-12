# CAPM Standard Review Report — *ILLUSTRATIVE / NON-PRODUCTION EXAMPLE*

> Fictional report over the [fixture](fixture.md), demonstrating the `/capm-review-milestone M3` output format. Not a real review.

| Field | Value |
|---|---|
| **Project** | granary-orders (fictional) |
| **Reviewed revision** | `example-fixture` |
| **Review date** | 2026-08-12 |
| **Reviewer** | Claude Code (worked example) |
| **Standard version** | 747b933 |
| **Scope** | Milestone gate review — M3 (Services/API) |
| **Review type** | initial |

## 1. Project profile
From [`capm-profile.yaml`](capm-profile.yaml), validated OK (no ✗ combinations; ⚠ none): Node.js, `@sap/cds` 10 (register current per docs/version-management.md, verified 2026-08-12), single-tenant, XSUAA, CF target, HANA production DB. Capability flags: custom_operations ✓, ui ✓, personal_data ✓, concurrent_edit ✓; drafts/media/mcp/eventing/remote_services/localized/temporal/pdm/extensibility ✗. Profile↔repository cross-check: consistent (runtime, packages, no MTX artifacts).

**Rule filtering:** M3 primary set (15) − CAP-SRV-007 (DRAFTS=false) − CAP-SRV-009 (MEDIA=false) − CAP-SEC-018 (MCP=false) − CAP-PERF-001 (CONCURRENT-PAGING=false) − CAP-SEC-010 (no instance-based-auth requirement) = **10 rules evaluated**.

## 2. Verdict summary

| Category | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| CAP-SRV | 5 | 1 | 3 | 1 |
| CAP-SEC | 2 | 1 | 2 | 0 |
| CAP-PERF | 0 | 0 | 1 | 0 |
| **Total (of M3 set)** | **7** | **2** | **6** | **1** |

**Gate recommendation: FAIL** — one uncovered HARD-GATE violation (CAP-SEC-001); no exception record exists.

## 3. Defects (rule violations)

### CAP-SEC-001 — Model authorization explicitly for every exposed service — FAIL — Critical *(HARD-GATE)*
- **Authority:** SAP-REQ (source verified L2, see §10)
- **Evidence:** `srv/orders-service.cds` L3–L4 — `OrdersService` carries no `@requires`/`@restrict` at any level and re-exposes `Orders` incl. `internalMargin` and `buyerEmail`; no annotate file covers it (verified absence across `srv/**`). `srv/internal-recalc.cds` L3 — `RecalcHelper` is neither restricted nor `@protocol:'none'`.
- **Detection:** enumerated `service` definitions in `srv/**/*.cds`; checked annotations inline + annotate files.
- **Impact:** any authenticated production user gets full CRUD on all orders including internal margin data; the helper action `recalcAll` is publicly invocable.
- **Remediation:** see §11 item 1.

### CAP-SRV-006 — Make protocol exposure an explicit decision — FAIL — Medium *(SOFT-GATE)*
- **Authority:** SAP-REC
- **Evidence:** `srv/internal-recalc.cds` L3 — internal helper service without `@protocol: 'none'`.
- **Impact:** internal orchestration service served on the default OData endpoint.
- **Remediation:** §11 item 2. *Soft gate: milestone may proceed only with recorded justification — none recorded.*

## 4. Recommendations & observations (non-defects)
- `AI-REC`: `buyerEmail` is exposed nowhere after the CatalogService exclusion — consider whether OrdersService (once restricted) needs it either (data-minimization, feeds CAP-PRIV review at M6).

## 5. Cross-cutting assessments
- **Security:** the CAP-SEC-001 finding above; no secrets found in the fixture (checked `package.json`, cds configs).
- **Architecture:** none found — services are use-case-cut (CatalogService browse; OrdersService admin-intent), CAP-native throughout.
- **Production readiness:** not in scope at M3; no observations.
- **Missing tests:** authorization denial tests absent (started at M3, mandatory by M6/M7 — tracked, not a FAIL here).

## 6. NOT ASSESSABLE rules

### CAP-SRV-008 — Optimistic concurrency for concurrently edited entities — NOT ASSESSABLE
Profile declares `concurrent_edit: true` and the UI edits Orders, but the repository contains **no** `@odata.etag` annotation, **no** draft protection, and **no** recorded last-write-wins decision (searched `srv/**`, `docs/**`). The rule permits a *recorded decision* as compliance — none exists, and whether concurrent editing is a real requirement cannot be determined from the repository. **Needed:** either the `@odata.etag` annotation, or a recorded concurrency decision, or a corrected profile.

## 7. Applicability decisions
CAP-SRV-007 (DRAFTS=false), CAP-SRV-009 (MEDIA=false), CAP-SEC-018 (MCP=false), CAP-PERF-001 (CONCURRENT-PAGING=false), CAP-SEC-010 (no instance-auth requirement) — NOT APPLICABLE per profile; CAP-SRV-005 evaluated (ODATA=true).

## 8. Full per-rule results

| Rule | Verdict | Evidence pointer |
|---|---|---|
| CAP-SRV-001 | PASS | `srv/catalog-service.cds` L6 — tailored projection excluding internal fields |
| CAP-SRV-002 | PASS | no custom handlers reimplementing generic behavior (none exist yet) |
| CAP-SRV-003 | PASS | validation declared in `srv/annotations.cds` L4–L5 |
| CAP-SRV-004 | PASS | `reorder` modeled as action (`catalog-service.cds` L7) |
| CAP-SRV-005 | PASS | OData V4 default; no V2 adapter/annotations present |
| CAP-SRV-006 | **FAIL** | `internal-recalc.cds` L3 |
| CAP-SRV-008 | NOT ASSESSABLE | §6 |
| CAP-SEC-001 | **FAIL** | `orders-service.cds` L3–L4; `internal-recalc.cds` L3 |
| CAP-SEC-011 | PASS | exposure walk: `Orders.items` (inline composition, no stricter target); no cross-sensitivity paths |
| CAP-SEC-012 | PASS | `@assert.range`/`@readonly` on writable elements; keys untouched |

## 9. Exceptions honored
None on file (`docs/capm/exceptions/` empty). No ORG-PENDING rules in the M3 evaluated set.

## 10. Standards & CAPire evidence verification
Per [capire-verification.md](../../reviews/capire-verification.md); 5 unique URLs, fetched once each.

| Rule | Verdict | Evidence | CAPire URL | Source status | Verified |
|---|---|---|---|---|---|
| CAP-SEC-001 | FAIL | orders-service.cds L3 | cap.cloud.sap/docs/guides/security/authorization | CURRENT | L2, 2026-08-12 |
| CAP-SEC-011 | PASS | exposure walk | …/guides/security/authorization | CURRENT | L2, 2026-08-12 |
| CAP-SEC-012 | PASS | annotations.cds L4 | …/guides/services/constraints | CURRENT | L2, 2026-08-12 |
| CAP-SRV-001/-002/-003 | PASS | catalog-service.cds | …/guides/services/providing-services, …/served-ootb, …/custom-code | CURRENT | L1, 2026-08-12 |
| CAP-SRV-004 | PASS | catalog-service.cds L7 | …/guides/services/custom-actions | CURRENT | L1, 2026-08-12 |
| CAP-SRV-005 | PASS | package.json | …/guides/protocols/odata | CURRENT | L1+L3 (version-sensitive; register current) |
| CAP-SRV-006 | FAIL | internal-recalc.cds L3 | …/node.js/cds-serve | CURRENT | L1, 2026-08-12 |
| CAP-SRV-008 | NOT ASSESSABLE | §6 | …/guides/services/served-ootb | CURRENT **BUT EVOLVING** *(illustration of the status class)* | L1, 2026-08-12 |

Governance flags: none (no source materially changed).

## 11. Remediation plan
Per [remediation-plan-template](../../templates/remediation-plan-template.md); no fixes applied in this review.

**Item 1 — CAP-SEC-001 (HARD, blocks):** add `@requires: 'OrderManager'` (role per xs-security.json) to `OrdersService`; annotate `RecalcHelper` with `@protocol: 'none'` *and* `@requires: 'internal-user'`-appropriate restriction if it must stay service-callable. Files: `srv/orders-service.cds`, `srv/internal-recalc.cds`, `xs-security.json`. Test: allow/deny per role (403 for non-managers). Re-review: CAP-SEC-001, CAP-SRV-006, plus anything touched.
**Item 2 — CAP-SRV-006 (SOFT):** covered by item 1's `@protocol:'none'`.
**Item 3 — CAP-SRV-008 (NOT ASSESSABLE):** decide concurrency handling for Orders (recommend `@odata.etag: modifiedAt` — `managed` is already in place) or record last-write-wins; update profile if `concurrent_edit` was wrong.

## 12. Outstanding risks & next-milestone readiness
Risks: unrestricted admin service (item 1) — must not reach any deployed environment. M4 entry criteria **not met** (M3 gate FAIL). After remediation: re-review via `/capm-review-milestone re-review M3` scoped to CAP-SEC-001, CAP-SRV-006, CAP-SRV-008 + touched files.

---
*Scope statement: this report evaluates only the applicable M3 standards listed in §7/§8. It makes no claim about overall CAP best-practice compliance.*

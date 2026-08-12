# CAPM Standard Review Report — *ILLUSTRATIVE / NON-PRODUCTION EXAMPLE (CAP Java)*

> Fictional report over the [fixture](fixture.md), demonstrating `/capm-review-milestone M3` for a **CAP Java** project. Not a real review.

| Field | Value |
|---|---|
| **Project** | millhouse-approvals (fictional) |
| **Reviewed revision** | `example-fixture` |
| **Review date** | 2026-08-12 |
| **Reviewer** | Claude Code (worked example) |
| **Standard version** | 9688250 |
| **Scope** | Milestone gate review — M3 (Services/API) |
| **Review type** | initial |

## 1. Project profile
From [`capm-profile.yaml`](capm-profile.yaml), validated OK: **Java**, CAP Java 5 (register current, verified 2026-08-12), single-tenant, XSUAA, CF, HANA. Flags: custom_operations ✓, ui ✓, personal_data ✓; drafts/media/mcp/eventing/remote_services/localized/temporal/concurrent_edit/instance_authorization ✗. Profile↔repository cross-check: consistent (`pom.xml` L10 `cds-services-bom` 5.0.0, `java.version` 21; no `package.json`, no MTX artifacts).

**Rule filtering:** M3 primary set (15) − CAP-SRV-007 (DRAFTS) − CAP-SRV-008 (CONCURRENT-EDIT) − CAP-SRV-009 (MEDIA) − CAP-SEC-010 (INSTANCE-AUTH) − CAP-SEC-018 (MCP) − CAP-PERF-001 (CONCURRENT-PAGING) = **9 rules evaluated**. *(M3 contains no runtime-scoped rules; runtime filtering is exercised in the report's M4-selection appendix and the [validation results](../../docs/validation-results-2026-08.md).)*

## 2. Verdict summary

| Category | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| CAP-SRV | 5 | 1 | 3 | 0 |
| CAP-SEC | 1 | 1 | 2 | 0 |
| CAP-PERF | 0 | 0 | 1 | 0 |
| **Total (of M3 set)** | **6** | **2** | **6** | **0** |

**Gate recommendation: FAIL** — one uncovered HARD-GATE violation (CAP-SEC-011); no exception record exists.

## 3. Defects (rule violations)

### CAP-SEC-011 — Review authorization of every exposed association and composition — FAIL — High *(HARD-GATE)*
- **Authority:** SAP-REQ (documented runtime enforcement gaps; source verified L2, §10)
- **Evidence:** `srv/approval-service.cds` L5 — `Approvals` projection exposes the `payroll` association (`db/schema.cds` L9) whose target `PayrollRecords` is restricted to `PayrollAdmin` (`srv/access-control.cds` L4), **stricter** than the service's `Approver` requirement. No mitigation exists: not excluded from the projection, no equivalent root restriction, no custom authorization handler (verified absence in `srv/src/main/java/**`).
- **Detection:** exposure walk per the rule — associations/compositions of each exposed entity compared against target restrictions.
- **Impact:** an `Approver` can read salary data via `$expand=payroll` — the documented Java enforcement gap makes the model's `PayrollAdmin` restriction ineffective on this path.
- **Remediation:** §11 item 1.

### CAP-SRV-006 — Make protocol exposure an explicit decision — FAIL — Medium *(SOFT-GATE; Java-specific check)*
- **Authority:** SAP-REC
- **Evidence:** no `@protocol`/`@protocols` annotation in `srv/**`, no adapter restriction in `application.yaml` (L1–L4) — per the documented default, `ApprovalService` is served on **both** `odata-v4` and `odata-v2`; no recorded V2 decision (CAP-SRV-005 justification requirement would bind if V2 is kept).
- **Impact:** an undecided deprecated-protocol endpoint is live by default. *(This check exists only for Java reviews — Node.js defaults to V4 only.)*
- **Remediation:** §11 item 2.

## 4. Recommendations & observations (non-defects)
- `AI-REC`: `PayrollRecords.salary` will feed the M6 privacy review (personal_data ✓); consider whether the association is needed at all (data minimization).

## 5. Cross-cutting assessments
- **Security:** the CAP-SEC-011 finding; no secrets in fixture files.
- **Architecture:** none found — handler follows the documented registration pattern (`ApprovalHandler.java` L4), action implemented via `@On` (CAP-SRV-004 ✓ — Java requires it).
- **Production readiness:** not in M3 scope.
- **Missing tests:** unauthenticated-rejection test exists (`ApprovalServiceIT.java` L5 — early CAP-SEC-015/TEST-007 evidence); allow/deny per role still to come (M6/M7).

## 6. NOT ASSESSABLE rules
None in this review.

## 7. Applicability decisions
CAP-SRV-007 (DRAFTS=false), CAP-SRV-008 (CONCURRENT-EDIT=false), CAP-SRV-009 (MEDIA=false), CAP-SEC-010 (INSTANCE-AUTH=false), CAP-SEC-018 (MCP=false), CAP-PERF-001 (CONCURRENT-PAGING=false) — NOT APPLICABLE per profile.

## 8. Full per-rule results

| Rule | Verdict | Evidence pointer |
|---|---|---|
| CAP-SRV-001 | PASS | `approval-service.cds` L5 — use-case projection (single audience); no 1:1 model dump |
| CAP-SRV-002 | PASS | no generic-behavior reimplementation; only the mandatory action `@On` handler exists |
| CAP-SRV-003 | PASS | validation declared (`access-control.cds` L7–L8), not hand-coded |
| CAP-SRV-004 | PASS | `approve` modeled as action; Java `@On` handler present (`ApprovalHandler.java` L5) |
| CAP-SRV-005 | PASS* | V4 is served; *the V2 half is subsumed by the SRV-006 finding (undecided default exposure) |
| CAP-SRV-006 | **FAIL** | `application.yaml` L1–L4 + verified absence of `@protocol` |
| CAP-SEC-001 | PASS | `approval-service.cds` L3 — `@requires: 'Approver'`; no unannotated service exists |
| CAP-SEC-011 | **FAIL** | `approval-service.cds` L5 → `access-control.cds` L4 |
| CAP-SEC-012 | PASS | `@assert.range`/`@mandatory` on writable elements; keys untouched |

## 9. Exceptions honored
None on file. No ORG-PENDING rules in the evaluated set.

## 10. Standards & CAPire evidence verification
Per [capire-verification.md](../../reviews/capire-verification.md); 6 unique URLs; live checks of 2026-08-12 (see [validation results](../../docs/validation-results-2026-08.md) for the fetch record).

| Rule | Verdict | Evidence | CAPire URL | Source status | Verified |
|---|---|---|---|---|---|
| CAP-SEC-001 | PASS | approval-service.cds L3 | cap.cloud.sap/docs/guides/security/authorization | CURRENT (live fetch) | L2, 2026-08-12 |
| CAP-SEC-011 | FAIL | L5→L4 chain | …/guides/security/authorization (composition/association gaps re-confirmed live) | CURRENT | L2, 2026-08-12 |
| CAP-SEC-012 | PASS | access-control.cds L7 | …/guides/services/constraints | CURRENT (per register 2026-08-12) | L2 |
| CAP-SRV-001/-002/-003 | PASS | approval-service.cds | …/guides/services/providing-services, …/served-ootb, …/custom-code | CURRENT (per register) | L1 |
| CAP-SRV-004 | PASS | ApprovalHandler.java L5 | …/guides/services/custom-actions; …/java/cqn-services/application-services | CURRENT (live fetch: Java V4+V2 default & @On requirement re-confirmed) | L1, 2026-08-12 |
| CAP-SRV-005/-006 | FAIL (006) | application.yaml | …/guides/protocols/odata; …/java/cqn-services/application-services | CURRENT (live fetch) | L1+L3 (version-sensitive; register current) |

Governance flags: none.

## 11. Remediation plan
Per [remediation-plan-template](../../templates/remediation-plan-template.md); no fixes applied in this review.

**Item 1 — CAP-SEC-011 (HARD, blocks):** exclude `payroll` from the `Approvals` projection (`excluding { payroll }`) — the approval flow needs `amount`/`requester`/`note` only; alternatively add a custom `@Before` READ handler enforcing `PayrollAdmin` on the expand path. Files: `srv/approval-service.cds` (preferred). Test: `$expand=payroll` as `Approver` → 4xx/absent. Re-review: CAP-SEC-011 + touched files.
**Item 2 — CAP-SRV-006 (SOFT):** add `@protocol: 'odata-v4'` to `ApprovalService` (no V2 consumer exists — greenfield). Files: `srv/approval-service.cds`. Test: V2 endpoint no longer served. Re-review: CAP-SRV-006, CAP-SRV-005.

## 12. Outstanding risks & next-milestone readiness
Risk: salary exposure via expand until item 1 lands — must not reach a deployed environment. M4 entry criteria **not met** (M3 FAIL). Re-review: [re-review.md](re-review.md).

---
*Scope statement: this report evaluates only the applicable M3 standards listed in §7/§8. It makes no claim about overall CAP best-practice compliance.*

# Re-Review — millhouse-approvals M3 — *ILLUSTRATIVE / NON-PRODUCTION*

> Demonstrates `/capm-review-milestone re-review M3` after remediation. Scope: the prior report's FAIL rules (CAP-SEC-011, CAP-SRV-006) **plus anything the remediation touched** — resolution verified by re-running each rule's detection, never inferred from changed files.

## Remediation applied (fictional commit `fix-m3-findings`)

`srv/approval-service.cds` — the only file changed:

```cds
using { acme.millhouse as my } from '../db/schema';

@requires: 'Approver'
@protocol: 'odata-v4'                                  // L4 — item 2: V2 exposure decided OFF
service ApprovalService {
  entity Approvals as projection on my.Approvals
    excluding { payroll };                             // L7 — item 1: stricter target no longer reachable
  action approve (ID : UUID) returns String;
}
```

## Re-review scope determination

Prior FAILs: CAP-SEC-011, CAP-SRV-006. Remediation diff touched `srv/approval-service.cds` → rules whose subject that file carries are re-checked too: CAP-SEC-001, CAP-SRV-001, CAP-SRV-004, CAP-SRV-005, CAP-SEC-012 (annotations target the projection). **Re-evaluated: 7 rules.**

## Re-evaluation (detection re-run, not diff inference)

| Rule | Prior | Re-run of detection | Verdict |
|---|---|---|---|
| CAP-SEC-011 | FAIL | Exposure walk repeated: `Approvals` exposes **no** associations/compositions after `excluding { payroll }` (L7); `Approvals.items` does not exist; no other service entities. The stricter `PayrollRecords` is unreachable from this service | **PASS** |
| CAP-SRV-006 | FAIL | `@protocol: 'odata-v4'` present (L4) → per the documented Java behavior, explicit protocol serves **only** that adapter; V2 endpoint no longer served; decision is now visible in the model | **PASS** |
| CAP-SRV-005 | PASS | Re-confirmed: V4 only; no V2 exposure left to justify | PASS |
| CAP-SEC-001 | PASS | `@requires: 'Approver'` still present (L3) — remediation did not regress it | PASS |
| CAP-SRV-001 | PASS | Projection now *narrower* (payroll excluded) — still use-case-tailored | PASS |
| CAP-SRV-004 | PASS | Action unchanged; `@On` handler untouched | PASS |
| CAP-SEC-012 | PASS | `annotate ApprovalService.Approvals` still resolves (elements `amount`, `note` remain in the projection) — annotations not orphaned by the exclusion | PASS |

*Illustration of the non-inference principle: CAP-SEC-012 was re-checked although it previously passed, because the remediation edited the projection its annotations target — a careless exclusion could have orphaned them. The re-run, not the diff, established it still passes.*

## Gate evaluation

No HARD or SOFT findings remain in scope; no exceptions needed.

**Milestone result: PASS** — M4 entry criteria met (services defined, validation/authorization declared, M3 gate passed).

---
*Scope statement: this re-review evaluated the 7 rules above. Prior PASS verdicts outside the remediation's blast radius (e.g., CAP-SRV-002/-003) were not re-evaluated and carry forward from the [initial report](review-report.md).*

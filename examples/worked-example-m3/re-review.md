# Re-Review — granary-orders M3 — *ILLUSTRATIVE / NON-PRODUCTION*

> Demonstrates `/capm-review-milestone re-review M3` after remediating the [initial report](review-report.md)'s findings (CAP-SEC-001 HARD, CAP-SRV-006 SOFT, CAP-SRV-008 NOT ASSESSABLE). Resolution verified by re-running each rule's detection — never inferred from changed files.

## Remediation applied (fictional commit `fix-m3-findings`)

`srv/orders-service.cds`:

```cds
using { acme.granary as my } from '../db/schema';

@requires: 'OrderManager'                              // L3 — item 1: explicit authorization
service OrdersService {
  entity Orders as projection on my.Orders
    excluding { internalMargin };                      // L6 — margin no longer exposed
}
annotate OrdersService.Orders with @odata.etag: modifiedAt;   // L8 — item 3: concurrency decided
```

`srv/internal-recalc.cds`:

```cds
using { acme.granary as my } from '../db/schema';

@protocol: 'none'                                      // L3 — item 1/2: internal only
service RecalcHelper {
  action recalcAll();
}
```

`xs-security.json` (new): scope + role template `OrderManager` generated via `cds compile srv --to xsuaa`.

## Re-review scope determination

Prior FAIL/NOT-ASSESSABLE: CAP-SEC-001, CAP-SRV-001, CAP-SRV-006, CAP-SRV-008. Diff touched both service files + `xs-security.json` → additionally re-checked: CAP-SEC-011, CAP-SEC-012 (their subjects live in the touched files); the new `xs-security.json` is noted for M6 (CAP-SEC-008's subject, not M3's). **Re-evaluated: 6 rules.**

## Re-evaluation (detection re-run)

| Rule | Prior | Re-run of detection | Verdict |
|---|---|---|---|
| CAP-SEC-001 | FAIL | Service enumeration repeated: `CatalogService` `@requires:'authenticated-user'` ✓; `OrdersService` `@requires:'OrderManager'` (L3) ✓; `RecalcHelper` `@protocol:'none'` → internal, excluded from exposure ✓. No unannotated exposed service remains | **PASS** |
| CAP-SRV-006 | FAIL | `RecalcHelper` now `@protocol: 'none'` (L3) — internal service not served | **PASS** |
| CAP-SRV-008 | NOT ASSESSABLE | `@odata.etag: modifiedAt` present (L8) on the concurrently edited entity (`managed` provides `modifiedAt`) — the CAP-native concurrency mechanism is in place; 412-path test noted as due at M7 | **PASS** (test evidence tracked for M7) |
| CAP-SRV-001 | FAIL | Projection re-checked: `excluding { internalMargin }` (L6) removes the 1:1 internal-field exposure; contract now use-case-shaped | **PASS** |
| CAP-SEC-011 | PASS | Exposure walk repeated post-change: `Orders.items` inline composition unchanged, no stricter targets; no new association exposed | PASS |
| CAP-SEC-012 | PASS | Validation annotations re-resolved against the modified projection (`quantity`, `status` remain) — not orphaned | PASS |

## Gate evaluation

No HARD or SOFT findings remain; the previously NOT-ASSESSABLE rule is resolved by the CAP-native mechanism plus a recorded M7 test obligation.

**Milestone result: PASS** — M4 entry criteria met.

---
*Scope statement: this re-review evaluated the 6 rules above; untouched prior PASS verdicts carry forward from the [initial report](review-report.md). The new `xs-security.json` becomes CAP-SEC-008's evidence at its owning milestone (M6).*

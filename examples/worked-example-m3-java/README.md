# Worked Example — M3 Milestone Review (CAP Java)

> **ILLUSTRATIVE / NON-PRODUCTION.**
> Everything in this folder is **fictional**, created solely to validate runtime-specific behavior of the `/capm-review-milestone` workflow for **CAP Java** projects. Not a real review; not part of the normative standard; sets no precedent.

## What this fixture adds beyond the [Node.js example](../worked-example-m3/README.md)

| Aspect | Demonstrated here |
|---|---|
| Runtime detection | `capm-profile.yaml` `runtime: java` cross-checked against `pom.xml` |
| Runtime filtering (M3) | M3's primary set contains no runtime-scoped rules — Both-runtime rules apply to both fixtures; the runtime *content* differs (CAP-SRV-006's Java V2-by-default check) |
| Runtime filtering (M4 demonstration) | The rule-set selection for a hypothetical M4 review: **4 Java-only rules included** (CAP-LOGIC-004/-005, CAP-TXN-004, CAP-DB-010), **10 Node.js-only rules excluded** (CAP-LOGIC-003, CAP-DB-005/-006/-009, CAP-TXN-002/-003/-006, CAP-ERR-002/-004, CAP-PERF-005) — see the [validation results](../../docs/validation-results-2026-08.md) |
| Java-specific finding | CAP-SRV-006 SOFT FAIL: Java serves `odata-v4` **and** `odata-v2` by default — undecided V2 exposure (a check that must never appear in a Node.js review) |
| HARD gate | CAP-SEC-011 FAIL: exposed association to a stricter-restricted entity, unmitigated |
| Re-review cycle | [re-review.md](re-review.md): remediation applied → both findings re-verified per rule detection → PASS |

## Contents

[`capm-profile.yaml`](capm-profile.yaml) · [`fixture.md`](fixture.md) (fictional repository files) · [`review-report.md`](review-report.md) (initial M3 review — FAIL) · [`re-review.md`](re-review.md) (remediation + re-review — PASS)

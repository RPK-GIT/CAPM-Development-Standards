# M1 — Architecture

Operational checklist for the [M1 milestone](../lifecycle.md#m1--architecture). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md).

## Purpose
Fix the architectural decisions every later milestone builds on: runtime, versions, service cut, persistence, identity direction, tenancy, integration/eventing approach, deployment target — CAP-native-first throughout, ADR-recorded.

## Entry criteria
M0 PASS (requirements + capability profile available).

## Applicable standards

**Primary (15):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-ARCH-001 | SOFT | — | Standard layout & conventions |
| CAP-ARCH-002 | SOFT | — | No custom framework layers |
| CAP-ARCH-003 | SOFT | — | Single-use-case services |
| CAP-ARCH-004 | SOFT | — | Stateless, passive data |
| CAP-ARCH-005 | SOFT | — | Platform/protocol agnosticism |
| CAP-ARCH-006 | SOFT | — | Modulith first; cuts documented |
| CAP-ARCH-007 | SOFT *(ORG-PENDING)* | — | Material decisions → ADRs |
| CAP-SEC-006 | SOFT | NEW-PROJECT | IAS-first identity choice |
| CAP-MT-001 | **HARD** | MT | Streamlined MTX only |
| CAP-MT-002 | SOFT | MT | Sidecar architecture |
| CAP-EVT-006 | SOFT | EVENTING+BROKER | Broker selection (Event Hub default) |
| CAP-DB-001 | SOFT | — | HANA Cloud as production DB |
| CAP-DB-003 | SOFT | NODE | Current `@cap-js` DB services |
| CAP-VER-002 | **HARD** | — | Active CAP release line |
| CAP-VER-003 | **HARD** | — | Supported runtime baselines |

**Supporting:** CAP-INT-002 (consumption approach sketched), CAP-EXT-001 (reuse strategy: extend, don't fork), CAP-VER-004 (wave-consistent package selection).

**Conditional filter:** apply MT/EVENTING/NEW-PROJECT/NODE flags from the M0 profile.

## Required evidence
Architecture description + ADRs (runtime rationale, service cut, persistence, identity, tenancy, deployment target, eventing); project skeleton (`cds init` layout) that builds; `package.json`/`pom.xml` showing the version baseline against the [version register](../../docs/version-management.md) (confirm the register's verified date first); requirement→CAP-capability mapping (AI-DEV-004 style: capability named per requirement, custom residual justified).

## Required tests
None mandatory; a walking-skeleton build (`cds build` clean) is the smoke evidence.

## Review procedure
1. Confirm the capability profile (M0) — mark non-applicable conditional rules NOT APPLICABLE with the flag as reason.
2. Evaluate the 15 primary rules per their catalog detection guidance (config- and document-level at this stage).
3. Check every requirement maps to a CAP-native capability or a justified custom approach (CAP-ARCH-002 pre-check).
4. Verify HARD gates: MT-001 (mtxs line, if MT), VER-002/-003 (Active line, supported runtimes, consistent pins).
5. Verify each material decision has an ADR (ARCH-007 — report as ORG-PENDING policy finding if violated).

## Remediation expectations
HARD-gate findings (wrong MTX line, unsupported/EOL versions) are corrected before PASS — exceptions per [AI-DOC-002] only. Soft findings (layout deviations, undocumented cuts) remediated or justified in this checklist.

## Exit criteria
Architecture + ADRs approved; skeleton builds; version baseline on the Active line; HARD gates clear. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).

# M8 — Deployment & CI/CD

Operational checklist for the [M8 milestone](../lifecycle.md#m8--deployment). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). Heavy FINAL-GATE milestone: **11 production-configuration re-verifications** land here alongside 15 primary rules.

## Purpose
Reproducible pipeline deployment to a staging space with production-grade configuration: the point where design-time compliance meets deployed reality.

## Entry criteria
M7 PASS (green suite in CI).

## Applicable standards

**Primary (15):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-DEP-001 | SOFT | CF | MTA from production facets; `cds up`/mbt |
| CAP-DEP-002 | SOFT | MULTI-LANDSCAPE | `.mtaext` landscape overlays |
| CAP-DEP-003 | SOFT | KYMA | Generated chart + buildpacks + RO pull secrets |
| CAP-CICD-001 | SOFT | — | Pipeline builds/tests/deploys |
| CAP-CICD-002 | SOFT | — | Protected, release-triggered production stage |
| CAP-CICD-003 | SOFT *(ORG-PENDING)* | — | Gates block; enforcement classes honest |
| CAP-VER-001 | SOFT | NODE | Lockfile frozen + refresh automation |
| CAP-VER-004 | **HARD** | NODE | No mixed CAP package majors |
| CAP-VER-005 | SOFT | MAJOR-UPGRADE | Official migration path |
| CAP-VER-006 | SOFT | LEGACY-HDBCDS | `hdbtable` only |
| CAP-DB-002 | **HARD** | — | No dev databases in production config |
| CAP-MT-005 | **HARD** | MT | Tenant upgrade before traffic (automated, ordered) |
| CAP-LOG-002 | SOFT | — | JSON logs in production config |
| CAP-LOG-005 | **HARD** | JAVA | Only health actuator public |
| CAP-OPS-001 | SOFT | — | Health probes wired in descriptors |

**FINAL-GATE re-verifications (production config against deployed reality):** CAP-SEC-002, CAP-SEC-003 (NODE), CAP-SEC-004/-005 (JAVA), CAP-SEC-015 (deployed unauthenticated-rejection check runs), CAP-SEC-017, CAP-DB-001, CAP-DB-003 (NODE), CAP-MT-002 (MT), CAP-VER-002, CAP-VER-003, CAP-EVT-002 (EVENTING).

**Supporting:** CAP-SEC-008 (deployed xs-security artifact), CAP-INT-006 (destination strategy in deployed config), CAP-EXT-004 (production toggles provider), CAP-TEST-006 (hybrid stage wired), CAP-DB-007 (journal artifacts committed), CAP-OPS-002 (UI module present — verified live at M9).

## Required evidence
`mta.yaml`/chart + overlays; pipeline definitions with blocking gates and the post-deploy 401/403 smoke; lockfile + update automation; version pins consistent across every location ([register](../../docs/version-management.md) current); a green pipeline run deploying to staging; tenant-upgrade step in the automation (MT); rollback documentation (application-artifact scope); secret scan clean.

## Required tests
Pipeline-executed suite; post-deploy smoke tests (health endpoints, unauthenticated rejection); hybrid stage where CLOUD-ONLY.

## Review procedure
1. Filter by CF/KYMA/MT/NODE/JAVA/EVENTING/etc. flags.
2. **HARD gates:** VER-004 (wave-consistent package majors), DB-002 (production DB per profile resolution), MT-005 (upgrade step exists and precedes traffic), LOG-005 (actuator exposure config).
3. Run all 11 FINAL-GATE re-verifications against the *deployment artifacts and the staging deployment* — config-level PASS from M6 does not transfer; the deployed evidence decides.
4. Verify pipeline gating (CICD-003 — report as ORG-PENDING finding class): deploy `needs` test, no bypasses, secret scanning, smoke check present.
5. Confirm the deployment came from the pipeline, not a workstation (CICD-001).

## Remediation expectations
HARD/FINAL-GATE findings block promotion beyond staging. Landscape-config drift (DEP-002) and probe wiring (OPS-001) fixed before M9's operational drill depends on them.

## Exit criteria
Green pipeline deploys to staging; all FINAL-GATE re-verifications pass; deployment review PASS. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).

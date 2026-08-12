# M2 — Domain / CDS Model

Operational checklist for the [M2 milestone](../lifecycle.md#m2--domain--cds-model). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md).

## Purpose
A clean, performant, privacy-classified domain model in `db/` — the artifact every service, handler, and tenant database derives from.

## Entry criteria
M1 PASS (architecture, skeleton, version baseline).

## Applicable standards

**Primary (18):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-CDS-001 | ADV | — | SAP naming conventions |
| CAP-CDS-002 | SOFT | — | `cuid` technical keys |
| CAP-CDS-003 | SOFT | — | UUIDs opaque |
| CAP-CDS-004 | SOFT | — | `@sap/cds/common` reuse |
| CAP-CDS-005 | SOFT | — | Managed associations |
| CAP-CDS-006 | SOFT | — | Compositions for containment; no composition-of-one |
| CAP-CDS-007 | SOFT | — | Concerns separated via aspects |
| CAP-CDS-008 | SOFT | LOCALIZED | `localized` for translatable text |
| CAP-CDS-009 | SOFT | TEMPORAL | `temporal` aspect; writes in handlers |
| CAP-CDS-010 | ADV | — | Flat models |
| CAP-CDS-011 | SOFT | — | Stable namespaces |
| CAP-PRIV-001 | **HARD** | PERSONAL-DATA | Complete `@PersonalData` annotations |
| CAP-EXT-001 | SOFT | — | Extend — never modify reuse artifacts |
| CAP-PERF-002 | SOFT | — | Associations over JOIN views |
| CAP-PERF-003 | SOFT | — | Calculated fields out of filter/sort paths |
| CAP-PERF-004 | SOFT | — | No UNION modeling |
| CAP-PERF-006 | SOFT | DRAFTS | Draft-manageable composition trees |
| CAP-DB-007 | SOFT | HANA-LARGE | `@cds.persistence.journal` for large entities |

**Supporting:** CAP-ARCH-001 (layout now live: `db/schema.cds`, CSV naming), CAP-VER-006 (no legacy `hdbcds` remnants, LEGACY-HDBCDS).

## Required evidence
`db/**/*.cds` compiling clean (`cds compile`); privacy annotation file (e.g., `srv/data-privacy.cds`) covering the M0 personal-data inventory — EntitySemantics + DataSubjectID per annotated entity; seed data in `db/data/<ns>-<Entity>.csv`; deployment smoke (`cds deploy` to dev DB); model-review notes for composition-vs-association decisions on non-obvious relationships.

## Required tests
Deploy smoke test; model assertions where applicable.

## Review procedure
1. `cds compile` clean — else NOT READY.
2. Evaluate the 11 CDS rules per catalog detection (naming, keys, aspects, containment semantics — CDS-006's semantic walk per relationship).
3. **HARD gate:** diff the privacy annotations against the M0 inventory — any unannotated personal-data element FAILs CAP-PRIV-001 (report structural gaps per entity).
4. Check performance-modeling rules on entities expected to grow (PERF-002/-003/-004/-006; DB-007 for HANA-LARGE).
5. Verify no forked/patched reuse artifacts (EXT-001).

## Remediation expectations
CAP-PRIV-001 findings block (exception only via AI-DOC-002 with legal-aware approver). Modeling soft-gates fixed now — they are an order of magnitude cheaper before services and data exist.

## Exit criteria
Model compiles and deploys; privacy annotations complete vs inventory; model review PASS. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).

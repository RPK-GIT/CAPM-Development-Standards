# Phase 4 — Pilot 1: Real-Project M3 Review (scs)

First real-project pilot of the CAPM Engineering Standard and the `/capm-review-milestone` workflow. **This is a pilot review against the applicable CAPM Development Standards** — it is NOT a certification, NOT a production-compliance claim, and NOT a full CAP-compliance assessment.

| Field | Value |
|---|---|
| Target | `scs` (Supply Coverage Simplified) — an internal, deployed SAP CAP application |
| Runtime | Node.js, `@sap/cds` 7.9.5, HANA Cloud, XSUAA, Cloud Foundry (MTA), 6 Fiori/UI5 apps, 20+ S/4HANA remote services |
| Milestone reviewed | **M3 (Services/API) only** — retrospective (the application is live) |
| Standard version | `791de1e` |
| Review date | 2026-08-12 |
| Result | **Milestone FAIL** — 1 Critical HARD finding (CAP-SEC-001), 10 SOFT findings |
| Pilot classification | **AMBER** — framework works; meaningful calibration items recorded |

## Contents

- [capm-profile.yaml](capm-profile.yaml) — the profile created for the project (copy of the one placed at the project root)
- [review-report.md](review-report.md) — the full M3 review report, including the **Phase 4 Pilot Calibration** section (SRV-005/006 analysis, SUPPORTING-rule analysis, false-positive/negative/noise/ambiguity pass, standard & workflow change proposals)

## Anonymization & data handling

The application source is **not** copied into this repository. Artifacts here contain only: file paths, line numbers, entity/service names, rule verdicts, and minimal paraphrased evidence. Deliberately omitted: HANA host/database identifiers, destination names, S/4 system identifiers, any credentials or `.env` content (the project's `.env` and `.cdsrc-private.json` are gitignored in the project and were not read beyond key-structure verification).

## Pilot scope guards

- One milestone (M3); no M0–M9 sweep, no certification attempt.
- Read-only: no application code was modified. The only file written to the project is `capm-profile.yaml` (the documented adoption artifact).
- No rules were modified as a result of this pilot; defects in the standard are recorded as **STANDARD CHANGE PROPOSALS** in the report for controlled follow-up.

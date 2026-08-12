# Re-Verification Cadence

The standard's SAP evidence must remain current or the rules that cite it drift silently. This document defines the sustainable cadence for re-verifying evidence, sources, and rule applicability.

**Dates in this document are governance cadence, not SAP requirements.** SAP does not require any specific verification schedule; these are organizational practices to keep the standard honest.

## Cadence at a glance

| Trigger | Scope | Owner | Artifact |
|---|---|---|---|
| **Quarterly** (every 3 months) | Full CAPire evidence sweep | Standards maintainer | Bumped `Last verified` dates + governance entries in [CHANGELOG.md](../CHANGELOG.md) |
| **CAP release** (major or minor) | Version-sensitive rules only ([version-management.md](version-management.md)) | Standards maintainer | Version register update + affected rule re-verification |
| **Material source change** (source removed, redirected, or load-bearing content changed) | Immediate — affected rules only | Whoever notices | Immediate governance entry + affected reviews re-evaluated |
| **Security-critical source** review (11 Critical rules + all SAP-REQ security/tenant/privacy sources) | Full L2 re-fetch | Standards maintainer | Bumped `Last verified` + governance entry if changed |
| **Annually** | Rule-catalog governance review | Maintainer + org rep | Retirement/deprecation decisions; ORG-ratification refresh; deferred-items disposition |

## Quarterly CAPire evidence sweep

**Purpose.** Catch source drift between release-triggered checks. Every quarter, re-verify every unique URL cited in `references/sap-cap-sources.md`.

**Procedure.**

1. Load the [source map](../references/sap-cap-sources.md) — one entry per URL, per subject.
2. Live-fetch each URL. Classify status per [capire-verification.md](../reviews/capire-verification.md):
   - `CURRENT` — load-bearing content unchanged; bump the URL's verification date in the source map.
   - `CURRENT BUT EVOLVING` — content added but not contradicting; bump the date; note the addition.
   - `REDIRECTED` — follow, treat as new URL, verify content match.
   - `STALE` / `REMOVED` / `UNAVAILABLE` — trigger the material-source-change path (below).
3. For each rule that cites a source: bump `Last verified` **only if** the source status is `CURRENT`/`CURRENT BUT EVOLVING`. If STALE/REMOVED, do NOT silently bump — the rule enters the [governance case](rule-governance.md).
4. Record the sweep result in [CHANGELOG.md](../CHANGELOG.md) with a governance entry:
   ```
   ### Quarterly CAPire sweep — YYYY-QN
   - N sources verified CURRENT
   - N sources CURRENT BUT EVOLVING (list)
   - N sources STALE/REMOVED (list; affected rules; disposition)
   - Rule verified-date bumps: <bulk>
   ```

**Not a research sweep.** This is verification of *existing* citations. Do not extend the catalog or invent new rules based on what's found — that path is [rule governance](rule-governance.md) with proposal + evidence.

**Cost.** ~2–4 hours per sweep with the current source map (~40 unique URLs). Scales linearly with the catalog.

## CAP release triggers

CAP releases quarterly (Node.js + Java, roughly aligned). Not every release is a rule event, but every release triggers a version-register check:

1. Read the release notes (Node.js: `@sap/cds` changelog; Java: CAP Java release notes; MTX / `@cap-js` sub-packages).
2. Update [version-management.md](version-management.md) — CURRENT / MAINTENANCE / UNRELEASED / LEGACY-DEPRECATED table.
3. For every rule marked ⏱ *version-sensitive*:
   - Re-fetch its cited source at L3.
   - Confirm the mechanism it names still applies at the new version.
   - Where the mechanism has changed (property name renamed; feature moved out of preview; adapter package changed like the SRV-005 Node V2-adapter deprecation caught in Pilot 1), amend the rule via [rule governance](rule-governance.md) with the SAP release note as evidence.
4. Where a rule refers to a mechanism marked "Beta / Preview" in the release notes: check whether it moved to `Stable` or was withdrawn.
5. Governance entry in the CHANGELOG:
   ```
   ### CAP release — <version> (YYYY-MM-DD)
   - Version register updated: CURRENT <old>→<new>, MAINTENANCE <old>→<older>, …
   - Version-sensitive rules re-verified: <list>
   - Amendments: <rule + one-line delta or "none">
   ```

## Material source change (immediate)

Discovered by: quarterly sweep, review-time fetch, developer/reviewer report, SAP announcement.

Actions:

1. Confirm the change (a second fetch; a link to the SAP page's changelog if available).
2. Identify affected rules (grep the source map + rule bodies).
3. For each affected rule:
   - If the rule is materially dependent on the changed statement → the rule enters `NOT ASSESSABLE` for reviews performed until the rule is amended.
   - If the change is a mechanism update only (like SEC-014's Node `$batch` limit surfacing in v2 docs → v3) → rule text amended via [rule governance](rule-governance.md); rule's Detection guidance updated; validation scenarios re-run.
4. Governance flag in the CHANGELOG.
5. Notify adopting projects (organizational channel — this repository does not push notifications).

**Silent re-write is prohibited.** The rule's `Last verified` date and the CHANGELOG entry make the change auditable.

## Security-critical source review

The 11 Critical rules and every SAP-REQ security / tenant-isolation / privacy source get an L2 explicit re-fetch on a schedule tighter than the general quarterly sweep — recommend **monthly**.

- Sources to include: `guides/security/authorization`, `guides/security/data-protection`, `guides/security/best-practices`, `guides/multitenancy/*`, `guides/personal-data`, `node.js/cds-ql`, `java/cqn-services/*`, plus any URL cited by a Critical rule.
- Procedure: same as the quarterly sweep for these sources only.
- Deliverable: monthly governance entry in the CHANGELOG.

Rationale: Critical rules define the standard's blocking edge for security posture. A material change in their sources changes what the framework blocks or lets through — this deserves faster feedback than quarterly.

## Annual rule-catalog governance review

Once a year, a full review of the catalog's shape:

- Rules marked `Deprecated` and past their effective date → move to `Retired` (leave file in place — IDs never reused).
- Deferred items from pilot calibration ([pilots/round-1-calibration.md § Deferred / open items](../pilots/round-1-calibration.md)): re-evaluate with the year's accumulated pilot evidence.
- ORG-PENDING rules → check ratification status; if still pending, record why.
- Validation-suite audit: validation-suite scenarios still relevant? new scenarios needed for accumulated calibration items?
- CHANGELOG's governance entries for the year: consolidated summary at the top of the annual section.

Deliverable: an annual `## Annual governance review — YYYY` block in [CHANGELOG.md](../CHANGELOG.md) plus any resulting rule/matrix/documentation changes.

## What to do when a review flags a source problem mid-review

Per [capire-verification.md](../reviews/capire-verification.md) and [review command step 10](../.claude/commands/capm-review-milestone.md):

- The rule affected becomes `NOT ASSESSABLE` in *this* review — the reviewer does not amend the rule during the review (AI-REVIEW-012).
- A governance flag is raised in report §10.
- The material-source-change path (above) fires after the review completes.
- The adopting project's report is not held up by this; the standard is.

## What NOT to re-verify

- Individual per-rule text unless the source it cites has changed. Rules do not decay because time passes; sources do.
- Third-party summaries of SAP docs. Only SAP official sources count.
- Community-maintained mirrors of CAPire. `cap.cloud.sap/docs` is the authority.

## Ownership

- **Standards maintainer** — the human role that owns quarterly and release-triggered sweeps.
- **Adopting project reviewers** — trigger the ad-hoc material-source-change path when they hit it in their reviews.
- **Organization security lead** — owns the monthly security-critical sweep (or delegates to a security-engineering rotation).

The maintainer role is not "Claude Code". Claude executes the mechanics under a maintainer's direction; the maintainer's judgment is what admits changes into the catalog.

## Where to go next

- The verification protocol itself → [../reviews/capire-verification.md](../reviews/capire-verification.md).
- Version-register model → [version-management.md](version-management.md).
- Rule lifecycle (what changes a rule versus what changes a date) → [rule-governance.md](rule-governance.md).
- What v1.0 does not guarantee (bounds this cadence) → [adoption-boundaries.md](adoption-boundaries.md).

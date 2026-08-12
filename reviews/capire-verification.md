# CAPire Source Verification at Review Time

Protocol for verifying the official SAP documentation (capire) evidence behind the rules evaluated in a milestone review, executed **before the final report is generated** (step 12 of [/capm-review-milestone](../.claude/commands/capm-review-milestone.md)). The goal is verifying the sources relevant to **this review's evaluated rules** — never crawling all of capire. The at-rest evidence base remains [references/sap-cap-sources.md](../references/sap-cap-sources.md); this protocol governs the *live check* at report time.

## Verification levels

Collect the **deduplicated set of SAP reference URLs** from the rules actually evaluated (PASS/FAIL — NOT APPLICABLE rules need no source check), then:

| Level | Applies to | Check |
|---|---|---|
| **1 — Standard** | All other evaluated rules | Per unique URL: the page resolves; spot-confirm it still contains the load-bearing statement quoted/paraphrased in the rule; authority label still appropriate |
| **2 — Critical/security/privacy** | Every evaluated Critical rule + all `CAP-SEC-*`, `CAP-PRIV-*`, `CAP-MT-003` (tenant isolation) | Explicit fetch of each cited page; verify the specific supporting statement (not just page existence) before the verdict is finalized |
| **3 — Version-sensitive** | Every evaluated rule with a ⏱/version-bound `CAP version` field | First confirm [docs/version-management.md](../docs/version-management.md)'s verified date is within the quarterly cadence; then check the current release/changelog source for changes affecting the rule. Never rely on an old cached baseline |

## Source status classification

| Status | Meaning | Effect on rule verdicts |
|---|---|---|
| **CURRENT** | Page live; statement present; authority holds | Verdict stands |
| **CURRENT BUT EVOLVING** | Page live and supporting, but marked under-construction/beta or visibly reworked | Verdict stands; status noted; rule flagged for the next re-verification sweep |
| **REDIRECTED** | Content moved; new URL supports the rule | Verdict stands; **governance event: update the rule's URL** in the standards repo (do not leave the stale link) |
| **STALE** | Page live but the supporting statement is materially changed/absent | See *material dependence* below |
| **REMOVED** | Page 404/section gone; no successor found | See *material dependence* below |
| **UNAVAILABLE** | Fetch impossible in this environment (offline/blocked) | If the rule's `Last verified` (or the source map entry) is within the quarterly cadence: verdict stands, report states *"live check unavailable; relying on register verification of <date>"*. Outside the cadence: treat as STALE for material-dependence purposes |

**Material dependence:** if a rule's verdict *depends on* the SAP claim (any SAP-REQ/SAP-REC rule does) and its source is STALE or REMOVED, the rule's verdict for this review becomes **NOT ASSESSABLE** — never a silent PASS — with the source finding attached. Exception: where the rule's own text already anchors an alternative live source (as CAP-ARCH-002 does for the removed Bad-Practices page), verify against that anchor instead.

## Change handling (governance events)

When a current source **materially differs** from the evidence the rule was built on:

1. Do **not** rewrite the rule during the review (AI-REVIEW-012: reviews are read-only — this extends to the standards repo).
2. Flag the rule in the report's CAPire section: source, nature of the change, possible impact on the rule.
3. Recommend controlled re-validation: a standards-maintenance change (Mode 1 of [CLAUDE.md](../CLAUDE.md)) that re-verifies the claim, updates the rule/source map with a new `Last verified` date, and records the disposition — the same discipline as the Phase 2 verification corrections logged in [candidate-dispositions.md](../references/candidate-dispositions.md).
4. Until re-validated, subsequent reviews inherit the flag (the rule is not silently trusted again).

## Report output

Every milestone report includes the **Standards & CAPire Evidence Verification** section (see the [review-report template](../templates/review-report-template.md)) — one row per evaluated rule with its project evidence, source URL(s), source status, verification timestamp, and verdict, providing the mandatory traceability chain:

```
Rule → Project evidence → CAPire source → Current source status → Verdict
```

Only sources relevant to the evaluated rules appear — never the whole registry. Where several rules share a URL, the fetch is performed once and the status applied to each row.

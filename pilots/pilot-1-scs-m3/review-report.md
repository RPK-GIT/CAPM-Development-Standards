# CAPM Standard Review Report — Pilot 1 — scs — M3

> **Phase 4 pilot review against the applicable CAPM Development Standards.** Not a certification; not a production-compliance or full-CAP-compliance claim. *(Report per [review-report-template](../../templates/review-report-template.md); command: `/capm-review-milestone M3`.)*

| Field | Value |
|---|---|
| **Project** | scs (Supply Coverage Simplified) — internal, deployed CAP application |
| **Reviewed revision** | `c1f51ac` (2026-07-06) + uncommitted working tree of 2026-08-12 (differences are UI `dist/` build output only) |
| **Review date** | 2026-08-12 |
| **Reviewer** | Claude Code (Phase 4 Pilot 1) |
| **Standard version** | `791de1e` |
| **Scope** | Milestone gate review — M3 (Services/API), retrospective |
| **Review type** | initial |
| **Review duration** | ≈ 55 minutes wall clock (project identification through report generation, including live CAPire verification) |

## 1. Project profile

From [`capm-profile.yaml`](capm-profile.yaml) (created at M0-equivalent during this pilot; validated OK — no ✗ combinations; identity/deployment/database all decided). Node.js, `@sap/cds` 7.9.5 (major 7 — **behind the register's current major 10**; upgrade is an M8/VER-rule topic, out of M3 scope), single-tenant, XSUAA, CF/MTA, HANA Cloud, 6 UI5/Fiori apps, 20+ S/4 remote services. Capability flags: remote_services ✓, mashups ✓, custom_operations ✓, instance_authorization ✓, drafts ✓, personal_data ✓, ui ✓, concurrent_edit ✓; eventing/media/localized/temporal/pdm/extensibility/mcp ✗; `concurrent_paging` **not establishable from the repository** (owner confirmation needed — open question O-1).

**Milestone plausibility:** profile says `LIVE`; reviewing M3 retrospectively is deliberate pilot scope — flagged per command step 3, proceeded.

**Profile↔repository cross-check:** consistent (runtime/packages, no MTX artifacts, no messaging config, personal-data plausibility via stored user IDs). `.env` and `.cdsrc-private.json` exist locally but are gitignored and untracked — noted for M6/M8 (CAP-SEC-017's milestones), not an M3 finding.

**Rule filtering:** M3 PRIMARY set (15) − CAP-SRV-009 (media ✗) − CAP-SEC-018 (mcp ✗) = **13 primaries evaluated** (CAP-PERF-001 evaluated → NOT ASSESSABLE, see §6). SUPPORTING per checklist: all five subjects (service cut, annotation placement, limits, error contract) are instantiated by the M3-scope artifacts of this retrospective review → **5 supporting evaluated** (selection rationale examined in the calibration section). No M3 FINAL-GATE rows exist in the matrix. No ORG rules in scope. Runtime filter: nodejs (no Java-only rules present in the M3 set). Version register consulted (verified 2026-08-12) before version-sensitive rules (SRV-005, PERF-001); `reliablePaging` support confirmed present in the installed cds 7.9.5 (`node_modules/@sap/cds/libx/odata/middleware/read.js`).

## 2. Verdict summary

| Set | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| PRIMARY (15) | 4 | 8 | 2 | 1 |
| SUPPORTING (5) | 2 | 3 | 0 | 0 |
| **Total (20 selected)** | **6** | **11** | **2** | **1** |

Gate-class distribution of the selected set: 3 HARD (SEC-001, SEC-011, SEC-018) / 17 SOFT / 0 ADVISORY. Findings: **1 Critical HARD FAIL, 10 SOFT FAILs** (7 primary + 3 supporting), 0 advisory.

**Gate recommendation: FAIL** — one uncovered HARD-GATE violation (CAP-SEC-001, Critical); no exception records exist (the project has no `docs/capm/exceptions/`, no ADRs, no docs directory at all).

## 3. Critical rules (explicit verification — command step 9)

### CAP-SEC-001 — Model authorization explicitly for every exposed service — **FAIL** — Critical *(HARD-GATE)*
- **Authority:** SAP-REQ · **Runtime:** Both (Node.js applies) · **Version:** all supported (holds for cds 7) · **CAPire:** `guides/security/authorization` — **CURRENT**, L2 fetch 2026-08-12, load-bearing statement re-confirmed ("By default, CDS services have no access control … without authorization modeling, authenticated users have access to all entities").
- **Evidence (DIRECT + verified absence):** `srv/interaction_srv.cds` L4–L7 (`loadbuilderV4` at `/AdminV4`) and L9–L172 (`loadbuilder` at `/Admin`, ~78 exposed entities + 7 operations) carry **no** `@requires`/`@restrict` at service, entity, or operation level; verified absence of authorization annotations across all `.cds` files in `srv/`, `db/`, `app/` (repo-wide grep). `xs-security.json` L4–L12 defines `Admin`/`User` scopes and role templates that are **never referenced by the model** — authorization stops at approuter *authentication* (`app/xs-app.json`: `authenticationType: xsuaa`), while the srv module has its own CF route (`mta.yaml`: `srv-url: ${default-url}`).
- **Why it violates the rule:** the rule prohibits authorization by omission; both served services rely on runtime fallback behavior.
- **Risk:** any authenticated user — regardless of assigned role collection — has full generic CRUD on all exposed entities, including system configuration (`Config` with destination names, `db/interactions.cds` L45–L49), other users' UI variants (`Variants`, L35–L42), and all S/4 replica data; and can invoke all custom operations.
- **Remediation:** §11 item 1. **Required before M3 exit: YES (blocking).**

CAP-SEC-011 (the second applicable HARD rule) was explicitly verified and **passed** — see §8. CAP-SEC-018 (third HARD) is NOT APPLICABLE (mcp ✗). No Critical rule was skipped.

## 4. Defects — SOFT-gate rule violations

Each: severity, evidence, why, CAPire basis, risk, remediation, M3-exit requirement. All CAPire sources CURRENT (§10).

### CAP-ARCH-003 *(supporting)* — Design services for single use cases — FAIL — High
- **Evidence:** `loadbuilder` exposes ~78 of the 88 domain entities (`srv/interaction_srv.cds` L11–L172) — configuration (`Config`, `CriticalityConfig`, `SalesOrderConfig`), master-data replicas, transactional data, and KPIs in one service consumed by **all six** UI apps and both roles (buyer users and admins).
- **Why:** detection step 2 fires exactly: entity count ≈ domain count, mixed concerns, single service for multiple distinct consumer groups. SAP: "We strongly recommend designing your services for single use cases"; 1:1 whole-model exposure "open[s] huge entry doors".
- **Risk:** authorization (once added) must be entity-by-entity instead of per-facade; every consumer sees every concern; API evolution couples all six apps.
- **Remediation:** §11 item 2 (service split is the strategic fix; §11 item 1 is the tactical containment). **Before M3 exit:** recommended; may proceed with recorded justification (retrospective reality: live API, split needs a deprecation path).

### CAP-SRV-001 — Expose use-case projections, not persistence entities — FAIL — High
- **Evidence:** every service entity is a bare `as projection on custom.X` with **zero** `excluding`/column selection anywhere in `srv/interaction_srv.cds`; internal fields exposed include `Variants.Owner` (user IDs, `db/interactions.cds` L39) and `Config.Destination` (L47–L48).
- **Why:** the whole persistence model is re-exposed 1:1; every element exposure is not a decision. (Nuance for calibration: much of the db model is itself a UI-shaped S/4 replica — see calibration item A-2.)
- **Risk:** API↔persistence coupling; internal/technical fields leak to all consumers; maximal writable surface for SEC-012 to defend.
- **Remediation:** §11 item 3. **Before M3 exit:** the field-level exposures (Owner, Destination) — yes; full re-cut — with item 2.

### CAP-SRV-002 — Rely on generic providers — FAIL — High
- **Evidence:** `srv/interaction_srv.js` L608–L618 — `on READ Variants` **replaces** the generic read with a hand-built `SELECT` that discards the client's query (`req.query` ignored → `$filter`/`$top`/`$skip`/`$search` silently lost) to implement row filtering that belongs in the model (see SEC-010); L540–L551 — `on READ CriticalityConfig` re-runs the query and re-implements sorting in memory (generic `$orderby` covers it).
- **Scoping note:** the ~30 further custom `on READ` handlers that delegate to S/4 remote services are **not** violations — federation genuinely requires custom handlers (calibration item SCP-1 proposes making this explicit in the rule).
- **Risk:** silently lost query semantics (a UI filter on Variants does nothing server-side); maintenance drift.
- **Remediation:** §11 item 4. **Before M3 exit:** with item 5 (same root cause).

### CAP-SRV-003 — Prefer declarative techniques — FAIL — Medium
- **Evidence:** declaratively expressible requirements implemented imperatively: Variants ownership filter (`@restrict … where` exists for exactly this — SEC-010), manual `CreatedAt` stamping (`srv/interaction_srv.js` L623 vs the `managed` aspect), and **no `@readonly` anywhere** although dozens of S/4 replica entities are read-only by nature yet served writable.
- **Consolidation:** the ownership-filter aspect of this finding is the same defect as CAP-SEC-010's — reported once, cross-referenced (calibration WCP-2).
- **Remediation:** §11 items 3–5. **Before M3 exit:** `@readonly` on replicas — yes (cheap, large risk reduction).

### CAP-SRV-005 — OData V4; V2 only as documented legacy — FAIL — High *(version-sensitive: register + L3 verification done)*
- **Evidence:** OData V2 is served via `@sap/cds-odata-v2-adapter-proxy` (`package.json`; wired in `srv/server.js` L7, L21–L35); consumer exists (`app/buyerapp/webapp/manifest.json`: `mainService` OData 2.0 at `/loadbuilder/Admin/`) — a legitimate SAP-documented reason (existing UI), **but no documented justification with review date exists anywhere** (the project has no docs/ADRs). Additionally the adapter package is the deprecated predecessor of the currently documented `@cap-js-community/odata-v2-adapter` (CAPire `guides/protocols/odata`, L3-verified 2026-08-12).
- **Risk:** compatibility debt on a deprecated protocol and a deprecated adapter package; undocumented → never revisited.
- **Remediation:** §11 item 6. **Before M3 exit:** the justification record — yes; adapter migration — plan with the cds upgrade.

### CAP-SRV-008 — Optimistic concurrency decision — FAIL — Medium *(CONCURRENT-EDIT)*
- **Evidence:** no `@odata.etag` anywhere; no recorded last-write-wins decision (no docs). Outside draft protection: `Variants` with `IsGlobal: true` are editable by every user **with no UPDATE guard at all** (handlers cover READ/CREATE/DELETE only — `srv/interaction_srv.js` L608–L632); `PurchaseReqnItemText` has a custom UPDATE path (L3849). Draft-enabled entities are correctly excluded (draft locking).
- **Risk:** lost updates on shared variants and item texts.
- **Remediation:** §11 item 7. **Before M3 exit:** decision required (annotation or recorded LWW).

### CAP-SEC-010 — Model instance-based authorization declaratively — FAIL — High *(INSTANCE-AUTH)*
- **Evidence:** row-level requirement exists (personal vs global variants) and is hand-rolled: READ filter L608–L618, DELETE owner check L627–L632, CREATE owner stamping L620–L625 — with **no model `@restrict`**, and two gaps the declarative form would not have: **UPDATE is entirely unguarded** (any user can overwrite another user's personal variant via generic UPDATE), and the DELETE check lets any user delete **global** variants (`variant.Owner !== req.user.id && !variant.IsGlobal` → global ⇒ no rejection — possibly intended, but nowhere decided/recorded).
- **Why:** detection step 3 verbatim: handlers implementing row filters for entities lacking a model restriction.
- **Risk:** horizontal privilege: cross-user variant overwrite/deletion.
- **Remediation:** §11 item 5. **Before M3 exit: YES** (real access-control gap, cheap fix).

### CAP-SEC-012 — Validate externally writable input declaratively — FAIL — High
- **Evidence:** no `@assert.*` anywhere; `@mandatory` partial (RP/UoM areas of `db/interactions.cds`); replicas not `@readonly` (mass-assignment surface, cross-ref SRV-003); `CriticalityConfig` range/sequence validation is imperative and **CREATE-only** (`srv/interaction_srv.js` L553–L606) while the entity is draft-enabled and SAP's documented caveat — "active entities can be updated directly, bypassing drafts" (`guides/uis/fiori`, L2-verified) — applies: **UPDATE bypasses all of it**; action parameters `getBulkATP(requests: String)` / `getUomConversion(requests: String)` are stringly-typed JSON, bypassing CDS-typed validation (only a `JSON.parse` try/catch, L3621–L3626).
- **Risk:** invalid criticality ranges via update path; arbitrary writes to replica caches; weakly validated operation input.
- **Remediation:** §11 item 8. **Before M3 exit: YES** for the UPDATE-path validation and `@readonly`; typed action signatures may be scheduled (V2-consumer compatibility check needed).

### CAP-SEC-014 *(supporting)* — Request-flooding limits deliberately configured — FAIL — Medium
- **Evidence:** pagination cap explicitly decided (`@cds.query.limit.max: 10`, `srv/interaction_srv.cds` L9 — see ambiguity A-3); but no `$expand` guard, no `$batch` limit, and no rate-limiting decision anywhere (no config, no route service, no ADR).
- **Why:** the rule enforces recorded decisions; three of four controls are undecided.
- **Remediation:** §11 item 9. **Before M3 exit:** no (M6/M8 re-verify; sketch expected at M3).

### CAP-ERR-005 *(supporting)* — Stable error codes / targets — FAIL — Medium
- **Evidence:** all 43 `req.error`/`req.reject` sites use hardcoded English message strings, no stable codes, no i18n bundles for messages (no `_i18n`/`srv/i18n`), no `target` on field-level validation errors consumed by Fiori UIs (e.g., `srv/interaction_srv.js` L558, L851).
- **Remediation:** §11 item 10. **Before M3 exit:** no (error contract matures by M4; single-locale decision may downscope it — needs owner decision).

## 5. Recommendations & observations (non-defects)

- **Cross-cutting security observation (out of M3 scope, do not lose):** `getBulkATP` builds remote OData URLs by string-interpolating client input (`srv/interaction_srv.js` ≈L3640: `…?Material='${Material}'&SupplyingPlant='${Plant}'…`) — an injection-pattern subject of **CAP-SEC-013 (M4 primary)**. Recorded here as a calibration false-negative-by-scoping (item C-1) and flagged for the M4 review.
- `loadbuilder.testPurchaseOrder()` (`srv/interaction_srv.cds` L17) is declared but has **no handler** — dead API surface returning runtime errors; remove or implement.
- `@cds.query.limit.max: 10` (see ambiguity A-3) — verify intent: as written it caps every generic query on the service at 10 rows.
- CAP major 7 is behind the register's current major (10); Node 26 vs cds 7 support matrix should be checked at M8 (VER rules own this).
- `AtpDetails`/KPI entities marked `@cds.persistence.skip` with custom handlers — correct CAP-native pattern, noted approvingly.

## 6. NOT ASSESSABLE rules

### CAP-PERF-001 — Reliable pagination — NOT ASSESSABLE *(version-applicability confirmed; workload fact missing)*
The feature exists in the installed runtime (cds 7.9.5) and five apps page V4 collections; but whether collections are **paged while concurrently modified** (the rule's applicability condition) cannot be established from the repository, and the profile spec forbids inferring workload flags. **Needed:** owner confirmation of the paging-under-modification scenario (open question **O-1**). If confirmed: `reliablePaging` is not enabled and no decision is recorded → would be FAIL; if denied: NOT APPLICABLE with the profile updated.

## 7. Applicability decisions

CAP-SRV-009 (media ✗), CAP-SEC-018 (mcp ✗) — NOT APPLICABLE per profile. CAP-SRV-004/-005/-007, CAP-SEC-010, CAP-PERF-001 conditionals triggered (custom_operations/odata/drafts/instance_authorization ✓). Supporting selection: all five checklist rows (see calibration WCP-1).

## 8. Full per-rule results

| Rule | Class | Gate | Verdict | Evidence pointer |
|---|---|---|---|---|
| CAP-SRV-001 | PRIMARY | SOFT | **FAIL** | `interaction_srv.cds` — zero tailoring across ~78 projections; `Variants.Owner`, `Config.Destination` exposed |
| CAP-SRV-002 | PRIMARY | SOFT | **FAIL** | `interaction_srv.js` L608 (client query discarded), L540 (reimplemented sort); federation handlers exempted |
| CAP-SRV-003 | PRIMARY | SOFT | **FAIL** | no `@readonly` model-wide; ownership filter imperative; manual timestamps |
| CAP-SRV-004 | PRIMARY | SOFT | PASS | 7 ops modeled as actions/functions with CDS types (obs.: read-only ops as actions for payload size — undocumented; `testPurchaseOrder` unimplemented) |
| CAP-SRV-005 | PRIMARY | SOFT | **FAIL** | V2 via deprecated proxy pkg, real V2 consumer (buyerapp manifest), no recorded justification |
| CAP-SRV-006 | PRIMARY | SOFT | PASS | exposure deliberate & visible: explicit `@path` per service, V2 proxy explicitly wired (`server.js` L21–35), dedicated V4 service for the V2 gap; obs.: intent in code, not `@protocol` annotations |
| CAP-SRV-007 | PRIMARY | SOFT | PASS | `@odata.draft.enabled` on 6 entities; no hand-built draft mechanisms; SEC-012 carries the active-entity caveat finding |
| CAP-SRV-008 | PRIMARY | SOFT | **FAIL** | no `@odata.etag`, no recorded decision; unguarded shared-entity UPDATE paths |
| CAP-SRV-009 | PRIMARY | SOFT | NOT APPLICABLE | media ✗ |
| CAP-SEC-001 | PRIMARY | **HARD** | **FAIL** | §3 |
| CAP-SEC-010 | PRIMARY | SOFT | **FAIL** | Variants row-auth in handlers only; UPDATE unguarded |
| CAP-SEC-011 | PRIMARY | **HARD** | PASS | exposure walk: `RP.items` composition → `RPItems` (`db/interactions.cds` L823) and `RPItems.RestrictionProfile` → `RP` (L828), both in-service; **no stricter-restricted target exists anywhere** (nothing is restricted — SEC-001). Caveat recorded: this walk MUST be repeated in the SEC-001 re-review, since remediation introduces the restriction differentials this rule guards |
| CAP-SEC-012 | PRIMARY | SOFT | **FAIL** | §4 |
| CAP-SEC-018 | PRIMARY | **HARD** | NOT APPLICABLE | mcp ✗ |
| CAP-PERF-001 | PRIMARY | SOFT | NOT ASSESSABLE | §6 (O-1) |
| CAP-ARCH-003 | SUPPORTING | SOFT | **FAIL** | §4 |
| CAP-CDS-007 | SUPPORTING | SOFT | PASS | UI annotation blocks correctly factored into `app/services.cds` + per-app annotation files; obs.: 269 inline `@title` labels in `db/interactions.cds` (scattered singles per detection step 2 → observation, not FAIL) |
| CAP-SEC-014 | SUPPORTING | SOFT | **FAIL** | §4 |
| CAP-ERR-001 | SUPPORTING | SOFT | PASS | 43 `req.error`/`req.reject` sites; the single raw `throw` (L4959) is an internal config failure where 500 is correct |
| CAP-ERR-005 | SUPPORTING | SOFT | **FAIL** | §4 |

## 9. Exceptions honored

None on file — the project has no exception records (no `docs/` directory). No ORG-PENDING rules in the evaluated set.

## 10. Standards & CAPire evidence verification

Per [capire-verification.md](../../reviews/capire-verification.md). 12 unique URLs; L2 explicit fetches for security rules, L3 for the two version-sensitive rules, L1 liveness+skim otherwise. All fetches live on 2026-08-12.

| Source (cap.cloud.sap/docs/…) | Rules | Level | Status |
|---|---|---|---|
| guides/security/authorization | SEC-001/-010/-011 | L2 | **CURRENT** (no-default-access statement re-confirmed; fetched earlier same day, recorded in [validation results](../../docs/validation-results-2026-08.md)) |
| guides/services/constraints | SEC-012, SRV-003 | L2 | **CURRENT** ("enforced automatically … request is ultimately rejected and the transaction rolled back") |
| guides/uis/fiori | SRV-007, SEC-012 | L2 | **CURRENT** (draft-bypass caveat re-confirmed verbatim) |
| guides/security/data-protection | SEC-014 | L2 | **CURRENT** — with a **governance note**: the page now also documents a Node.js `$batch` limit (`cds.odata.batch_limit`), which CAP-SEC-014's mechanism parenthetical (Java-only for batch) predates → SCP-2. Non-material to this verdict (the rule requires decisions; none exist) |
| guides/protocols/odata | SRV-005 | **L3** | **CURRENT** ("OData V2 is deprecated…" verbatim; current adapter `@cap-js-community/odata-v2-adapter` — project uses the deprecated predecessor) |
| guides/services/served-ootb | SRV-002/-008, PERF-001 | **L3** | **CURRENT** (skip-token dup/missing-rows statement, `reliablePaging` config keys, "only for OData V4", `@odata.etag` mechanism re-confirmed) |
| guides/services/providing-services | SRV-001/-003, ARCH-003 | L2 | **CURRENT** ("1:1 fashion", "huge entry doors", "single use cases" re-confirmed verbatim) |
| guides/services/custom-code | SRV-002/-003 | L1 | CURRENT (declarative-first statement re-confirmed) |
| guides/services/custom-actions | SRV-004 | L1 | CURRENT ("Actions modify data…, Functions retrieve data") |
| node.js/cds-serve | SRV-006 | L1 | CURRENT (same-day fetch, recorded in validation results) |
| node.js/events | ERR-001/-005 | L1 | CURRENT (req.reject/req.error, code→i18n lookup, `target` re-confirmed) |
| cds/aspects + guides/domain/ | CDS-007 | L1 | CURRENT |

Version register (`docs/version-management.md`, verified 2026-08-12) consulted for SRV-005/PERF-001. **Governance flags: 1 note** (SEC-014 mechanism evolution → SCP-2); no material source change; no STALE/REMOVED/UNAVAILABLE; no rule rewritten during review.

## 11. Remediation plan

Per [remediation-plan-template](../../templates/remediation-plan-template.md). No fixes applied (read-only review). Ordered by risk:

1. **CAP-SEC-001 (HARD, blocks):** introduce a dedicated authorization aspect file (e.g., `srv/access-control.cds`): `@requires` per service bound to the existing `Admin`/`User` scopes; operation-level restrictions for the S/4-writing actions. Test: allow/deny per role. Re-review: SEC-001 + a **repeat SEC-011 walk** + touched files.
2. **CAP-ARCH-003:** decide the target service cut (config/admin vs buyer vs data-maintenance facades); record as ADR with a deprecation path for the shared `/Admin` contract. May be justified-deferred at M3; the ADR is the M3 deliverable.
3. **CAP-SRV-001:** exclude `Variants.Owner` from the projection (server-set only) and stop exposing `Config.Destination` to non-admin consumers; introduce `excluding`/column lists opportunistically with item 2.
4. **CAP-SRV-002:** delete the `Variants` READ/DELETE handlers once item 5 lands; drop the CriticalityConfig sort handler (use `$orderby`/default order).
5. **CAP-SEC-010 (+SRV-003):** replace ownership handlers with `@restrict: [{ grant: '*', where: 'Owner = $user or IsGlobal = true' }]`-style model restrictions **covering UPDATE**; decide and record who may edit/delete global variants.
6. **CAP-SRV-005:** record the V2 justification (buyerapp, named controls, review date) — an ADR or annotation comment; plan migration to `@cap-js-community/odata-v2-adapter` together with the cds-major upgrade.
7. **CAP-SRV-008:** add `@odata.etag` (needs a `managed`/`modifiedAt` element) on `Variants` + `PurchaseReqnItemText`, or record last-write-wins.
8. **CAP-SEC-012:** `@readonly` on all replica entities; move CriticalityConfig range checks so they hold on UPDATE (or declare per-element `@assert.range` where expressible); typed signatures for `getBulkATP`/`getUomConversion` (check V2-consumer impact first).
9. **CAP-SEC-014:** record limit decisions ($expand guard or platform alternative, batch limit, rate limiting); re-examine `@cds.query.limit.max: 10` intent (A-3).
10. **CAP-ERR-005:** stable message codes + i18n bundles + `target` on field errors — schedule with M4 handler work; a documented single-locale decision would downscope the translation element.
11. **O-1 (owner):** confirm/deny paging-under-concurrent-modification (resolves PERF-001) and the `[OWNER-CONFIRM]` profile flags.

## 12. Outstanding risks & next-milestone readiness

**Risks:** unrestricted services on a deployed application (item 1) — highest priority; cross-user variant overwrite (item 5); injection-pattern observation for M4 (§5). **M4 entry criteria NOT met** (M3 gate FAIL). After remediation of items 1 and 5 (+3 minimum): `/capm-review-milestone re-review M3` scoped to the FAILED/NOT-ASSESSABLE rules + touched files, including the mandatory SEC-011 re-walk.

---

# Phase 4 Pilot Calibration

*(Pilot-specific section per the Phase 4 brief — observational; no rules were changed.)*

## Execution metrics

| Metric | Value |
|---|---|
| Review duration | ≈ 55 min wall clock (identification → profile → selection → evidence → CAPire → report) |
| Applicable rule count | 20 selected (15 primary + 5 supporting); 18 evaluated; 13 primary evaluated |
| Findings | 11 FAIL (1 Critical/HARD + 10 SOFT) · 0 ADVISORY · 1 NOT ASSESSABLE · 2 NOT APPLICABLE |
| CAPire | 12/12 sources CURRENT; 1 governance note (SEC-014 mechanism evolution); 0 STALE/REMOVED/UNAVAILABLE |
| Rule-text defects blocking evaluation | 0 |

## Step 8 — CAP-SRV-005 / CAP-SRV-006 calibration

**Finding: both rules are independently useful — confirmed by divergent verdicts on the same project.** SRV-005 FAILED (deprecated protocol version without recorded justification; deprecated adapter package) while SRV-006 PASSED (protocol exposure is a visible, deliberate decision — explicit `@path`s, explicitly wired proxy, a dedicated V4 service created to cover a V2 gap). They captured **different failure modes**: *which protocol generation and why* (005) vs *is exposure decided at all* (006). In the fixtures both failed together only because the fixtures bundled both defects into one service. **Recommendation:** keep both PRIMARY at M3; current wording sufficient; add a report convention (WCP-2) to consolidate into one finding block only when both fail for the same service with the same root cause. No rule change needed.

## Step 9 — SUPPORTING-rule selection calibration

Selected: all five checklist supporting rules (ARCH-003, CDS-007, SEC-014, ERR-001, ERR-005). **Why:** the command's criterion — "where this milestone's changes touch their subject" — is defined for *incremental* development flow; in a retrospective review of an existing codebase every subject is 'touched', so the criterion degenerates to 'all'. **Value observed:** ARCH-003 was the highest-value selection (it names the root cause behind SRV-001/SEC-001's breadth); SEC-014 produced a legitimate sketch-level finding; ERR-001 gave a cheap PASS (worth having); **ERR-005 at M3 is noise-adjacent** — a real defect, but its remediation belongs to M4 handler work and it lengthened the report without changing the gate; CDS-007 was cheap and produced a useful observation. **Missed supporting rule:** none from the checklist; but SEC-013 (M4) would have caught the injection pattern (C-1). **Recommendation (WCP-1):** define selection for retrospective reviews as "all checklist supporting rows, reported non-gating"; consider demoting ERR-005's M3 supporting appearance to M4-only. Matrix unchanged pending Round 2.

## Step 11 — False positive / false negative / noise / ambiguity

| # | Class | Rule | Evidence | Explanation | Recommendation |
|---|---|---|---|---|---|
| C-1 | **FALSE NEGATIVE** (by milestone scoping) | CAP-SEC-013 (M4, not in M3 set) | `interaction_srv.js` ≈L3640: client input interpolated into remote OData URL | A real injection-pattern defect visible at M3 that no M3-scope rule flags; the framework catches it only at M4 | SCP-3: consider SEC-013 as M3 SUPPORTING when `custom_operations ∧ remote_services`; interim: cross-cutting-observation section carried it (§5) |
| C-2 | **NOISE** (duplicate finding) | CAP-SRV-003 vs SEC-010/SEC-012 | Variants ownership filter counted by both SRV-003 and SEC-010 | Overlapping rule subjects produce two findings for one defect | WCP-2 consolidation convention: report once under the most specific rule, cross-reference the other verdicts |
| C-3 | **NOISE** (milestone fit) | CAP-ERR-005 at M3 | §4 last block | Valid finding, wrong milestone for actionability | Consider M4-only supporting appearance (with WCP-1) |
| A-1 | **AMBIGUITY** (workflow) | CAP-PERF-001 | §6 | Workload flags (`concurrent_paging`) cannot be established from a repository, and the spec forbids inference — without an owner, the conditional rule can neither fire nor be dismissed | WCP-3: codify "workload flags need owner confirmation; otherwise dependent rules are NOT ASSESSABLE, never silently filtered" |
| A-2 | **AMBIGUITY** (rule detection) | CAP-SRV-001 | whole-model 1:1 exposure of a *UI-shaped replica* model | Detection step "1:1 → FAIL" fires correctly, but the canonical remediation (excluding fields) misses the real fix when the persistence model **is** the use-case model (replica/cache architectures) — the defect is the service cut + writable surface | Add a detection note pointing replica architectures to ARCH-003 + `@readonly` as primary remediation; wording change deferred |
| A-3 | **AMBIGUITY** (evidence interpretation) | CAP-SEC-014 | `@cds.query.limit.max: 10` | An explicit-but-anomalous value (caps all generic queries at 10 rows; most lists are served by custom handlers, so effective behavior is unclear) — the rule records decisions but has no lens for "decided value looks wrong" | Acceptable: rule stays decision-focused; report carries it as an observation (§5). No change proposed |
| — | **FALSE POSITIVE** | — | — | **None identified.** Each FAIL was re-checked against its detection steps and concrete evidence; the SRV-002 federation exemption (SCP-1) prevented the one likely FP class | — |

## Step 13 — Developer experience

**Not assessed.** No project developer participated in this pilot; no feedback was collected and none is fabricated. *Reviewer-side* observations (not developer feedback): profile creation from repository evidence took ~10 minutes and the template comments were sufficient; rule selection via checklist+matrix+profile was mechanical except the SUPPORTING criterion (WCP-1); the 20-rule report is long but the findings are individually actionable; CAPire verification added real value twice (deprecated-adapter confirmation for SRV-005; the SEC-014 governance note). Whether a developer would adopt `/capm-develop`: unknown — Round 2 should include a developer.

## Step 14 — STANDARD CHANGE PROPOSALS (no changes applied)

**SCP-1 — CAP-SRV-002 (federation exemption made explicit).** *Current:* detection targets custom `on` CRUD handlers reimplementing generic behavior; remote-backed entities are not mentioned. *Problem:* in federation-heavy projects (~30 legitimate remote-delegating `on READ` handlers here), naive detection over-fires; this pilot exempted them by judgment. *Evidence:* `srv/interaction_srv.js` remote handlers vs L608 (genuine violation). *Proposed:* add detection step "custom handlers whose body delegates to a remote service for remote-backed entities are required custom logic, not reimplementation — skip; still check they honor `req.query` semantics". *SAP evidence:* consuming-services guide (remote delegation pattern). *Severity/authority/runtime/milestone impact:* none (detection clarification only).

**SCP-2 — CAP-SEC-014 (mechanism list update).** *Current:* parenthetical lists `$batch` limit as Java-only and Node `$expand` as custom-handler-only. *Problem:* CAPire (`guides/security/data-protection`, fetched 2026-08-12) now documents Node.js `cds.odata.batch_limit`. *Evidence:* live fetch quote in §10. *Proposed:* add the Node batch mechanism; re-verify the Node `$expand` stance on the same page. *Impact:* none on severity/authority/runtime/milestones; verdict here unaffected (decisions absent either way).

**SCP-3 — Matrix: CAP-SEC-013 conditional SUPPORTING at M3.** *Current:* SEC-013 first appears at M4. *Problem:* injection-relevant surface (custom operations building remote queries) is reviewable at M3 and was found only via an out-of-scope observation. *Evidence:* C-1. *Proposed:* SUPPORTING row at M3 with condition `CUSTOM-OPS ∧ REMOTE`. *Impact:* milestone mapping only; rule text unchanged.

## Workflow change proposals

**WCP-1:** define SUPPORTING selection for retrospective reviews (all checklist rows, non-gating). **WCP-2:** finding-consolidation convention for overlapping rules (answers Step 8 too). **WCP-3:** workload-flag confirmation rule (owner input or NOT ASSESSABLE — never reviewer inference, never silent filtering). **WCP-4:** name "retrospective review of a LIVE project" explicitly in the command's milestone-plausibility step (step 3).

## Step 15 — Pilot outcome: **AMBER**

**Justification:** the framework reliably assessed a real, messy, federation-heavy production codebase — correct gate result, 11 actionable findings, zero false positives, all 12 CAPire sources verified, no rule-text defect blocked evaluation (GREEN territory). AMBER rather than GREEN because the calibration items are *meaningful*, not cosmetic: the SUPPORTING-selection criterion is undefined for the pilot's own review mode (WCP-1), the workload-flag rule changed a verdict class (A-1/WCP-3), one detection would over-fire without reviewer judgment (SCP-1), and a milestone-scoping false negative hid a security-relevant pattern (SCP-3). None of these made the review unreliable (not RED); all of them should be resolved before Round 2 treats results as repeatable.

---
*Scope statement: this pilot report evaluates only the applicable M3 standards listed in §8 at standard version `791de1e`. It makes no claim about overall CAP best-practice compliance, production readiness, or any other milestone.*

# CAP-PRIV — Data privacy & audit

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-16 (legal sufficiency/retention periods), G-17 (DRM guide in-flux).

**Rules:** 4 active (0 Critical, 3 High, 1 Medium). All SAP references verified against official CAP documentation on **2026-08-12** — including a page-status check: the **Data Retention Manager guide (`dpp-drm`) is explicitly "Under construction"** with no rendered normative content; no rule below claims SAP authority from it.

**Privacy vs security:** this category owns personal-data *classification, data-subject rights, audit of personal-data events, and retention* — unauthorized access, authentication, and tenant isolation remain [CAP-SEC](security.md)/[CAP-MT](multitenancy.md). SAP's own disclaimer applies throughout: decisions related to data protection "must be made on a case-by-case basis, considering the given system landscape and the applicable legal requirements" — these rules operationalize the documented mechanisms, they do not constitute legal compliance (G-16).

**Applicability precondition:** rules apply to projects processing personal data (data about identifiable natural persons). Projects verifiably without personal data are NOT APPLICABLE throughout — state the model-review evidence.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-PRIV-001 | Annotate personal data completely with `@PersonalData` | High | SAP-REC | Both |
| CAP-PRIV-002 | Run audit logging for personal-data events in production | High | SAP-REC | Both |
| CAP-PRIV-003 | Protect and flatten Personal Data Manager services | High | SAP-REQ | Both |
| CAP-PRIV-004 | Maintain a documented retention and erasure approach | Medium | GEN | Both |

---

## CAP-PRIV-001 — Annotate personal data completely with `@PersonalData`

| Field | Value |
|---|---|
| **Rule ID** | CAP-PRIV-001 |
| **Title** | Annotate personal data completely with `@PersonalData` |
| **Category** | Data privacy & audit |
| **Severity** | High |
| **Authority** | SAP-REC (the "first and frequently only task" of the developer; the DataSubjectID element requirement is documented REQ-substance: each annotated entity "needs to identify a DataSubjectID element") |
| **Applicability** | All entities/elements holding personal data |
| **Runtime** | Both (model-level annotations) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-PRIV-002 (annotations drive audit logging), CAP-PRIV-003/-004 (and PDM/retention integration), CAP-SEC-016 (PII in logs), CAP-CDS-007 (the separate-file pattern); **absorbs deferred candidate CAP-SEC #17 and candidate CAP-PRIV #2** |
| **Last verified** | 2026-08-12 |

### Rule statement
Every entity and element holding personal data MUST be annotated with `@PersonalData` — SAP: "The first and frequently only task to do as an application developer is to identify entities and elements (potentially) holding personal data using `@PersonalData` annotations." Annotations MUST be structurally complete: `EntitySemantics` (`DataSubject` / `DataSubjectDetails` / `Other`) per annotated entity, each of which "needs to identify a DataSubjectID element"; element-level `IsPotentiallyPersonal` (audit on modification) and `IsPotentiallySensitive` (audit on read) per the data's nature. Following SAP's documented practice, the annotations live in a dedicated file (e.g., `srv/data-privacy.cds` — "Following the best practice of separation of concerns"). The personal-data inventory from milestone M0 is the completeness baseline.

### Rationale
The annotations are the switch for CAP's entire privacy automation: audit logging (CAP-PRIV-002), Personal Data Manager (CAP-PRIV-003), and retention integration all key on `@PersonalData`. Unannotated personal data is invisible to all of it — modifications unaudited, data-subject access reports incomplete — while structurally incomplete annotations (missing DataSubjectID) break the association between data and data subject that every downstream mechanism needs. Note on `IsPotentiallySensitive`: each read of a sensitive-tagged element produces an audit event — tag deliberately per the data's actual sensitivity (this operational consequence is our rationale; SAP documents the mechanics, not a "be selective" instruction). **High:** unannotated personal data defeats the documented protection mechanisms wholesale — a compliance-exposure class — below Critical because the annotation itself grants no access (that is CAP-SEC's domain).

### Implementation guidance
- Start from the M0 personal-data inventory; walk every entity: data *about* a person (`DataSubject` — e.g., Customers), data *belonging to* a subject's context (`DataSubjectDetails`/`Other` — e.g., their orders), transactional references.
- Remember indirect stores: `managed` audit fields (`createdBy`/`modifiedBy` hold user IDs), message payloads persisted via the queue, draft copies.
- Keep the annotation file under model review whenever entities change (drift check below).

### Evidence expected in code
A dedicated privacy-annotation file covering the personal-data inventory; `EntitySemantics` + DataSubjectID per annotated entity; element-level tags on personal/sensitive elements; review notes tying the annotations to the M0 inventory.

### Detection guidance
1. Locate `@PersonalData` annotations (dedicated file or inline). None while the domain plausibly holds personal data (names, emails, user references, addresses in the model) → FAIL (High).
2. For each annotated entity: verify `EntitySemantics` present and a DataSubjectID element identified → missing → FAIL per entity (documented structural need).
3. Completeness sweep: search the model for personal-data-typed elements (name/email/phone/address/birthdate patterns, user-ID references incl. `createdBy`/`modifiedBy` on personal contexts) not covered by annotations → each gap → FAIL.
4. Check element-level tagging plausibility: sensitive-by-nature fields (health, financial identifiers) lacking `IsPotentiallySensitive` → FAIL; blanket sensitive-tagging of everything → observation (audit-volume consequence, our rationale).
5. Verify the dedicated-file pattern (CAP-CDS-007 consistency) → inline scattering → observation.
6. NOT APPLICABLE only with model-review evidence of no personal data.

### Good example
```cds
// srv/data-privacy.cds — separate concern, complete semantics
using { acme.bookshop as my } from '../db/schema';
annotate my.Customers with @PersonalData: {
  EntitySemantics: 'DataSubject'
} {
  ID           @PersonalData.FieldSemantics: 'DataSubjectID';
  email        @PersonalData.IsPotentiallyPersonal;
  creditRating @PersonalData.IsPotentiallySensitive;   // read-audited — deliberate
};
annotate my.Orders with @PersonalData: {
  EntitySemantics: 'Other'
} {
  customer @PersonalData.FieldSemantics: 'DataSubjectID';
};
```

### Bad example
```text
Customers, Orders, and SupportTickets hold names, emails, and user IDs;
no @PersonalData anywhere. Audit logging (CAP-PRIV-002) is installed
but logs nothing — the switch it keys on was never set.
```

### Exception guidance
Truly non-personal domains (pure reference/measurement data) are the NOT APPLICABLE path, evidenced by model review. Pseudonymized data requires a documented assessment (whether re-identification is possible) before dropping annotations — a G-16 legal question, not a developer shortcut.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/data-privacy ("first and frequently only task"; automation driven by the annotations; case-by-case disclaimer)
- https://cap.cloud.sap/docs/guides/security/dpp-annotations (EntitySemantics values; DataSubjectID element "needs to identify"; IsPotentiallyPersonal/IsPotentiallySensitive semantics; `srv/data-privacy.cds` separation)

---

## CAP-PRIV-002 — Run audit logging for personal-data events in production

| Field | Value |
|---|---|
| **Rule ID** | CAP-PRIV-002 |
| **Title** | Run audit logging for personal-data events in production |
| **Category** | Data privacy & audit |
| **Severity** | High |
| **Authority** | SAP-REC (documented plugin/service mechanism; production binding is the documented setup) |
| **Applicability** | Production deployments processing personal data |
| **Runtime** | Both — Node.js: `@cap-js/audit-logging` plugin (documented in detail); Java: CAP Java's own AuditLog service (the DPP page defers to the Java documentation at `/docs/java/auditlog` — Java specifics verified there before citing details) |
| **CAP version** | Current plugin/service state per docs |
| **Status** | Active |
| **Related rules** | CAP-PRIV-001 (the annotations that drive it), CAP-EVT-002 (audit events ride the transactional queue by default — do not disable), CAP-SEC-016 (application logs are NOT audit logs — the distinction this rule embodies), G-06 (authorization-decision logging remains an open gap — not covered by this mechanism) |
| **Last verified** | 2026-08-12 |

### Rule statement
Projects processing personal data MUST run the documented audit-logging mechanism in production: Node.js — the `@cap-js/audit-logging` plugin, which provides "automatic audit logging of data privacy-related events, in particular changes to *personal data* and reads of *sensitive* data" (old/new values logged for modifications), delivered "through a transactional outbox" by default (keep it so — CAP-EVT-002), with a bound SAP Audit Log service instance in production (`audit-log-to-restv2`; console variant is development-only); Java — the CAP Java AuditLog service per its documentation. Application logs MUST NOT be presented as the audit trail (CAP-SEC-016 hygiene ≠ audit evidence).

### Rationale
Audit logging is the enforcement half of the annotation work: with CAP-PRIV-001 in place, the mechanism automatically records who changed personal data and who read sensitive data — the evidence data-protection processes require. Console-only or absent audit logging in production means personal-data events leave no durable, tamper-resistant trail; routing around the outbox reintroduces lost-audit-events on rollback/crash (the exact failure CAP-EVT-002 prevents). **High:** missing production audit trail for personal-data processing is a compliance-evidence failure discovered exactly when it's needed (incident, data-subject request, audit) — below Critical as it exposes nothing by itself.

### Evidence expected in code
Node: `@cap-js/audit-logging` in dependencies; `[production]` audit-log kind `audit-log-to-restv2` (or current documented production kind); `auditlog` service instance (premium plan per docs) in `mta.yaml`/chart. Java: AuditLog feature configured per the Java docs with the service binding. Queue delivery not disabled.

### Detection guidance
1. Confirm personal data is processed (CAP-PRIV-001 results) → else NOT APPLICABLE.
2. Node: check `package.json` for `@cap-js/audit-logging`; production profile kind (`audit-log-to-restv2` vs console); `auditlog` service resource bound in deployment descriptors → plugin absent or console-only in production → FAIL.
3. Java: check the AuditLog service configuration and binding per `/docs/java/auditlog` → absent while personal data is processed → FAIL (verify specifics against the Java page — NOT ASSESSABLE if that page's current content can't be checked).
4. Verify queue delivery intact: audit-log service not configured `outboxed: false`/immediate in production → violation → FAIL here and under CAP-EVT-002.
5. Spot-check the wiring end-to-end where a test exists (modify an annotated field → audit event asserted) → note as positive evidence; absence → missing-evidence element.
6. Confirm no documentation claims application logs as the audit trail → such claims → FAIL (misrepresentation).

### Good example
```jsonc
// package.json (Node.js)
{ "dependencies": { "@cap-js/audit-logging": "^1" },
  "cds": { "requires": { "audit-log": {
      "[development]": { "kind": "audit-log-to-console" },
      "[production]":  { "kind": "audit-log-to-restv2" }
} } } }
```

### Bad example
```text
@PersonalData annotations complete; audit plugin left on
audit-log-to-console in production — every sensitive read "audited"
into stdout, rotated away within hours. No Audit Log service instance
exists. The audit trail is a log file that no longer exists.
```

### Exception guidance
Landscapes with a mandated central audit system may substitute it — documented per CAP-ARCH-007, with equivalent coverage of the annotation-driven events demonstrated. Development/test profiles legitimately use the console variant.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/dpp-audit-logging (automatic logging of personal-data changes and sensitive reads; old/new values; transactional outbox by default; production `auditlog` service with `audit-log-to-restv2`; Java pointer to `/docs/java/auditlog`)

---

## CAP-PRIV-003 — Protect and flatten Personal Data Manager services

| Field | Value |
|---|---|
| **Rule ID** | CAP-PRIV-003 |
| **Title** | Protect and flatten Personal Data Manager services |
| **Category** | Data privacy & audit |
| **Severity** | High |
| **Authority** | SAP-REQ (documented integration requirements: flat structures; `@requires: 'PersonalDataManagerUser'` with the role granted to the PDM service instance) |
| **Applicability** | Projects integrating SAP Personal Data Manager (enterprise accounts only — documented); NOT APPLICABLE otherwise |
| **Runtime** | Both (documented walkthrough is Node.js; the service-modeling and protection requirements are model/config-level) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-PRIV-001 (prerequisite annotations), CAP-SEC-001 (this is its PDM-specific instantiation), CAP-SRV-001 (a documented legitimate flat-view case), CAP-PERF-002 (ditto for real joins) |
| **Last verified** | 2026-08-12 |

### Rule statement
PDM-facing services MUST follow the documented integration shape: **flat** projections ("SAP Personal Data Manager needs flattened out structures" — helper views flattening compositions), and the endpoint protected with `@requires: 'PersonalDataManagerUser'`, with that role granted to the **PDM service instance** (via the security configuration) — not to business users. The service exposes exactly what PDM needs for data-subject information retrieval, nothing broader.

### Rationale
A PDM service is, by construction, an aggregation endpoint over a data subject's personal data across the model — the most privacy-dense surface in the application. The documented role gate scopes it to the PDM service instance; without it, the aggregation endpoint is reachable by whoever satisfies weaker defaults, and CAP-SEC-001's generic review may under-rate its sensitivity. **High:** an unprotected or over-broad PDM endpoint is concentrated personal-data exposure — below Critical only because the documented setup makes the gate a single, checkable annotation.

### Evidence expected in code
A dedicated PDM service with flat views over annotated entities; `@requires: 'PersonalDataManagerUser'` on it; the role granted to the PDM instance in the security descriptor; PDM subscription/binding artifacts.

### Detection guidance
1. Confirm PDM integration exists (PDM service instance in descriptors, PDM-facing CDS service) → absent → NOT APPLICABLE.
2. Check the PDM service definition: `@requires: 'PersonalDataManagerUser'` present → missing/weaker (`authenticated-user`) → FAIL (High; also report under CAP-SEC-001).
3. Verify the role grant targets the PDM service instance (xs-security/grant configuration), not business role collections → business-user assignment → FAIL (cross-ref CAP-SEC-009's separation discipline).
4. Check flatness: PDM views flatten compositions/associations into the documented flat shape → deep/nested exposure → FAIL (documented integration need).
5. Check exposure breadth: only data-subject-relevant projections in the PDM service → unrelated entities → FAIL (CAP-SRV-001 instantiation).

### Good example
```cds
@requires: 'PersonalDataManagerUser'      // granted to the pdm instance only
service PDMService {
  entity CustomerView as projection on my.Customers { ID, email, name };
  // flattened helper view — PDM needs flat structures
  entity ConversationView as projection on my.IncidentConversations {
    key ID, incident.customer.ID as customerID, timestamp, author
  };
}
```

### Bad example
```cds
@requires: 'authenticated-user'           // any logged-in user reaches the
service PDMService {                       // aggregated personal-data surface
  entity Everything as projection on my.Customers;   // nested, over-broad
}
```

### Exception guidance
None on the protection requirement. Alternative data-subject-access implementations (not PDM) are assessed under CAP-SEC-001/CAP-SRV-001 generally plus PRIV-004's documented-approach duty.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/dpp-pdm (flat structures; `@requires: 'PersonalDataManagerUser'`; role granted to the PDM service instance; enterprise-account constraint)

---

## CAP-PRIV-004 — Maintain a documented retention and erasure approach

| Field | Value |
|---|---|
| **Rule ID** | CAP-PRIV-004 |
| **Title** | Maintain a documented retention and erasure approach |
| **Category** | Data privacy & audit |
| **Severity** | Medium |
| **Authority** | GEN (general compliance-engineering practice). **No SAP authority is claimed:** the DRM guide (`dpp-drm`) is under construction with no rendered normative content (verified 2026-08-12); the data-privacy hub names Data Retention Manager integration for the right to be forgotten as the direction, and SAP's disclaimer delegates decisions case-by-case. Retention periods/legal grounds are ORG/legal (G-16); the SAP mechanism is in-flux (G-17) |
| **Applicability** | Projects processing personal data with a production target |
| **Runtime** | Both |
| **CAP version** | ⏱ Mechanism landscape in-flux — re-verify the DRM/`@cap-js/data-privacy` state at adoption time (G-17) |
| **Status** | Active |
| **Related rules** | CAP-PRIV-001 (classification is the prerequisite), CAP-PRIV-002 (erasure actions are themselves audit-relevant), CAP-MT (tenant offboarding intersects — unsubscribe data handling), G-16/G-17 |
| **Last verified** | 2026-08-12 |

### Rule statement
Projects processing personal data MUST have a *documented* retention-and-erasure approach before production: which personal data is retained how long (periods = ORG/legal policy, G-16), how data-subject erasure ("right to be forgotten") is fulfilled technically, and how erasure interacts with audit logs, drafts, queue contents, and backups. The approach names its mechanism — SAP's documented direction is Data Retention Manager integration (guide currently under construction — verify its state at adoption, G-17), or a documented custom erasure implementation. This rule requires the *documented decision and mechanism*, not specific periods, and claims no SAP mandate.

### Rationale
Erasure obligations exist independent of SAP's documentation state — but CAP applications have personal data in more places than the primary tables (managed audit fields, drafts, `cds.outbox.Messages` payloads, audit logs themselves), so an undesigned "we'll delete the row" answer is incomplete by construction. Recording the approach forces the inventory-to-mechanism mapping while the SAP-native mechanism matures. **Medium:** a process/readiness control (M9 gate material); actual unlawful retention is a legal determination outside this standard (G-16).

### Implementation guidance
- Map the CAP-PRIV-001 inventory to erasure paths: primary entities, `createdBy`/`modifiedBy` occurrences, draft copies, queued payloads (CAP-EVT-004's header rules already limit spread), tenant-offboarding data (CAP-MT lifecycle).
- Watch the DRM guide and `@cap-js/data-privacy` plugin state at each re-verification cycle (G-17) — adopt the SAP-native path when it stabilizes rather than growing custom machinery (CAP-native-first).

### Evidence expected in code
A retention/erasure document (runbook/ADR) covering the inventory; the implementing mechanism (DRM artifacts when adopted, or custom erasure services/jobs) matching the document; erasure actions audit-logged.

### Detection guidance
1. Confirm applicability (personal data + production target).
2. Locate the retention/erasure document → none → FAIL (Medium: undecided approach).
3. Check coverage against the CAP-PRIV-001 inventory: primary data, managed-field user IDs, drafts, queue payloads, backups/tenant offboarding addressed → material omissions → FAIL element per omission.
4. Verify the implementing mechanism exists and matches the document (erasure service/job/DRM integration) → document without implementation → FAIL (paper approach).
5. Check erasure actions produce audit evidence (CAP-PRIV-002) → observation if absent.
6. Retention *periods* are reviewed for existence, not correctness (legal question, G-16) — mark period-adequacy NOT ASSESSABLE.

### Good example
```text
docs/privacy/retention.md — inventory-mapped approach (ADR-0027):
  Customers: erase on request via anonymization job (IDs → tombstone),
  cascaded to Orders.customer refs; drafts purged; outbox drained check;
  audit-log entries retained per legal hold (G-16 decision 2026-05);
  tenant offboarding: HDI container drop per CAP-MT lifecycle.
srv/privacy-service.cds — erasure action, audit-logged, role-restricted.
```

### Bad example
```text
Production SaaS processing customer data; deletion story: "customers
can email support". No document, no mechanism, no answer for drafts,
outbox payloads, or createdBy fields.
```

### Exception guidance
Organizations with a central privacy-operations platform document the hand-off instead of a per-project mechanism. Short-lived pilots without real personal data fall under the NOT APPLICABLE precondition — with the evidence stated.

### SAP reference
None normative (authority: GEN; gaps G-16/G-17). Related SAP reading: https://cap.cloud.sap/docs/guides/security/data-privacy (DRM named for the right to be forgotten; case-by-case disclaimer); https://cap.cloud.sap/docs/guides/security/dpp-drm (**under construction** — verified 2026-08-12).

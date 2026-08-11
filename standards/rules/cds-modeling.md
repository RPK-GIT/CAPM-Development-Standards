# CAP-CDS — Domain modeling & CDS

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-18, G-19, G-20 in [research-gaps.md](../../references/research-gaps.md).

**Rules:** 11 active (0 Critical, 1 High, 8 Medium, 2 Low). All SAP references verified against official CAP documentation on **2026-08-11**.

Scope boundaries: what services *expose* of the model is [CAP-SRV-001](services-apis.md); performance-motivated modeling (calculated fields in filters, key shape for JOINs) belongs to the future `CAP-PERF` category; `@PersonalData` modeling to the future `CAP-PRIV` category.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-CDS-001 | Follow SAP's CDS naming conventions | Low | SAP-REC | Both |
| CAP-CDS-002 | Prefer canonic UUID keys via `cuid` | Medium | SAP-REC | Both |
| CAP-CDS-003 | Treat UUIDs as opaque values | Medium | SAP-REC | Both |
| CAP-CDS-004 | Use `@sap/cds/common` reuse aspects and types | Medium | SAP-REC | Both |
| CAP-CDS-005 | Prefer managed associations | Medium | SAP-REC | Both |
| CAP-CDS-006 | Model containment with compositions — and avoid composition-of-one | High | SAP-REC | Both |
| CAP-CDS-007 | Separate secondary concerns via aspects | Medium | SAP-REC | Both |
| CAP-CDS-008 | Use `localized` for translatable text | Medium | SAP-REC | Both |
| CAP-CDS-009 | Model time-dependent data with the `temporal` aspect — writes in custom handlers | Medium | SAP-REQ | Both |
| CAP-CDS-010 | Prefer flat models; custom types only with reuse value | Low | SAP-REC | Both |
| CAP-CDS-011 | Choose stable namespaces | Medium | SAP-REC | Both |

---

## CAP-CDS-001 — Follow SAP's CDS naming conventions

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-001 |
| **Title** | Follow SAP's CDS naming conventions |
| **Category** | Domain modeling & CDS |
| **Severity** | Low |
| **Authority** | SAP-REC (documented naming checklist) |
| **Applicability** | All CDS models |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-011; org-specific naming vocabulary beyond SAP's checklist is gap G-19 |
| **Last verified** | 2026-08-11 |

### Rule statement
CDS models SHOULD follow SAP's documented naming conventions: entity and type names start with capital letters; entities are plural (`Authors`), types singular (`Genre`); element names start lowercase; technical primary keys are named `ID`; names are concise and do not repeat their context (`Authors.name`, not `Authors.authorName`).

### Rationale
SAP's domain-modeling guide states these as an imperative checklist ("Start entity and type names with capital letters… Use plural form for entities… Use `ID` for technical primary keys… Don't repeat contexts"). Consistent naming is what makes models "concise and comprehensible" — the guide's stated goal — and lets every CAP-literate reader parse a model instantly. **Low:** style-level; violations cost readability, nothing functional. (Multi-word casing style and service/action naming vocabulary are NOT prescribed by SAP — organization additions belong to gap G-19, not this rule.)

### Evidence expected in code
Models in `db/**/*.cds` (and service models) following the checklist.

### Detection guidance
1. Parse entity/type/element names from `db/**/*.cds` and `srv/**/*.cds`.
2. Flag: lowercase entity names, singular collection entities (`entity Book` holding many books), uppercase-first elements, technical keys not named `ID`, context-repeating elements (`Books.bookTitle`) → each an observation; systematic violation across the model → FAIL (Low).
3. Do not flag externally imported models (`srv/external/**`) — their naming follows the remote system.
4. Report representative examples with file:line, not every instance.

### Good example
```cds
entity Authors : cuid {        // capitalized, plural, ID via cuid
  name  : String(111);          // lowercase, no context repetition
  books : Association to many Books on books.author = $self;
}
type Genre : String enum { fiction; nonFiction; }   // singular type
```

### Bad example
```cds
entity book {                   // lowercase, singular
  key book_id : UUID;           // not ID
  BookTitle   : String;         // uppercase element, repeats context
}
```

### Exception guidance
Imported/external models and models mirroring an existing API contract keep their source naming. Domain terms that are naturally singular collectives (e.g., `Equipment`) are fine — the plural rule follows language, not grammar pedantry.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ (naming checklist)

---

## CAP-CDS-002 — Prefer canonic UUID keys via `cuid`

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-002 |
| **Title** | Prefer canonic UUID keys via `cuid` |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC ("Prefer canonic primary keys" / "Prefer UUIDs for primary keys") |
| **Applicability** | New domain entities requiring technical keys |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-003, CAP-CDS-004; key shape vs JOIN performance → future CAP-PERF |
| **Last verified** | 2026-08-11 |

### Rule statement
New entities needing technical primary keys SHOULD use the canonic `cuid` aspect from `@sap/cds/common` (`key ID : UUID`, auto-filled by generic providers). Database sequences SHOULD be used only for genuinely high data volumes, per SAP's guidance. Natural/semantic keys remain legitimate where the domain provides stable ones (code lists, ISO codes) — the rule targets *technical* keys.

### Rationale
SAP: "Prefer canonic primary keys", "Prefer UUIDs for primary keys", and "Use DB sequences only if you really deal with high data volumes. Otherwise, prefer UUIDs." Canonic keys make entities uniformly addressable (generic providers fill them automatically), avoid central key coordination in distributed/offline scenarios, and keep reuse aspects and integrations interoperable. **Medium:** inconsistent key strategies create integration friction and boilerplate, but nothing breaks outright.

### Evidence expected in code
`: cuid` includes (or `key ID : UUID`) on technical-keyed entities; sequences only where a recorded volume rationale exists; semantic keys only where the domain justifies them.

### Detection guidance
1. List entities in `db/**/*.cds` with their key definitions.
2. Flag technical keys deviating from the pattern: auto-increment/sequence-based keys (`@cds.on.insert` counters, `Integer` keys with sequence config) without a recorded high-volume rationale → FAIL per entity.
3. Distinguish semantic keys (currency codes, country codes, document numbers from a business rule) — compliant, note as such.
4. Check `cuid` is used rather than hand-typed `key ID : UUID` where `@sap/cds/common` is already imported (consistency, cross-ref CAP-CDS-004) — observation only.
5. Report with file:line.

### Good example
```cds
using { cuid, managed } from '@sap/cds/common';
entity Orders : cuid, managed { /* … */ }
```

### Bad example
```cds
entity Orders {
  key orderNo : Integer;   // sequence-managed technical key, no volume
                           // rationale recorded — coordination + interop cost
}
```

### Exception guidance
High-volume entities may use sequences per SAP's own carve-out — record the volume rationale (ADR or model comment). Entities with genuine natural keys (code lists per CAP-CDS-004's `CodeList`) are not exceptions; they're the other compliant case.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ (prefer canonic keys/UUIDs; sequences only for high volumes)

---

## CAP-CDS-003 — Treat UUIDs as opaque values

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-003 |
| **Title** | Treat UUIDs as opaque values |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented as an "anti pattern" with Don't/Avoid wording — not literal must/never) |
| **Applicability** | All code and models handling UUID keys |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-002, CAP-SEC-012 (don't add `@assert.format` to UUID keys) |
| **Last verified** | 2026-08-11 |

### Rule statement
UUID key values MUST be treated as opaque: no validation against RFC 4122 or hyphen/format expectations, no upper/lowercase assumptions or normalization, no string↔binary conversions. Integrations MUST accept foreign key formats (e.g., 32-character ABAP GUIDs) unchanged.

### Rationale
SAP: "It is an unfortunate anti pattern to validate UUIDs, such as for compliance with RFC 4122… **UUIDs are unique opaque values!**" — with explicit bullets to avoid case assumptions, conversions, and format validations. Violations surface exactly at integration time: valid foreign keys get rejected or corrupted by normalization, breaking data exchange with SAP systems. **Medium:** an integration-correctness defect class; scoped, not systemic. (Authority is SAP-REC per the verified "anti pattern / Don't / Avoid" wording; the Phase 1 inventory's REQ label was downgraded accordingly.)

### Evidence expected in code
No UUID-format regexes/validators on key elements; no `.toLowerCase()`/`.toUpperCase()`/re-formatting of key values; no binary UUID storage conversions.

### Detection guidance
1. Search handlers and utilities for UUID validation patterns: regexes matching `[0-9a-f]{8}-…`, libraries like `uuid.validate` applied to inbound keys → FAIL per site.
2. Search for case normalization or reformatting applied to `ID`/`*_ID` values before storage/comparison → FAIL.
3. Check models: `@assert.format` with UUID regexes on key elements → FAIL (also mis-scoped per CAP-SEC-012 guidance).
4. Check integration import code accepts foreign key formats without transformation.
5. Report with file:line.

### Good example
```js
// keys pass through untouched — equality and storage only
const order = await SELECT.one.from(Orders).where({ ID: req.data.orderID });
```

### Bad example
```js
// rejects valid ABAP GUIDs and "normalizes" stored keys
if (!/^[0-9a-f]{8}-[0-9a-f]{4}-.*$/.test(req.data.orderID))
  return req.reject(400, 'invalid UUID');
req.data.orderID = req.data.orderID.toLowerCase();
```

### Exception guidance
Generating well-formed UUIDs for *new* records (framework default) is unaffected — the rule governs treatment of *incoming/stored* values. A documented boundary that must enforce a partner-contracted format may validate at that boundary only, with the contract referenced.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ ("UUIDs are unique opaque values!"; avoid validations, case assumptions, conversions)

---

## CAP-CDS-004 — Use `@sap/cds/common` reuse aspects and types

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-004 |
| **Title** | Use `@sap/cds/common` reuse aspects and types |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All domain models with audit fields, code-list-like data, or country/currency/language elements |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-002 (`cuid`), CAP-CDS-009 (`temporal`), CAP-SRV-002 |
| **Last verified** | 2026-08-11 |

### Rule statement
Models SHOULD use the pre-defined aspects and types from `@sap/cds/common` instead of hand-modeled equivalents: `managed` for created/modified audit elements (instead of hand-written `@cds.on.insert/update` fields), `cuid` for canonic keys, `Country`/`Currency`/`Language` (and `CodeList`) for their respective concepts. Hand-rolled equivalents require a reason.

### Rationale
SAP documents `@sap/cds/common` as "proven best practices captured from real applications", fostering "interoperability between all applications" with concise models, minimal footprint, and out-of-the-box support (localized code lists, value helps). The guide shows hand-modeled `@cds.on.insert/update` fields only as the verbose alternative that `managed` replaces "to keep our core domain model clean and comprehensible". Hand-rolled variants diverge in naming and semantics, breaking reuse and tooling expectations. **Medium:** interoperability and conciseness; functionally the hand-rolled annotations work — the loss is consistency and ecosystem fit.

### Evidence expected in code
`using { cuid, managed, Country, Currency, … } from '@sap/cds/common'` with aspects applied; absence of hand-modeled `createdAt/changedBy/...` audit fields and hand-built country/currency entities duplicating the common ones.

### Detection guidance
1. Check imports of `@sap/cds/common` in `db/**/*.cds`.
2. Search for hand-modeled audit fields (`createdAt`, `createdBy`, `modifiedAt`, `changedOn`, … with `@cds.on.insert/update`) on entities not using `managed` → FAIL per entity (name the aspect that replaces them).
3. Search for custom country/currency/language entities or plain `String` elements semantically holding those concepts → FAIL/observation depending on clarity.
4. Custom code-list-like entities (code + localized text) not based on `CodeList` → observation with recommendation.
5. Report with file:line.

### Good example
```cds
using { cuid, managed, Currency } from '@sap/cds/common';
entity Orders : cuid, managed {
  currency : Currency;    // association to sap.common.Currencies incl. value help
}
```

### Bad example
```cds
entity Orders {
  key ID       : UUID;
  createdOn    : Timestamp @cds.on.insert: $now;   // hand-rolled managed
  creator      : String    @cds.on.insert: $user;  // divergent naming
  currCode     : String(3);                        // stringly-typed currency
}
```

### Exception guidance
Domain-specific semantics that genuinely differ (e.g., a business-defined "approvedBy/At" that is not technical audit data) are domain fields, not `managed` replacements — model them separately *in addition to* `managed`. Reuse models imported from other products keep their own aspects.

### SAP reference
- https://cap.cloud.sap/docs/cds/common (benefits: proven best practices, interoperability; `managed`, `cuid`, reuse types)
- https://cap.cloud.sap/docs/guides/domain/ (`managed` keeps the core model clean vs verbose hand-modeling)

---

## CAP-CDS-005 — Prefer managed associations

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-005 |
| **Title** | Prefer managed associations |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC ("always prefer managed Associations" for to-one) |
| **Applicability** | All relationship modeling in CDS |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-006, CAP-SEC-011 (exposure of associations) |
| **Last verified** | 2026-08-11 |

### Rule statement
To-one relationships SHOULD be modeled as managed associations (`Association to <Target>`, foreign keys generated by the compiler), not as hand-modeled foreign-key elements plus unmanaged `on` conditions. To-many associations use the documented backlink pattern (`on <assoc>.<backlink> = $self`). Unmanaged associations are reserved for cases the compiler cannot manage (existing/legacy foreign keys, composite join conditions).

### Rationale
SAP: "For the sake of conciseness and comprehensibility of your models **always prefer managed Associations** for to-one associations." Managed associations generate correct foreign keys, enable path expressions and `$expand` uniformly, and keep models free of join boilerplate; hand-managed FK pairs drift (FK without association, association whose `on` doesn't match the FK) and lose tooling support. **Medium:** model quality and comprehensibility; wrong-but-consistent unmanaged modeling still functions.

### Evidence expected in code
`Association to <Target>` for to-one relationships; explicit FK elements only where an unmanaged association's documented reason exists; to-many with proper backlinks.

### Detection guidance
1. Scan `db/**/*.cds` for relationship patterns.
2. Flag hand-modeled foreign keys: elements like `author_ID : UUID` *without* a corresponding managed association, or unmanaged associations (`on` conditions) over self-maintained FK elements where a managed association would express the same → FAIL per relationship.
3. Verify to-many associations declare `on <assoc>.<backlink> = $self` backlinks (compiler enforces most of this — flag modeling that works around it).
4. Accept unmanaged associations with a visible reason (legacy table mapping, composite conditions) — note as compliant.
5. Report with file:line.

### Good example
```cds
entity Books : cuid {
  author : Association to Authors;                  // managed; FK generated
}
entity Authors : cuid {
  books  : Association to many Books on books.author = $self;
}
```

### Bad example
```cds
entity Books : cuid {
  authorId : UUID;                                   // hand-managed FK
  author   : Association to Authors on author.ID = authorId;  // unmanaged for no reason
}
```

### Exception guidance
Mapping to existing database tables/legacy schemas with fixed column layouts legitimately uses unmanaged associations — note the reason at the entity. Composite or conditional join semantics that managed associations cannot express are the other legitimate case.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ ("always prefer managed Associations")

---

## CAP-CDS-006 — Model containment with compositions — and avoid composition-of-one

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-006 |
| **Title** | Model containment with compositions — and avoid composition-of-one |
| **Category** | Domain modeling & CDS |
| **Severity** | High |
| **Authority** | SAP-REC (managed compositions for document structures; composition-of-one explicitly "discouraged") |
| **Applicability** | All parent-child / document-structure modeling |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-005, CAP-CDS-010, CAP-SEC-011 (composition authorization), CAP-SRV-007 (drafts operate on composition trees) |
| **Last verified** | 2026-08-11 |

### Rule statement
Contained-in relationships (document structures whose children live and die with the parent) MUST be modeled as compositions — preferably managed `Composition of many` (with inline targets or named aspects) — so deep insert/update, cascaded delete, and draft handling apply. Independent references MUST be associations, not compositions. `Composition of one` for entities SHOULD NOT be used (SAP: "discouraged… the information can be placed in the root entity"); inline the data into the root instead (see CAP-CDS-010).

### Rationale
SAP documents managed compositions of aspects as the way document structures are modeled ("contained-in relationships", Orders/OrderItems), and explicitly discourages composition-of-one for entities: no added value, limited Fiori draft support (custom creation handlers needed), and modifications over paths requiring manual foreign-key population. The association/composition choice is semantic, not stylistic — it decides cascaded delete, deep-operation, and draft behavior. **High justification:** mis-modeling containment produces wrong lifecycle behavior — orphaned children where deletes should cascade, or cascaded deletion of shared data referenced as a composition — i.e., data-integrity defects, plus the documented authorization enforcement gap on composition children (CAP-SEC-011).

### Implementation guidance
- Ask the ownership question per relationship: *can the child exist without this parent?* No → composition; yes → association.
- Prefer managed compositions of aspects (`Composition of many { … }` / named aspect) for classic header-item documents — scoped entities like `Orders.Items` are generated for you.
- Data that "would be a composition-of-one" belongs as elements in the root (flat models, CAP-CDS-010).

### Evidence expected in code
Compositions exactly on owned children; associations on references; no `Composition of one <Entity>`; deep structures using managed compositions of aspects where new.

### Detection guidance
1. List all `Composition` and `Association` usages in `db/**/*.cds`.
2. Flag `Composition of one` targeting entities → FAIL per occurrence (documented discouragement; recommend inlining).
3. Semantic check per composition: target shared/referenced elsewhere (other associations point to it, master data) → composition is wrong (cascade delete would destroy shared data) → FAIL.
4. Semantic check per association to weak child entities (children with no independent lifecycle, never referenced elsewhere, keyed by parent) → should be composition (no cascade/deep-ops today) → FAIL.
5. Verify document structures get deep behavior via composition (deep insert tests, drafts) rather than hand-coded child handling (cross-ref CAP-SRV-002).
6. Report per relationship with file:line.

### Good example
```cds
entity Orders : cuid, managed {
  Items : Composition of many { key pos : Integer; book : Association to Books; quantity : Integer; };
  buyer : Association to Customers;      // reference, NOT owned → association
}
```

### Bad example
```cds
entity Orders : cuid {
  header  : Composition of one OrderHeader;   // discouraged — inline into Orders
  items   : Association to many OrderItems on items.order = $self;  // owned children as association: no cascade, no deep ops, no draft coverage
}
entity OrderItems : cuid { order : Association to Orders; /* … */ }
```

### Exception guidance
Legacy schemas where children are physically shared tables may keep associations with explicitly hand-implemented lifecycle handling — documented at the model (ADR per CAP-ARCH-007). Separating rarely-used or very large sub-structures into associated entities for performance is legitimate when recorded (future CAP-PERF cross-ref).

### SAP reference
- https://cap.cloud.sap/docs/cds/cdl ("Using compositions of one for entities is discouraged"; managed compositions of aspects for contained-in document structures)
- https://cap.cloud.sap/docs/guides/domain/ (compositions for contained-in relationships)

---

## CAP-CDS-007 — Separate secondary concerns via aspects

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-007 |
| **Title** | Separate secondary concerns via aspects |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All projects with UI, authorization, or other cross-cutting annotations |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-001, CAP-SEC-001 (authorization file guidance), CAP-CDS-010 |
| **Last verified** | 2026-08-11 |

### Rule statement
Secondary concerns — UI/Fiori annotations, authorization restrictions, analytics metadata, localization overlays — SHOULD be factored into separate files using `annotate`/aspects rather than inlined into the core domain model, keeping `db/schema.cds` (and service definitions) concise. Consumers see the merged effective model, so separation costs nothing functionally.

### Rationale
SAP: aspects are "a very powerful means to organize your models… by factoring out secondary concerns into separate files", demonstrated by moving UI annotations to a separate `app/fiori-layout.cds`, "keep[ing] the core model concise and comprehensible". A domain model buried under hundreds of UI annotation lines is unreadable and couples the domain to one UI (a reusability defect the review should catch). **Medium:** readability/coupling; the compiler merges either way.

### Evidence expected in code
Core `db/` models mostly free of `@UI`/`@Common` UI annotations; dedicated annotation files (`app/*-layout.cds`, `srv/access-control.cds` per CAP-SEC-001 guidance) using `annotate`.

### Detection guidance
1. Measure annotation density in `db/**/*.cds`: count `@UI.*`, `@Common.*` (UI-semantic), `@restrict/@requires` inline in domain entities.
2. Substantial UI annotation blocks inline in `db/` models → FAIL (Medium) with file:line; scattered single annotations → observation.
3. Verify separate annotation files exist and are wired (`using` from an index or app folder).
4. Check the domain model remains UI-agnostic: elements existing *only* to serve one UI layout (labels-as-elements, UI ordering fields) → observation of UI coupling.
5. Report representative locations.

### Good example
```cds
// db/schema.cds — pure domain
entity Books : cuid, managed { title : localized String(111); author : Association to Authors; }
```
```cds
// app/admin/fiori-layout.cds — UI concern, separate file & lifecycle
using { AdminService } from '../../srv/admin-service';
annotate AdminService.Books with @UI.LineItem: [ { Value: title }, { Value: author.name } ];
```

### Bad example
```cds
// db/schema.cds — domain entity drowned in UI layout
entity Books : cuid, managed {
  @UI.Hidden: false @UI.LineItem: [{Position:10}] @Common.Label: 'Title'
  title  : String;
  /* …dozens more UI annotation lines… */
}
```

### Exception guidance
Semantic annotations that *are* domain facts (`@mandatory`, `@assert.*`, `@readonly`, `@title` used as domain vocabulary) belong with the model — do not exile them. Tiny projects with one small UI may inline pragmatically; note the trade-off.

### SAP reference
- https://cap.cloud.sap/docs/cds/aspects (factoring out secondary concerns; effective models)
- https://cap.cloud.sap/docs/guides/domain/ (keep the core domain model clean)

---

## CAP-CDS-008 — Use `localized` for translatable text

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-008 |
| **Title** | Use `localized` for translatable text |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC (mechanism); the modeling restrictions are documented constraints (SAP-REQ strength) |
| **Applicability** | Entities with end-user-visible translatable text; NOT APPLICABLE where no translation requirement exists |
| **Runtime** | Both |
| **CAP version** | All currently supported versions; restriction: `localized` in entity sub-elements is currently unsupported/ignored |
| **Status** | Active |
| **Related rules** | CAP-CDS-004 (`CodeList` texts come localized), CAP-SRV-001 |
| **Last verified** | 2026-08-11 |

### Rule statement
Translatable business text MUST be modeled with the `localized` modifier — generating the `.texts` entity, composition, and fallback views automatically — never with hand-modeled translation tables. Documented restrictions apply and MUST be respected: the entity's keys must not be associations, and `localized` in sub-elements/structured types is ignored.

### Rationale
SAP documents `localized` as the mechanism: the compiler "unfolds" it into a `Books.texts` entity, `texts`/`localized` associations, and views with locale fallback — served automatically with the user's locale. Hand-built translation tables re-implement all of that (fallback logic, locale propagation, write paths) and won't integrate with generated value helps and `CodeList` texts. **Medium:** framework duplication and i18n correctness; the restrictions, if violated, simply don't compile or are silently ignored — catching the silent case is the review's job.

### Evidence expected in code
`localized` on translatable elements; generated `.texts` handling used for translations; no custom `*Texts`/`*Translations` entities duplicating the mechanism.

### Detection guidance
1. Search `db/**/*.cds` for hand-modeled translation structures: entities named `*Texts`/`*Translation*` with locale+FK keys *not* generated by `localized` → FAIL per structure.
2. Identify user-visible text elements (names, titles, descriptions on master data exposed to UIs); translatable-by-requirement but plain `String` → FAIL/observation depending on requirement evidence.
3. Check restriction compliance: entities using `localized` whose keys are associations (compiler error — should not occur) and `localized` inside structured types (silently ignored — flag as ineffective annotation) → FAIL the latter.
4. NOT APPLICABLE if the project has no translation requirement — state the evidence (single-locale product decision).

### Good example
```cds
entity Books : cuid {
  title : localized String(111);   // .texts entity, fallback views generated
  descr : localized String(1111);
}
```

### Bad example
```cds
entity Books : cuid { title : String(111); }
entity BookTexts {                       // hand-built translation table:
  key book   : Association to Books;     // custom fallback, custom write path,
  key locale : String(14);               // invisible to generated value helps
  title      : String(111);
}
```

### Exception guidance
Externally mastered translations (texts replicated from another system) may keep their replication structures — document the boundary and don't double-model. Technical strings never shown to end users need no `localized`.

### SAP reference
- https://cap.cloud.sap/docs/guides/uis/localized-data (`localized` unfolding; keys must not be associations; sub-element restriction)

---

## CAP-CDS-009 — Model time-dependent data with the `temporal` aspect — writes in custom handlers

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-009 |
| **Title** | Model time-dependent data with the `temporal` aspect — writes in custom handlers |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REQ for the write path ("Writing temporal data must be done in custom handlers"); SAP-REC for the aspect itself |
| **Applicability** | Entities with date-effective/time-sliced data (validity periods); NOT APPLICABLE otherwise |
| **Runtime** | Both |
| **CAP version** | All currently supported versions; dev-parity note: time-travel/period queries are not supported on SQLite |
| **Status** | Active |
| **Related rules** | CAP-CDS-004, CAP-SRV-002 (this is a documented case where custom handlers are *required*) |
| **Last verified** | 2026-08-11 |

### Rule statement
Date-effective data (validity-period records) SHOULD be modeled with the `temporal` aspect (`validFrom`/`validTo` with `@cds.valid.from/to`), giving as-of-now and time-travel reads generically. Writes of time slices MUST be implemented in custom handlers — SAP documents no generic write support — maintaining non-overlapping, closed-open intervals. Hand-modeled validity columns without the annotations forgo the generic read support and MUST NOT be presented as temporal data.

### Rationale
SAP documents the aspect and states verbatim: "Writing temporal data must be done in custom handlers." The aspect buys generic time-aware reading; the write-side rule prevents the opposite failure — assuming the framework maintains slice integrity (it doesn't), yielding overlapping or gapping validity intervals, i.e., wrong query results for past/future dates. **Medium:** scoped to temporal entities; defects are data-quality issues in those entities.

### Evidence expected in code
`temporal` aspect (or `@cds.valid.from/to`) on time-sliced entities; custom write handlers maintaining interval discipline; tests covering slice writes.

### Detection guidance
1. Identify time-sliced entities: `temporal` includes, `@cds.valid.from/to`, or semantic pairs (`validFrom/validTo`, `effectiveDate/expiryDate`).
2. Semantic validity pairs *without* the annotations, where as-of/time-travel reads are required → FAIL (generic read support forgone).
3. For annotated temporal entities: locate custom write handlers for slice maintenance; writable via generic CRUD only (no handler) → FAIL (framework won't maintain intervals).
4. Check handler logic targets non-overlapping closed-open intervals (inspect and cross-check tests); no tests → missing-evidence element.
5. Note SQLite dev-parity limitation as an observation where relevant.

### Good example
```cds
using { temporal } from '@sap/cds/common';
entity WorkAssignments : temporal, cuid { role : String; dept : Association to Departments; }
```
```js
// slice write maintained by custom handler (close current, insert next)
srv.on('UPDATE', 'WorkAssignments', closeAndInsertSlice);
```

### Bad example
```cds
entity WorkAssignments : cuid {
  validFrom : Date; validTo : Date;   // no @cds.valid annotations: no generic
}                                      // time-aware reads — and generic UPDATE
                                       // happily creates overlapping slices
```

### Exception guidance
Simple effective-dating without historical query needs (only "current row" semantics) may use plain date fields deliberately — record that time-travel is out of scope. Full bi-temporal requirements beyond CAP's mechanism are custom by nature; document the design (CAP-ARCH-007).

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/temporal-data (`temporal` aspect; "Writing temporal data must be done in custom handlers")

---

## CAP-CDS-010 — Prefer flat models; custom types only with reuse value

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-010 |
| **Title** | Prefer flat models; custom types only with reuse value |
| **Category** | Domain modeling & CDS |
| **Severity** | Low |
| **Authority** | SAP-REC |
| **Applicability** | All domain models |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-006 (inline instead of composition-of-one), CAP-CDS-008 (`localized` ignored in sub-elements) |
| **Last verified** | 2026-08-11 |

### Rule statement
Models SHOULD stay flat: avoid nested structured types for entity elements (SAP marks the nested variant "Bad"), and define custom types only where they have a decent reuse ratio — not one type per element. Structure that would be a composition-of-one or a deep struct usually belongs as plain elements in the entity.

### Rationale
SAP: "Prefer Flat Models" (nested structured example explicitly labeled "Bad:") and "Avoid overly excessive use of custom-defined types. They're valuable when you have a decent reuse ratio." Flat models integrate better across protocols/UIs and avoid the documented sub-element limitations (e.g., `localized` ignored inside structs). **Low:** readability/integration friction — a style-level modeling preference with documented grounding.

### Evidence expected in code
Entities with flat element lists; custom types used repeatedly (reuse ratio > 1); no single-use struct types wrapping two fields.

### Detection guidance
1. List `type … {}` structured type definitions and their usage counts across the model.
2. Single-use structured types nested into entities → observation/FAIL (Low) per SAP's "Bad" example.
3. Deeply nested structures in entity elements → flag with the documented limitations (localized, UI integration).
4. Custom scalar types: flag only excessive one-off definitions; reused semantic types (e.g., `type Quantity : Decimal(9,3)`) are compliant.
5. Report representative examples.

### Good example
```cds
entity Addresses : cuid {          // flat elements — no nested structs
  street : String; town : String; country : Country;
}
```

### Bad example
```cds
type Amount { value : Decimal; currency : String(3); }   // single-use struct
entity Orders : cuid {
  total : Amount;                  // nested struct element — SAP's "Bad" shape
}
```

### Exception guidance
Genuinely reused structures (an `Amount` used across a dozen entities, standard address structs from reuse packages) are the documented good case for custom types. External models keep their shapes.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ ("Prefer Flat Models"; reuse-ratio guidance on custom types)

---

## CAP-CDS-011 — Choose stable namespaces

| Field | Value |
|---|---|
| **Rule ID** | CAP-CDS-011 |
| **Title** | Choose stable namespaces |
| **Category** | Domain modeling & CDS |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All models declaring namespaces (recommended for reusable models) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-001, CAP-ARCH-001 (CSV file names embed the namespace) |
| **Last verified** | 2026-08-11 |

### Rule statement
Namespaces MUST NOT contain short-lived ingredients — current organization names, project code names, team names — per SAP: "Don't use short-lived ingredients in namespaces… such as your current organization's name, or project code names." Reverse-domain-style, product-oriented namespaces are the documented good pattern. Namespaces are effectively immutable once data exists: they prefix persistence names and seed-data file names.

### Rationale
SAP's guidance is explicit, and the cost asymmetry justifies Medium rather than Low: a namespace bleeds into database artifact names, CSV data file names (CAP-ARCH-001), external API metadata, and extension models — renaming later is a data-migration event, not a refactoring. **Medium:** a wrong choice is cheap today and expensive forever.

### Evidence expected in code
`namespace` declarations using stable, product/domain-oriented reverse-domain names; no org/project codenames.

### Detection guidance
1. Extract `namespace` declarations from all `*.cds`.
2. Flag namespaces embedding organization names, project code names, temporary campaign names, or team identifiers → FAIL per namespace (Medium), with the migration-cost note.
3. Check CSV seed files and any deployed artifacts consistently reflect the namespace (cross-ref CAP-ARCH-001 detection).
4. For existing shipped products with legacy namespaces: FAIL only new models introducing fresh unstable names; existing ones → observation (migration is its own decision, CAP-ARCH-007).

### Good example
```cds
namespace acme.retail.ordering;     // product/domain oriented, stable
```

### Bad example
```cds
namespace acme.project_phoenix.q3poc;   // project codename + phase — will be
                                          // wrong before the tables have data
```

### Exception guidance
Company brand names that are themselves the stable product identity (the vendor prefix in `sap.common` style) are fine — the rule targets *short-lived* ingredients, not every organizational token. Legacy namespaces stay until a deliberate, recorded migration.

### SAP reference
- https://cap.cloud.sap/docs/guides/domain/ (avoid short-lived namespace ingredients; reverse-domain approach)

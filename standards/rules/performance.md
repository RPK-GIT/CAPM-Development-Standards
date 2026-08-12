# CAP-PERF — Performance & scalability

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-04 (performance budgets/SLOs/sizing — SAP prescribes none; none are invented here).

**Rules:** 7 active (0 Critical, 2 High, 5 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries: pagination *limit decisions* are [CAP-SEC-014](security.md) (absorbed candidate CAP-PERF #1); CQN UNION/JOIN support constraints are [CAP-DB-005](data-persistence.md); Decimal arithmetic is [CAP-DB-009](data-persistence.md) (relocated candidate #6); key-shape JOIN rationale is absorbed by [CAP-CDS-002](cds-modeling.md) (candidate #8); remote-call fan-out is [CAP-INT-005](integration.md); media streaming is [CAP-SRV-009](services-apis.md). No performance thresholds or SLOs are defined here — those are ORG policy (G-04, set per project at milestone M0).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-PERF-001 | Enable reliable pagination for concurrently modified OData V4 collections | Medium | SAP-REC | Both |
| CAP-PERF-002 | Prefer associations with on-demand expansion over static JOIN views | Medium | SAP-REC | Both |
| CAP-PERF-003 | Keep live-calculated fields out of filter, sort, and join paths | High | SAP-REC | Both |
| CAP-PERF-004 | Avoid UNIONs in models — remodel polymorphism | Medium | SAP-REC | Both |
| CAP-PERF-005 | Stream large result sets instead of materializing them | Medium | SAP-REC | Node.js |
| CAP-PERF-006 | Keep composition trees draft-manageable | Medium | SAP-REC | Both |
| CAP-PERF-007 | No per-row queries in loops — use set-based access | High | GEN | Both |

---

## CAP-PERF-001 — Enable reliable pagination for concurrently modified OData V4 collections

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-001 |
| **Title** | Enable reliable pagination for concurrently modified OData V4 collections |
| **Category** | Performance & scalability |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented opt-in feature with documented limitations) |
| **Applicability** | OData V4 collections paged by clients while concurrently modified; NOT APPLICABLE for stable/read-mostly sets or non-V4 protocols |
| **Runtime** | Both (Node: `cds.query.limit.reliablePaging: true`; Java: `cds.query.limit.reliablePaging.enabled: true`) |
| **CAP version** | ⏱ Documented as **OData V4 only** ("This feature is available only for OData V4 endpoints") |
| **Status** | Active |
| **Related rules** | CAP-SEC-014 (limit decisions), CAP-SRV-005 (V4), CAP-SEC-016 (the sensitive-data caveat below) |
| **Last verified** | 2026-08-12 |

### Rule statement
Collections that clients page through while the data changes SHOULD enable reliable pagination — SAP documents that the default numeric skip token "can result in duplicate or missing rows if the entity set is modified between the calls". The documented limitations MUST be respected when enabling it: no functions/arithmetic expressions in `$orderby`; `$orderby` elements must be of simple type and included in `$select` (if set); and — SAP's explicit warning — do not use it when sorting by sensitive elements, since "the skip token could reveal the values of these elements".

### Rationale
Duplicate or missing rows during paging are a *correctness* defect users experience as flaky lists and exports that silently skip records — the documented failure mode of value-based skip tokens under concurrent writes. The feature is opt-in per runtime configuration, so an undecided project silently has the problem. The sensitive-data caveat ties into CAP-SEC-016's disclosure discipline. **Medium:** scoped correctness/quality; the data itself is not corrupted.

### Evidence expected in code
`reliablePaging` configuration for the affected services/entities; `$orderby`-relevant UIs sorting by simple, selected, non-sensitive elements; or a recorded decision that concurrent-modification paging is not a scenario.

### Detection guidance
1. Identify collections paged by clients under concurrent modification (transactional lists, work queues — from requirements/UI usage).
2. None → NOT APPLICABLE (state the assessment).
3. For identified collections: check `reliablePaging` config (package.json / `application.yaml`) → absent with no recorded accept-the-risk decision → FAIL (Medium).
4. Where enabled: verify sort fields comply with the documented limitations (simple types, selected, no expressions) and are not sensitive per the project's data classification (cross-check CAP-PRIV-001 annotations) → sensitive sort field → FAIL (report also under CAP-SEC-016).
5. Report configuration locations.

### Good example
```jsonc
// package.json — work-queue lists paged under concurrent writes
{ "cds": { "query": { "limit": { "reliablePaging": true } } } }
```

### Bad example
```text
A task-inbox UI pages through 50k rows sorted by modifiedAt while
workers update tasks continuously — default skip tokens: users see
duplicated tasks and exports silently drop rows. No config, no decision.
```

### Exception guidance
Read-mostly collections and single-page result sets don't need it — the NOT APPLICABLE path with stated reasoning. Last-write-wins-style acceptance of paging anomalies is a recordable decision for low-stakes lists.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/served-ootb (reliable pagination: duplicate/missing-row problem; V4-only; config keys; limitations; sensitive-data skip-token warning)

---

## CAP-PERF-002 — Prefer associations with on-demand expansion over static JOIN views

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-002 |
| **Title** | Prefer associations with on-demand expansion over static JOIN views |
| **Category** | Performance & scalability |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | CDS view/projection modeling combining entities |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-005 (managed associations enable this), CAP-DB-005 (CQN constraint), CAP-SRV-001 |
| **Last verified** | 2026-08-12 |

### Rule statement
Cross-entity read models SHOULD be built from associations consumed on demand rather than static JOIN-based views: SAP documents that with associations "A JOIN will not be executed until you explicitly use the OData feature `$expand`" — consumers who don't expand pay nothing. Where sorted/filtered detail data is the goal, follow the documented pattern: query the detail entity first and join back to the header via the association, rather than sorting/filtering a header-based JOIN view.

### Rationale
A static JOIN view executes its join for every consumer on every access, whether or not the joined fields are needed; association-based models make the join opt-in per request. The detail-first pattern keeps sorting/filtering on the indexed detail table instead of a join product. **Medium:** systematic-but-bounded query cost; the model works either way.

### Evidence expected in code
Cross-entity service views modeled as projections with associations (expandable), not `select from A join B`; detail-oriented lists modeled on the detail entity with a back-association.

### Detection guidance
1. Search `db/**/*.cds` and `srv/**/*.cds` for explicit `join` in view definitions (`as select from … join …`).
2. For each JOIN view: is the join needed by *all* consumers, or would association+expand serve (some consumers don't need the joined fields)? → latter → FAIL (Medium) with the association alternative.
3. Identify detail-sorted/filtered list requirements implemented as header-JOIN views → FAIL with the detail-first pattern.
4. Joins genuinely needed by every consumer (flattening for an external contract, PDM-style flat views per CAP-PRIV-003) → PASS with note.
5. Report per view with file:line.

### Good example
```cds
// consumers expand on demand; no join cost for those who don't
entity Orders as projection on my.Orders;           // Items reachable via assoc
// detail-first list: sort/filter on items, join back via association
entity OrderItemsList as projection on my.OrderItems { *, order.buyer as buyer };
```

### Bad example
```cds
// every consumer of this view pays the 3-way join — including the
// mobile list that shows only order IDs
entity OrdersFull as select from my.Orders as o
  join my.OrderItems as i on i.order.ID = o.ID
  join my.Books as b on b.ID = i.book.ID { o.ID, o.date, i.quantity, b.title };
```

### Exception guidance
Flattened views required by consumers that cannot expand (external flat contracts, PDM per CAP-PRIV-003, analytics extracts) are the documented use case for real joins — scope them to those consumers instead of making them the general read model.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/performance ("A JOIN will not be executed until you explicitly use … `$expand`"; detail-first sort/join-back pattern)

---

## CAP-PERF-003 — Keep live-calculated fields out of filter, sort, and join paths

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-003 |
| **Title** | Keep live-calculated fields out of filter, sort, and join paths |
| **Category** | Performance & scalability |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | Models with calculated/derived elements on entities that grow beyond trivial size |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-010 (model shape), CAP-SRV-001; escalates through CAP-SEC-014 when client-triggerable at scale |
| **Last verified** | 2026-08-12 |

### Rule statement
Live-calculated fields (calculated-on-read elements, CASE-derived values, handler-computed values) MUST NOT participate in WHERE clauses, JOIN/association filter conditions, or sorting on non-trivial entities — SAP: "Database operations on calculated fields cannot leverage any DB indexes … calculated fields cause full table scans", with the explicit instructions to "Disable sorting, filtering … on **live** calculated fields" (via `@Capabilities` restrictions, e.g. `SortRestrictions.NonSortableProperties`) and "Don't use **live** calculated fields in `where` clauses, as in JOIN conditions or association filtering." Where filtering/sorting on the derived value is required, follow SAP's documented preference order: calculate in the UI where possible, otherwise **pre-calculate on write** (stored), with on-read calculation and read-event handlers as documented fallbacks only.

### Rationale
A calculated field in a filter or sort forces the database to evaluate the expression for every row — a full table scan per request, on exactly the entities that grow. This is the classic CAP performance incident: fine in dev (100 rows), a multi-second scan in production (10 million rows), triggered by any user sorting a column. **High justification:** the documented pattern is "capable of exhausting database resources" on the normal request path at production scale — the named High case — while remaining below Critical (degradation, not corruption or exposure).

### Implementation guidance
- Derived values used in lists' filters/sorts → compute on write into a stored element (a `before CREATE/UPDATE` handler or stored calculated element), index-friendly by construction.
- Display-only derivations may stay calculated-on-read — then annotate them non-sortable/non-filterable so UIs can't turn them into scans.

### Evidence expected in code
Stored (write-time) elements backing filterable/sortable derived values; `@Capabilities` restrictions on any remaining live-calculated elements; no live-calculated elements referenced in `where`/`on` conditions of views or handler queries.

### Detection guidance
1. Inventory calculated elements: CDS calculated-on-read elements, `virtual` elements filled by `after READ` handlers, CASE expressions in projections.
2. For each, check usage in filter/sort/join paths: view `where`/`on` clauses referencing them; UI annotations enabling sort/filter on them; handler queries filtering on them → each → FAIL (High) with file:line.
3. Live-calculated elements exposed on list-relevant entities without `@Capabilities` sort/filter restrictions → FAIL (the UI can trigger the scan).
4. Verify write-time pre-calculation for values that *are* filtered/sorted (stored element + write-path handler) → compliant pattern; note it.
5. Small, bounded entities (code lists) → observation only, not FAIL — state the size reasoning.

### Good example
```cds
entity Orders : cuid, managed {
  total  : Decimal;                 // pre-calculated on write → indexable
  @Capabilities.SortRestrictions.NonSortableProperties: [ageInDays]
  virtual ageInDays : Integer;      // display-only, sort disabled
}
```

### Bad example
```cds
// CASE-derived category filtered in a view over a 20M-row table:
// full scan on every request hitting this view
entity HotOrders as select from my.Orders {
  *, case when total > 10000 then 'HOT' else 'NORMAL' end as category : String
} where (case when total > 10000 then 'HOT' else 'NORMAL' end) = 'HOT';
```

### Exception guidance
Trivially small entities and admin-only rarely-used queries may tolerate live calculation — record the bounded-size assumption. Database-side computed/indexed columns (documented native features via CAP-DB-004's exception path) are an alternative compliant implementation.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/performance (calculated fields "cause full table scans"; preference order UI → on-write → on-read → handlers; `@Capabilities` restrictions; no live-calculated fields in `where`/JOIN)

---

## CAP-PERF-004 — Avoid UNIONs in models — remodel polymorphism

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-004 |
| **Title** | Avoid UNIONs in models — remodel polymorphism |
| **Category** | Performance & scalability |
| **Severity** | Medium |
| **Authority** | SAP-REC ("Using the UNION statement to merge data from different sources should be avoided") |
| **Applicability** | CDS model/view design merging heterogeneous sources |
| **Runtime** | Both (the CQN-level UNION constraint for Node.js custom queries is CAP-DB-005 — this rule is the *modeling* counterpart) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DB-005, CAP-CDS-006 (compositions), CAP-CDS-010 |
| **Last verified** | 2026-08-12 |

### Rule statement
Model-level UNIONs (views built with `union` to merge similar-but-different sources) SHOULD be avoided — SAP: "UNIONs in views come with a performance penalty and complex modelling", especially with sorting/filtering applied after the union. Polymorphic data SHOULD instead be remodeled per SAP's documented options: one entity with a type discriminator (enum) plus per-subtype compositions, or the denormalized-aspect variant yielding "a single, sparsely populated DB table, which is not an issue using modern databases with variable page sizes".

### Rationale
A UNION view re-computes the merge on every access and defeats index-based sorting/filtering across the branches; the remodeled single-entity forms restore ordinary indexed access and simpler authorization/annotation handling. **Medium:** chronic query cost and model complexity — a design-debt class, not an outage class.

### Evidence expected in code
No `union` in CDS views for polymorphic business data; discriminator-based single entities (or sparse aspect-based tables) where subtypes exist.

### Detection guidance
1. Search `db/**/*.cds` and `srv/**/*.cds` for `union` in view definitions.
2. For each: does it merge polymorphic variants of one business concept? → FAIL (Medium) with the documented remodeling options.
3. Check for sorting/filtering applied on top of union views (view `order by`/`where`, or exposure to sortable lists) → strengthen the finding (SAP's specific warning).
4. Genuine cross-source technical merges (e.g., archival + live for a migration window) → PASS with a documented time-bound reason.
5. Report per view.

### Good example
```cds
entity Incidents : cuid {              // one entity, discriminator + subtype data
  type    : String enum { complaint; request; incident };
  details : Composition of one IncidentDetails;
}
```

### Bad example
```cds
entity AllIncidents as select from Complaints { ID, title, 'C' as type : String }
  union select from Requests { ID, title, 'R' as type };   // merged + later sorted:
                                                            // performance penalty on
                                                            // every list request
```

### Exception guidance
Time-bound technical unions (migration overlays) with a recorded end date. Analytical models materialized outside the transactional path are a different discipline (record the design per CAP-ARCH-007).

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/performance (UNION avoidance; discriminator + compositions; sparse-table aspect variant)

---

## CAP-PERF-005 — Stream large result sets instead of materializing them

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-005 |
| **Title** | Stream large result sets instead of materializing them |
| **Category** | Performance & scalability |
| **Severity** | Medium |
| **Authority** | SAP-REC ("Use this API instead of `cds.run` if you expect large result sets") |
| **Applicability** | Node.js code processing potentially large result sets (exports, batch jobs, mass processing) |
| **Runtime** | Node.js (Java's `Result`/streaming semantics differ; no equivalent documented single-API rule — Java mass processing is governed by the general set-based discipline of CAP-PERF-007) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-PERF-007, CAP-SRV-009 (media streaming), CAP-MT-006 (such jobs also need tenant context) |
| **Last verified** | 2026-08-12 |

### Rule statement
Node.js code expecting large result sets MUST process them with `srv.foreach(query, callback)` — SAP: "Use this API instead of `cds.run` if you expect large result sets. Then they're processed in a streaming-like fashion instead of materializing the full result set in memory before processing." Loading unbounded sets fully into memory (`await SELECT.from(...)` without limits on mass data) for row-wise processing is prohibited in batch/export paths.

### Rationale
Materializing a million-row result set into a JS array is an out-of-memory event waiting for data growth — and in multitenant apps, one tenant's export can take down the instance serving all tenants. The streaming API bounds memory to per-row processing. **Medium:** resource-behavior defect in specific (batch/export) paths; request-path unboundedness is largely fenced by the framework's pagination defaults (CAP-SEC-014).

### Evidence expected in code
`srv.foreach`/`cds.foreach` in mass-processing paths; bounded (`.limit`) reads elsewhere; no full-table `SELECT` into arrays in jobs/exports.

### Detection guidance
1. Identify mass-data paths: exports, scheduled jobs, migrations, recalculation tasks in `srv/**`.
2. For each, inspect the read pattern: unbounded `await SELECT.from(<large entity>)` followed by iteration → FAIL (Medium) with the `foreach` replacement; chunked/limited reads or `foreach` → PASS.
3. Estimate "large" from entity nature (transactional data, logs) rather than current row counts — growth is the point; note the reasoning.
4. Cross-check such jobs for tenant context (CAP-MT-006) while there.
5. NOT APPLICABLE if no mass-processing paths exist.

### Good example
```js
// streaming-like: memory bounded to one row at a time
await srv.foreach(SELECT.from(Orders).where({ status: 'open' }),
  order => exporter.write(order));
```

### Bad example
```js
const all = await SELECT.from(Orders);        // 4M rows into one array
for (const o of all) exporter.write(o);       // OOM at production scale
```

### Exception guidance
Genuinely bounded sets (result of a selective filter with a known cap) may materialize — state the bound. Aggregations belong in the database (push-down), not in streamed application code at all.

### SAP reference
- https://cap.cloud.sap/docs/node.js/core-services (`srv.foreach`: "instead of materializing the full result set in memory")

---

## CAP-PERF-006 — Keep composition trees draft-manageable

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-006 |
| **Title** | Keep composition trees draft-manageable |
| **Category** | Performance & scalability |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Draft-enabled entities with large composition trees; NOT APPLICABLE without drafts or without large child sets |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-006 (composition semantics — its exception path lands here), CAP-SRV-007 (drafts) |
| **Last verified** | 2026-08-12 |

### Rule statement
Draft-enabled entities MUST NOT carry unbounded, very large composition trees: SAP documents that "Large documents, containing compositions with thousands of children, are copied entirely into draft state, even when only one little part is changed." Where child sets grow into the thousands, the documented remedy applies — decouple the large child set via an association (own lifecycle/editing) instead of keeping it inside the drafted composition.

### Rationale
Every draft edit of such a document copies the entire tree — thousands of rows written per "open for editing", multiplied by concurrent editors, on the framework's hot path. The containment *semantics* decision is CAP-CDS-006's; this rule adds the documented scale boundary for the draft mechanism. **Medium:** heavy but bounded operational cost, surfacing as slow editing and draft-table bloat.

### Evidence expected in code
Draft-enabled roots whose compositions are bounded (structural or business bounds); large collections modeled as associations with their own editing flow; a recorded size assumption for borderline cases.

### Detection guidance
1. List draft-enabled entities (per CAP-SRV-007 detection) and their composition trees.
2. For each composition target, assess growth: order-of-thousands children per document (measurement, requirements, or entity nature — e.g., log-like children) → composition inside the draft → FAIL (Medium) with the association-decoupling remedy.
3. Bounded children (line items with business caps) → PASS with the bound noted.
4. Check decoupled large children have their own consistent editing path (not silently losing draft protection where users need it) — observation.
5. NOT APPLICABLE without drafts.

### Good example
```cds
@odata.draft.enabled
entity Projects : cuid { 
  milestones : Composition of many Milestones;   // bounded (tens)
  logEntries : Association to many LogEntries    // thousands — own lifecycle,
               on logEntries.project = $self;    // NOT copied into drafts
}
```

### Bad example
```cds
@odata.draft.enabled
entity Projects : cuid {
  logEntries : Composition of many LogEntries;   // 20k children copied into
}                                                 // draft state on every edit
```

### Exception guidance
Documents whose business semantics genuinely require whole-tree draft editing at scale accept the cost by recorded decision (with the operational implications named). Bounded trees are simply compliant.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/performance ("copied entirely into draft state, even when only one little part is changed"; association decoupling)

---

## CAP-PERF-007 — No per-row queries in loops — use set-based access

| Field | Value |
|---|---|
| **Rule ID** | CAP-PERF-007 |
| **Title** | No per-row queries in loops — use set-based access |
| **Category** | Performance & scalability |
| **Severity** | High |
| **Authority** | GEN (general engineering practice; SAP documents the set-based primitives — expands, path expressions, deep reads, batch operations — but no explicit N+1 prohibition; authored from gap G-05) |
| **Applicability** | All custom handler/job code issuing queries |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-INT-005 (owns the *remote* fan-out variant), CAP-PERF-005, CAP-DB-004/-005 (the set-based query APIs), CAP-SRV-002 |
| **Last verified** | 2026-08-12 (GEN authority; SAP primitives verified) |

### Rule statement
Custom code MUST NOT issue one database query per element of a collection (the N+1 pattern): no `SELECT`/`UPDATE`/`INSERT` inside loops over result rows or request payloads where a set-based formulation exists. CAP's primitives make the set-based form available in every common case: `$expand`/nested `columns` for related data, `where: { ID: { in: [...] } }` for keyed lookups, batch `INSERT ... entries([...])`, expression-based mass `UPDATE`, and path expressions instead of join-loops. The remote-call variant of this pattern is governed by CAP-INT-005.

### Rationale
N+1 turns one request into row-count database round trips — 5,000 list rows become 5,000 queries, saturating the connection pool and the database for all users (and all tenants on shared resources). It is invisible at dev-scale and dominant at production scale. **High justification:** this is the named "severe query explosion … capable of exhausting application/database resources" case; on the normal request path it degrades the whole instance, not just the offending request. Authority is honestly **GEN**: SAP documents the primitives, not the prohibition — recorded as the resolution of gap G-05.

### Implementation guidance
- Need related data per row? Expand it in the original query (`columns(o => o.items(...))`) — one round trip.
- Need lookups for a set of keys? Collect keys, one `in`-query, then join in memory.
- Need to update many rows uniformly? One expression `UPDATE ... where`, not a loop (also atomic — CAP-DB-009's pattern).

### Evidence expected in code
Handlers/jobs with query counts independent of collection sizes; expands and `in`-queries where related data is assembled; batch inserts/updates.

### Detection guidance
1. Search `srv/**` for query calls inside loops: `for`/`forEach`/`map` bodies (Node) or loop bodies/streams (Java) containing `SELECT`/`UPDATE`/`INSERT`/`db.run`/`service.run` → each candidate site.
2. Classify: loop over a collection whose size scales with data (query results, request payload arrays) → FAIL (High) with the set-based rewrite; loops over small fixed sets (config constants) → PASS with note.
3. Check `after READ` handlers enriching rows one query per row (the classic case) → FAIL; the fix is expand or one `in`-query.
4. Remote calls in loops → report under CAP-INT-005, note here.
5. Report per site with file:line and the concrete rewrite.

### Good example
```js
srv.after('READ', 'Orders', async orders => {
  const ids = [...new Set(orders.map(o => o.book_ID))];
  const books = await SELECT.from(Books).where({ ID: { in: ids } });  // ONE query
  const byId = new Map(books.map(b => [b.ID, b]));
  orders.forEach(o => o.bookTitle = byId.get(o.book_ID)?.title);
});
```

### Bad example
```js
srv.after('READ', 'Orders', async orders => {
  for (const o of orders)                                   // N+1: one query per row —
    o.bookTitle = (await SELECT.one.from(Books, o.book_ID)).title;  // 5,000 rows = 5,000 queries
});
```

### Exception guidance
Loops over genuinely constant-size sets, and per-item processing where each item's query is unavoidable *and* the collection is contractually bounded (documented bound), are acceptable. Row-by-row semantics required by an external system belong in queued, throttled processing (CAP-EVT), not the request path.

### SAP reference
None normative (authority: GEN — gap G-05 resolution). Set-based primitives: https://cap.cloud.sap/docs/node.js/cds-ql, https://cap.cloud.sap/docs/java/working-with-cql/query-api, https://cap.cloud.sap/docs/guides/databases/performance (expand/path-expression patterns).

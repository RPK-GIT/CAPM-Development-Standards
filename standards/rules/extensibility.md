# CAP-EXT — Extensibility

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-35 (extension-allowlist content).

**Rules:** 4 active (0 Critical, 1 High, 3 Medium). All SAP references verified against official CAP documentation on **2026-08-12**. Note: the intrinsic-extensibility statements formerly on `/docs/about/best-practices` now live on the **get-started/concepts** page (the old URL is 404) — citations use the live page.

Scope boundaries: SaaS extension *authorization* (`cds.ExtensionDeveloper` role isolation) is [CAP-SEC-009](security.md); tenant lifecycle around extensions is [CAP-MT](multitenancy.md); model mechanics of aspects are [CAP-CDS-007](cds-modeling.md). Rules 002–004 apply to **SaaS providers** offering customer extensibility/feature toggles — single-tenant apps without extension offerings are NOT APPLICABLE there.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-EXT-001 | Extend — never modify base or reuse artifacts | Medium | SAP-REC | Both |
| CAP-EXT-002 | Gate customer extensions with a configured extension allowlist | High | SAP-REQ | Both |
| CAP-EXT-003 | Follow the supported extension-development workflow | Medium | SAP-REC | Both |
| CAP-EXT-004 | Keep toggled features isolated and schema-uniform | Medium | SAP-REQ | Both |

---

## CAP-EXT-001 — Extend — never modify base or reuse artifacts

| Field | Value |
|---|---|
| **Rule ID** | CAP-EXT-001 |
| **Title** | Extend — never modify base or reuse artifacts |
| **Category** | Extensibility |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented intrinsic-extensibility model: "you can extend any reuse definition that you might consume from reuse packages") |
| **Applicability** | All projects consuming reuse packages, base models, or `@sap/cds/common`; and all customizations of standard artifacts |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CDS-007 (aspects mechanics), CAP-ARCH-002 (no forked framework layers), CAP-VER candidates (upgrade path — future category) |
| **Last verified** | 2026-08-12 |

### Rule statement
Adaptations of models and artifacts the project does not own — reuse packages, `@sap/cds/common`, imported base models, generated artifacts — MUST be made through CDS extension mechanisms (`extend`, `annotate`, aspects), never by editing the artifacts in place: no patched files under `node_modules/`, no forked copies of reuse models maintained by hand, no edits to generated files that regeneration overwrites. SAP's documented model is intrinsic extensibility: "you can extend any reuse definition that you might consume from reuse packages, including the reuse models shipped with CAP itself."

### Rationale
In-place modifications are upgrade bombs: the next `npm install`/dependency update silently reverts patched `node_modules` content, while forked copies drift from their source and re-import every upstream fix by hand. The `extend`/`annotate` mechanism applies the same change *on top of* the consumed artifact, surviving upgrades by construction — it is the extension counterpart of CAP-ARCH-002's no-forked-frameworks discipline. **Medium:** upgrade-safety and maintainability; the failure surfaces at the next update, not immediately.

### Implementation guidance
- Additions/changes to consumed entities: `extend Entity with { x_myField : String; }` / `annotate Entity with @(...)` in your own files.
- Behavior changes to consumed services: event handlers registered on them (CAP-LOGIC-001) — CAP's documented way to extend framework services too.
- If an upstream artifact is genuinely wrong, raise it upstream; a temporary patch needs an ADR with removal condition (CAP-ARCH-007).

### Evidence expected in code
`extend`/`annotate` statements in project-owned files targeting consumed definitions; pristine dependencies (no patch-package artifacts against model/runtime packages without ADR); no maintained forks of reuse models.

### Detection guidance
1. Check for dependency patching: `patches/` folders (patch-package), `postinstall` scripts editing `node_modules`, vendored copies of `@sap/*`/reuse packages in the repo → each → FAIL with location (ADR-covered temporary patches → PASS with expiry check).
2. Diff-check suspicious model files: project files containing full copies of reuse-package entities (same names/namespaces as a consumed package) rather than `extend`/`using` references → FAIL (fork).
3. Verify adaptations of consumed models use `extend`/`annotate` in project files → compliant pattern; note it.
4. Check generated artifacts (`gen/`, generated static model classes) are not hand-edited (committed edits to generated paths that the build would overwrite) → FAIL.
5. Report per artifact.

### Good example
```cds
// srv/extensions.cds — adapting a consumed reuse model, upgrade-safe
using { sap.common.Currencies } from '@sap/cds/common';
extend Currencies with { x_cryptoFlag : Boolean; }
annotate Currencies with @cds.autoexpose;
```

### Bad example
```text
node_modules/@sap/cds/common.cds edited in place to add a field
(reverted by the next npm install), plus a hand-maintained copy of a
partner reuse model pasted into db/partner-model.cds and "improved".
```

### Exception guidance
Documented temporary patches (upstream bug, fix pending) via a visible patch mechanism with an ADR naming the removal condition. Copying a model as a deliberate hard fork (severing the dependency permanently) is an architecture decision — ADR per CAP-ARCH-007, after which the copy is owned code and out of this rule's scope.

### SAP reference
- https://cap.cloud.sap/docs/get-started/concepts (intrinsic extensibility; extending reuse definitions incl. CAP's own)
- https://cap.cloud.sap/docs/guides/extensibility/ (CDS aspects as the extension mechanism)

---

## CAP-EXT-002 — Gate customer extensions with a configured extension allowlist

| Field | Value |
|---|---|
| **Rule ID** | CAP-EXT-002 |
| **Title** | Gate customer extensions with a configured extension allowlist |
| **Category** | Extensibility |
| **Severity** | High |
| **Authority** | SAP-REQ (extensions are forbidden by default without configuration — documented; the `x_` prefix requirement is documented: "All new elements have to start with `x_`") |
| **Applicability** | SaaS providers offering customer (tenant) extensibility; NOT APPLICABLE otherwise |
| **Runtime** | Both (extension tooling is cds CLI; validation client-side at `cds build` as part of `cds push`) |
| **CAP version** | Streamlined MTX (`cds.xt.ExtensibilityService`) |
| **Status** | Active |
| **Related rules** | CAP-MT-001/-002 (MTX foundation), CAP-SEC-009 (`cds.ExtensionDeveloper` is a technical role), CAP-EXT-003; concrete limits are ORG gap G-35 |
| **Last verified** | 2026-08-12 |

### Rule statement
SaaS applications offering customer extensibility MUST ship a deliberately configured extension allowlist on `cds.xt.ExtensibilityService` — namespace allowlists, limits on new fields/entities, and the documented `x_` element-name prefix ("All new elements have to start with `x_` → to avoid naming conflicts") — as a reviewed artifact, not an afterthought: without configuration, extensions are forbidden by default, so *enabling* extensibility means *deciding* its boundaries. Validation of these constraints runs in `cds build` and aborts `cds push` on violations (documented behavior); the concrete limit values are ORG policy (G-35).

### Rationale
The allowlist is the contract between provider and extending customers: what tenants may add to the provider's model, deployed into every extended tenant's database. Over-permissive boundaries let tenant extensions collide with future base-model evolution (the `x_` prefix exists precisely "to avoid naming conflicts") or bloat tenant schemas; absent deliberate configuration, teams either can't extend or flip on permissive defaults ad hoc. **High:** the allowlist guards base-model integrity and upgrade compatibility for *all* tenants of a SaaS product — a wrong boundary is expensive to retract once customer extensions exist.

### Evidence expected in code
Extensibility configuration (`cds.requires.extensibility` / ExtensibilityService restrictions: namespaces, element prefixes `x_`, field/entity caps) in the provider's config; an ADR or product decision recording the chosen limits (G-35).

### Detection guidance
1. Confirm the project offers customer extensibility (`cds add extensibility` artifacts, extensibility enabled in MTX config) → not offered → NOT APPLICABLE.
2. Locate the ExtensibilityService restriction configuration: namespaces, `element-prefix` (`x_`), new-field/new-entity limits → extensibility enabled with no restriction configuration beyond defaults → FAIL (undecided boundaries).
3. Verify the `x_` prefix constraint is active (documented default — confirm not overridden to empty).
4. Locate the recorded decision for the limit values (ADR/product doc per G-35) → absent → FAIL element (configured but unowned numbers).
5. Cross-check `cds.ExtensionDeveloper` role handling under CAP-SEC-009 (technical role separation) — report findings there.
6. Report configuration locations.

### Good example
```jsonc
// provider config — deliberate, reviewed extension boundaries (ADR-0021)
{ "cds": { "requires": { "cds.xt.ExtensibilityService": {
    "element-prefix": "x_",
    "extension-allowlist": [
      { "for": ["acme.bookshop.Books"], "new-fields": 10 },
      { "for": ["acme.bookshop"], "new-entities": 5 }
    ]
} } } }
```

### Bad example
```text
Extensibility switched on for a pilot customer with a blanket
allowlist covering all namespaces and no field caps — six months later
the base model can't add a field without colliding with some tenant's
extension, and no one owns the decision.
```

### Exception guidance
Internal-only extensibility (provider's own teams as the only extension developers) may run wider limits — still configured and recorded. No exception for customer-facing extensibility without configured boundaries.

### SAP reference
- https://cap.cloud.sap/docs/guides/extensibility/customization (`x_` prefix requirement; ExtensibilityService restrictions; `cds build` validation aborting `cds push`)
- https://cap.cloud.sap/docs/guides/multitenancy/mtxs (extensions forbidden without allowlist — verified Batch 1)

---

## CAP-EXT-003 — Follow the supported extension-development workflow

| Field | Value |
|---|---|
| **Rule ID** | CAP-EXT-003 |
| **Title** | Follow the supported extension-development workflow |
| **Category** | Extensibility |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented workflow; the `cds.ExtensionDeveloper` role is a documented requirement) |
| **Applicability** | Extension development against extensible SaaS applications (provider guidance to extension developers, and providers' own extension projects) |
| **Runtime** | Both (tooling is cds CLI) |
| **CAP version** | Streamlined MTX |
| **Status** | Active |
| **Related rules** | CAP-EXT-002, CAP-MT-005 (base upgrades interact with extensions), CAP-SEC-009 |
| **Last verified** | 2026-08-12 |

### Rule statement
Extensions MUST be developed through the supported workflow: authenticate with `cds login` (OAuth2 via XSUAA, requiring the `cds.ExtensionDeveloper` role), fetch the base model with `cds pull --from <app>`, develop and test locally (`cds watch` against the pulled base model), then `cds push` to a **test tenant first** and only then to production (the documented flow pushes as distinct tenant users for test and production). Extensions MUST NOT be applied by side channels — direct database changes, hand-deployed artifacts, or pushes straight to production tenants.

### Rationale
The workflow is what makes extensions validated (against the allowlist at `cds build`/`push`, CAP-EXT-002), versioned (extension projects are code), and staged (test tenant before production). Side-channel changes bypass allowlist validation and the provider's upgrade machinery — precisely the uncontrolled customization the extensibility mechanism exists to prevent. **Medium:** process integrity for a scoped audience; hard failures (broken tenants) are caught by the staging step it mandates.

### Evidence expected in code
Extension projects with pulled base models and `.cds` extension files; push scripts/runbooks showing test-tenant-first staging; no direct-DB or hand-deployment paths for extensions.

### Detection guidance
1. Locate extension projects (provider templates, customer-extension repos in scope).
2. Verify structure: base model via `cds pull` artifacts, extensions as `.cds` files validating with `cds build` → ad-hoc artifacts without the pulled base → FAIL.
3. Check the staging path: documented push sequence (test tenant → production) in scripts/runbook → production-only push paths → FAIL.
4. Look for side channels: SQL scripts or manual schema changes applied to tenant databases for "extensions" → FAIL (also cross-report CAP-MT-003).
5. NOT APPLICABLE where no extension development exists.

### Good example
```text
cds login extend.acme.com --to t-test        (cds.ExtensionDeveloper role)
cds pull --from https://acme-bookshop.app
# develop x_ratings.cds; cds watch → local verification
cds push --to t-test                          → verify in test tenant
cds push --to t-prod                          → production
```

### Bad example
```text
ALTER TABLE executed directly on a tenant's HDI container to add a
"customer extension" column — invisible to the allowlist, destroyed by
the next tenant upgrade (CAP-MT-005), unversioned.
```

### Exception guidance
Providers may wrap the flow in their own portal/automation — compliant when the same stages (validation, staging) execute underneath. No exception for unvalidated production pushes.

### SAP reference
- https://cap.cloud.sap/docs/guides/extensibility/customization (`cds login`/`pull`/`push`; test-tenant-first flow; `cds.ExtensionDeveloper` role; local testing with `cds watch`)

---

## CAP-EXT-004 — Keep toggled features isolated and schema-uniform

| Field | Value |
|---|---|
| **Rule ID** | CAP-EXT-004 |
| **Title** | Keep toggled features isolated and schema-uniform |
| **Category** | Extensibility |
| **Severity** | Medium |
| **Authority** | SAP-REQ (documented hard limitations in an explicit warning box) |
| **Applicability** | Projects using feature toggles (`fts/` features); NOT APPLICABLE otherwise |
| **Runtime** | Both — production toggle determination differs: Java requires a custom Feature Toggles Info Provider (mock users not for production); Node.js has ⏱ "no out-of-the-box feature toggles provider for production yet" — custom middleware setting `req.features` is required |
| **CAP version** | Current streamlined MTX feature-toggle support |
| **Status** | Active |
| **Related rules** | CAP-EXT-002, CAP-MT-003 (features deploy to every tenant DB — no per-tenant schema divergence), CAP-SEC-002 (no mock users in production) |
| **Last verified** | 2026-08-12 |

### Rule statement
Feature-toggled model extensions MUST respect the documented constraints: features live in `fts/<feature>` folders with no `.cds` files in nested subfolders; **no `using` dependencies between features** — anything a feature refers to or extends must be part of the base model; and no per-feature database schemas — "all features will be deployed to each tenant database" with activation controlled per tenant/user/request at runtime. Production deployments MUST provide a real toggle determination: Java via a custom Feature Toggles Info Provider (mock-user feature assignment "must not be used" in production); Node.js via custom middleware (no out-of-the-box production provider exists — a documented gap to plan for).

### Rationale
The constraints are hard framework limitations, not style: inter-feature `using` breaks the build in toggle combinations the developer never tested; nested `.cds` files are silently ignored; and because every feature's artifacts reach every tenant database, features are a *runtime visibility* mechanism, not a schema-isolation mechanism — designs assuming per-tenant schema divergence are unimplementable on this feature. The production-provider requirement prevents shipping with dev-time mock toggle assignment (which would also violate CAP-SEC-002). **Medium (downgraded from candidate High):** violations fail loudly at build/deploy or in the first toggle-combination test; the review value is catching the design assumptions early.

### Evidence expected in code
`fts/<feature>/` folders with flat `.cds` content; features referencing only base-model definitions; a production toggle provider (Java class / Node middleware) wired; no design docs assuming per-tenant schemas via features.

### Detection guidance
1. Check `fts/` structure: `.cds` files in nested subfolders → FAIL (silently ignored — documented).
2. Parse feature models for `using` references to other features' definitions → FAIL per dependency (base-model refactoring required).
3. Verify the schema-uniformity assumption: design/requirement documents expecting tenant-specific schema via features → FAIL the design (all features deploy everywhere — documented).
4. Production toggle determination: Java — custom Feature Toggles Info Provider implementation present; Node — middleware setting `req.features` → mock-user-based assignment reachable in production → FAIL (also CAP-SEC-002).
5. NOT APPLICABLE without `fts/` features.

### Good example
```text
fts/
├── isbn/isbn-extension.cds        ← flat; extends base-model Books only
└── reviews/reviews-extension.cds  ← independent of fts/isbn
srv/feature-provider.js            ← production middleware: req.features from
                                      the tenant's entitlement config
```

### Bad example
```cds
// fts/reviews/model.cds — depends on another feature: breaks whenever
// 'isbn' is toggled off for a tenant/request
using { sap.capire.bookshop.Books } from '../../db/schema';
using { ISBNInfo } from '../isbn/isbn-extension';   // inter-feature dependency
extend Books with { reviewOfIsbn : Association to ISBNInfo; }
```

### Exception guidance
None on the documented limitations (they are framework constraints). Genuine per-tenant schema needs are customer *extensions* (CAP-EXT-002/-003) or separate deployments — different mechanisms with different trade-offs; record the choice.

### SAP reference
- https://cap.cloud.sap/docs/guides/extensibility/feature-toggles (fts/ structure; no nested `.cds`; no `using` between features; all features deployed to each tenant database; Java Feature Toggles Info Provider for production; Node.js no out-of-the-box production provider)

# CAP-SEC — Security

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions for this category: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-06…G-15, G-41, G-43 in [research-gaps.md](../../references/research-gaps.md).

**Rules:** 18 active (7 Critical, 8 High, 3 Medium). All SAP references verified against official CAP documentation on **2026-08-11** (post-January-2026 docs restructure URLs), including a targeted re-verification pass of every SAP-REQ claim.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-SEC-001 | Model authorization explicitly for every exposed service | Critical | SAP-REQ | Both |
| CAP-SEC-002 | Production runs a real identity service — never mock authentication | Critical | SAP-REQ | Both |
| CAP-SEC-003 | Keep Node.js production deny-by-default authentication | Critical | SAP-REQ | Node.js |
| CAP-SEC-004 | Do not weaken CAP Java authentication defaults | High | SAP-REC | Java |
| CAP-SEC-005 | Verify CAP Java security auto-configuration is actually active | Critical | SAP-REQ | Java |
| CAP-SEC-006 | Prefer IAS for new projects | Medium | SAP-REC | Both |
| CAP-SEC-007 | Provision authorization explicitly when authenticating with IAS | High | SAP-REQ | Both |
| CAP-SEC-008 | Keep XSUAA scope definitions generated from the model | High | SAP-REQ | Both |
| CAP-SEC-009 | Never assign technical roles to business users | Critical | SAP-REQ | Both |
| CAP-SEC-010 | Model instance-based authorization declaratively | High | SAP-REC | Both |
| CAP-SEC-011 | Review authorization of every exposed association and composition | High | SAP-REQ | Both |
| CAP-SEC-012 | Validate externally writable input declaratively | High | SAP-REC | Both |
| CAP-SEC-013 | Construct queries injection-safe | Critical | SAP-REQ | Both |
| CAP-SEC-014 | Configure request-flooding limits deliberately | Medium | SAP-REQ | Both |
| CAP-SEC-015 | Backends enforce authentication independently of the App Router | Critical | SAP-REQ | Both |
| CAP-SEC-016 | Keep security-sensitive data out of logs | Medium | SAP-REC | Both |
| CAP-SEC-017 | No secrets in the repository or development artifacts | High | SAP-REC | Both |
| CAP-SEC-018 | Govern MCP exposure of CAP services | High | SAP-REC | Both |

---

## CAP-SEC-001 — Model authorization explicitly for every exposed service

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-001 |
| **Title** | Model authorization explicitly for every exposed service |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | Every CDS service served over any protocol (i.e., not marked internal-only) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions (verified against cds 10 / CAP Java 5 docs) |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-003, CAP-SEC-009, CAP-SEC-010, CAP-SEC-011, CAP-SEC-015 |
| **Last verified** | 2026-08-11 |

### Rule statement
Every externally served CDS service MUST carry an explicit, deliberate authorization model: `@requires` and/or `@restrict` at service level, or entity-level restrictions covering all exposed entities, actions, and functions. Authorization by omission (relying on runtime fallbacks) is prohibited; deliberately public endpoints MUST be explicitly annotated as such and documented.

### Rationale
SAP documents that "By default, CDS services have no access control", so "without authorization modeling, authenticated users have access to all entities" — modeling authorization is the application's responsibility. An exposed service without explicit authorization grants any (authenticated) caller full generic CRUD access to all exposed data. **Critical justification:** violation directly enables unauthorized read/write of business data.

### Implementation guidance
- Design roles and restrictions early, together with the service cut — one use-case service per user group keeps authorization simple (Layer 1 principles P3/P6).
- Keep authorization annotations in a dedicated aspect file (e.g., `srv/access-control.cds`) so the domain model stays readable.
- Use conceptual business roles (`Vendor`, `Auditor`), not technical scopes.

### Evidence expected in code
`@requires`/`@restrict` annotations on every service definition (or on all its exposed entities/actions) in `srv/**/*.cds` or a dedicated authorization aspect file; documented rationale for any endpoint intentionally open.

### Detection guidance
1. Enumerate service definitions: search `srv/**/*.cds` (and all files using `annotate <Service> with`) for `service <Name>`.
2. Exclude services marked internal-only (`@protocol: 'none'`).
3. For each remaining service, verify an explicit `@requires`/`@restrict` at service level, **or** that every exposed entity/action/function carries its own restriction (including via `annotate` files).
4. Flag services with no annotation anywhere (implicit runtime fallback only) as FAIL.
5. Flag `@requires: 'any'` without an adjacent documented justification as FAIL.
6. Verify authorization tests exist for at least one allowed and one denied case per restricted service (cross-check `CAP-TEST` evidence).
7. Report per service with file:line references.

### Good example
```cds
// srv/admin-service.cds
@requires: 'admin'
service AdminService {
  entity Books as projection on my.Books;
}

// srv/catalog-service.cds — deliberately public, documented
@requires: 'any'  // public catalog by design — see ADR-0007
service CatalogService {
  @readonly entity Books as projection on my.Books excluding { supplierCost };
}
```

### Bad example
```cds
// srv/order-service.cds — no authorization anywhere; any authenticated
// user (production) or anyone (dev defaults) gets full CRUD
service OrderService {
  entity Orders as projection on my.Orders;
}
```

### Exception guidance
Deliberately public services are not exceptions — they must be *explicitly* modeled (`@requires: 'any'`) with a documented decision (ADR or exception record naming the approver). No other exception is acceptable.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authorization ("By default, CDS services have no access control"; `@requires`/`@restrict` semantics)

---

## CAP-SEC-002 — Production runs a real identity service — never mock authentication

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-002 |
| **Title** | Production runs a real identity service — never mock authentication |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | All projects with a production deployment target |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-003, CAP-SEC-005, CAP-SEC-006, CAP-SEC-015, CAP-MT-003 |
| **Last verified** | 2026-08-11 |

### Rule statement
Production profiles MUST authenticate against a bound identity service (IAS or XSUAA). `mocked`, `basic`, and `dummy` authentication (Node.js) and mock users (Java `cds.security.mock.*`) MUST NOT be active or reachable in any production configuration.

### Rationale
SAP states verbatim: "Mocked authentication is not suitable for production!"; `dummy` additionally disables `@requires`/`@restrict` evaluation. Mock users are well-known shared identities — leaving them active lets anyone authenticate with published credentials and acquire the roles configured for them. **Critical justification:** violation is a direct, trivially exploitable authentication bypass.

### Implementation guidance
- Scaffold production auth with `cds add xsuaa` or the IAS facets — this wires the `[production]` profile and binding artifacts.
- Define mock users only inside `[development]`/test profiles, never profile-neutrally.
- Java: mock users are disabled in production by default (`cds.security.mock.enabled` is `false` in the production profile) — never override this.

### Evidence expected in code
Node.js: `cds.requires.auth` resolving to `jwt`/`xsuaa`/`ias` under the production profile (package.json / `.cdsrc.json`), with `xsuaa`/`identity` resources in `mta.yaml` or Helm values. Java: `cds-feature-identity` (or an including starter) in `pom.xml`, identity binding in deployment descriptors, no `cds.security.mock.enabled: true` in a production/cloud profile.

### Detection guidance
1. Node.js: resolve the production auth kind (`cds env get requires.auth --profile production`, or statically read `package.json`/`.cdsrc.json` including `[production]` blocks); FAIL on `mocked`, `basic`, or `dummy`.
2. Search all configuration for mock user definitions (`users:` under `cds.requires.auth`; `cds.security.mock.users` in `application.yaml`) and verify they are scoped to non-production profiles.
3. Verify `mta.yaml` / Helm `values.yaml` binds an `xsuaa` or `identity` service instance to the CAP module.
4. Java: check no production/cloud profile sets `cds.security.mock.enabled: true`.
5. Report the effective production auth kind with file:line evidence.

### Good example
```jsonc
// package.json (Node.js)
{ "cds": { "requires": {
    "auth": "mocked",                       // dev default
    "[production]": { "auth": { "kind": "xsuaa" } }
} } }
```

### Bad example
```jsonc
// package.json — mocked auth with privileged mock user in ALL profiles
{ "cds": { "requires": { "auth": {
    "kind": "mocked", "users": { "admin": { "roles": ["admin"] } } } } } }
```

### Exception guidance
None. There is no legitimate production use of mocked/dummy authentication.

### SAP reference
- https://cap.cloud.sap/docs/node.js/authentication ("Mocked authentication is not suitable for production!")
- https://cap.cloud.sap/docs/java/security (mock users; `cds.security.mock.enabled` production default)

---

## CAP-SEC-003 — Keep Node.js production deny-by-default authentication

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-003 |
| **Title** | Keep Node.js production deny-by-default authentication |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ (documented secure production default; the rule forbids disabling it) |
| **Applicability** | Node.js projects with a production deployment target |
| **Runtime** | Node.js (Java counterpart: CAP-SEC-004) |
| **CAP version** | Production deny-by-default per current `@sap/cds` documentation (verified on cds 10 docs) |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-002, CAP-SEC-004 |
| **Last verified** | 2026-08-11 |

### Rule statement
Projects MUST NOT set `cds.requires.auth.restrict_all_services: false`. In production, services without explicit restrictions retain the framework's implicit `@requires: 'authenticated-user'`. This fallback is a safety net, not a design — explicit authorization per CAP-SEC-001 is still required.

### Rationale
SAP documents that in production "all services without `@restrict` or `@requires` implicitly get `@requires: 'authenticated-user'`", disableable via `cds.requires.auth.restrict_all_services: false`. Disabling it exposes every unannotated service to unauthenticated callers. **Critical justification:** one configuration line silently makes all unannotated services public.

### Evidence expected in code
Absence of `restrict_all_services: false` in all cds configuration sources (package.json, `.cdsrc.json`, `.cdsrc-private.json`, env-style config in `mta.yaml`/Helm values).

### Detection guidance
1. Search the repository for `restrict_all_services` (JSON/YAML config and `CDS_REQUIRES_AUTH_RESTRICT_ALL_SERVICES`-style env variables in deployment descriptors).
2. Any occurrence set to `false` in a profile that can reach production → FAIL with file:line.
3. If absent → PASS (default preserved); note in the report that this does not substitute for CAP-SEC-001.

### Good example
```jsonc
// package.json — no restrict_all_services override anywhere
{ "cds": { "requires": { "[production]": { "auth": { "kind": "xsuaa" } } } } }
```

### Bad example
```jsonc
// .cdsrc.json — disables the production authentication fallback globally
{ "requires": { "auth": { "restrict_all_services": false } } }
```

### Exception guidance
None. A service that must be public is declared public explicitly (`@requires: 'any'`, per CAP-SEC-001), never by disabling the global default.

### SAP reference
- https://cap.cloud.sap/docs/node.js/authentication (implicit `@requires: 'authenticated-user'` in production; `restrict_all_services` flag)

---

## CAP-SEC-004 — Do not weaken CAP Java authentication defaults

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-004 |
| **Title** | Do not weaken CAP Java authentication defaults |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REC (SAP: override security defaults "only when absolutely necessary") |
| **Applicability** | CAP Java projects with a production deployment target |
| **Runtime** | Java (Node.js counterpart: CAP-SEC-003) |
| **CAP version** | Property names/defaults verified on CAP Java 5 docs (`cds.security.authentication.mode` default `model-relaxed`) |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-003, CAP-SEC-005 |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP Java security configuration MUST NOT be weakened below its documented defaults in any profile reachable from production: `cds.security.authentication.mode` MUST NOT be set to `never`, and `authenticateUnknownEndpoints` / `authenticateMetadataEndpoints` MUST NOT be set to `false`. Adopting the stricter `model-strict` mode is recommended but remains an ORG decision (gap G-15).

### Rationale
SAP's security guidance is to deviate from security defaults only when absolutely necessary. These switches control whether unknown endpoints, `$metadata`, and (in `never` mode) all endpoints skip authentication; lowering them silently widens the unauthenticated surface. **High (not Critical):** with CAP-SEC-001/-002/-005 in place, weakened defaults expose metadata/unknown endpoints rather than directly granting business-data access — but combined with any modeling gap it escalates.

### Implementation guidance
- Leave `cds.security.*` unset unless there is a reviewed reason; absence of configuration is compliance here.
- If your organization ratifies `model-strict` (G-15), set it globally and annotate deliberately public endpoints with `@requires: 'any'`.

### Evidence expected in code
No weakening overrides of `cds.security.authentication.*` in `application.yaml`/`application.properties`/env config; if `model-strict` is adopted, it appears consistently in the default or cloud profile.

### Detection guidance
1. Search `srv/src/main/resources/application*.{yaml,yml,properties}` and deployment descriptors for `cds.security`.
2. FAIL on `mode: never`, `authenticateUnknownEndpoints: false`, or `authenticateMetadataEndpoints: false` in any profile that can reach production.
3. Record (as observation, not FAIL) whether `model-strict` is adopted, referencing G-15.

### Good example
```yaml
# application.yaml — hardened beyond default per ORG decision (G-15)
cds:
  security.authentication.mode: model-strict
```

### Bad example
```yaml
# application.yaml — $metadata and unknown endpoints now unauthenticated
cds:
  security.authentication:
    authenticateUnknownEndpoints: false
    authenticateMetadataEndpoints: false
```

### Exception guidance
A single public endpoint never justifies weakening these globals — model it explicitly per endpoint. Any exception requires a documented security review naming the approver.

### SAP reference
- https://cap.cloud.sap/docs/java/security (authentication modes; defaults)
- https://cap.cloud.sap/docs/guides/security/overview (override defaults only when absolutely necessary)

---

## CAP-SEC-005 — Verify CAP Java security auto-configuration is actually active

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-005 |
| **Title** | Verify CAP Java security auto-configuration is actually active |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ (documented activation precondition) |
| **Applicability** | CAP Java projects with a production deployment target |
| **Runtime** | Java |
| **CAP version** | CAP Java 2.x–5.x (`cds-feature-identity` since Java 3; predecessor `cds-feature-xsuaa` deprecated) |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-004, CAP-SEC-015 |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP Java projects MUST include the security dependencies (`cds-feature-identity` plus Spring Security, directly or via a cds starter bundle) **and** bind an XSUAA/IAS service instance in every production deployment — and MUST verify both, because automatic authentication enforcement activates only when both are present. A deployed verification (test rejecting unauthenticated requests) MUST exist.

### Rationale
SAP documents: "Only if both, the library dependencies and an XSUAA or IAS service binding are in place, the CAP Java SDK activates a Spring security configuration, which enforces authentication for all endpoints automatically." Missing either one fails **open**: the application starts and serves all endpoints unauthenticated, with no visible error. **Critical justification:** silent, total authentication bypass in production caused by an omission that produces no failure signal.

### Implementation guidance
- Use `cds add xsuaa`/IAS facets so dependencies and binding artifacts stay in sync.
- Add a pipeline smoke test: unauthenticated `GET` to a protected endpoint must return 401/403 after deployment (also satisfies CAP-SEC-015).

### Evidence expected in code
`pom.xml` containing `cds-feature-identity` (directly or via `cds-starter-cloudfoundry`/equivalent); `mta.yaml`/Helm values binding an `xsuaa` or `identity` instance to the Java module; a CI/post-deploy test asserting 401/403 for unauthenticated access.

### Detection guidance
1. Inspect `pom.xml` (all modules): confirm `cds-feature-identity` or a starter bundle including it, plus Spring Security on the classpath.
2. Inspect `mta.yaml` / chart values: confirm the Java module requires an xsuaa/identity resource.
3. Locate the unauthenticated-rejection test (integration test or pipeline smoke step); absent → FAIL this verification element.
4. FAIL the rule if either dependency or binding is missing; report file:line for each element found/missing.

### Good example
```xml
<!-- srv/pom.xml -->
<dependency>
  <groupId>com.sap.cds</groupId>
  <artifactId>cds-feature-identity</artifactId>
</dependency>
```
```yaml
# mta.yaml
modules:
  - name: bookshop-srv
    requires: [ bookshop-auth ]
resources:
  - name: bookshop-auth
    type: org.cloudfoundry.managed-service
    parameters: { service: xsuaa, service-plan: application }
```

### Bad example
```yaml
# mta.yaml — Java module deployed with NO identity binding:
# the app starts fine and serves every endpoint unauthenticated
modules:
  - name: bookshop-srv
    type: java
    path: srv
```

### Exception guidance
None for production. Even network-isolated internal deployments require a documented security review before omitting the identity binding.

### SAP reference
- https://cap.cloud.sap/docs/java/security (enforcement activates only with dependencies **and** service binding)

---

## CAP-SEC-006 — Prefer IAS for new projects

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-006 |
| **Title** | Prefer IAS for new projects |
| **Category** | Security |
| **Severity** | Medium |
| **Authority** | SAP-REC ("Start new projects with IAS") |
| **Applicability** | New projects choosing an identity service; existing XSUAA apps out of scope (migration = ORG decision, gap G-43) |
| **Runtime** | Both |
| **CAP version** | Version-sensitive direction from the Jan 2026 security-guide restructure; re-verified on cds 10 / Java 5 docs — re-check at each CAP major |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-007, CAP-SEC-008 |
| **Last verified** | 2026-08-11 |

### Rule statement
New CAP projects SHOULD use SAP Cloud Identity Services (IAS) as their identity service, paired with explicit authorization provisioning (CAP-SEC-007). Choosing XSUAA for a new project requires a documented reason; existing XSUAA applications SHOULD plan against SAP's cross-consumption path rather than deepening new XSUAA investment.

### Rationale
SAP's authentication guide states: "Start new projects with IAS to take advantage of the best integration options", explicitly labelling XSUAA-based services "legacy" with IAS cross-consumption as the bridge. **Medium:** choosing XSUAA creates strategic/technical debt against SAP's documented direction, not a vulnerability.

### Evidence expected in code
For new projects: `ias` auth kind / `identity` binding plus AMS artifacts per CAP-SEC-007 — or an ADR documenting why XSUAA was chosen.

### Detection guidance
1. Determine greenfield status (git history start, ADRs).
2. Read the effective production auth configuration (Node: `cds.requires.auth`; Java: binding type in `mta.yaml`/values).
3. New project on XSUAA without a documented decision → FAIL (Medium); with documented reason → PASS with observation.
4. Existing XSUAA project → NOT APPLICABLE (note G-43 migration-policy status).

### Good example
```jsonc
// package.json — new project on IAS (+ AMS per CAP-SEC-007)
{ "cds": { "requires": { "[production]": { "auth": { "kind": "ias" } } } } }
```

### Bad example
```text
New 2026 project scaffolded on XSUAA with no recorded decision, no
landscape constraint, and no migration consideration.
```

### Exception guidance
Legitimate: landscape/service availability constraints, mandated enterprise IdP architecture, reuse of an established XSUAA authorization landscape. Record reason and re-evaluation date in an ADR.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authentication ("Start new projects with IAS"; XSUAA labelled legacy)

---

## CAP-SEC-007 — Provision authorization explicitly when authenticating with IAS

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-007 |
| **Title** | Provision authorization explicitly when authenticating with IAS |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REQ ("Closing this gap is up to you as application developer") |
| **Applicability** | All projects using IAS authentication |
| **Runtime** | Both |
| **CAP version** | AMS integration per current docs (Node `@sap/ams` v3; CAP Java AMS support libraries) |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-006, CAP-SEC-008 |
| **Last verified** | 2026-08-11 |

### Rule statement
Projects authenticating with IAS MUST provide an explicit authorization source for CAP roles — SAP's documented path is Authorization Management Service (AMS) policies (`cds add ams`) — or a documented, reviewed alternative mapping. Code MUST NOT assume IAS tokens convey roles or scopes.

### Rationale
SAP documents: "JWT tokens issued by IAS service don't contain authorization information. In particular, no scopes are included. Closing this gap is up to you as application developer." Without explicit provisioning, `@requires`/`@restrict` either lock everyone out (pressuring developers to loosen restrictions "to make it work") or a hand-rolled mapping becomes an unreviewed security-critical component. **High:** the direct failure is a broken or improvised authorization layer; it becomes unauthorized access only through a secondary mistake.

### Evidence expected in code
With IAS auth: AMS artifacts (DCL policy files, `@sap/ams` in `package.json` or the CAP Java AMS support dependency in `pom.xml`, AMS instance in deployment descriptors) — or an ADR plus the implemented alternative role mapping with tests.

### Detection guidance
1. Confirm IAS is the effective production auth (per CAP-SEC-002/-006 detection).
2. Look for AMS: `@sap/ams` dependency / Java AMS support artifact; DCL policy files; AMS consumption in `mta.yaml`/values.
3. If no AMS: look for a documented custom mapping (ADR + middleware/handler assigning roles) and its tests.
4. Neither found → FAIL: modeled roles can never be granted.
5. Verify at least one test proves a role arrives end-to-end (else mark that element NOT ASSESSABLE, naming the runtime evidence needed).

### Good example
```text
package.json:         "@sap/ams": "^3"
ams/basePolicies.dcl  POLICY admin { GRANT admin; }
mta.yaml:             identity instance with AMS consumption configured
```

### Bad example
```text
Auth kind "ias"; services annotated @requires:'admin'; no AMS and no
custom role mapping — no user can ever hold 'admin', and the applied
"fix" was deleting the @requires annotation.
```

### Exception guidance
A documented custom mapping (e.g., enterprise IdP group sync) is acceptable if reviewed as security-critical code and covered by tests; record it as the AMS alternative in an ADR.

### SAP reference
- https://cap.cloud.sap/docs/node.js/authentication (IAS tokens contain no scopes; gap is the developer's)
- https://cap.cloud.sap/docs/guides/security/cap-users (AMS policies inject CAP roles for IAS)

---

## CAP-SEC-008 — Keep XSUAA scope definitions generated from the model

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-008 |
| **Title** | Keep XSUAA scope definitions generated from the model |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REQ (scope names must exactly match CDS role names — checked at runtime) |
| **Applicability** | Projects using XSUAA-based authorization |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-007, CAP-SEC-009 |
| **Last verified** | 2026-08-11 |

### Rule statement
`xs-security.json` MUST be generated from the CDS model (`cds compile srv --to xsuaa`) and regenerated whenever roles in the model change. Scope/role-template names MUST match the CDS role names exactly; hand-maintained drift between model roles and XSUAA scopes is prohibited.

### Rationale
SAP documents that scope names in `xs-security.json` must "exactly match the role names in the CDS model, as these scope names will be checked at runtime". A model role with no matching scope can never be granted (silently broken authorization); orphaned scopes indicate stale or misassigned privileges. **High:** drift breaks or corrupts the authorization model, though exploitation requires an additional assignment error.

### Implementation guidance
- Regenerate on every model change touching `@requires`/`@restrict` roles; keep the command in build docs or a script.
- Review the generated file before deployment — generation does not replace reviewing role templates and attributes.

### Evidence expected in code
`xs-security.json` whose scopes/role-templates correspond 1:1 to the roles used in CDS annotations (minus documented platform-technical entries).

### Detection guidance
1. Extract role names from all `@requires` and `@restrict.to` usages in `*.cds` (exclude pseudo-roles `any`, `authenticated-user`, `system-user`, `internal-user`).
2. Extract scope and role-template names from `xs-security.json` (strip `$XSAPPNAME.`).
3. Diff both sets; any mismatch → FAIL with the missing/orphaned names and locations.
4. Check build docs/git history that regeneration follows model changes (observation if not determinable).

### Good example
```cds
@requires: 'Auditor' service AuditService { /* … */ }
```
```jsonc
// xs-security.json (generated)
{ "scopes": [ { "name": "$XSAPPNAME.Auditor" } ],
  "role-templates": [ { "name": "Auditor", "scope-references": [ "$XSAPPNAME.Auditor" ] } ] }
```

### Bad example
```text
CDS uses @requires:'Auditor'; xs-security.json only defines scope
"$XSAPPNAME.Audit" (hand-renamed) — the Auditor role can never be granted.
```

### Exception guidance
Hand-added technical entries (e.g., scopes required by platform services, `grant-as-authority-to-apps` grants) are legitimate; document them in the file/README and list them explicitly so the diff check can exclude them.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/cap-users (exact scope↔role match; `cds compile srv --to xsuaa`)

---

## CAP-SEC-009 — Never assign technical roles to business users

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-009 |
| **Title** | Never assign technical roles to business users |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | All projects; heightened relevance for multitenant SaaS |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-008, CAP-MT-004 |
| **Last verified** | 2026-08-11 (technical-role statement re-verified on the current data-protection page) |

### Rule statement
Technical roles — `cds.Subscriber`, `mtcallback`, `emcallback`, `cds.ExtensionDeveloper`, and any role gating `internal-user`/`system-user` endpoints — MUST never be included in business role templates or role collections. Service instances used for `internal-user` access MUST NOT be shared with untrusted clients.

### Rationale
SAP states: "Ensure that technical roles such as `cds.Subscriber`, `mtcallback`, or `emcallback` are never included in business roles", and separately warns not to mix CAP roles for business and technical users. These roles gate tenant lifecycle operations (subscribe/unsubscribe/upgrade), extension development, and internal service-to-service channels. **Critical justification:** violation enables privilege escalation up to destructive tenant-lifecycle operations and cross-tenant exposure.

### Evidence expected in code
`xs-security.json` (and role-collection definitions in deployment artifacts) where technical scopes appear only app-to-app (`grant-as-authority-to-apps`) or in dedicated technical role templates never meant for user assignment.

### Detection guidance
1. In `xs-security.json` and deployment descriptors, list all role-templates and their scope references.
2. Flag any user-assignable role-template containing `mtcallback`, `emcallback`, `cds.Subscriber`, `cds.ExtensionDeveloper`, or scopes used by `@requires: 'internal-user'`-guarded services.
3. Check that SaaS callback scopes are granted via `grant-as-authority-to-apps`, not user role collections.
4. Any technical scope reachable through a business role collection → FAIL with file:line.
5. Where role-collection assembly happens only in the BTP cockpit (not in the repo), mark that portion NOT ASSESSABLE and name the needed evidence (cockpit export).

### Good example
```jsonc
// xs-security.json — SaaS callback scope granted app-to-app only
{ "scopes": [
    { "name": "$XSAPPNAME.mtcallback",
      "grant-as-authority-to-apps": [ "$XSAPPNAME(application,sap-provisioning,tenant-onboarding)" ] }
] }
```

### Bad example
```jsonc
// role template intended for humans includes the SaaS callback scope
{ "role-templates": [ { "name": "PowerUser",
    "scope-references": [ "$XSAPPNAME.admin", "$XSAPPNAME.mtcallback" ] } ] }
```

### Exception guidance
None. Operations personnel needing lifecycle actions get dedicated, audited technical role collections separate from any business role.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/data-protection ("technical roles such as cds.Subscriber, mtcallback, or emcallback are never included in business roles")
- https://cap.cloud.sap/docs/guides/security/cap-users (don't mix business and technical roles; `internal-user` instance isolation)

---

## CAP-SEC-010 — Model instance-based authorization declaratively

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-010 |
| **Title** | Model instance-based authorization declaratively |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | Services requiring row-level (instance-based) access control |
| **Runtime** | Both |
| **CAP version** | All currently supported versions (CAP Java ≥ 4: deep authorization / input checking active by default) |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-011, CAP-SEC-013 |
| **Last verified** | 2026-08-11 |

### Rule statement
Row-level access control MUST be modeled declaratively via `@restrict … where` (using `$user`, `$user.<attr>`, association paths, `exists` predicates), not improvised in handlers. Where custom handlers run their own queries for restricted data, they MUST preserve the equivalent filter semantics — custom code MUST NOT widen access beyond the modeled restriction.

### Rationale
SAP's authorization guide provides instance-based authorization as a declarative model feature enforced by the generic handlers. Hand-rolled row filtering in handlers is invisible to the model, unenforced on generic paths, untested by default, and drifts from the declared rules — the classic source of horizontal-privilege leaks. **High:** the declarative mechanism exists and is enforced; bypassing it makes correctness a per-handler accident.

### Implementation guidance
- Express ownership/tenancy filters in the model: `@restrict: [{ grant: '*', to: 'Vendor', where: 'createdBy = $user' }]`.
- Empty attribute value lists fully restrict access (documented behavior) — design attribute provisioning accordingly.
- Keep restriction aspects in the dedicated authorization file (CAP-SEC-001 guidance).

### Evidence expected in code
`@restrict` with `where` clauses for entities with row-level requirements; absence of ad-hoc user-based filtering logic in handlers that substitutes for (rather than supplements) model restrictions.

### Detection guidance
1. Identify entities with row-level requirements (from requirements/roles docs and from any handler code filtering by `req.user`/`UserInfo`).
2. For each, check the model for `@restrict … where` covering the requirement.
3. Flag handlers implementing row filters for entities lacking a model restriction → FAIL (restriction belongs in the model).
4. Flag custom queries (e.g., `SELECT` in `on` handlers) on restricted entities that drop the modeled filter → FAIL.
5. Verify tests cover both the allowed and the filtered-out case per restricted entity.

### Good example
```cds
annotate OrderService.Orders with @(restrict: [
  { grant: ['READ','UPDATE'], to: 'Vendor', where: 'vendor.ID = $user.vendorID' }
]);
```

### Bad example
```js
// srv/order-service.js — filter exists only inside one handler;
// generic READs, drafts, and future handlers bypass it
srv.on('READ', 'Orders', req =>
  SELECT.from('Orders').where({ vendorID: req.user.attr.vendorID }))
```

### Exception guidance
Filters genuinely inexpressible in `where` clauses (complex cross-service conditions) may live in `before`/`on` handlers — documented at the entity with a comment referencing the exception record, and covered by dedicated authorization tests.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authorization (instance-based authorization; `where` conditions; attribute semantics)

---

## CAP-SEC-011 — Review authorization of every exposed association and composition

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-011 |
| **Title** | Review authorization of every exposed association and composition |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REQ (documented runtime enforcement gaps requiring application action) |
| **Applicability** | All services exposing entities with associations/compositions |
| **Runtime** | Both (gap details differ per runtime — see statement) |
| **CAP version** | Verified on current docs; CAP Java ≥ 4 checks associated entities by default, but composition-children restrictions remain unchecked |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-010, CAP-SEC-018 |
| **Last verified** | 2026-08-11 |

### Rule statement
Every association and composition reachable through an exposed entity MUST have a deliberate authorization decision, because the runtimes do not fully enforce target restrictions: SAP documents that restrictions on compositions are not checked by the CAP Java runtime (custom handlers required for child-level rules), and that Node.js evaluates security annotations only on the request's target entity — restrictions of associated entities in expands/deep operations are not regarded. For each reachable target either: (a) rely on the root/target restriction being sufficient, (b) exclude the path from the projection, or (c) add a custom authorization handler — and record which.

### Rationale
Navigation and `$expand` make reachable data readable regardless of annotations the runtime does not evaluate on that path. Without a per-path decision, a properly restricted entity can leak through a neighboring exposure. **High:** real unauthorized-read potential, but exploitation requires a reachable path whose target restriction actually differs — a review-scale problem rather than an unconditionally open door.

### Implementation guidance
- Prefer structural prevention: tailor projections so sensitive targets are simply not reachable from broader services (P3 — use-case facades).
- Where child-level rules matter (e.g., approval items stricter than the order root), add explicit `before` handlers checking the child restriction.

### Evidence expected in code
Projection design excluding non-needed navigation targets; custom authorization handlers where child-level restrictions exist; a recorded exposure review (ADR/review notes) for services with mixed-sensitivity graphs.

### Detection guidance
1. For each exposed entity, enumerate associations/compositions and their targets (transitively, within the service).
2. For each reachable target, compare its own `@restrict`/`@requires` against the path's effective enforcement: flag targets whose restrictions are stricter than the root's — these are unenforced on expand/deep paths.
3. For flagged paths, look for mitigation: exclusion from projection, equivalent root restriction, or a custom handler implementing the check.
4. No mitigation found → FAIL with the path (`Service.Entity.assoc → Target`) and file:line.
5. Verify auto-exposed composition targets (`@cds.autoexpose`, drafts) are included in the walk.

### Good example
```cds
service ProjectService {
  // salaries stricter than projects → not reachable here at all
  entity Projects as projection on my.Projects excluding { staffCosts };
}
```

### Bad example
```cds
service ProjectService {
  entity Projects as projection on my.Projects;   // Projects → members → salary
}
// my.Salaries carries @restrict to:'HR' — but READ Projects?$expand=members($expand=salary)
// is authorized only against Projects (Node.js), leaking salary data
```

### Exception guidance
Graphs where all reachable targets share the root's exact restriction level need no per-path mitigation — state this once in the exposure review. Everything else requires mitigation or a documented, approved acceptance.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authorization ("Restrictions on compositions are not checked by the runtime" [Java]; Node.js: "security annotations are only evaluated on the target entity of the request")

---

## CAP-SEC-012 — Validate externally writable input declaratively

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-012 |
| **Title** | Validate externally writable input declaratively |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | All service entities/actions accepting external write input |
| **Runtime** | Both |
| **CAP version** | `@assert.range` open intervals since `@sap/cds` 8.5 / CAP Java 3.5; Java validates action/function parameters (deep) since cds-java 1.28 |
| **Status** | Active |
| **Related rules** | CAP-SEC-013, CAP-SEC-014; future CAP-SRV validation rule will cross-reference, not duplicate |
| **Last verified** | 2026-08-11 |

### Rule statement
Every externally writable element and action/function parameter MUST have a deliberate validation decision, implemented declaratively where CDS can express it — `@mandatory`, `@assert.format`, `@assert.range`, `@assert.target`, `@assert:(…)` constraints — with imperative handler validation only for rules annotations cannot express. Elements that must not be client-writable are excluded from the projection or marked `@readonly` (never on key elements).

### Rationale
CDS constraints are enforced uniformly by the generic handlers on every write path (including drafts on activation), rejecting the request and rolling back the transaction — handler-only validation covers only the paths the handler author remembered. Unvalidated input is the entry point for data-integrity and downstream injection issues. This also addresses mass-assignment: fields absent from the projection or `@readonly` cannot be set by clients. **High:** first-line defense whose absence multiplies the impact of every other flaw.

### Implementation guidance
- Add constraints in the service model where consumers see them (they also generate UI metadata like `FieldControl/Mandatory`).
- Remember draft caveat (see SAP Fiori guide): validations must hold for direct updates of active entities too, not only draft activation.
- Validate user-controlled values embedded into messages/URLs before echoing them (see CAP-SEC-016 for logs).

### Evidence expected in code
Validation annotations on writable elements in `srv/**/*.cds` (or annotate files); `@readonly`/exclusion for protected fields; handler validation only for genuinely non-declarative rules.

### Detection guidance
1. Enumerate exposed entities accepting CREATE/UPDATE and all action/function parameters.
2. For each writable element, check for an applicable constraint annotation or an explicit decision that none is needed (e.g., free-text field).
3. Flag: externally writable elements with format/range semantics but no `@assert.*`; mandatory business fields without `@mandatory`; client-settable fields that should be system-controlled (not `@readonly`, not excluded) → FAIL per finding.
4. Flag `@readonly` on key elements → FAIL (documented prohibition).
5. Check tests include rejection cases for invalid input (cross-check CAP-TEST evidence).

### Good example
```cds
entity Orders as projection on my.Orders { *, } excluding { internalRating } actions {
  action cancel(reason : String @mandatory);
};
annotate OrderService.Orders with {
  quantity @assert.range: [ 1, 1000 ];
  email    @assert.format: '^[^@]+@[^@]+$';
  status   @readonly;                     // system-controlled, not client-writable
};
```

### Bad example
```cds
// Everything writable, nothing validated; 'discount' is client-settable
// though it must only be computed server-side
entity Orders as projection on my.Orders;
```

### Exception guidance
Rules inexpressible declaratively (cross-entity checks, remote lookups) belong in `before` handlers — with a comment at the element noting where validation lives. Free-form fields may reasonably have no constraint; the decision must be visible in review notes for security-relevant entities.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/constraints (constraint annotations; enforcement semantics; `@readonly` never on keys)
- https://cap.cloud.sap/docs/guides/uis/fiori (validations must also cover direct updates of active entities)

---

## CAP-SEC-013 — Construct queries injection-safe

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-013 |
| **Title** | Construct queries injection-safe |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | All custom code constructing or modifying queries (CQL/CQN, native SQL, HANA hints) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-010, CAP-SEC-012; absorbs candidate CAP-DB-6 (the future CAP-DB category will cross-reference, not restate) |
| **Last verified** | 2026-08-11 |

### Rule statement
User input MUST enter queries only as parameter values, never as query text or structure:
- **Node.js:** never build queries by string concatenation; never surround tagged template strings with parentheses (this degrades them to plain concatenation); use `cds.ql` fluent APIs or tagged templates.
- **Java:** pass runtime/external values via `CQL.val()`/`CQL.param()`/named parameters; never as `constant()` literals; `hdb.`-prefixed hints are rendered directly into SQL and MUST NOT contain external input.
- **Both:** where query *structure* (entity/element names, orderby targets) derives from request input, validate it against a positive list before use. The same rules apply to any (justified) native SQL.

### Rationale
SAP renders CQL to prepared statements — safe for values — and explicitly warns: "Never use string concatenation when constructing queries!", tagged templates must not be parenthesized, constant literals "must not contain external input", `hdb.` hints "must not contain external input", and input-dependent query structure needs positive-list validation. **Critical justification:** violation is SQL/CQL injection — arbitrary read/write across the database, including cross-tenant data on shared infrastructure.

### Implementation guidance
- Prefer the fluent APIs (`SELECT.from(Books).where({ ID }))` / Java static model with `CQL.param()`), which make injection-unsafe forms hard to write.
- For dynamic sorting/filtering features, map allowed client tokens to model element names via a hard-coded lookup table.

### Evidence expected in code
Handler code free of string-built queries; parameterized values throughout; a positive-list mapping wherever query structure is input-dependent.

### Detection guidance
1. Search handler/service code (`srv/**/*.js|ts`, `srv/src/main/java/**`) for: string concatenation or template interpolation inside `cds.run`/`cds.ql`/`db.run` arguments; parenthesized tagged templates (`SELECT(` … `` ` `` patterns); `CQL.constant(`/inline literals fed from request data; `.hints(` / `hdb.` strings containing variables.
2. Search for native SQL (`JdbcTemplate`, raw `run("SELECT …` strings) and apply the same checks.
3. Trace any query whose entity/column/orderby names come from `req.data`/`req._queryOptions`/parameters: verify a positive-list validation precedes use; absent → FAIL.
4. Report each finding with file:line and the tainted variable.
5. Where taint flow can't be statically determined, mark NOT ASSESSABLE for that site and name what's needed (test or trace).

### Good example
```js
// Node.js — values as parameters; structure from a positive list
const allowed = { title: 'title', date: 'createdAt' };
const orderBy = allowed[req.data.sortBy] ?? 'title';
const books = await SELECT.from(Books).where({ author_ID: req.data.author }).orderBy(orderBy);
```

### Bad example
```js
// string-built query — injectable via req.data.author
const books = await cds.run(`SELECT * from my_Books where author='${req.data.author}'`);
// parenthesized tagged template — same problem in disguise
const also = await SELECT.from(Books).where(`author=${req.data.author}`);
```

### Exception guidance
None for the value rules. Input-dependent structure is itself the exception path and is only permitted with positive-list validation as stated.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-ql ("Never use string concatenation…"; no parenthesized tagged templates)
- https://cap.cloud.sap/docs/java/working-with-cql/query-api (`val()`/`param()` vs `constant()`; constants must not contain external input)
- https://cap.cloud.sap/docs/java/working-with-cql/query-execution (`hdb.` hints must not contain external input)
- https://cap.cloud.sap/docs/guides/security/data-protection (prepared statements; positive-list validation for input-dependent structure)

---

## CAP-SEC-014 — Configure request-flooding limits deliberately

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-014 |
| **Title** | Configure request-flooding limits deliberately |
| **Category** | Security |
| **Severity** | Medium |
| **Authority** | SAP-REQ (SAP names these developer responsibilities; concrete values are ORG — gaps G-07/G-08) |
| **Applicability** | All projects serving OData/HTTP APIs in production |
| **Runtime** | Both (mechanisms differ — see statement) |
| **CAP version** | Java property names per current CAP Java docs; Node.js `$expand` limiting has no framework switch (custom handler required) |
| **Status** | Active |
| **Related rules** | CAP-SEC-012, CAP-SEC-018; future CAP-PERF pagination rule will cross-reference |
| **Last verified** | 2026-08-11 |

### Rule statement
Projects MUST make deliberate, recorded decisions for the documented request-amplification controls: `$batch` size (Java: `cds.odataV4.batch.maxRequests`), `$expand` depth (Java: `cds.query.restrictions.expand.maxLevels`; Node.js: a custom handler per SAP), and pagination limits (`@cds.query.limit`, keeping the framework default cap). A rate-limiting strategy MUST exist (application- or platform-level); its concrete thresholds are ORG policy (G-07), and the Node.js `$expand` handler pattern is ORG-defined (G-08).

### Rationale
SAP's data-protection guide assigns these to the application: batch limits, expand limits ("CAP applications have to limit the amount of $expands per request in a custom handler" for Node.js), and "Applications need to establish an adequate rate limiting strategy." Unbounded batch/expand turns one request into thousands of operations. **Medium:** availability/resource impact, not confidentiality or integrity.

### Evidence expected in code
Java: the two properties set in `application.yaml`. Node.js: an expand-guard in a `before` handler (or documented platform control). Either: pagination limits configured or the default cap intentionally kept; a documented rate-limiting decision (route service, gateway, or app middleware).

### Detection guidance
1. Java: check `application*.yaml` for `cds.odataV4.batch.maxRequests` and `cds.query.restrictions.expand.maxLevels`; absent → FAIL (undecided), present → record values.
2. Node.js: search `srv/` for a `$expand`-depth/count guard in `before` handlers; check docs for the platform-level alternative; neither → FAIL.
3. Check pagination: `@cds.query.limit` settings or confirmation the default cap is intentionally kept (no `limit: 0` disabling).
4. Look for the rate-limiting decision (mta route-service binding, gateway config, middleware, or ADR); none → FAIL (undecided).
5. Report as "undecided" vs "decided with values" — this rule enforces the decision, not specific numbers.

### Good example
```yaml
# application.yaml (Java)
cds:
  odataV4.batch.maxRequests: 100
  query.restrictions.expand.maxLevels: 3
```

### Bad example
```text
No batch or expand limits anywhere; pagination disabled with
@cds.query.limit: 0 on a mass-data entity; no rate-limiting decision
recorded — a single crafted $batch with deep $expands DoSes the app.
```

### Exception guidance
Internal-only services behind an already-rate-limited gateway may document the platform control instead of app-level limits. The pagination default may be raised per entity for legitimate bulk APIs — recorded with the reason.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/data-protection (batch/expand/pagination responsibilities; rate limiting)

---

## CAP-SEC-015 — Backends enforce authentication independently of the App Router

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-015 |
| **Title** | Backends enforce authentication independently of the App Router |
| **Category** | Security |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | All deployments fronted by SAP Application Router or any gateway/proxy |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-003, CAP-SEC-005 |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP backend services MUST enforce authentication themselves, never relying on the App Router or any fronting proxy as the security boundary. Every deployment pipeline MUST include a verification that unauthenticated requests to backend endpoints are rejected (SAP: add authentication tests that the deployed application rejects unauthenticated requests).

### Rationale
SAP is explicit: "Application Router as a frontend proxy does not shield the backend from incoming traffic. Therefore, you must secure the backend independently", and "Without security middleware configured, CDS services are exposed to public." Backend routes on CF/Kyma are directly reachable. **Critical justification:** treating the router as the boundary leaves backends publicly accessible — full unauthorized access.

### Evidence expected in code
Backend auth configured per CAP-SEC-002/-005; a post-deploy smoke test or integration test asserting 401/403 on direct unauthenticated backend calls (pipeline step or test file).

### Detection guidance
1. Confirm backend authentication configuration exists independent of approuter config (CAP-SEC-002/-005 detection).
2. Search the pipeline (`.github/workflows/`, Piper config) and test suites for an unauthenticated-rejection check against the backend URL (not the approuter URL).
3. No such verification anywhere → FAIL (missing mandated test), even if configuration looks correct.
4. Inspect `xs-app.json` for routes with `authenticationType: 'none'` forwarding to CAP endpoints — flag each for justification.
5. Report evidence with file:line; runtime-only aspects (actual deployed behavior) → NOT ASSESSABLE if no pipeline evidence exists, naming the needed check.

### Good example
```yaml
# .github/workflows/deploy.yml — post-deploy smoke test
- name: verify backend rejects unauthenticated requests
  run: |
    code=$(curl -s -o /dev/null -w '%{http_code}' "$SRV_URL/odata/v4/orders/Orders")
    test "$code" = "401" || test "$code" = "403"
```

### Bad example
```text
xs-app.json protects UI routes, but the srv app's own CF route is public
and serves /odata/v4/orders without a token — "the approuter handles auth."
```

### Exception guidance
None. Even fully internal services keep backend authentication (see also CAP-SEC-005 exception stance).

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authentication ("does not shield the backend"; secure independently; add authentication tests)

---

## CAP-SEC-016 — Keep security-sensitive data out of logs

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-016 |
| **Title** | Keep security-sensitive data out of logs |
| **Category** | Security |
| **Severity** | Medium |
| **Authority** | SAP-REC (Java escaping phrased as an application need) |
| **Applicability** | All projects; every log statement handling user input, credentials, tokens, or personal data |
| **Runtime** | Both (mechanics differ — see statement) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-012, CAP-SEC-017; future CAP-LOG rules will cross-reference |
| **Last verified** | 2026-08-11 |

### Rule statement
Logs MUST NOT contain secrets, tokens, credentials, or unnecessary personal data; production log level stays at the INFO default. Node.js code MUST log through the CAP logging API (`cds.log`, CRLF-safe per SAP) — not `console.log`. Java code MUST escape user-controlled data before logging (SAP: applications "need to care for escaping user data that is used as input parameter for application logging"). Framework defaults that mask sensitive headers/secrets MUST NOT be disabled.

### Rationale
SAP's data-protection guide names log injection and information disclosure as application responsibilities, with runtime-specific mechanics: a CRLF-safe API in Node.js, manual escaping in Java, and INFO default level to avoid disclosure. **Medium:** exposure/forgery through logs is real but indirect — it requires log access or downstream processing to exploit.

### Evidence expected in code
`cds.log('<component>')` usage (no `console.*` in srv code); Java log calls with escaped/parameterized user input; no logging of `req.headers.authorization`, tokens, passwords, or full payload dumps of personal data; masking config untouched.

### Detection guidance
1. Node.js: search `srv/**` for `console.log|error|warn` → flag each; verify `cds.log` is the logging path.
2. Both: search log statements for tainted values (`req.data`, `req.headers`, exception objects carrying credentials) logged raw; flag secrets/token/authorization-header logging → FAIL.
3. Java: for each log call embedding user input, check for escaping (e.g., OWASP encoder) or structured parameterization with sanitization; raw concatenation of user input → FAIL.
4. Check config: log level overrides to DEBUG/TRACE in production profiles; disabled header masking (`cds.log.mask_headers` emptied) → FAIL.
5. Report per statement with file:line.

### Good example
```js
const LOG = cds.log('orders');
LOG.info('order created', { orderID: order.ID });   // IDs, not payload dumps
```

### Bad example
```js
console.log(`login for ${req.data.user}: token=${req.headers.authorization}`);
// secret in the log, unsanitized user input, bypasses cds.log masking
```

### Exception guidance
Diagnostic logging of business identifiers is fine. Temporarily elevated log levels for incident analysis are acceptable when time-bound and reverted — not committed as defaults. Audit-relevant personal-data access belongs in the audit log (CAP-PRIV rules, future batch), not application logs.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/data-protection (log injection; escaping in Java; CRLF-safe Node API; INFO default)
- https://cap.cloud.sap/docs/node.js/cds-log (masking, structured logging)

---

## CAP-SEC-017 — No secrets in the repository or development artifacts

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-017 |
| **Title** | No secrets in the repository or development artifacts |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | All projects, all lifecycle stages |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-015; future CAP-INT destination rule cross-references (no credentials in destination config) |
| **Last verified** | 2026-08-11 |

### Rule statement
Credentials, service keys, tokens, and certificates MUST NOT be committed to the repository or materialized into local files: hybrid access to cloud services uses `cds bind` (which stores only pointers in `.cdsrc-private.json`, resolving credentials on demand); `default-env.json`/`.env` files with credentials MUST NOT be committed; application-defined destination configuration MUST NOT embed credentials (provide programmatically/via env). Local development servers bind to localhost, and production data is not used in development/tests.

### Rationale
SAP's security overview instructs using `cds bind` instead of copying credentials into `default-env.json`, binding local servers to localhost only, and never testing with production data; the consuming-services guide warns against sensitive credentials in destination configuration. Committed secrets outlive their commit (forks, clones, history). **High:** leaked credentials are directly abusable, but exploitation requires repository/file access — one step removed from an open endpoint.

### Implementation guidance
- `.gitignore` must cover `default-env.json`, `.env*`, `.cdsrc-private.json`, service-key exports.
- For CI, resolve bindings at runtime (`cds bind --exec`, `cds env get requires --resolve-bindings`) rather than storing credentials in the repo.

### Evidence expected in code
Clean repository history scan; `.gitignore` entries for credential files; `cds bind`-based hybrid setup; destinations without embedded credentials.

### Detection guidance
1. Search the working tree for `default-env.json`, `.env*`, `*-key.json`, `VCAP_SERVICES` dumps, and inline credential patterns (`"clientsecret"`, `"password"`, private key blocks) in config files including `mta.yaml`/`.mtaext`/Helm values → any hit with real-looking values → FAIL with file:line.
2. Check `.gitignore` covers the credential file patterns; missing → FAIL (prevention control absent).
3. Inspect destination definitions (package.json `cds.requires`, Java config): embedded `clientsecret`/password → FAIL.
4. Optionally scan git history for previously committed secrets (report as observation with remediation note — rotation, not just deletion).
5. Check test fixtures/CSVs for production-looking personal data (observation → escalate to CAP-PRIV review).

### Good example
```text
.gitignore:            default-env.json, .env*, .cdsrc-private.json
hybrid setup:          cds bind -2 bookshop-db   (pointer stored, no secret)
CI:                    cds bind --exec -- npm run test:integration
```

### Bad example
```jsonc
// committed default-env.json with a live HANA service key
{ "VCAP_SERVICES": { "hana": [ { "credentials": {
    "user": "SBSS_86…", "password": "Aa3…", "certificate": "-----BEGIN…" } } ] } }
```

### Exception guidance
Placeholder/dummy values in committed templates (`.env.example`) are fine when obviously non-functional. No exception for real credentials, including "test-only" cloud service keys.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/overview (use `cds bind`, don't copy secrets to default-env.json; localhost; no production data)
- https://cap.cloud.sap/docs/tools/cds-bind (pointer-only binding storage)
- https://cap.cloud.sap/docs/guides/services/consuming-services (no sensitive credentials in destination configuration)

---

## CAP-SEC-018 — Govern MCP exposure of CAP services

| Field | Value |
|---|---|
| **Rule ID** | CAP-SEC-018 |
| **Title** | Govern MCP exposure of CAP services |
| **Category** | Security |
| **Severity** | High |
| **Authority** | SAP-REC (documented beta status, explicit limitation warnings, and SAP API Policy statement; production-control specifics are ORG — gap G-41) |
| **Applicability** | Projects annotating services with `@mcp` (MCP protocol adapter) or attaching AI/MCP tooling |
| **Runtime** | Both (Java adapter availability lagging Node.js per current docs) |
| **CAP version** | MCP protocol adapter is **beta** as of cds 10 (June 2026); Java config property renamed `cds.mcp.autoConfig` → `cds.mcp.autoWired` at cds-java 5.0 — re-verify at each release |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SEC-002, CAP-SEC-011, CAP-SEC-014 |
| **Last verified** | 2026-08-11 |

### Rule statement
Services exposed via the MCP protocol adapter (`@mcp`) MUST be treated as a governed, beta exposure surface:
1. The full authorization model of CAP-SEC-001/-010/-011 applies to MCP-exposed services; because the MCP guide does not itself guarantee enforcement semantics, effective authentication/authorization on MCP endpoints MUST be explicitly verified by test.
2. Dev-mode auto-injected mock credentials (mock user `alice` in Node.js / `privileged` in Java) MUST NOT be reachable outside development profiles (this is CAP-SEC-002 applied to MCP).
3. MCP MUST NOT be used as a gateway/proxy for SAP Application APIs (SAP: "must not be used as a gateway or proxy for SAP Application APIs"; "not an SAP-endorsed architecture" for that purpose).
4. Production exposure additionally requires ORG-approved compensating controls for the protections SAP documents as absent — prompt-injection validation, rate limiting, audit logging, approval workflows (gap G-41).

### Rationale
SAP's MCP guide (beta) is explicit about missing safeguards: "The MCP adapter does not perform any input validation or output validation to prevent prompt injection attacks", no automatic rate limiting, no agent-action audit logging, no approval workflows or policy enforcement. MCP clients are LLM-driven — an unguarded MCP endpoint combines a powerful programmatic query surface (`query` tool) with a caller that can be manipulated by content. **High:** CAP authorization still gates data access (when verified per point 1); the missing layers are amplification and abuse controls rather than an unconditional bypass.

### Implementation guidance
- Prefer exposing narrow, read-only, use-case-specific services via `@mcp` — never broad admin services.
- Write an authorization test that calls the MCP endpoint (`query` tool) as an unauthorized user and asserts denial.
- Keep `cds.mcp.autowire`/`autoWired` client auto-configuration out of production configuration.

### Evidence expected in code
`@mcp` annotations only on reviewed services; MCP-specific authorization tests; no SAP Application API proxying through MCP-exposed services; an ORG approval record for any production MCP exposure (G-41).

### Detection guidance
1. Search all `*.cds` for `@mcp` annotations; none → NOT APPLICABLE.
2. For each `@mcp` service: verify explicit authorization per CAP-SEC-001 detection, and locate an MCP-endpoint authorization test (denied case); missing test → FAIL point 1.
3. Check profiles/config: MCP client auto-configuration and mock credentials confined to development; reachable in production config → FAIL point 2 (Critical escalation via CAP-SEC-002).
4. Check whether the exposed service consumes/proxies imported SAP Application APIs (`srv/external` models re-exposed) → FAIL point 3.
5. For production deployments exposing MCP: locate the ORG approval + compensating-controls record (G-41); absent → FAIL point 4.
6. Report per service with file:line references.

### Good example
```cds
// Narrow read-only service, explicitly authorized, MCP-exposed
@requires: 'CatalogViewer'
@mcp
service CatalogFacts {
  @readonly entity Books as projection on my.Books { ID, title, genre };
}
```

### Bad example
```cds
// Broad admin service with imported S/4 API re-exposed over MCP,
// no MCP-specific tests, autowired mock credentials in default profile
@mcp
service AdminService {
  entity Orders as projection on my.Orders;
  entity BusinessPartners as projection on external.API_BUSINESS_PARTNER.A_BusinessPartner;
}
```

### Exception guidance
Local-development-only usage (never deployed) needs no ORG approval — verified by absence of MCP config in production profiles. No exception to the SAP Application API prohibition.

### SAP reference
- https://cap.cloud.sap/docs/guides/protocols/mcp (beta; documented missing protections; dev mock credentials; "must not be used as a gateway or proxy for SAP Application APIs")

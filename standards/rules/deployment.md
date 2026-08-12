# CAP-DEP — Deployment (Cloud Foundry, Kyma/Kubernetes)

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md).

**Rules:** 3 active (0 Critical, 0 High, 3 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries — deployment *content* requirements live with their owners and are verified there: production identity binding ([CAP-SEC-002/-005](security.md)), no dev databases in production ([CAP-DB-001/-002](data-persistence.md)), secrets/credentials ([CAP-SEC-017](security.md)), MT sidecar and tenant upgrades ([CAP-MT-002/-005](multitenancy.md)), health probes ([CAP-OPS-001](production-readiness.md)), pipelines ([CAP-CICD](cicd.md)), dependency freezing ([CAP-VER-001](versions-dependencies.md)). These rules govern the deployment *mechanism and structure*. No zero-downtime/blue-green rules are authored — SAP's deployment pages document none (verified 2026-08-12); such strategies are ORG territory (G-39).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-DEP-001 | Deploy to Cloud Foundry as an MTA built from production facets | Medium | SAP-REC | Both |
| CAP-DEP-002 | Keep landscape-specific settings in MTA extension descriptors | Medium | SAP-REC | Both |
| CAP-DEP-003 | Deploy to Kyma with the generated Helm chart and buildpack images | Medium | SAP-REC | Both |

---

## CAP-DEP-001 — Deploy to Cloud Foundry as an MTA built from production facets

| Field | Value |
|---|---|
| **Rule ID** | CAP-DEP-001 |
| **Title** | Deploy to Cloud Foundry as an MTA built from production facets |
| **Category** | Deployment |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented deployment path; the facet *content* — HANA, identity — is SAP-REQ and owned by CAP-DB-001/CAP-SEC-002) |
| **Applicability** | Projects deploying to Cloud Foundry; NOT APPLICABLE for Kyma-only targets (CAP-DEP-003) |
| **Runtime** | Both |
| **CAP version** | Current tooling (`cds up`, `cds add mta`, MBT + CF CLI with multiapps plugin) per current docs |
| **Status** | Active |
| **Related rules** | CAP-SEC-002/-005 (identity binding content), CAP-DB-001/-002 (database content), CAP-VER-001 (frozen dependencies), CAP-CICD-001 (pipeline executes this), CAP-OPS-002 (UI entry point) |
| **Last verified** | 2026-08-12 |

### Rule statement
Cloud Foundry deployments MUST be Multitarget Applications generated from the documented production facets — `cds add hana`, `cds add xsuaa` (or the IAS path), `cds add mta`, plus a UI option where applicable (`approuter`, `workzone`, `portal` for multitenant, or `app-frontend`) — and built/deployed via the documented toolchain: `cds up` (which automates `mbt build -t gen --mtar mta.tar` + `cf deploy`) or the equivalent manual `mbt build` + `cf deploy`. Hand-maintained `cf push` of individually assembled artifacts, bypassing the MTA descriptor, MUST NOT be the production path.

### Rationale
The MTA descriptor is where CAP's production wiring lives: service instances (HANA/HDI, identity), bindings, module relationships, sidecar modules (CAP-MT-002), and health-check configuration (CAP-OPS-001) — the facets generate it consistently and `cds add` keeps it maintainable. Ad-hoc `cf push` paths lose the service-binding orchestration and drift from what the facets would regenerate, producing deployments whose configuration exists only in someone's shell history. **Medium:** mechanism/maintainability — the dangerous *content* failures (mock auth, SQLite in prod) are Critical/High in their owning rules.

### Implementation guidance
- Scaffold with the facets rather than hand-writing `mta.yaml`; re-run `cds add` after structural changes (it merges rather than overwrites).
- Inspect the production assembly with `cds build --production` before first deployment.
- The pipeline (CAP-CICD-001) is what runs the build/deploy — not developer machines.

### Evidence expected in code
`mta.yaml` with modules for srv/db(-deployer)/UI and resources for HANA + identity; facet-generated structure recognizable; deployment scripts/pipeline invoking `cds up` or `mbt build`+`cf deploy`.

### Detection guidance
1. Confirm CF is the target (deployment scripts, pipeline, `mta.yaml` presence) → else NOT APPLICABLE.
2. `mta.yaml` present with the expected modules/resources (srv module, DB deployer, identity resource, UI module where a UI exists) → structural PASS; missing → FAIL (name the facet that generates it).
3. Search deployment scripts/pipeline for raw `cf push` as the production path (not MTA-mediated) → FAIL with location.
4. Cross-check resource *content* under the owning rules (identity: CAP-SEC-002/-005; database: CAP-DB-001) — do not re-report here.
5. Report with file:line into `mta.yaml`/scripts.

### Good example
```yaml
# mta.yaml (facet-generated) — modules + services wired declaratively
modules:
  - name: bookshop-srv
    type: nodejs
    requires: [ bookshop-db, bookshop-auth ]
  - name: bookshop-db-deployer
    type: hdb
    requires: [ bookshop-db ]
resources:
  - name: bookshop-db
    type: com.sap.xs.hdi-container
  - name: bookshop-auth
    type: org.cloudfoundry.managed-service
    parameters: { service: xsuaa, service-plan: application }
```

### Bad example
```bash
# "deployment" — hand-pushed artifact, bindings created ad hoc in the shell,
# no descriptor, unreproducible next month
npm run build && cf push bookshop-srv -p gen/srv
cf bind-service bookshop-srv some-hana-instance
```

### Exception guidance
Non-MTA deployment mechanisms mandated by an organization's platform team (e.g., org-standard manifests) are an ADR-documented deviation (CAP-ARCH-007) — the binding completeness then gets verified manually against the owning rules. Scratch/dev spaces may `cf push` freely.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/to-cf (production facets incl. UI options; `cds up` automating `mbt build` + `cf deploy`)

---

## CAP-DEP-002 — Keep landscape-specific settings in MTA extension descriptors

| Field | Value |
|---|---|
| **Rule ID** | CAP-DEP-002 |
| **Title** | Keep landscape-specific settings in MTA extension descriptors |
| **Category** | Deployment |
| **Severity** | Medium |
| **Authority** | SAP-REC ("This allows you to keep landscape-specific deployment settings outside your base mta.yaml") |
| **Applicability** | CF/MTA deployments targeting more than one landscape/environment |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DEP-001, CAP-ARCH-005 (config-not-code for environment differences), CAP-SEC-017 (no credentials in the descriptors either) |
| **Last verified** | 2026-08-12 |

### Rule statement
Landscape-/environment-specific deployment settings — scaling parameters, routes, landscape endpoints — MUST live in MTA extension descriptors (`.mtaext`, applied via `cds up --overlay <file>.mtaext` or `cf deploy -e`), not in per-environment copies or hand-edits of the base `mta.yaml`. One base descriptor, one overlay per landscape. Credentials do not belong in either (CAP-SEC-017).

### Rationale
Per-environment `mta.yaml` forks are the deployment version of copy-paste inheritance: every structural change must be replayed across copies, and drift between them becomes invisible until a landscape breaks. The documented overlay mechanism keeps the structure single-sourced and the per-landscape delta explicit and reviewable — the deployment-artifact instance of CAP-ARCH-005's config-not-code principle. **Medium:** maintainability/drift risk.

### Evidence expected in code
A single `mta.yaml`; `.mtaext` files per landscape (e.g., `.deploy/eu10-prod.mtaext`); deploy invocations applying the overlay.

### Detection guidance
1. Confirm multiple landscapes exist (pipeline stages, deploy docs) → single-landscape projects → NOT APPLICABLE.
2. Look for environment forks: multiple `mta*.yaml` variants, environment-named copies, or pipeline steps `sed`-editing the descriptor → each → FAIL with location.
3. Verify overlays: `.mtaext` files present and referenced in deploy commands (`--overlay`/`-e`) → compliant pattern.
4. Scan `.mtaext` contents for credentials → report under CAP-SEC-017.
5. Report per landscape.

### Good example
```text
mta.yaml                      ← single base descriptor
.deploy/eu10-prod.mtaext      ← scaling/routes for prod
.deploy/eu10-test.mtaext
deploy:  cds up --overlay .deploy/eu10-prod.mtaext
```

### Bad example
```text
mta.yaml, mta-test.yaml, mta-prod.yaml — three diverging copies;
prod got the new sidecar module, test silently didn't.
```

### Exception guidance
Fully identical landscapes (no per-environment delta) need no overlays — the absence is compliant. Helm-based targets use `values`-file layering instead (CAP-DEP-003's mechanism).

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/to-cf (`cds up --overlay …mtaext`; keep landscape-specific settings outside the base descriptor)

---

## CAP-DEP-003 — Deploy to Kyma with the generated Helm chart and buildpack images

| Field | Value |
|---|---|
| **Rule ID** | CAP-DEP-003 |
| **Title** | Deploy to Kyma with the generated Helm chart and buildpack images |
| **Category** | Deployment |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented chart/buildpack path); the pull-secret clause carries imperative documented wording ("use a technical user with read-only permissions") |
| **Applicability** | Projects deploying to Kyma/Kubernetes; NOT APPLICABLE for CF-only targets |
| **Runtime** | Both |
| **CAP version** | Current `cds add kyma` chart generation per current docs |
| **Status** | Active |
| **Related rules** | CAP-DEP-001 (CF counterpart), CAP-SEC-017 (registry credentials handling), CAP-OPS-001 (probes wired by the kyma facet), CAP-MT-002 (sidecar module in the chart) |
| **Last verified** | 2026-08-12 |

### Rule statement
Kyma/Kubernetes deployments MUST use the CAP-generated Helm chart (`cds add kyma`, deployed via `cds up -2 k8s`) with container images built by Cloud Native Buildpacks (`pack` — documented properties: reproducible builds, unprivileged user, SBoM baked in). Chart customization MUST follow the documented safe paths — edit `chart/values.yaml`/`Chart.yaml` (preserved on regeneration: "will not be modified. Only new or missing properties will be added"), copy templates for overrides, or use Kustomize as a post-processor — never edits inside `gen/chart`. Private-registry image pull secrets MUST use "a technical user with read-only permissions" (documented imperative — anyone with cluster access can read the secret).

### Rationale
The generated chart carries CAP's Kubernetes wiring (bindings for XSUAA/HANA/messaging, probes per CAP-OPS-001, sidecar modules per CAP-MT-002) and stays regenerable; hand-rolled manifests or edits in generated folders are overwritten or drift. Buildpack images provide the documented supply-chain properties (reproducibility, unprivileged user, SBoM) that hand-written Dockerfiles must re-earn. The pull-secret clause bounds the blast radius of a leaked secret to read-only. **Medium:** mechanism/maintainability, with the pull-secret clause carrying the security-hygiene edge (cross-report to CAP-SEC-017 when violated with write-capable credentials).

### Evidence expected in code
`chart/` from `cds add kyma` with customizations in `values.yaml` (or copied templates/Kustomize); image build via `pack`/buildpacks in the pipeline; pull-secret provisioning documented with a read-only technical user.

### Detection guidance
1. Confirm Kyma/K8s target → else NOT APPLICABLE.
2. Chart present and facet-generated (structure matches `cds add kyma` output); hand-rolled manifests replacing it → FAIL (name the facet path) unless ADR-documented (CAP-ARCH-007).
3. Check for edits inside `gen/chart` committed as customization → FAIL (overwritten on rebuild; use values/copied templates/Kustomize).
4. Verify image build uses buildpacks (`pack` in pipeline/scripts); custom Dockerfiles → observation/FAIL depending on documented reason (supply-chain properties must be re-established).
5. Pull secret: check provisioning docs/scripts for the registry user's permissions → write-capable user in the pull secret → FAIL (documented read-only requirement; cross-report CAP-SEC-017).
6. Report with file locations.

### Good example
```text
cds add kyma                  → chart/ (values.yaml customized, survives updates)
pack build acme/bookshop-srv  → reproducible, unprivileged, SBoM included
registry pull secret          → user "registry-ro" (read-only) per runbook §2
cds up -2 k8s -n bookshop
```

### Bad example
```text
Hand-written deployment.yaml copied from a blog; image built from an
ad-hoc Dockerfile running as root; pull secret uses the registry admin
account — cluster readers can now push images.
```

### Exception guidance
Org-mandated deployment tooling (fleet GitOps, mandated base images) is an ADR-documented deviation; the chart's binding/probe content must then be replicated and verified manually. Kustomize post-processing is a documented compliant path, not an exception.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/to-kyma (chart generation; `cds up -2 k8s`; buildpack properties; values.yaml preservation; Kustomize; read-only technical user for pull secrets)

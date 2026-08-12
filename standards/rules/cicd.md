# CAP-CICD — CI/CD

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-39 (pipeline standards: branch policies, scanning depth, blue-green/canary).

**Rules:** 3 active (0 Critical, 1 High, 2 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries: no CI/CD *product* is prescribed — SAP documents GitHub Actions scaffolding as the worked example with SAP Continuous Integration and Delivery and Project Piper as documented alternatives; the rules bind capabilities, not tools. Pipeline *credentials* hygiene is [CAP-SEC-017](security.md); hybrid test execution in CI is [CAP-TEST-006](testing.md) (absorbed candidate CAP-CICD #3); frozen dependencies are [CAP-VER-001](versions-dependencies.md); tenant-upgrade sequencing is [CAP-MT-005](multitenancy.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-CICD-001 | Build, test, and deploy through an automated pipeline | High | SAP-REC | Both |
| CAP-CICD-002 | Gate production deployment behind a protected, release-triggered stage | Medium | SAP-REC | Both |
| CAP-CICD-003 | Enforce the standard's automatable checks as pipeline gates | Medium | ORG | Both |

---

## CAP-CICD-001 — Build, test, and deploy through an automated pipeline

| Field | Value |
|---|---|
| **Rule ID** | CAP-CICD-001 |
| **Title** | Build, test, and deploy through an automated pipeline |
| **Category** | CI/CD |
| **Severity** | High |
| **Authority** | SAP-REC (documented scaffolding and alternatives; no product mandated) |
| **Applicability** | All projects with deployment targets |
| **Runtime** | Both |
| **CAP version** | Current scaffolding: `cds add github-actions` (alias `gha`) per current docs |
| **Status** | Active |
| **Related rules** | CAP-DEP-001/-003 (what the pipeline executes), CAP-VER-001 (`npm ci` from the frozen tree), CAP-CICD-002/-003, CAP-SEC-017 (pipeline secrets) |
| **Last verified** | 2026-08-12 |

### Rule statement
Every deployable project MUST build, test, and deploy through an automated pipeline — production artifacts MUST NOT be assembled or deployed from developer machines. The documented starting points are preferred over hand-rolled pipelines: `cds add github-actions` (generates workflows that "test, deploy and release new versions", with staging deployment on `main`), the SAP Continuous Integration and Delivery service (ready-made CAP pipeline), or Project Piper (flexible, incl. on-premise). Pipeline platform credentials follow the documented pattern (CF variables + `CF_PASSWORD`/`KUBE_CONFIG` as secrets — hygiene per CAP-SEC-017).

### Rationale
The pipeline is what makes every other gate in this standard *enforceable*: reproducible builds from the frozen tree (CAP-VER-001), tests that actually ran (CAP-TEST), deployments that match reviewed configuration (CAP-DEP). Laptop deployments produce production artifacts nobody can reproduce, from working trees nobody reviewed. SAP documents the scaffolds precisely so teams don't hand-roll the plumbing. **High:** unpipelined production deployment materially affects production reliability and traceability — every artifact is a one-off.

### Implementation guidance
- Scaffold with `cds add github-actions` and adapt, or adopt SAP CI/CD service / Piper per landscape — the rule binds the capability (automated build+test+deploy), not the product.
- The pipeline runs `npm ci` (Node) / locked Maven builds and executes the full default test suite before any deploy stage.

### Evidence expected in code
Pipeline definitions in the repository (`.github/workflows/`, SAP CI/CD config, Piper config) covering build + test + deploy; deployment credentials as platform secrets; no documented laptop-deploy path for production.

### Detection guidance
1. Locate pipeline definitions → none while deployment targets exist → FAIL (High).
2. Verify stages: build from the frozen dependency tree, full test execution, deployment via the CAP-DEP-001/-003 mechanisms → deploy stage bypassing tests → FAIL (also CAP-CICD-003).
3. Check credentials handling: platform secret store (Actions secrets, service credentials), not committed values → violations → report under CAP-SEC-017.
4. Look for production deploy runbooks instructing manual `cf deploy`/`cds up` from workstations → FAIL (name the pipeline path).
5. Report with file locations.

### Good example
```yaml
# .github/workflows/main.yml (scaffolded by cds add github-actions, adapted)
jobs:
  test:   { steps: [ npm ci, npm test ] }
  deploy: { needs: test, environment: staging, steps: [ cds up … ] }
```

### Bad example
```text
"Deployment process": senior dev runs `mbt build && cf deploy` from
their laptop when QA says ok. CI exists but only lints.
```

### Exception guidance
Emergency break-glass deployment procedures may exist — documented, audited, and exceptional (their use is an incident artifact, not a routine). Non-deployable libraries need build+test only.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/cicd (`cds add github-actions`; generated test/deploy/release workflows; CF/Kyma credential pattern; SAP CI/CD service and Project Piper as documented alternatives)

---

## CAP-CICD-002 — Gate production deployment behind a protected, release-triggered stage

| Field | Value |
|---|---|
| **Rule ID** | CAP-CICD-002 |
| **Title** | Gate production deployment behind a protected, release-triggered stage |
| **Category** | CI/CD |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented pattern: protected "Production" environment; deployment triggered by tagged releases, e.g. `v1.0.0`) |
| **Applicability** | Projects with a production stage in their pipeline |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CICD-001/-003, CAP-DEP-002 (per-landscape overlays feed the stages), CAP-MT-005 (tenant upgrade rides the production stage) |
| **Last verified** | 2026-08-12 |

### Rule statement
Production deployment MUST be a distinct, protected pipeline stage — separated from staging, guarded by an environment-protection mechanism (SAP's documented pattern: a protected "Production" environment), and triggered deliberately (documented pattern: publishing a tagged release such as `v1.0.0`) — never as an automatic side effect of every merge to `main` unless that policy is explicitly recorded. The production stage MUST identify what it deploys (immutable tag/artifact) so the deployed version is always attributable, and the rollback approach (redeploy of a previous tag, at minimum) MUST be documented.

### Rationale
Staging-on-merge plus release-triggered production is the documented flow the scaffolds generate: it makes production deployment a decision with an audit trail (the release), a protection point (environment approval), and an addressable artifact (the tag) — which is also the minimal rollback story. Continuous production deployment is a legitimate alternative policy, but only as an explicit, recorded choice with equivalent attribution. **Medium:** process/traceability quality; the content it deploys is guarded by the other rules.

### Evidence expected in code
Pipeline with separated staging/production stages; environment protection on production; release/tag trigger (or a recorded continuous-deployment ADR); rollback procedure documented (runbook).

### Detection guidance
1. Inspect pipeline definitions for the production path: distinct stage/workflow → merged into the per-push flow with no recorded policy → FAIL.
2. Verify protection: environment protection rules / approval requirement on the production stage → absent → FAIL.
3. Verify trigger + attribution: tagged-release trigger (or recorded alternative) and immutable artifact identification → untagged "deploy latest main" → FAIL.
4. Locate the rollback documentation (previous-tag redeploy or better) → none → FAIL element (note: DB schema rollback is *not* claimed — see exception guidance).
5. Report with file locations.

### Good example
```text
push → staging deploy (auto)          release v1.4.2 published →
Production environment (2 approvers) → deploy tag v1.4.2
runbook: rollback = re-release previous tag (schema-compatible window
per CAP-DB-007 migrations)
```

### Bad example
```text
Every push to main deploys straight to production; no environment
protection, no tags — "what's in prod?" is answered by timestamps
and hope.
```

### Exception guidance
Recorded continuous-deployment policies (with equivalent artifact attribution and protection) are compliant alternatives. **No rollback guarantees are invented for database schemas** — schema changes roll *forward* (journal-based evolution per CAP-DB-007); the documented rollback scope is the application artifact.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/cicd (protected "Production" environment; tagged-release deployment flow)

---

## CAP-CICD-003 — Enforce the standard's automatable checks as pipeline gates

| Field | Value |
|---|---|
| **Rule ID** | CAP-CICD-003 |
| **Title** | Enforce the standard's automatable checks as pipeline gates |
| **Category** | CI/CD |
| **Severity** | Medium |
| **Authority** | ORG (our policy for operationalizing this standard; SAP prescribes no gate set — gap G-39 holds the open specifics: scanning depth, branch policies, promotion strategies) |
| **Applicability** | All pipelines (CAP-CICD-001) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-CICD-001/-002; gates enforce (at minimum): CAP-TEST-001/-002/-007 (suite incl. security tests), CAP-VER-001 (lockfile-exact builds), CAP-SEC-015/-005 (deployed unauthenticated-rejection check), CAP-SEC-017 (secret scanning); the enforceability classification below feeds Phase 3 |
| **Last verified** | 2026-08-12 (ORG policy — no SAP claim to verify) |

### Rule statement
Pipelines MUST enforce, as blocking gates, the standard's automatable checks — at minimum: (1) build from the frozen dependency tree (`npm ci`/locked Maven — CAP-VER-001); (2) the full default test suite including the security-behavior tests (CAP-TEST-007) — a failing suite blocks every deploy stage; (3) secret scanning on the repository (CAP-SEC-017's prevention control); (4) the post-deploy unauthenticated-rejection smoke check (CAP-SEC-015). Gate failures MUST block promotion — no manual "deploy anyway" path outside the documented break-glass procedure. The standard's rules classify for enforcement as: **automatically enforceable** (the gates above; lint-style config checks), **manually reviewable** (design/model rules — the review model's territory), **deployment-time validation** (binding/descriptor completeness), and **operational verification** (runtime behavior, hybrid checks) — pipelines automate the first class and MUST NOT claim to cover the others (Phase 3 maps rules to classes explicitly).

### Rationale
A pipeline that runs checks but deploys on failure is documentation, not enforcement. Conversely, a pipeline that claims "standard compliance" from automated checks alone overstates — most of this catalog needs review-time judgment. The four-class distinction keeps both errors out: automated gates block what they can actually verify; the rest is routed to the review gates of the [lifecycle](../../development/lifecycle.md). **Medium:** process integrity; the individual gate subjects carry their own severities. **ORG authority:** SAP documents pipeline mechanics, not a gate policy — this is our operationalization decision (the remaining specifics stay in G-39).

### Evidence expected in code
Pipeline stages wired as blocking (deploy `needs` test; branch protection requiring checks); secret-scanning step/config; post-deploy smoke check; no bypass flags in the workflow.

### Detection guidance
1. Verify test→deploy dependency: deploy stages require the test job (workflow `needs`, or platform equivalents) → deploy runs regardless → FAIL.
2. Check for bypass mechanisms (`continue-on-error` on gate jobs, `[skip ci]` culture on release branches, manual dispatch skipping tests) → each → FAIL unless it is the documented break-glass path.
3. Verify the gate content: `npm ci` (not floating install), full suite (not a smoke subset) incl. the CAP-TEST-007 security tests, secret scanning present, post-deploy 401/403 check (CAP-SEC-015's evidence) → missing element → FAIL per element.
4. Check the pipeline does not *claim* full standard compliance from automation (badges/docs asserting "standard-compliant" solely via CI) → misclaim → observation (AI-DOC-004 discipline).
5. Report per gate with file locations.

### Good example
```yaml
jobs:
  test:    { steps: [ npm ci, npm test ] }              # full suite incl. authz tests
  scan:    { steps: [ gitleaks detect ] }               # CAP-SEC-017 prevention
  deploy:  { needs: [test, scan], environment: staging, steps: [ cds up … ] }
  verify:  { needs: deploy, steps: [ ./smoke-unauthenticated-401.sh ] }  # CAP-SEC-015
```

### Bad example
```yaml
jobs:
  test:   { continue-on-error: true }    # "gates"
  deploy: { steps: [ cds up ] }          # runs unconditionally
```

### Exception guidance
Quarantining a *specific known-flaky test* with an issue link is acceptable; disabling the suite gate is not. Projects on SAP CI/CD service/Piper satisfy the rule through those tools' equivalent gate configuration — the review checks the configuration, not the product.

### SAP reference
None normative (authority: ORG; open specifics in gap G-39). Related SAP reading: https://cap.cloud.sap/docs/guides/deploy/cicd (pipeline mechanics the gates run on).

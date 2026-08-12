# CAP-VER — Dependency & version management

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). The live version baseline these rules bind against is [docs/version-management.md](../../docs/version-management.md) (**verified 2026-08-12**: June 2026 wave current; the August 2026 minor is listed but unreleased) — rules reference the baseline document rather than hardcoding numbers, so the register is updated on SAP releases, not the rules. Related ORG gap: G-40 (dependency-update SLA).

**Rules:** 6 active (0 Critical, 4 High, 2 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundary: the **cds 8→10 queue-drain requirement (candidate CAP-VER #9) is owned by [CAP-EVT-002](events-messaging.md)**'s version note — CAP-VER-005 references release-note-driven upgrade steps generically instead of duplicating it.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-VER-001 | Freeze dependencies for deployment — and refresh them regularly | High | SAP-REQ | Node.js |
| CAP-VER-002 | Stay on an Active CAP release line | High | SAP-REQ | Both |
| CAP-VER-003 | Run supported runtime baselines — pinned consistently everywhere | High | SAP-REQ | Both |
| CAP-VER-004 | Never mix CAP package major lines | High | SAP-REQ | Node.js |
| CAP-VER-005 | Execute major upgrades with the official migration path | Medium | SAP-REC | Both |
| CAP-VER-006 | Deploy HANA artifacts as `hdbtable` — never `hdbcds` | Medium | SAP-REQ | Both |

---

## CAP-VER-001 — Freeze dependencies for deployment — and refresh them regularly

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-001 |
| **Title** | Freeze dependencies for deployment — and refresh them regularly |
| **Category** | Dependency & version management |
| **Severity** | High |
| **Authority** | SAP-REQ (freezing: "Deployed applications should freeze all their dependencies, including transient ones"); the refresh cadence is SAP-REC ("We recommend setting up Dependabot, Renovate or similar… one-by-one"); the concrete SLA is ORG (G-40) |
| **Applicability** | Node.js projects with deployment targets (Java's Maven resolution is deterministic by versioned coordinates; no counterpart rule needed) |
| **Runtime** | Node.js |
| **CAP version** | All currently supported versions (`cds up` creates the lockfile on initial deploy; refresh via `npm install --package-lock-only`) |
| **Status** | Active |
| **Related rules** | CAP-VER-002 (what to refresh toward), CAP-CICD-001 (the pipeline builds from the frozen tree), CAP-SEC-017; absorbs candidate CAP-VER #2 |
| **Last verified** | 2026-08-12 |

### Rule statement
Deployed Node.js applications MUST freeze all dependencies including transient ones — a committed `package-lock.json` (SAP: "Deployed applications should freeze all their dependencies, including transient ones"; refresh with `npm install --package-lock-only`) — and MUST refresh them deliberately and regularly: SAP recommends "Dependabot, Renovate or similar automated solutions to update dependencies one-by-one to easily identify breaking changes". Freezing without refreshing rots (missed security fixes); refreshing without freezing makes every deployment a lottery. The concrete update SLA (patch within N days) is ORG policy (G-40).

### Rationale
An unfrozen dependency tree means the artifact deployed tomorrow differs from what was tested today — transitive updates land silently in production. A frozen-but-stale tree accumulates unpatched vulnerabilities. The documented pair — lockfile plus automated one-by-one updates — is the reproducibility *and* currency mechanism. **High:** unreproducible production builds and stale security patches both materially affect production reliability; kept below Critical (candidate downgrade) because the failure is probabilistic drift, not a directly exploitable condition.

### Evidence expected in code
`package-lock.json` committed and current (matching `package.json`); an update automation config (`.github/dependabot.yml`, `renovate.json`) or documented manual cadence; CI installing from the lockfile (`npm ci`).

### Detection guidance
1. Verify `package-lock.json` exists, is committed (not gitignored), and is in sync with `package.json` (`npm ci` would succeed) → missing/ignored/out-of-sync → FAIL.
2. Verify the pipeline installs with `npm ci` (lockfile-exact) rather than bare `npm install` → FAIL if builds float.
3. Look for update automation (dependabot/renovate config) or a documented refresh process → neither → FAIL the refresh element (frozen-and-forgotten).
4. Check update PRs/commits actually flow (recent lockfile churn) → stale for many months → observation escalating to FAIL against the ORG SLA once defined (G-40).
5. Report file locations.

### Good example
```text
package-lock.json          ← committed, current
.github/dependabot.yml     ← daily npm updates, one dependency per PR
CI:  npm ci && npm test    ← builds exactly the frozen tree
```

### Bad example
```text
package-lock.json in .gitignore "to avoid merge conflicts";
CI runs npm install — every pipeline run resolves a different tree;
last dependency update: whenever someone remembers.
```

### Exception guidance
Alternative package managers with equivalent committed lockfiles (pnpm-lock.yaml, yarn.lock) satisfy the freezing requirement — name the `npm ci` equivalent in CI. No exception for floating production dependencies.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/to-cf ("should freeze all their dependencies, including transient ones"; `npm install --package-lock-only`; Dependabot/Renovate one-by-one — re-verified verbatim 2026-08-12)

---

## CAP-VER-002 — Stay on an Active CAP release line

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-002 |
| **Title** | Stay on an Active CAP release line |
| **Category** | Dependency & version management |
| **Severity** | High |
| **Authority** | SAP-REQ (documented maintenance model: Maintenance = "at most twelve more months", "critical bug fixes only"; EOL = no fixes, freeze dependencies); the consumption cadence is SAP-REC (CAP Java: latest minor monthly, patches ASAP) |
| **Applicability** | All projects |
| **Runtime** | Both |
| **CAP version** | ⏱ The Active/Maintenance lines are tracked in [docs/version-management.md](../../docs/version-management.md) (2026-08-12: cds 10.x / CAP Java 5.x Active; cds 9 Maintenance policy-derived; CAP Java 4.9.x Maintenance) |
| **Status** | Active |
| **Related rules** | CAP-VER-003, CAP-VER-005 (how to move), CAP-MT-001 (the MTX instance of this rule); absorbs candidate CAP-VER #4 |
| **Last verified** | 2026-08-12 |

### Rule statement
Projects MUST run on an Active CAP release line and plan major upgrades inside the maintenance window: SAP's model gives a superseded major at most twelve months of critical-fixes-only Maintenance, then End of life with no fixes at all (SAP's EOL instruction: freeze Node.js/Spring Boot/Java versions, patch updates only — an emergency posture, not a strategy). Within the Active line, minors/patches SHOULD be consumed per SAP's cadence (CAP Java: "consume the latest minor version on a monthly basis", patches "as soon as possible"). A project on a Maintenance line MUST have a dated upgrade plan; a project on an EOL line is a standing High finding until resolved.

### Rationale
The maintenance model is the support contract: security fixes, new Node/JDK compatibility, and defect resolution exist only on Active (fully) and Maintenance (critically, briefly). Production software outside that window accumulates unfixable vulnerabilities and blocks runtime upgrades (Maintenance lines get no new Node/Java support — the runtimes EOL underneath them). **High:** unsupported foundations in production; consistent with CAP-MT-001's calibration (candidate Critical downgraded — the risk is unpatched exposure over time, not an immediate breach).

### Evidence expected in code
CAP dependency majors (`package.json`/`pom.xml`) on the Active line per the baseline document; for Maintenance-line projects: a dated upgrade plan (ADR/roadmap); minor/patch currency within the line.

### Detection guidance
1. Read the project's CAP majors (`@sap/cds` in package-lock; `com.sap.cds` in the effective POM).
2. Compare against [version-management.md](../../docs/version-management.md)'s verified Active/Maintenance lines (re-verify the baseline's date first — if stale, re-verify upstream before failing anyone).
3. Active line → PASS; check minor/patch lag (many minors behind → observation per the SAP-REC cadence).
4. Maintenance line → locate the dated upgrade plan → present → PASS with observation; absent → FAIL.
5. EOL line → FAIL (High), with the EOL freeze posture as interim mitigation only.
6. Report versions with file:line.

### Good example
```jsonc
// package.json — Active line, minors consumed
{ "dependencies": { "@sap/cds": "^10" } }
// docs/adr/0031-cds9-upgrade.md exists for the sibling still on 9:
// "upgrade to 10 by 2026-12, before Maintenance ends" — dated plan
```

### Bad example
```jsonc
// production SaaS on a line that left Maintenance long ago —
// no fixes, no plan, Node upgrade blocked underneath it
{ "dependencies": { "@sap/cds": "^7" } }
```

### Exception guidance
A frozen legacy system in documented decommissioning (dated end-of-life of the *application*) may ride out Maintenance/EOL with the SAP freeze posture and a risk acceptance — recorded, time-boxed. No exception for systems with a future.

### SAP reference
- https://cap.cloud.sap/docs/releases/schedule (Active/Maintenance/EOL model — wording re-verified 2026-08-12)
- https://cap.cloud.sap/docs/java/versions (consume latest minor monthly, patches ASAP; Active 5.x / Maintenance 4.9.x)

---

## CAP-VER-003 — Run supported runtime baselines — pinned consistently everywhere

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-003 |
| **Title** | Run supported runtime baselines — pinned consistently everywhere |
| **Category** | Dependency & version management |
| **Severity** | High |
| **Authority** | SAP-REQ (documented minimum runtime requirements per CAP line) |
| **Applicability** | All projects |
| **Runtime** | Both |
| **CAP version** | ⏱ Baseline per [docs/version-management.md](../../docs/version-management.md) (2026-08-12 for cds 10 / CAP Java 5: Node ≥ 22, rec. 24/26; JDK ≥ 21, rec. 25; Spring Boot ≥ 4.0; Maven ≥ 3.9.14) |
| **Status** | Active |
| **Related rules** | CAP-VER-002 (line determines baseline), CAP-VER-005 (pin updates are a documented migration step), CAP-CICD-001 (CI matrices are pin locations) |
| **Last verified** | 2026-08-12 |

### Rule statement
Projects MUST run runtimes meeting the documented minimums for their CAP line (per the live baseline in [version-management.md](../../docs/version-management.md); no numbers are hardcoded here — the register is authoritative), and MUST pin the runtime **consistently across every pin location**: `package.json` `engines` / `.nvmrc`, `pom.xml` (`java.version`, Spring Boot parent), Dockerfiles/base images, CI matrices, and BTP runtime configuration (buildpack/`mta.yaml` settings). Divergent pins — CI testing on one Node major, production running another — are a violation even when each is individually supported.

### Rationale
SAP drops runtime support at CAP majors (cds 10 dropped Node 20; CAP Java 5 dropped JDK 17) because the framework starts *using* newer runtime features (`node:sqlite` needs ≥ 22.5) — below-minimum runtimes fail unsupported, sometimes subtly. The consistency clause is where real incidents live: every pin location is a chance for "tested on 24, deployed on 22". SAP's own upgrade guide instructs updating all pin locations. **High:** unsupported or inconsistent runtime baselines undermine everything above them; usually loud, occasionally subtle (feature-level gaps).

### Evidence expected in code
All pin locations present and mutually consistent, meeting the baseline: engines/.nvmrc, POM properties, Docker base images, CI matrix versions, deployment runtime config.

### Detection guidance
1. Collect every pin: `package.json` engines, `.nvmrc`, `pom.xml` java/Spring versions, `Dockerfile` `FROM` tags, CI workflow matrices, `mta.yaml`/chart buildpack-runtime settings.
2. Check each against the baseline document's minimums (after confirming the baseline's verification date is current) → below-minimum → FAIL per location.
3. Cross-compare the pins → divergence (different majors across locations) → FAIL with all locations listed.
4. Missing pins (no engines field, untagged `FROM node`) → FAIL element (floating runtime).
5. Report as a pin-location table.

### Good example
```text
package.json engines: ">=22"    .nvmrc: 24
Dockerfile: FROM node:24-slim   CI matrix: [24]
mta.yaml: nodejs buildpack, node 24        ← one story everywhere
```

### Bad example
```text
engines: ">=18"  (stale)        CI matrix: [20, 22]
Dockerfile: FROM node:latest    production buildpack: default
— four locations, four answers, one of them below the supported minimum
```

### Exception guidance
Transitional dual-version CI matrices *during* a runtime upgrade are legitimate — time-boxed, with production pinned to one supported version throughout. No exception for below-minimum production runtimes.

### SAP reference
- https://cap.cloud.sap/docs/releases/2026/jun26 (runtime minimums for the current majors — re-verified 2026-08-12)
- https://cap.cloud.sap/docs/java/versions (JDK/Spring Boot/Maven baselines per line)
- https://cap.cloud.sap/docs/node.js/upgrading (update every pin location)

---

## CAP-VER-004 — Never mix CAP package major lines

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-004 |
| **Title** | Never mix CAP package major lines |
| **Category** | Dependency & version management |
| **Severity** | High |
| **Authority** | SAP-REQ (documented pairing for the cds 10 wave: `@sap/cds` 10.x with `@sap/cds-dk` 10.x, `@sap/cds-compiler` 7.x, `@sap/cds-mtxs` 4.x, `@cap-js/*` 3.x — mixing unsupported) |
| **Applicability** | Node.js projects (CAP Java's BOM-style versioning makes the Maven side largely self-consistent; the cds-dk pairing for Java tooling is noted in the baseline) |
| **Runtime** | Node.js |
| **CAP version** | ⏱ Pairing matrix per [docs/version-management.md](../../docs/version-management.md) (updated at each major) |
| **Status** | Active |
| **Related rules** | CAP-VER-002, CAP-VER-005; CAP-EVT-002 (whose documented 8→10 queue hazard is a concrete mixed-generation failure mode during rolling upgrades), CAP-MT-001 (mtxs pairing) |
| **Last verified** | 2026-08-12 |

### Rule statement
All CAP packages in a project MUST come from one coherent major wave per the documented pairing (current matrix in the baseline document): `@sap/cds`, `@sap/cds-dk`, `@sap/cds-compiler`, `@sap/cds-mtxs`, and the `@cap-js/*` services move together. Mixed major lines (e.g., cds 10 with mtxs 3.x, or `@cap-js/hana` 2.x) are unsupported configurations and MUST NOT be deployed. During upgrades, the transition MUST be completed across all packages — and across all *running instances* (see the documented rolling-upgrade queue hazard owned by CAP-EVT-002).

### Rationale
The packages are released as a coordinated wave; cross-generation combinations are explicitly outside what SAP tests ("do not mix major lines"). Failures range from loud (install/boot errors) to subtle — the documented cds 8/10 queue double-processing shows mixed *generations of running instances* corrupting behavior at the data level. **High (candidate Critical downgraded):** most mixes fail fast; the subtle instance-level case is owned and documented at CAP-EVT-002 — this rule prevents the configuration class.

### Evidence expected in code
`package.json`/lockfile with all CAP packages on the paired majors; upgrade PRs moving the wave together; deployment strategy for upgrades honoring release-note instance guidance.

### Detection guidance
1. Extract CAP package majors from `package-lock.json`: `@sap/cds`, `@sap/cds-dk` (also global/CI usage), `@sap/cds-compiler` (usually transitive), `@sap/cds-mtxs`, `@cap-js/*`.
2. Compare against the baseline document's pairing matrix → any cross-wave combination → FAIL with the package list.
3. Check CI/global tooling (`npx @sap/cds-dk` versions in workflows) matches the project line → mismatch → FAIL.
4. For in-flight upgrades: verify the upgrade changes the whole wave in one change set → partial bumps merged to main → FAIL.
5. Report as a package/major table.

### Good example
```text
@sap/cds 10.0.2 · @sap/cds-dk 10.0.1 · compiler 7.0.1 · mtxs 4.0.1 · @cap-js/hana ^3
— one wave (per version-management.md matrix, verified 2026-08-12)
```

### Bad example
```text
@sap/cds ^10 with @sap/cds-mtxs ^3 ("mtx upgrade looked risky") —
unsupported pairing; MTX behavior now untested territory.
```

### Exception guidance
None for deployed configurations. Short-lived mixed states inside an upgrade branch are the working state of CAP-VER-005's process — never merged/deployed as such.

### SAP reference
- https://cap.cloud.sap/docs/releases/2026/jun26 (wave pairing; do not mix major lines — baseline re-verified 2026-08-12)

---

## CAP-VER-005 — Execute major upgrades with the official migration path

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-005 |
| **Title** | Execute major upgrades with the official migration path |
| **Category** | Dependency & version management |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented guides and tooling; the Node tool is explicitly **alpha** — manual review is part of the documented reality) |
| **Applicability** | Every CAP major upgrade |
| **Runtime** | Both (Node: upgrade guide + `cds upgrade` (alpha); Java: migration guides + OpenRewrite recipes `com.sap.cds:cds-services-recipes`) |
| **CAP version** | ⏱ Per-major guides (current chain: Node cds 9→10 guide; Java 1.x→…→5.0) |
| **Status** | Active |
| **Related rules** | CAP-VER-002/-003/-004 (what an upgrade must land on), **CAP-EVT-002** (owns the 8→10 queue-drain step — the model for release-note-driven operational steps), CAP-MT-005 (tenant upgrades ride along), CAP-TEST (full suite after) |
| **Last verified** | 2026-08-12 |

### Rule statement
CAP major upgrades MUST follow the official migration path: read the runtime's upgrade/migration guide for the target major; apply the official tooling (Node: `cds upgrade` — alpha, its findings manually reviewed; Java: the OpenRewrite recipes); update all version pins together (CAP-VER-003); execute **version-specific operational steps from the release notes** (e.g., the documented queue-drain/step-through-9 requirement for cds 8→10 — owned by CAP-EVT-002); and re-run the full test suite plus an M8-scoped review before production. Upgrades MUST NOT be improvised as bare version bumps.

### Rationale
Majors change defaults (compat flags become behavior), drop APIs, and occasionally require operational sequencing — the documented artifacts (guides, recipes, release notes) are the difference between a managed migration and archaeology after production breaks. The alpha status of `cds upgrade` is itself documented (false positives possible) — the tool assists, the review decides. **Medium:** a process-quality rule; the concrete breakage classes it prevents are owned by the referenced rules.

### Evidence expected in code
Upgrade PRs referencing the migration guide; recipe/tool runs visible (OpenRewrite in the build log, `cds upgrade` output attached); pins moved together; release-note operational steps in the deployment plan; green suite + review record.

### Detection guidance
1. Identify the last major upgrade (git history of `@sap/cds`/`com.sap.cds` majors).
2. Check the upgrade change set: all-wave pin movement (CAP-VER-004), engines/CI/Docker updates (CAP-VER-003) → piecemeal → FAIL.
3. Look for migration-path evidence: guide reference, recipe execution, `cds upgrade` report → none → FAIL (bare bump).
4. For upgrades crossing documented operational hazards (8→10 queues): verify the runbook step existed (CAP-EVT-002's element — report there, note here).
5. Verify post-upgrade test-suite run and review record (completion report/M8 gate) → absent → FAIL element.
6. NOT APPLICABLE if no major upgrade has occurred in the assessable history.

### Good example
```text
PR "cds 9 → 10": migration guide checklist in description; cds upgrade
report attached (2 false positives dismissed with reasons); engines,
.nvmrc, Dockerfile, CI matrix moved to Node 24; queue drained per
CAP-EVT-002 note; full suite green; M8 review linked.
```

### Bad example
```text
Commit "bump cds to 10" touching only package.json — mtxs still 3.x,
CI still Node 20, no guide consulted; deployed as a rolling update
over v8 instances with a hot queue.
```

### Exception guidance
Greenfield projects starting on the current major have no migration to run (NOT APPLICABLE). Skipping the official tooling is acceptable when the guide itself is fully applied manually and the review documents it.

### SAP reference
- https://cap.cloud.sap/docs/node.js/upgrading (cds 10 upgrade guide; `cds upgrade` alpha; pin locations)
- https://cap.cloud.sap/docs/java/migration (migration chain; OpenRewrite recipes)
- https://cap.cloud.sap/docs/guides/events/event-queues (example of a release-note operational step — owned by CAP-EVT-002)

---

## CAP-VER-006 — Deploy HANA artifacts as `hdbtable` — never `hdbcds`

| Field | Value |
|---|---|
| **Rule ID** | CAP-VER-006 |
| **Title** | Deploy HANA artifacts as `hdbtable` — never `hdbcds` |
| **Category** | Dependency & version management |
| **Severity** | Medium |
| **Authority** | SAP-REQ ("The deploy format `hdbcds` for SAP HANA has been deprecated with cds8 … and can now no longer be used" — re-verified on the live May 2025 release note, 2026-08-12) |
| **Applicability** | HANA-deployed projects — practically, those migrated from older/on-prem-origin setups (SAP HANA Cloud never used `hdbcds`; new projects get `hdbtable` by default) |
| **Runtime** | Both |
| **CAP version** | ⏱ `hdbcds` unusable since cds 9; `hdbtable` is the default (migration path documented: `.hdbcds` → `.hdbtable` → `.hdbmigrationtable`) |
| **Status** | Active |
| **Related rules** | CAP-DB-001 (HANA Cloud production), CAP-DB-007 (`.hdbmigrationtable` for large tables), CAP-VER-005 (migration execution) |
| **Last verified** | 2026-08-12 |

### Rule statement
HANA deployment MUST use the `hdbtable` format (the default) — the `hdbcds` format "can now no longer be used" since cds 9 (deprecated with cds 8). Projects carrying legacy `hdbcds` configuration or artifacts MUST complete the documented migration (`hdbcds` → `hdbtable`, with `@cds.persistence.journal`/`.hdbmigrationtable` per CAP-DB-007 where large tables warrant it) before or as part of any move to a current CAP line.

### Rationale
This is a removed capability, not a preference: builds configured for `hdbcds` fail on current toolchains, and the constraint surfaces exactly during major upgrades (CAP-VER-005) of long-lived projects — the worst moment to discover a deploy-format migration is also due. New HANA Cloud projects are unaffected by construction (the default is `hdbtable`; HANA Cloud never supported `hdbcds`). **Medium:** loud build/deploy failure; the review value is catching the legacy configuration *before* the upgrade forces it.

### Evidence expected in code
No `hdbcds` deploy-format configuration (`cds.hana.deploy-format` overrides, `.hdbcds` artifacts under `db/src`); `hdbtable` (default) in effect; migration completed for formerly-`hdbcds` projects.

### Detection guidance
1. Search configuration for deploy-format overrides (`deploy-format`, `hdbcds` in `.cdsrc*`/package.json/build config) → `hdbcds` configured → FAIL.
2. Search `db/` for committed `.hdbcds` artifacts → present → FAIL (legacy remnants; name the migration guide).
3. New projects on defaults → PASS (typically NOT APPLICABLE in practice — state it).
4. For projects with migration history: verify the journal decision for large tables was made during migration (cross-check CAP-DB-007).

### Good example
```text
db/src/**.hdbtable (+ selected .hdbmigrationtable per CAP-DB-007) —
default format, no overrides anywhere.
```

### Bad example
```jsonc
// .cdsrc.json carried since 2023 — build now fails on current cds majors
{ "hana": { "deploy-format": "hdbcds" } }
```

### Exception guidance
None on current CAP lines — the format is removed. Projects frozen on pre-9 lines (under a CAP-VER-002 exception) inherit this as a mandatory step of their eventual upgrade plan.

### SAP reference
- https://cap.cloud.sap/docs/releases/2025/may25 ("deprecated with cds8 … can now no longer be used" — re-verified 2026-08-12)
- https://cap.cloud.sap/docs/guides/databases/hana (hdbcds → hdbtable → hdbmigrationtable migration flow)

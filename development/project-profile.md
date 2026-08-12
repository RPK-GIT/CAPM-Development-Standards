# CAPM Project Profile (`capm-profile.yaml`)

The machine-readable capability profile that adopting CAP projects place in their **repository root** as `capm-profile.yaml`. It is the single input that makes the [rule-milestone matrix](rule-milestone-matrix.md)'s CONDITIONAL and runtime filtering decidable **without parsing prose**. Copyable template: [templates/capm-profile-template.yaml](../templates/capm-profile-template.yaml). Created/updated at M0 (see [M0 checklist](milestones/m0-requirements.md)); kept current as capabilities change.

## Schema

Every field maps to matrix §1.4 flags or rule-applicability conditions — nothing else belongs here.

```yaml
standard:
  version: ""            # REQUIRED — commit SHA or tag of the CAPM standards repo this project follows

project:
  name: ""               # REQUIRED
  milestone: M0          # REQUIRED — current lifecycle position: M0..M9 | LIVE

cap:
  runtime: ""            # REQUIRED — nodejs | java          → runtime-scoped rule filtering
  version: ""            # REQUIRED — @sap/cds major (nodejs, e.g. "10") or CAP Java major (java, e.g. "5")
                         #   checked against docs/version-management.md (CAP-VER-002/-003)
  odata: true            # OData-served APIs                  → CAP-SRV-005, CAP-PERF-001

tenancy:
  multitenant: false     # REQUIRED                           → CAP-MT-*, CAP-INT-006
  sidecar: null          # only if multitenant: true | false  → CAP-MT-002

identity:
  service: ""            # REQUIRED from M1 — ias | xsuaa | undecided   → CAP-SEC-007/-008
  new_project: false     # greenfield                          → CAP-SEC-006

capabilities:            # all REQUIRED as explicit true/false (no silent defaults for these)
  eventing: false        #                                    → CAP-EVT-001..005/-007
  broker: false          # message broker in use/planned      → CAP-EVT-006
  remote_services: false #                                    → CAP-INT-001..007
  mashups: false         # cross local/remote reads           → CAP-INT-005
  custom_operations: false #                                  → CAP-SRV-004
  instance_authorization: false # row-level access requirements  → CAP-SEC-010
  drafts: false          #                                    → CAP-SRV-007, CAP-PERF-006
  media: false           #                                    → CAP-SRV-009
  localized_data: false  #                                    → CAP-CDS-008, CAP-DB-006
  temporal_data: false   #                                    → CAP-CDS-009
  personal_data: false   # NO DEFAULT — must be an explicit decision → CAP-PRIV-*
  pdm: false             # SAP Personal Data Manager           → CAP-PRIV-003
  extensibility: false   # SaaS customer extensions offered    → CAP-EXT-002/-003
  feature_toggles: false # fts/ features                       → CAP-EXT-004
  mcp: false             # @mcp-exposed services               → CAP-SEC-018
  telemetry: false       # tracing/metrics adopted             → CAP-LOG-004
  ui: false              # end-user UI                         → CAP-OPS-002, CAP-ERR-005

deployment:
  target: ""             # REQUIRED from M1 — cloud-foundry | kyma | both | undecided → CAP-DEP-001/-003
  multi_landscape: false #                                    → CAP-DEP-002

database:
  production: ""         # REQUIRED from M1 — hana | postgres | undecided → CAP-DB-001
  legacy_hdbcds: false   # migrated on-prem-origin artifacts   → CAP-VER-006

workload:                # OPTIONAL — values true | false | unknown; see "Workload flag evidence states"
  mass_data: false       #                                    → CAP-PERF-005
  concurrent_edit: false #                                    → CAP-SRV-008
  concurrent_paging: false #                                  → CAP-PERF-001
  hana_large_volume: false #                                  → CAP-DB-007
  pessimistic_locking: false #                                → CAP-DB-008
  cloud_only_features: false # locking/MTX/HANA-specific       → CAP-TEST-006
  major_upgrade_in_scope: false #                              → CAP-VER-005
```

## Workload flag evidence states *(added 2026-08-12, Round 1 calibration — Pilot 1, ambiguity A-1)*

Workload characteristics are usually **not derivable from a repository** (e.g., whether collections are paged while concurrently modified). To keep dependent rules honest, each `workload` flag carries one of three values plus an optional source note:

| Value | Meaning | Effect on dependent rules |
|---|---|---|
| `true` / `false` | Decided. An inline `# source:` comment records the basis: `# source: owner-declared (<who/role>, <date>)` or `# source: repo-evidence (<pointer>)` | Normal CONDITIONAL filtering |
| `unknown` | Nobody has established the characteristic yet | Rules conditioned on the flag are **NOT ASSESSABLE**, naming the flag — never silently filtered to NOT APPLICABLE, never guessed |

Evidence discipline:

- **Owner declaration is a hypothesis, not repository evidence.** Reviewers cross-check every `owner-declared` flag against the repository (Validation step 4); where the repository contradicts the declaration (e.g., `concurrent_edit: false` but shared draft-enabled entities are writable by multiple roles), the review reports **CONTRADICTORY profile evidence** and stops — the profile must be corrected, not overridden.
- Repository evidence MAY confirm a declaration (upgrade the source note to `repo-evidence`), and MAY establish a flag on its own where the artifact is unambiguous.
- **Claude never sets or assumes a workload value by inference.** It MAY propose a value with the supporting evidence; the value enters the profile only by user acceptance (same rule as Validation step 2). Where no owner input and no unambiguous repository evidence exist, the correct value is `unknown`.

## Repository-derived conditions (deliberately NOT in the profile)

Two matrix conditions are determined by repository inspection, not declaration, because they are trivially and reliably detectable from code: **ERROR-HOOKS** (custom `srv.on('error')` handlers exist → CAP-ERR-004) and, where ambiguous, actual capability usage always overrides a stale profile flag (see Validation step 4). Everything requirement-derived (not code-derivable) belongs in the profile.

## Field rules

| Aspect | Rule |
|---|---|
| **Required always** | `standard.version`, `project.name`, `project.milestone`, `cap.runtime`, `cap.version`, `tenancy.multitenant`, the entire `capabilities` block (explicit values) |
| **Required from M1** | `identity.service`, `deployment.target`, `database.production` (`undecided` is a legal value *only before* M1 exit — see [M1 checklist](milestones/m1-architecture.md)) |
| **Conditionally required** | `tenancy.sidecar` when `multitenant: true` |
| **Optional (safe default false)** | the `workload` block, `cap.odata` (default true), `identity.new_project`, `deployment.multi_landscape`, `database.legacy_hdbcds` |
| **Allowed values** | `runtime`: `nodejs`\|`java` · `milestone`: `M0`–`M9`\|`LIVE` · `identity.service`: `ias`\|`xsuaa`\|`undecided` · `deployment.target`: `cloud-foundry`\|`kyma`\|`both`\|`undecided` · `database.production`: `hana`\|`postgres`\|`undecided` · booleans strictly `true`/`false` (exception: `workload` flags allow `true`\|`false`\|`unknown` — see Workload flag evidence states) |
| **Version format** | `cap.version` = the major line as a string (matched against the [version register](../docs/version-management.md), never against hard-coded numbers) |

## Review modes (milestone position vs reviewed milestone) *(added 2026-08-12, Round 1 calibration)*

The reviewed milestone and `project.milestone` together determine the **review mode**, recorded in every report header:

| Mode | When | Meaning for findings |
|---|---|---|
| **DEVELOPMENT** | Reviewed milestone = the milestone currently being developed (`project.milestone` equals it, application not yet complete) | Findings block forward progress; SUPPORTING selection = rows whose subject the milestone's changes touch |
| **RETROSPECTIVE** | Reviewed milestone lies behind the project's actual position (`project.milestone` later, or `LIVE`) | Findings assess decisions already made; SUPPORTING selection is evidence-driven (see the review command) |

`project.milestone: LIVE` additionally means the application is **deployed**: the report MUST state that findings represent **current operational risk**, not historical/pre-release risk. No other process changes — the same rules, gates, and evidence model apply in both modes.

## Invalid / flagged combinations

Validation fails (✗) or warns (⚠) on:

- ✗ `tenancy.multitenant: true` + `database.production: postgres` — documented unsupported combination (CAP-DB-001/CAP-MT precondition).
- ✗ `tenancy.multitenant: true` + `tenancy.sidecar` missing.
- ✗ `project.milestone` ≥ M1 with `identity.service`/`deployment.target`/`database.production` still `undecided`.
- ✗ `capabilities.pdm: true` + `capabilities.personal_data: false` (PDM without personal data is meaningless).
- ✗ `capabilities.mashups: true` + `capabilities.remote_services: false`.
- ⚠ `capabilities.mcp: true` + `cap.runtime: java` — the Java MCP adapter is not publicly released per the version register; verify before relying on it.
- ⚠ `capabilities.broker: true` + `capabilities.eventing: false` — a broker without eventing is usually a profile error.
- ⚠ `capabilities.feature_toggles: true` + `tenancy.multitenant: false` — per-tenant toggling implies MT; verify intent.
- ⚠ `workload` flags all false on a non-trivial project — reviewers spot-check plausibility (e.g., transactional entities usually imply `concurrent_edit`); where a characteristic genuinely cannot be established, the honest value is `unknown`, not `false`.
- ⚠ any `workload` flag `unknown` at M4+ — the dependent rules stay NOT ASSESSABLE until the flag is decided; the review names them.

## Validation procedure (executed by both commands before any rule work)

1. Parse the YAML; malformed → **NOT-ASSESSABLE** (name the parse error).
2. Check required fields for the declared milestone; any missing → **NOT-ASSESSABLE**, listing exactly the missing fields — **never guess or infer a missing value from the codebase silently**. (Claude MAY *propose* values derived from repository inspection — e.g., runtime from `package.json` vs `pom.xml` — but they enter the profile only by the user accepting the proposed edit.)
3. Check allowed values and the invalid-combination table; ✗ violations → **NOT-ASSESSABLE** until corrected; ⚠ warnings → proceed, reported in the output.
4. Cross-check plausibility against the repository (runtime vs manifests; `multitenant` vs `@sap/cds-mtxs`; `eventing` vs messaging config; `personal_data` vs obvious personal fields in the model). Contradictions → report as **CONTRADICTORY profile evidence** and stop: the profile must be corrected, not overridden.
5. Check `standard.version` against the standards repo in use; a mismatch is reported (review runs against the standard version actually loaded).

## Security constraints

The profile MUST NOT contain passwords, tokens, credentials, service keys, secrets, personal data, or system URLs with embedded credentials — it is a **capability declaration only** and is committed to version control like any source file (CAP-SEC-017 applies to it). Reviews treat a profile containing secret-like values as a CAP-SEC-017 finding.

## Relationship to existing artifacts

The profile is the machine-readable form of the review model's step 2 ("Profile the project") and the M0 checklist's capability-profile deliverable. Where profile and repository disagree, the repository is the evidence of record (AI-REVIEW-001) and the disagreement is itself a finding — the profile is an *index*, never a substitute for inspection.

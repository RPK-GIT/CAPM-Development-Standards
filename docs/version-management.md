# Version Management

CAP changes monthly; guidance that was correct last quarter may be wrong today. This document defines (a) how this standard handles version sensitivity, and (b) the currently verified version baseline. **Facts below carry the date they were verified against official SAP pages; re-verify before relying on them.** The normative version rules are [CAP-VER-001…006](../standards/rules/versions-dependencies.md).

## Policy

1. **No unverified "always use X".** Version recommendations appear in this standard only with an official SAP source and a verification date. Where SAP is silent, the entry is marked as an [org-policy gap](../references/research-gaps.md).
2. **Rules carry version applicability.** Every Layer 2 rule states the CAP version range it was verified for (template field `CAP version`). Rules invalidated by a release are updated or deprecated — never silently left stale.
3. **Version-sensitive rule register.** Rules whose substance depends on versions are flagged in their category files; the register below tracks the themes.
4. **Re-verification cadence (ORG).** The source map and this baseline are re-verified against <https://cap.cloud.sap/docs/releases/> at least quarterly and at every CAP major release. The January 2026 documentation restructuring (URL churn; several pages later removed outright, e.g. `/docs/about/*`) shows why: cited URLs rot.
5. **Follow SAP's maintenance model (SAP-REC).** Per the official [release schedule](https://cap.cloud.sap/docs/releases/schedule) (wording re-verified 2026-08-12): monthly minor releases on the Active major; a superseded major enters *Maintenance* ("at most twelve more months", "critical bug fixes only", no new Node/Java version support), then *End of life* (no fixes; SAP: freeze Node.js/Spring Boot/Java versions, patch updates only). Node.js support tracks "the two Active and Maintenance LTS versions of Node.js". Enforced by [CAP-VER-002/-003](../standards/rules/versions-dependencies.md).

## Verified version baseline — as of **2026-08-12**

Sources: <https://cap.cloud.sap/docs/releases/> (index), <https://cap.cloud.sap/docs/releases/2026/jun26>, <https://cap.cloud.sap/docs/releases/2026/changelog>, <https://cap.cloud.sap/docs/java/versions>, <https://cap.cloud.sap/docs/releases/schedule>.

> **Baseline status:** the **June 2026 wave remains current**. The **August 2026 minor (cds 10.1 / CAP Java 5.1) is listed on the releases index but explicitly UNRELEASED as of 2026-08-12** (its release-notes page 404s; the changelog has no July/August entries). Re-check the index later in August.

| Component | Current (verified 2026-08-12) | Notes |
|---|---|---|
| `@sap/cds` (Node.js) | **10.x** (10.0.2 in changelog) | June 2026 major. Do not mix major lines: pairs with `@sap/cds-dk` 10.x (10.0.1), `@sap/cds-compiler` 7.x (7.0.1), `@sap/cds-mtxs` 4.x (4.0.1), `@cap-js/*` 3.x (SAP-REQ; see CAP-VER-004) |
| `@sap/cds` 9 | Maintenance (policy-derived) | ⚠ Not stated verbatim on a page — derived from the documented maintenance model (major shipped June 2026 → prior major in Maintenance ≤ 12 months, critical fixes only) |
| `@cap-js/{sqlite,hana,postgres}` | **3.x line** | ⚠ Exact patch versions not published on the docs pages (only the line); do not cite concrete patch numbers |
| CAP Java (`com.sap.cds`) | **5.x** (5.0.0) Active; **4.9.x** Maintenance (patches only) | Java 5: Spring Boot ≥ 4.0, Spring Security 7, Maven ≥ 3.9.14; 4.9.x: Spring Boot 3.x, Maven ≥ 3.8.8 (SAP-REQ) |
| Node.js | **≥ 22 required**; 24 (Active LTS) or 26 recommended | Node 20 dropped with cds 10; `node:sqlite` default driver needs ≥ 22.5. Java tooling: Node 22 with cds-dk 10.x (cds-dk 9.x for CAP Java 4.9.x) |
| JDK | **≥ 21 required**; 25 recommended (SapMachine) | JDK 17 dropped with CAP Java 5 (SAP-REQ) |
| Databases — production | SAP HANA Cloud (PostgreSQL "in edge cases") (SAP-REC) | Unchanged 2026-08-12. Deploy format `hdbtable`; `hdbcds` "can now no longer be used" (may25 release note, re-verified 2026-08-12; HANA Cloud never used hdbcds — the constraint bites migrated on-prem-origin projects). See CAP-DB-001/-002, CAP-VER-006 |
| Databases — development | SQLite via `@cap-js/sqlite` (Node), H2 **up to 2.3.x only** (Java) (SAP-REQ for the H2 bound) | H2 wording: "we only support until version 2.3.x" (re-verified 2026-08-12 in Batch 5) |
| MCP protocol adapter | **Beta** (Node `@cap-js/mcp`; Java `cds-adapter-mcp` "not yet released publicly") | Re-verified 2026-08-12 — status unchanged; governed by CAP-SEC-018 |
| Express (Node) | `express^5` default when unpinned (since Jan 2026) | Flagged by SAP as potentially breaking |

## Version-sensitive themes (register)

Tracked in detail in [references/sap-cap-sources.md](../references/sap-cap-sources.md); headline items (all re-checked or carried with their original verification dates):

- **Defaults that flipped:** JSON log formatter production default (cds ≥ 7.5 — re-verified 2026-08-12, CAP-LOG-002); production deny-by-default auth (`restrict_all_services`, CAP-SEC-003); CAP Java 4+: deep authorization/input checking on by default (CAP-SEC-011); CAP Java 5 default flips (`preferServiceException` true → CAP-ERR-001, outbox unordered, auth mode model-relaxed, HANA fuzzy true); cds 10 makes former opt-in compat flags the defaults.
- **Package migrations:** built-in DB services → `@cap-js/*` (cds 8+, "highly recommended" — CAP-DB-003); `cds.test` → `@cap-js/cds-test` (CAP-TEST-001); MTX classic → `@sap/cds-mtxs` (CAP-MT-001); `cds-feature-xsuaa` → `cds-feature-identity` (CAP-SEC-005); Vitest supported / Jest has documented ESM/Chai friction (no deprecation statement — CAP-TEST-004); ESM default for new Node projects (cds 10).
- **Deprecated/removed:** `hdbcds` deploy format ("can now no longer be used" since cds 9 — CAP-VER-006); `install-cdsdk` Maven goal (removed in cds-maven-plugin 5); `cds.tx(req)` "not required nor recommended anymore" (CAP-TXN-003).
- **Identity direction:** "Start new projects with IAS" (+ AMS); XSUAA legacy with cross-consumption (CAP-SEC-006; existing-app migration = ORG gap G-43).
- **Messaging/queues:** transactional event queue default-on; **cds 8 → 10 rolling upgrade can double-process queued messages** — plan downtime, drain the queue, or upgrade through v9 (re-verified verbatim 2026-08-12; owned by **CAP-EVT-002**'s version note — candidate CAP-VER #9 resolved as a merge there); `#succeeded`/`#failed` callbacks still Alpha; Event Hub = "the new default offering" (CAP-EVT-006).
- **Migration tooling:** `cds upgrade` (Node, **alpha** — manual review required); OpenRewrite recipes `com.sap.cds:cds-services-recipes` (Java), migration chain 1.x→…→5.0 (CAP-VER-005).
- **AI/MCP:** MCP protocol adapter beta (CAP-SEC-018); `@cap-js/mcp-server` dev-time assistant without stability label; AI Core plugin/agents/HCQL alpha-beta — re-verify before standardizing (gap G-41).

## Upgrade & migration approach (ORG)

1. Watch monthly release notes; assess impact against the version-sensitive register. (Next known event: the pending cds 10.1 / CAP Java 5.1 minor.)
2. For major upgrades: read the official upgrade/migration guide for the runtime, run official tooling (`cds upgrade` / OpenRewrite recipes), then manually review — SAP itself flags the Node tool as alpha with false positives. Check the release notes for version-specific operational steps (e.g., the queue-drain requirement for cds 8→10, CAP-EVT-002).
3. Update *every* version pin location together: `package.json` engines / `.nvmrc` / `pom.xml`, Dockerfiles, CI matrices, BTP runtime config (per SAP upgrade guide; enforced by CAP-VER-003).
4. Re-run the full test suite and an M8-scoped review after any major upgrade.
5. After EOL of a CAP major without upgrade: freeze dependencies (SAP policy) and treat the situation as a High `CAP-VER-002` finding until resolved.

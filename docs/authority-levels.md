# Authority Levels

Every normative statement in this standard — every rule, principle, and recommendation — carries exactly one authority level. This prevents the single most damaging failure mode of an engineering standard: presenting opinion as vendor requirement.

## The five levels

| Level | Meaning | Evidence required | May be overridden by |
|---|---|---|---|
| **SAP-REQ** | SAP-documented **requirement**: SAP documentation states it as mandatory, or the platform/runtime enforces it | Official SAP page stating the requirement (URL + verification date) | Nothing (only a documented SAP change) |
| **SAP-REC** | SAP-documented **recommendation**: SAP documentation explicitly recommends, advises, or presents it as best practice | Official SAP page containing the recommendation (URL + verification date) | Documented, approved exception |
| **GEN** | **General engineering practice**: widely accepted software engineering practice not specific to CAP and not stated by SAP | Rationale in the rule itself | Documented, approved exception |
| **ORG** | **Organization-specific standard**: our policy, typically filling a gap where SAP is silent (see [research-gaps.md](../references/research-gaps.md)) | Decision recorded in the rule; linked gap entry | Organization decision |
| **AI-REC** | **Claude recommendation**: guidance originating from Claude Code analysis, not yet ratified as ORG policy | Explicit label in the rule/report | Reviewer judgment |

## Usage rules

1. **Default down, never up.** When in doubt between two levels, choose the weaker one. A statement is only `SAP-REQ`/`SAP-REC` if an official SAP page verifiably says so.
2. **Citation or demotion.** A `SAP-REQ`/`SAP-REC` statement without a working official-SAP reference must be demoted to `GEN`/`ORG`/`AI-REC` or removed.
3. **Version-scoped.** SAP guidance changes across CAP versions. `SAP-REQ`/`SAP-REC` statements record the CAP version(s)/date at which they were verified; see [version-management.md](version-management.md).
4. **Severity ≠ authority.** A rule's severity (impact if violated) is independent of its authority (who says so). An `ORG` rule can be Critical; an `SAP-REC` rule can be Low.
5. **In review reports**, findings must state the rule's authority level so readers can distinguish "SAP requires this" from "our organization decided this" from "Claude suggests this".
6. **Official sources only** for SAP levels: `cap.cloud.sap`, `help.sap.com`, official CAP release notes/changelogs, and official SAP repositories referenced by those docs. Blog posts (including SAP Community blogs), Stack Overflow, and third-party books are never sufficient for `SAP-REQ`/`SAP-REC`.

# Remediation Plan Template

Produced by a milestone review for its FAIL findings ([/capm-review-milestone](../.claude/commands/capm-review-milestone.md) step 12). Remediation itself is later development work (`/capm-develop` under the [development model](../development/development-model.md)) followed by re-review of the failed rules — **the review never applies fixes** (AI-REVIEW-012).

```markdown
# Remediation Plan — <project> — <milestone> review of <date>

Source review report: <link>

## Item <n>: <RULE-ID> — <rule title>

| Field | Value |
|---|---|
| **Finding** | What is non-compliant (one sentence) |
| **Evidence** | file:line reference(s) or verified absence, from the review |
| **Gate** | HARD / SOFT / ADVISORY — and whether it currently blocks the milestone |
| **Risk** | Concrete consequence if unremediated (from the rule's rationale, applied to this project) |
| **Recommended change** | CAP-native-first correction (what to change, not yet the diff) |
| **Files likely affected** | Paths |
| **Test required** | The test that will prove the fix (new or existing) |
| **Re-review required** | Yes (rule IDs to re-verify) / covered by full next-gate review |
| **Alternative** | Exception per AI-DOC-002 (owner, compensating controls, expiry) — only where legitimate |

## Sequencing & ownership
Order items (HARD gates first), name owners, note dependencies between fixes.

## Re-review scope
Rule IDs to re-verify + "anything touched by the remediation" (per the re-review mode of /capm-review-milestone).
```

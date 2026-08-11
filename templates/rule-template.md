# Rule Template

Every Layer 2 (`CAP-*`) and Layer 3 (`AI-*`) rule uses this exact structure. All fields are mandatory; write `None` explicitly rather than omitting a field.

```markdown
## <RULE-ID> — <Title>

| Field | Value |
|---|---|
| **Rule ID** | CAP-XXX-000 |
| **Title** | Short imperative title |
| **Category** | One of the categories in standards/rules/README.md |
| **Severity** | Critical / High / Medium / Low |
| **Authority** | SAP-REQ / SAP-REC / GEN / ORG / AI-REC (see docs/authority-levels.md) |
| **Applicability** | When the rule applies (e.g., all projects; only multitenant apps; only OData services) |
| **Runtime** | Node.js / Java / Both |
| **CAP version** | Version range the rule is verified for (e.g., `@sap/cds >= 8`, `CAP Java >= 3`) |
| **Status** | Active / Deprecated (→ successor ID) / Draft |
| **Last verified** | YYYY-MM-DD against the cited SAP reference |

### Rule statement
One or two sentences, testable, phrased as MUST / SHOULD per severity and authority.

### Rationale
Why the rule exists; consequences of violating it. Grounded in the SAP reference where authority is SAP-REQ/SAP-REC.

### Evidence expected in code
What a compliant repository visibly contains (files, annotations, configuration, patterns). This is what a reviewer looks FOR.

### Detection guidance
How to find violations: files/globs to inspect, annotations or API calls to grep for, configurations to read, questions to answer. This is what a reviewer looks AT.

### Good example
Minimal compliant snippet (CDS / JS / Java / YAML as applicable).

### Bad example
Minimal violating snippet, with one line on why it violates the rule.

### Exception guidance
Legitimate reasons not to comply and what a documented exception must record. `None` if no exception is acceptable.

### SAP reference
Official URL(s) supporting the rule. For GEN/ORG/AI-REC authority: `None (authority: ...)` plus any related SAP reading.
```

## Field conventions

- **MUST** is used for Critical/High rules and all SAP-REQ rules; **SHOULD** for Medium/Low recommendations.
- **Runtime** is `Both` only if the rule is meaningful in Node.js *and* Java; otherwise name the runtime and, where a counterpart rule exists for the other runtime, cross-link it.
- **CAP version**: if guidance predates or changes with a CAP release, state the boundary explicitly (e.g., "≥ cds 7: use `@cap-js/sqlite`; older: deprecated `sqlite3` service"). Never write "current version" — name it.
- **Detection guidance** must be executable by a reviewer with repository access only (no runtime environment assumed); note where dynamic verification (running tests, deploying) is additionally required.

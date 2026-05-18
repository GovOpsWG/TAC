# Design: Enterprise Capability Catalog Use-Cases Document

**Spec:** `enterprise-capability-catalog-use-cases`
**Workflow:** Requirements-First
**Output artifact:** `orbit/enterprise-capability-catalog-use-cases.md`
**Companion design:** `orbit/enterprise-capability-catalog-design.md`

---

### Purpose of the use-cases document

The design document answers *what* the system is. The use-cases document answers *how*
it is used. Each use case is a self-contained narrative scenario grounded in:

- The Gemara artifact model (`#CapabilityCatalog`, `#ControlCatalog`,
  `#AssessmentRequirement`, `#EvaluationLog`, `#MappingDocument`, `#Lexicon`)
- The PARC authorization request shape (Principal, Action, Resource, Context)
- The TIGER metric (five independent per-pillar ordinal scores; no aggregate)
- The GovOps toolchain (`govops lint`, `govops coverage`, `govops drift`,
  `govops prove`, `govops tiger`, IGA Exporter)

### Running example domain

All nine use cases share the **Acme Bank payments scenario** established in the seed
design document (§10). The shared domain ensures that artifact excerpts, tool outputs,
and cross-references are internally consistent across use cases.

**Canonical capabilities used throughout:**

| Capability id | Action | Resource | Risk tier | Data sensitivity |
|---|---|---|---|---|
| `payments:read:invoice` | read | Invoice | medium | pii |
| `payments:transfer:bank-account` | transfer | BankAccount | critical | confidential |
| `governance:policy-write:governance-policy` | policy-write | GovernancePolicy | critical | — |
| `iam:assume:role` | assume | Role | high | — |

**Canonical TIGER controls used throughout:**

| Control id | Pillar | Sub-requirements |
|---|---|---|
| `GOVOPS-TR-01` | Transparency | `GOVOPS-TR-01.01`, `GOVOPS-TR-01.02` |
| `GOVOPS-ID-01` | Identity | `GOVOPS-ID-01.01`, `GOVOPS-ID-01.02` |
| `GOVOPS-GV-01` | Governance | `GOVOPS-GV-01.01`, `GOVOPS-GV-01.02`, `GOVOPS-GV-01.03` |
| `GOVOPS-EV-01` | Events | `GOVOPS-EV-01.01`, `GOVOPS-EV-01.02`, `GOVOPS-EV-01.03` |
| `GOVOPS-RS-01` | Resilience | `GOVOPS-RS-01.01`, `GOVOPS-RS-01.02`, `GOVOPS-RS-01.03` |

---

## Architecture

The use-cases document is a **static Markdown document** — it is not executable software.
Its "architecture" is therefore the logical organization of its content and the
consistency rules that govern how artifact excerpts, tool outputs, and cross-references
relate to one another.

### Document layers

```
orbit/enterprise-capability-catalog-use-cases.md
│
├── Executive Summary (≤ 500 words)
├── Table of Contents
├── Persona Summary Table
│
├── UC-01 … UC-09  (nine use case sections)
│   Each section:
│   ├── Context
│   ├── Actors
│   ├── Preconditions
│   ├── Step-by-Step Workflow  (numbered steps + artifact excerpts + tool output)
│   ├── Artifacts Produced or Modified
│   ├── Correctness Properties Demonstrated
│   └── Cross-References
│
└── Summary Table (use case → persona, TIGER pillars, Policy Coverage, commands, design §§)
```

### Consistency model

Each use case is a **closed consistency unit**: every capability id, assessment
requirement id, TIGER score, PARC Context predicate, and tool output shown within a
use case section must be internally consistent with the catalog excerpts shown in that
same section. Cross-use-case consistency is enforced by the shared Acme Bank domain
and the canonical id tables above.

---

## Components and Interfaces

### 9 Use Case Sections

Each use case section is a component of the document with a defined interface:

| Component | Primary persona | Secondary personas | Toolchain commands |
|---|---|---|---|
| UC-01: Capability Catalog Authoring | Platform Security Engineer | — | `govops lint` |
| UC-02: TIGER Control Authoring and Policy Coverage | GRC Practitioner | Platform Security Engineer | `govops coverage` |
| UC-03: Provable Claims and Proof Workflow | Platform Security Engineer | GRC Practitioner | `govops prove` |
| UC-04: TIGER Pillar Score Computation and Reporting | GRC Practitioner | Compliance Auditor | `govops tiger` |
| UC-05: Compliance Mapping and Audit | Compliance Auditor | GRC Practitioner | `govops coverage`, `govops tiger` |
| UC-06: IGA Integration | IGA Team | Platform Security Engineer | `govops drift`, IGA Exporter |
| UC-07: Policy Drift Detection | Platform Security Engineer | Policy Engine Operator | `govops drift` |
| UC-08: Continuous TIGER in CI/CD | Platform Security Engineer | GRC Practitioner | `govops lint`, `govops coverage`, `govops drift`, `govops prove`, `govops tiger` |

> Note: UC-08 is the ninth use case (the table above lists 8 distinct use cases; the
> ninth is UC-08 itself, which exercises the full pipeline). The Policy Engine Operator
> persona appears as a secondary actor in UC-07 and UC-08.

### Section template interface

Every use case section MUST expose the following subsections (the "section interface"):

1. **Context** — narrative motivation (2–4 sentences)
2. **Actors** — primary persona + secondary actors with roles
3. **Preconditions** — bulleted list of what must be true before the workflow starts
4. **Step-by-Step Workflow** — numbered steps; each step that produces or consumes an
   artifact MUST include an indented artifact excerpt or tool output block
5. **Artifacts Produced or Modified** — table of Gemara artifact types and file paths
6. **Correctness Properties Demonstrated** — list of Req 10 properties satisfied
7. **Cross-References** — design §§, Gemara artifact types, TIGER controls, commands

### Artifact excerpt conventions

- YAML excerpts use `# ...` to elide unchanged fields
- Tool output blocks use `$ govops <command>` as the prompt line
- Proof artifacts use JSON
- All ids in excerpts MUST match the canonical id tables in the Overview section

---

## Data Models

### Capability id format

```
<service>:<action>:<resource>
```

- `service`: lowercase kebab-case service or domain name (e.g., `payments`, `iam`, `governance`)
- `action`: lowercase kebab-case action verb from the `#Lexicon` (e.g., `read`, `transfer`, `assume`, `policy-write`)
- `resource`: lowercase kebab-case resource type from the `#Lexicon` (e.g., `invoice`, `bank-account`, `role`, `governance-policy`)

Regex: `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$`

### AssessmentRequirement id format

```
GOVOPS-<PILLAR>-<NN>.<NN>
```

- `PILLAR`: uppercase abbreviation — `TR` (Transparency), `ID` (Identity), `GV` (Governance), `EV` (Events), `RS` (Resilience), `ID` (Identity — contents defined by GovOps Initiative Orbit WG)
- `NN`: two-digit zero-padded integer

Regex: `^GOVOPS-[A-Z]+-[0-9]+\.[0-9]+$`

### TIGER score shape

The `tiger-score.yaml` artifact produced by `govops tiger` contains **five independent
per-pillar ordinal scores**. There is **no aggregate score field**. Each pillar score
is an integer in {1, 2, 3, 4, 5}.

```yaml
# tiger-score.yaml — normative shape for use-cases document examples
metadata:
  id: tiger.<org>.<date>
  generated-at: "<ISO-8601 timestamp>"
  policy-versions:
    - engine: <engine-id>
      version: "<version>"
  catalog-version: "<catalog-version>"
score:
  transparency: <integer 1–5>
  identity: <integer 1–5>
  governance: <integer 1–5>
  events: <integer 1–5>
  resilience: <integer 1–5>
  # NO aggregate field
notes: "<per-pillar gap notes in catalog terms>"
```

> **Critical constraint:** The design document (§9.2) shows a legacy `aggregate` field
> in its worked example. The use-cases document MUST NOT reproduce that field. The
> requirements (Req 5.1, Req 10.3) are explicit: five independent scores, no aggregate.
> The use-cases document uses the normative shape above.

### Policy Coverage shape

Policy Coverage is the **weighted fraction of declared `#AssessmentRequirement`s
currently satisfied** by deployed policy and runtime evidence. It is computed by
`govops coverage` and is **distinct from the TIGER pillar scores**.

```
Policy Coverage = (satisfied requirements, weighted by risk-tier)
                  ─────────────────────────────────────────────────
                  (total declared requirements, weighted by risk-tier)
```

`govops coverage` output reports:
- Overall Policy Coverage percentage
- Per-risk-tier breakdown (critical, high, medium, low)
- Capabilities with no governing `#AssessmentRequirement` (coverage gaps)
- Controls with no current proof or runtime evidence

### Proof artifact shape

```json
{
  "claim-id": "<GOVOPS-PILLAR-NN.NN>",
  "capability-id": "<service>:<action>:<resource>",
  "formula": "<engine-neutral claim statement>",
  "result": "UNSAT | SAT",
  "counterexample": null,
  "analyzer": "<analyzer-id>",
  "policy-hash": "sha256:<hex>",
  "generated-at": "<ISO-8601 timestamp>",
  "signature": "<optional signing key reference>"
}
```

- `result: "UNSAT"` means the negation of the claim is unsatisfiable — the claim holds
  for all reachable states of the deployed policy.
- `result: "SAT"` means a satisfying assignment (counterexample) exists — the claim
  does NOT hold; `counterexample` MUST be non-null.

### EvaluationLog evidence entry shape

```yaml
evidence:
  - type: Proof
    artifact:
      reference-id: proofs/<claim-id>-<date>.json
      hash: sha256:<hex>
    claim-id: <GOVOPS-PILLAR-NN.NN>
    result: UNSAT | SAT
    policy-hash: sha256:<hex>
  - type: DecisionLog
    artifact:
      reference-id: evidence/<log-id>.json
    window: "<ISO-8601 interval>"
    summary: "<human-readable summary>"
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid
executions of a system — essentially, a formal statement about what the system should
do. Properties serve as the bridge between human-readable specifications and
machine-verifiable correctness guarantees.*

*In the context of this design, the "system" is the use-cases document itself. The
properties below are invariants that every example, excerpt, and tool output in the
document must satisfy. They are suitable for property-based testing of the document's
internal consistency using a document-parsing test harness.*

---

### Property 1: Capability ID Format

*For any* capability id string appearing in the use-cases document (in YAML excerpts,
tool output blocks, prose references, or tables), the id MUST match the regex
`^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$` — that is, exactly three
colon-separated lowercase kebab-case segments.

**Validates: Requirements 10.1**

---

### Property 2: AssessmentRequirement ID Format

*For any* `#AssessmentRequirement` id string appearing in the use-cases document
(in YAML excerpts, tool output blocks, prose references, or tables), the id MUST
match the regex `^GOVOPS-[A-Z]+-[0-9]+\.[0-9]+$` — that is, the prefix `GOVOPS-`,
followed by an uppercase pillar abbreviation, a hyphen, a numeric major segment, a
dot, and a numeric minor segment.

**Validates: Requirements 10.2**

---

### Property 3: TIGER Score Integrity

*For any* TIGER score example appearing in the use-cases document:

1. The score object MUST contain exactly five per-pillar score fields: `transparency`,
   `identity`, `governance`, `events`, `resilience`.
2. Each per-pillar score MUST be an integer in {1, 2, 3, 4, 5}.
3. The score object MUST NOT contain an `aggregate` field or any single combined
   TIGER number.
4. Policy Coverage (a percentage or fraction produced by `govops coverage`) MUST be
   presented as a separate metric and MUST NOT appear as a field within the TIGER
   score object.

**Validates: Requirements 5.1, 10.3, 3.3**

---

### Property 4: PARC Context Predicate Consistency

*For any* use case section that shows both a capability excerpt (containing a
`documented-context-expectations` field) and a PARC Context predicate (in a proof
claim, a policy excerpt, or a tool output), the predicate MUST be logically consistent
with the `documented-context-expectations` of the capability shown in the same use
case section.

Specifically: if a capability's `documented-context-expectations` states
`context.acr == 'urn:mfa'`, then any PARC Context predicate in the same use case
that references that capability MUST use `context.acr` with the value `urn:mfa` (or
a strictly stronger assertion). No use case may show a PARC Context predicate that
contradicts the capability's documented expectations.

**Validates: Requirements 10.4**

---

### Property 5: Proof Logical Consistency

*For any* proof result shown in the use-cases document:

1. If the result is `UNSAT`, the accompanying claim statement MUST be phrased such
   that its negation is the formula being checked for satisfiability — i.e., the
   claim holds because no satisfying assignment exists for the negation.
2. If the result is `SAT`, a concrete counterexample MUST be shown — a specific
   assignment of values to PARC fields (Principal, Action, Resource, Context) that
   satisfies the negation of the claim, demonstrating that the claim does NOT hold
   for the deployed policy.
3. No use case may show `result: UNSAT` alongside a counterexample, nor `result: SAT`
   without a counterexample.

**Validates: Requirements 10.5, 4.4**

---

### Property 6: Lint and Coverage Output Consistency

*For any* use case section that shows `govops lint` or `govops coverage` output, every
capability id and applicability-group id referenced in that tool output MUST appear in
the catalog excerpt shown in the same use case section. The tool output MUST NOT
reference phantom capability ids or group ids that are absent from the catalog excerpt.

**Validates: Requirements 10.7, 2.3, 3.3**

---

### Property 7: Use Case Section Structure Completeness

*For any* use case section in the use-cases document, the section MUST contain all
seven required subsections in order: Context, Actors, Preconditions, Step-by-Step
Workflow, Artifacts Produced or Modified, Correctness Properties Demonstrated, and
Cross-References. No use case section may omit any of these subsections.

**Validates: Requirements 11.5, 1.2**

---

### Property 8: Identity Pillar Non-Prescription

*For any* use case section that references the Identity pillar (TIGER pillar I) or
shows an Identity pillar score, the section MUST include a note stating that the
scoring criteria and control catalog contents for the Identity pillar are defined by
the GovOps Initiative Orbit WG and are not prescribed by this document. No use case
may present Identity pillar scores as authoritative or prescribe the contents of
`GovOps-Identity.yaml`.

**Validates: Requirements 3.6, 5.7**

---

## Error Handling

The use-cases document is a static Markdown document, not executable software. "Error
handling" in this context means how the document handles scenarios where things go
wrong in the workflows it describes. Each use case that involves a failure mode MUST
show:

1. **The failure signal** — what the tool outputs or what artifact state indicates the
   problem (e.g., `govops lint` exit code 1 with a specific error message, a `SAT`
   proof result with a counterexample, a stale proof hash mismatch).
2. **The diagnosis step** — how the persona identifies the root cause from the failure
   signal.
3. **The remediation step** — what the persona does to resolve the failure and return
   the system to a valid state.

### Failure modes covered per use case

| Use case | Failure mode | Signal | Remediation |
|---|---|---|---|
| UC-01 | Lexicon resolution failure | `govops lint` error: unknown action/resource term | Add term to `#Lexicon`, re-run lint |
| UC-02 | Coverage gap: critical capability with no control | `govops coverage` gap report | Author `#AssessmentRequirement` in relevant pillar catalog |
| UC-03 | Proof returns SAT (counterexample) | `govops prove` SAT result with counterexample | Fix deployed policy, re-run `govops prove` |
| UC-03 | Stale proof (policy hash mismatch) | `govops prove` stale-proof warning | Re-run `govops prove` against current policy |
| UC-04 | Pillar score decrease after capability added | `govops tiger` score drop with gap note | Author control for new capability |
| UC-04 | Pillar score decrease after proof invalidated | `govops tiger` score drop with stale-proof note | Re-run `govops prove`, refresh evaluation log |
| UC-07 | Drift: catalog entry with no policy rule | `govops drift` type-A report | Update policy or accept exception |
| UC-07 | Drift: policy rule with no catalog entry | `govops drift` type-B report | Add capability to catalog or remove policy rule |
| UC-07 | Drift: context-expectations diverge | `govops drift` type-C report | Align policy or update documented expectations |
| UC-08 | Analyzer unavailable during CI/CD run | Pipeline step failure with degraded-mode flag | Fail-closed for critical capabilities; degrade gracefully for others; affected pillar score reflects missing proof |

---

## Testing Strategy

### Applicability of property-based testing

This feature produces a **static Markdown document**, not executable application code.
However, the document contains structured content (YAML excerpts, JSON blocks, tool
output blocks, id strings) that can be parsed and validated programmatically. Property-
based testing is applicable to the **internal consistency of the document's structured
content** — specifically, the six correctness properties (P1–P6) that make universal
claims about all instances of a pattern in the document.

Property P7 (section structure completeness) and P8 (Identity pillar non-prescription)
are also testable as properties over the document's section structure.

### Property-based testing approach

**Library:** Use a property-based testing library appropriate for the implementation
language of the document validation harness. For Python: `hypothesis`. For TypeScript:
`fast-check`. For Haskell: `QuickCheck`.

**Test harness design:** The test harness parses the use-cases document and extracts:
- All capability id strings (from YAML `id:` fields, prose, and tool output blocks)
- All `#AssessmentRequirement` id strings
- All TIGER score objects (from YAML blocks)
- All capability excerpts with `documented-context-expectations` fields
- All proof result blocks
- All `govops lint` / `govops coverage` output blocks paired with their catalog excerpts
- All use case section structures (subsection headings)
- All Identity pillar references

Each property test runs the relevant extractor and asserts the property over all
extracted instances. Minimum 100 iterations is not applicable here (the document is
fixed, not randomly generated); instead, each property test exhaustively checks all
instances in the document.

**Tag format for property tests:**
```
# Feature: enterprise-capability-catalog-use-cases, Property N: <property_text>
```

### Unit tests (example-based)

Unit tests verify specific content requirements that are not universal properties:

- UC-01 contains both profile A (convention-only) and profile B (schema extension) examples
- UC-01 shows lexicon definition before capability entry authoring
- UC-01 shows both passing and failing `govops lint` output
- UC-02 shows both configuration-assertion and provable-claim `#AssessmentRequirement` for the same capability
- UC-02 shows Policy Coverage as distinct from TIGER pillar scores
- UC-03 shows a complete provable claim with capability id, PARC Context predicate, and policy reference
- UC-03 shows `#EvaluationLog` update with `#ArtifactMapping` including policy hash
- UC-03 shows proof workflow for at least one PDP class (Cedar or OpenFGA)
- UC-04 shows a `tiger-score.yaml` sample with five per-pillar scores and maturity level semantics
- UC-04 shows a `tiger-evidence.json` sample with per-pillar inputs and notes
- UC-05 shows a `#MappingDocument` linking to NIST 800-53 AC-3 and ISO 27001 A.5.15
- UC-05 shows the full evidence trace from compliance control id to proof artifact
- UC-06 shows IGA Exporter output in at least one format (CSV, SCIM, or OSCAL)
- UC-07 shows all three drift scenarios (type A, B, C)
- UC-08 shows the pipeline sequence: lint → coverage → drift → prove → tiger
- UC-08 shows gate conditions for all four failure modes
- Document contains executive summary ≤ 500 words
- Document contains persona summary table
- Document contains cross-reference table
- Document contains summary table with all required columns

### Integration tests

Not applicable — the use-cases document is a static artifact. Integration testing
of the GovOps toolchain itself is out of scope for this spec.

### Review checklist

Before the use-cases document is published, a reviewer MUST verify:

1. All nine use cases are present and follow the section template (P7)
2. All capability ids match the format regex (P1)
3. All `#AssessmentRequirement` ids match the format regex (P2)
4. All TIGER score examples have five per-pillar integer scores and no aggregate (P3)
5. All PARC Context predicates are consistent with capability `documented-context-expectations` (P4)
6. All proof results are logically consistent (P5)
7. All tool output blocks reference only ids present in the same-section catalog excerpt (P6)
8. All Identity pillar references include the WG-ownership note (P8)
9. Executive summary is ≤ 500 words
10. Cross-reference table and summary table are present and complete

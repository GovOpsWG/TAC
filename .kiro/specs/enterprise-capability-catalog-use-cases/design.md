# Design: Enterprise Capability Catalog Use-Cases Document

**Spec:** `enterprise-capability-catalog-use-cases`
**Workflow:** Requirements-First
**Output artifact:** `orbit/enterprise-capability-catalog-use-cases.md`
**Companion design:** `orbit/enterprise-capability-catalog-design.md` (§8 worked example, §10 tooling)

---

## Purpose

The design document answers *what* the system is. The use-cases document answers *how* it is used. Each use case is a self-contained narrative scenario grounded in:

- The Gemara artifact model (`#CapabilityCatalog`, `#MappingDocument`, `#Lexicon`)
- The PARC authorization request shape (Principal, Action, Resource, Context)
- The GovOps toolchain (`govops lint`, `govops drift`, IGA Exporter)

The focus is the **`GovOps-AC` capability catalog** — how it is authored, validated, mapped to compliance frameworks, fed into IGA platforms, and kept aligned with deployed policy.

---

## Running Example Domain

All use cases share the **Acme Bank payments scenario** from the seed design document (§10). The shared domain ensures artifact excerpts and tool outputs are internally consistent across use cases.

**Canonical capabilities used throughout:**

| Capability id | Action | Resource | Risk tier | Data sensitivity |
|---|---|---|---|---|
| `payments:read:invoice` | read | Invoice | medium | pii |
| `payments:transfer:bank-account` | transfer | BankAccount | critical | confidential |
| `lending:approve:loan` | approve | Loan | critical | confidential |
| `fraud:flag:transaction` | flag | Transaction | high | internal |

---

## Architecture

The use-cases document is a **static Markdown document**. Its architecture is the logical organization of its content and the consistency rules that govern how artifact excerpts and tool outputs relate to one another.

### Document structure

```
orbit/enterprise-capability-catalog-use-cases.md
│
├── Executive Summary (≤ 300 words)
├── Table of Contents
├── Persona Summary Table
│
├── UC-01: Capability Catalog Authoring
├── UC-02: Compliance Mapping and Audit
├── UC-03: IGA Integration
├── UC-04: Policy Drift Detection
├── UC-05: Continuous Governance in CI/CD
├── UC-06: Automated Fraud Flagging (Fraud System)
│
│   Each section:
│   ├── Context
│   ├── Actors
│   ├── Preconditions
│   ├── Step-by-Step Workflow
│   ├── Artifacts Produced or Modified
│   ├── Correctness Properties Demonstrated
│   └── Cross-References
│
└── Summary Table (use case → persona, commands, design §§)
```

### Consistency model

Each use case is a **closed consistency unit**: every capability id, PARC Context predicate, and tool output shown within a section must be internally consistent with the catalog excerpts shown in that same section. Cross-use-case consistency is enforced by the shared Acme Bank domain and the canonical capability table above.

---

## Components

### Use Case Sections

| Use case | Primary persona | Secondary personas | Toolchain commands |
|---|---|---|---|
| UC-01: Capability Catalog Authoring | Platform Security Engineer | — | `govops lint` |
| UC-02: Compliance Mapping and Audit | Compliance Auditor | Lending Officer | — |
| UC-03: IGA Integration | IGA Team | Platform Security Engineer | IGA Exporter, `govops drift` |
| UC-04: Policy Drift Detection | Platform Security Engineer | Policy Engine Operator | `govops drift` |
| UC-05: Continuous Governance in CI/CD | Platform Security Engineer | Lending Officer | `govops lint`, `govops drift` |
| UC-06: Automated Fraud Flagging (Fraud System) | Fraud System (non-human) | Platform Security Engineer, Policy Engine Operator | `govops drift` |

### Section template

Every use case section MUST contain these subsections in order:

1. **Context** — narrative motivation (2–4 sentences)
2. **Actors** — primary persona + secondary actors with roles
3. **Preconditions** — bulleted list of what must be true before the workflow starts
4. **Step-by-Step Workflow** — numbered steps; each step that produces or consumes an artifact MUST include an indented artifact excerpt or tool output block
5. **Artifacts Produced or Modified** — table of Gemara artifact types and file paths
6. **Correctness Properties Demonstrated** — which properties (P1–P3) this use case satisfies
7. **Cross-References** — design §§, Gemara artifact types, toolchain commands

### Artifact excerpt conventions

- YAML excerpts use `# ...` to elide unchanged fields
- Tool output blocks use `$ govops <command>` as the prompt line
- All capability ids in excerpts MUST match the canonical table above

---

## Data Models

### Capability id format

```
<service>:<action>:<resource>
```

- `service`: lowercase kebab-case domain name (e.g., `payments`, `lending`, `fraud`)
- `action`: lowercase kebab-case verb from the `#Lexicon` (e.g., `read`, `transfer`, `approve`, `flag`, `escalate`, `batch-transfer`)
- `resource`: lowercase kebab-case resource type from the `#Lexicon` (e.g., `invoice`, `bank-account`, `loan`, `transaction`, `invoice-summary`)

Extended capabilities (`payments:batch-transfer:bank-account`, `payments:read:invoice-summary`, `fraud:escalate:transaction`) appear from UC-03 onward; see **catalog state by use case** in the use-cases document.

Regex: `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$`

### `GovOps-AC.yaml` shape

The capability catalog is a `#CapabilityCatalog` artifact. Each entry is either a convention-only `#Capability` (Profile A) or a `#EnterpriseCapability` schema extension (Profile B).

**Profile A** (convention-only, no schema change):
```yaml
- id: <service>:<action>:<resource>
  title: <human label>
  group: <group-id>
  description: |
    ---
    action: <Action>
    resource: <ResourceType>
    data-sensitivity: <value>
    risk-tier: <low|medium|high|critical>
    ---
    <prose description>
```

**Profile B** (`#EnterpriseCapability` schema extension):
```yaml
- id: <service>:<action>:<resource>
  title: <human label>
  group: <group-id>
  action: <action>
  resource: <ResourceType>
  documented-context-expectations:
    - id: <predicate-id>
      text: <human-readable expectation>
      expression: <optional machine-readable form>
      expression-language: natural-language
  data-sensitivity: <value>
  risk-tier: <low|medium|high|critical>
  description: <prose>
```

### `#MappingDocument` shape

Maps capability entries or GovOps controls to external framework controls:

```yaml
metadata:
  id: map.<org>.<frameworks>
  type: MappingDocument
  mapping-references:
    - id: ec
      title: <catalog title>
    - id: <framework-id>
      title: <framework title>
mappings:
  - id: <mapping-id>
    source: <capability-id or control-id>
    relationship: implements | supports | related
    targets:
      - entry-id: <framework-control-id>
        rationale: <text>
        confidence-level: High | Medium | Low
```

### Drift report shape

`govops drift` produces a structured report with three finding types:

| Type | Meaning |
|---|---|
| A | Capability in catalog with no corresponding policy rule |
| B | Policy rule with no corresponding catalog entry |
| C | Capability whose `documented-context-expectations` diverge from deployed policy |

---

## Correctness Properties

The properties below are invariants that every example, excerpt, and tool output in the use-cases document must satisfy.

### Property 1: Capability ID Format

For any capability id string appearing in the use-cases document, the id MUST match `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$` — exactly three colon-separated lowercase kebab-case segments.

**Validates: Requirement 7.1**

### Property 2: PARC Context Predicate Consistency

For any use case section that shows both a capability excerpt (with a `documented-context-expectations` field) and a PARC Context predicate, the predicate MUST be logically consistent with the `documented-context-expectations` of the capability shown in the same section.

**Validates: Requirement 7.2**

### Property 3: Lint and Drift Output Consistency

For any use case section that shows `govops lint` or `govops drift` output, every capability id referenced in that output MUST appear in the catalog excerpt shown in the same section. No phantom ids.

**Validates: Requirement 7.3**

### Property 4: Section Structure Completeness

Every use case section MUST contain all seven required subsections in order: Context, Actors, Preconditions, Step-by-Step Workflow, Artifacts Produced or Modified, Correctness Properties Demonstrated, and Cross-References.

**Validates: Requirement 8.5**

---

## Error Handling

Each use case that involves a failure mode MUST show: the failure signal, the diagnosis step, and the remediation step.

| Use case | Failure mode | Signal | Remediation |
|---|---|---|---|
| UC-01 | Lexicon resolution failure | `govops lint` error: unknown action/resource term | Add term to `#Lexicon`, re-run lint |
| UC-04 | Drift: catalog entry with no policy rule | `govops drift` Type-A finding | Update policy or mark capability as design-time |
| UC-04 | Drift: policy rule with no catalog entry | `govops drift` Type-B finding | Add capability to catalog or remove policy rule |
| UC-04 | Drift: context-expectations diverge | `govops drift` Type-C finding | Align policy or update documented expectations |
| UC-05 | Drift detected in CI/CD | Pipeline gate failure | Resolve drift before merging |

---

## Testing Strategy

The use-cases document is a static Markdown document. Testing validates the internal consistency of its structured content.

### Property tests (P1–P4)

A document-parsing harness extracts all capability ids, PARC Context predicates, tool output blocks, and section headings, then asserts each property exhaustively over all instances found.

### Content checklist (unit tests)

- UC-01 shows both Profile A and Profile B capability entries
- UC-01 shows lexicon definition before capability authoring
- UC-01 shows both passing and failing `govops lint` output
- UC-02 shows a `#MappingDocument` linking to NIST 800-53 AC-3 and ISO 27001 A.5.15
- UC-02 shows the trace from compliance control id to capability entry
- UC-03 shows IGA Exporter output in at least one format (CSV, SCIM, or OSCAL)
- UC-03 shows capability deprecation and IGA catalog update
- UC-04 shows all three drift scenarios (Type A, B, C)
- UC-04 shows multi-engine drift composition
- UC-05 shows the pipeline sequence: lint → drift
- UC-05 shows gate conditions for lint errors and drift findings
- UC-06 shows Fraud System as non-human `software-agent` principal on `fraud:flag:transaction`
- UC-06 shows PARC request with Context consistent with documented-context-expectations
- UC-06 contrasts automated vs. human invocation of the same capability id
- Document contains executive summary ≤ 300 words
- Document contains persona summary table
- Document contains cross-reference table and summary table

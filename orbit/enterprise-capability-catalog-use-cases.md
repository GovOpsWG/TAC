# Enterprise Capability Catalog: Use Cases

**Status:** Draft for discussion

**Audience:** Platform Security Engineers, IGA Teams, Compliance Auditors, Policy Engine Operators

**Companion design:** [Cataloging Enterprise Capabilities with Gemara](./enterprise-capability-catalog-design.md)

---

## Executive Summary

This document shows how enterprise personas use the **GovOps authorization capability catalog** (`GovOps-AC`) and Phase 1 toolchain in real workflows. The design document specifies the artifact model; this document shows **how** it is used.

Six use cases at **Acme Bank** share a payments, lending, and fraud authorization domain. **UC-01** establishes four base capabilities; later use cases show catalog evolution (additional ids, deprecations) as called out below.

| Capability id | Domain | Risk (initial) | Role in scenario |
|---|---|---|---|
| `payments:read:invoice` | Payments | medium | Profile A example; deprecated in UC-03 |
| `payments:transfer:bank-account` | Payments | critical | High-value transfer; MFA in context |
| `lending:approve:loan` | Lending | critical | Human approvers; SoD in context |
| `fraud:flag:transaction` | Fraud | high | Fraud System + analysts (UC-06) |

Governance metadata (risk-tier, sensitivity, documented context expectations) lives **on capability entries** — not in a separate metrics framework.

Personas: **Platform Security Engineer** (UC-01, UC-04, UC-05), **Compliance Auditor** (UC-02), **IGA Team** (UC-03), **Lending Officer** (UC-02, UC-05), **Fraud System** (non-human actor, UC-06), and **Policy Engine Operator** (secondary in several flows).

Tooling in scope: `govops lint`, `govops drift`, IGA exporter. CI/CD runs lint and drift gates (UC-05).

---

## Table of Contents

1. [Persona Summary](#persona-summary)
2. [UC-01: Capability Catalog Authoring](#uc-01-capability-catalog-authoring)
3. [UC-02: Compliance Mapping and Audit](#uc-02-compliance-mapping-and-audit)
4. [UC-03: IGA Integration](#uc-03-iga-integration)
5. [UC-04: Policy Drift Detection](#uc-04-policy-drift-detection)
6. [UC-05: Continuous Governance in CI/CD](#uc-05-continuous-governance-in-cicd)
7. [UC-06: Automated Fraud Flagging (Fraud System)](#uc-06-automated-fraud-flagging-fraud-system)
8. [Summary Table](#summary-table)

---

## Persona Summary

| Persona | Role | Use Cases (primary) | Use Cases (secondary) |
|---|---|---|---|
| Platform Security Engineer | Deploys PDPs and maintains the GovOps repository | UC-01, UC-04, UC-05 | UC-03 |
| Compliance Auditor | NIST / ISO / SOC 2 conformance | UC-02 | — |
| Lending Officer | Loan approval policy and lending capability stewardship | — | UC-02, UC-05 |
| IGA Team | Access reviews and entitlement lifecycle | UC-03 | UC-04 |
| Fraud System | Non-human fraud detection service (software agent) | UC-06 | — |
| Policy Engine Operator | PDP policy lifecycle | — | UC-04, UC-05, UC-06 |

**Catalog state by use case** (avoid mixing excerpts from different points in time):

| Use case | `GovOps-AC` state |
|---|---|
| UC-01, UC-02, UC-06 | Four base capabilities only |
| UC-03 | Adds `payments:batch-transfer:bank-account`, `payments:read:invoice-summary`; deprecates `payments:read:invoice` |
| UC-04, UC-05 | Assumes UC-03 catalog plus `fraud:escalate:transaction` (UC-04) |

---

## UC-01: Capability Catalog Authoring

### Context

The enterprise must have a well-formed capability catalog before mappings, IGA export, or drift checks can run. This use case shows how a Platform Security
Engineer builds `GovOps-AC.yaml` from scratch: defining canonical vocabulary in the
`#Lexicon`, authoring capability entries in both supported profiles, assigning
applicability groups, and validating the result with `govops lint`. Getting the catalog
right at this stage is the foundation for every downstream workflow.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank — payments, lending, and fraud surfaces)
- **Secondary:** None

### Preconditions

- The `govops/` repository directory exists and is version-controlled.
- The GovOps toolchain (`govops lint`) is installed and on the `PATH`.
- The engineer has identified the set of (action, resource) pairs Acme exposes across
  payments, lending, and fraud services.
- No `GovOps-AC.yaml` or `lexicon.yaml` exists yet in the repository.

### Step-by-Step Workflow

**Step 1 — Define canonical vocabulary in the `#Lexicon`.**

The engineer creates `govops/lexicon.yaml`. Every action verb and resource type name
that will appear in capability ids MUST be defined here before being referenced in
`GovOps-AC.yaml`. This is the single source of truth for canonical terms; `govops lint`
will reject any capability whose `action` or `resource` does not resolve to a lexicon
term id.

```yaml
# govops/lexicon.yaml
title: GovOps Action and Resource Lexicon
metadata:
  id: lex.govops.actions-resources
  type: Lexicon
  gemara-version: "0.x"
  description: Canonical verbs and resource type names for Acme enterprise capabilities.
  author: { id: acme-platform-security, name: Acme Platform Security, type: Software Assisted }
terms:
  - id: action.read
    title: read
    definition: Retrieve a resource by identifier without modifying it.
    synonyms: [get, fetch, view]
  - id: action.transfer
    title: transfer
    definition: Move value or ownership from a source resource to a target resource.
    synonyms: [send, move, remit]
  - id: action.approve
    title: approve
    definition: Grant final authorization for a lending decision on a loan application.
    synonyms: [authorize, sign-off]
  - id: action.flag
    title: flag
    definition: Mark a transaction for fraud review or investigation.
    synonyms: [mark-suspicious, alert]
  - id: resource.invoice
    title: Invoice
    definition: A demand for payment issued by the payments service.
  - id: resource.bank-account
    title: BankAccount
    definition: A demand-deposit account at a financial institution.
  - id: resource.loan
    title: Loan
    definition: A loan application or facility in the lending domain.
  - id: resource.transaction
    title: Transaction
    definition: A payment or money-movement event subject to fraud monitoring.
  # Terms below are added when catalog evolves in UC-03 / UC-04 (not used in initial lint pass)
  - id: action.batch-transfer
    title: batch-transfer
    definition: Transfer funds via the batch payments rail (distinct from interactive transfer).
  - id: action.escalate
    title: escalate
    definition: Escalate a flagged transaction to a senior fraud queue.
  - id: resource.invoice-summary
    title: InvoiceSummary
    definition: Invoice header fields only (reduced PII vs. full Invoice).
  - id: resource.payment-order
    title: PaymentOrder
    definition: A payment order awaiting settlement in the payments rail.
```

**Step 2 — Author the capability catalog using the convention-only profile (Profile A).**

The engineer creates `govops/GovOps-AC.yaml` and adds the first capability entry using
the convention-only profile (design §6.2(A)). This profile requires no schema change
and validates against the stable `#CapabilityCatalog` today. The structured payload is
encoded in the `id` field and a YAML front-matter block inside `description`.

```yaml
# govops/GovOps-AC.yaml  (excerpt — Profile A entry)
title: Acme Enterprise Capability Catalog
metadata:
  id: cat.acme.ec
  type: CapabilityCatalog
  gemara-version: "0.x"
  version: "2026.1"
  date: "2026-05-01T00:00:00Z"
  description: (action, resource) capabilities across Acme services.
  author:
    id: acme-platform-security
    name: Acme Platform Security
    type: Software Assisted
  lexicon:
    reference-id: lex.govops.actions-resources
  applicability-groups:
    - id: sensitivity
      title: Data Sensitivity
      description: Classification of data exposed by the action.
      values: [public, internal, confidential, pii, regulated]
    - id: risk-tier
      title: Risk Tier
      description: Coarse risk classification used for IGA scoping and compliance queries.
      values: [low, medium, high, critical]
    - id: lifecycle-stage
      title: Lifecycle Stage
      description: When in the SDLC this capability is exercised.
      values: [design-time, build-time, deploy-time, runtime]
groups:
  - id: g.payments
    title: Payments
    description: Capabilities exposed by the payments domain.
  - id: g.lending
    title: Lending
    description: Capabilities exposed by the lending domain.
  - id: g.fraud
    title: Fraud
    description: Capabilities exposed by the fraud detection service.
capabilities:
  # --- Profile A: convention-only (no schema change required) ---
  - id: payments:read:invoice
    title: Read invoice
    group: g.payments
    description: |
      ---
      action: read
      resource: Invoice
      documentation:
        typical-requester-classes: [interactive-subject, software-agent, autonomous-actor]
      data-sensitivity: pii
      risk-tier: medium
      lifecycle-stage: runtime
      ---
      Retrieve an existing Invoice by id. Returns invoice header and line items.
      Callers must be authenticated; no additional context requirements documented.
```

**Step 3 — Author a capability entry using the schema extension profile (Profile B).**

The engineer adds the `payments:transfer:bank-account` capability using the
`#EnterpriseCapability` schema extension (design §6.2(B)). This profile makes
`action`, `resource`, and optional fields first-class typed fields rather than
front-matter strings. It is the recommended profile once the GovOps WG overlay package
is available.

```yaml
  # --- Profile B: #EnterpriseCapability schema extension ---
  - id: payments:transfer:bank-account
    title: Transfer funds
    group: g.payments
    action: transfer
    resource: BankAccount
    documented-requester-classes: [interactive-subject, software-agent]
    documented-context-expectations:
      - id: c.mfa
        text: "Policy MUST require strong authentication evidence in PARC Context"
        expression: "context.acr == 'urn:mfa'"
        expression-language: natural-language
    engine-bindings:
      - engine: cedar
        policy-artifact: govops/policies/payments-cedar.cedar
    data-sensitivity: confidential
    risk-tier: critical
    lifecycle-stage: runtime
    description: >
      Initiate a funds transfer between bank accounts in the payments domain.
      This is the highest-risk payments capability.

  - id: lending:approve:loan
    title: Approve loan
    group: g.lending
    action: approve
    resource: Loan
    documented-requester-classes: [interactive-subject]
    documented-context-expectations:
      - id: c.approvals
        text: "Two distinct approvers required in PARC Context"
        expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
        expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical
    lifecycle-stage: runtime
    description: >
      Approve a loan application or facility change. Lending Officers steward this
      capability; requires separation of duties.

  - id: fraud:flag:transaction
    title: Flag transaction
    group: g.fraud
    action: flag
    resource: Transaction
    documented-requester-classes: [software-agent, interactive-subject]
    documented-context-expectations:
      - id: c.risk-score
        text: "Automated flags MUST include model risk score in PARC Context"
        expression: "context.fraud.risk-score >= 0.85"
        expression-language: natural-language
    engine-bindings:
      - engine: opa
        policy-artifact: govops/policies/fraud-detection-opa.rego
    data-sensitivity: internal
    risk-tier: high
    lifecycle-stage: runtime
    description: >
      Flag a payment transaction for fraud review. Primary non-human requester is the
      Acme Fraud System (batch scoring pipeline).
```

**Step 4 — Run `govops lint` (passing case).**

With the lexicon and catalog in place, the engineer runs `govops lint` to validate
lexicon resolution, group membership, and engine-binding well-formedness.

```
$ govops lint govops/GovOps-AC.yaml --lexicon govops/lexicon.yaml

govops lint v0.9.0
Checking: govops/GovOps-AC.yaml

  ✓ Lexicon resolved: action.read       → payments:read:invoice
  ✓ Lexicon resolved: action.transfer   → payments:transfer:bank-account
  ✓ Lexicon resolved: action.approve    → lending:approve:loan
  ✓ Lexicon resolved: action.flag       → fraud:flag:transaction
  ✓ Lexicon resolved: resource.invoice  → payments:read:invoice
  ✓ Lexicon resolved: resource.bank-account → payments:transfer:bank-account
  ✓ Lexicon resolved: resource.loan     → lending:approve:loan
  ✓ Lexicon resolved: resource.transaction → fraud:flag:transaction
  ✓ Group membership: all capabilities reference defined groups
  ✓ Applicability groups: all values within declared group value sets
  ✓ No engine-binding well-formedness errors

4 capabilities validated. 0 errors. 0 warnings.
```

**Step 5 — Introduce a lexicon resolution failure and observe the lint error.**

The engineer temporarily adds a capability that references an action term not in the
lexicon — `payments:settle:payment-order` — where `settle` has not been defined.

```yaml
  # Intentionally broken entry (not in final catalog)
  - id: payments:settle:payment-order
    title: Settle payment order
    group: g.payments
    action: settle
    resource: PaymentOrder
    risk-tier: high
```

Running `govops lint` surfaces the violation:

```
$ govops lint govops/GovOps-AC.yaml --lexicon govops/lexicon.yaml

govops lint v0.9.0
Checking: govops/GovOps-AC.yaml

  ✓ Lexicon resolved: action.read       → payments:read:invoice
  ✓ Lexicon resolved: action.transfer   → payments:transfer:bank-account
  ✓ Lexicon resolved: action.approve    → lending:approve:loan
  ✓ Lexicon resolved: action.flag       → fraud:flag:transaction
  ✗ Lexicon resolution FAILED: action 'settle' not found in lex.govops.actions-resources
      capability: payments:settle:payment-order
      suggestion: Did you mean 'transfer'? Or add action.settle to lexicon.yaml.
  ✗ Lexicon resolution FAILED: resource 'PaymentOrder' not found in lex.govops.actions-resources
      capability: payments:settle:payment-order
      suggestion: Add resource.payment-order to lexicon.yaml.

5 capabilities checked. 2 errors. 0 warnings.
EXIT 1
```

**Resolution:** The engineer adds `action.settle` and `resource.payment-order` to
`lexicon.yaml`, then re-runs `govops lint` to confirm the error is resolved. The
broken entry is removed from this use case's final catalog (it was only a lint exercise).

**Step 6 — Verify applicability-group assignments drive downstream use.**

The engineer confirms that `risk-tier` and `sensitivity` assignments will scope IGA access reviews (UC-03) and compliance mapping queries (UC-02). `govops lint` only checks that values are within declared applicability-group value sets.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Lexicon | `#Lexicon` | `govops/lexicon.yaml` | Created; defines canonical action verbs and resource type names |
| Capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | Created; 4 capabilities across 3 groups |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All four capability ids (`payments:read:invoice`,
  `payments:transfer:bank-account`, `lending:approve:loan`,
  `fraud:flag:transaction`) match `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$`.
- **P3 (Lint Output Consistency):** The `govops lint` passing output references only
  the four capability ids present in the catalog excerpt above; the failing output
  references only `payments:settle:payment-order`, which appears in the broken entry
  shown in the same step.
- **P4 (Section Structure):** This section contains all seven required subsections.

### Cross-References

| Item | Reference |
|---|---|
| Convention-only profile (Profile A) | Design §6.2(A) |
| Schema extension profile (Profile B) | Design §6.2(B) |
| `#EnterpriseCapability` CUE definition | Design §6.2(B) |
| `#Lexicon` artifact | Design §6.4 |
| `#Group` and applicability-groups | Design §6.3 |
| `govops lint` tool | Design §10 |
| Gemara artifact: `#CapabilityCatalog` | ADR-0019 |
| Gemara artifact: `#Lexicon` | ADR-0021 |


## UC-02: Compliance Mapping and Audit

### Context

Auditors need to trace from framework controls (NIST 800-53, ISO 27001) to **capability ids** in `GovOps-AC`, not through a parallel control or metrics layer.

### Actors

- **Primary:** Compliance Auditor (Acme Bank)
- **Secondary:** Lending Officer

### Preconditions

- UC-01 complete: `GovOps-AC.yaml` passes `govops lint`.
- `govops/mappings/acme-capabilities-nist-iso.yaml` exists.

### Step-by-Step Workflow

**Step 1 — Review `#MappingDocument` from capabilities to frameworks.**

```yaml
# govops/mappings/acme-capabilities-nist-iso.yaml (excerpt)
metadata:
  id: map.acme.ec.nist-iso
  type: MappingDocument
source-reference:
  reference-id: ec
  entry-type: Capability
target-reference:
  reference-id: nist80053r5
  entry-type: Control
mappings:
  - id: m.transfer.ac3
    source: payments:transfer:bank-account
    relationship: implements
    targets:
      - entry-id: AC-3
        rationale: Critical transfer capability; MFA in documented-context-expectations.
      - entry-id: IA-2(1)
  - id: m.lending.ac5
    source: lending:approve:loan
    relationship: implements
    targets:
      - entry-id: AC-5
        rationale: Separation of duties via documented approver context.
```

**Step 2 — Query: high-risk capabilities without AC-3 mapping.**

Manual review or catalog query: `fraud:flag:transaction` (`risk-tier: high`) has no mapping to AC-3 while `payments:transfer:bank-account` and `lending:approve:loan` do. The **Lending Officer** adds an AC-3 mapping for `fraud:flag:transaction` (e.g., to AU-6 automated monitoring) or documents risk acceptance before audit close.

**Step 3 — Evidence trace for a capability.**

For NIST AC-3 on `payments:transfer:bank-account`:

```
NIST AC-3
  → MappingDocument m.transfer.ac3
  → Capability payments:transfer:bank-account
  → risk-tier: critical, documented-context-expectations (context.acr == urn:mfa)
  → (optional) policies/payments-cedar.cedar verified by govops drift (UC-04)
```

The auditor relies on **catalog fields** and mapping rationale; separate pillar scorecards are not used.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path |
|---|---|---|
| Compliance mapping | `#MappingDocument` | `govops/mappings/acme-capabilities-nist-iso.yaml` |

### Correctness Properties Demonstrated

- **P1:** All capability ids in mappings match the three-segment format.
- **P4:** All required subsections present.

### Cross-References

| Item | Reference |
|---|---|
| `#MappingDocument` | Design §9 |
| Worked example mapping | Design §8.2 |

---

## UC-03: IGA Integration

### Context

IGA platforms manage access reviews and entitlement lifecycle, but they traditionally
operate on opaque permission strings that are hard to interpret in a governance context.
This use case shows how the IGA Exporter translates `GovOps-AC` into IGA-ingestible
formats, how access reviews are scoped by risk-tier and sensitivity using
applicability-groups, how recertification campaigns reference canonical capability ids,
and how `govops drift` detects capabilities present in deployed policy but absent from
the catalog.

### Actors

- **Primary:** IGA Team (Acme Bank, identity governance)
- **Secondary:** Platform Security Engineer (resolves drift findings)

### Preconditions

- `govops/GovOps-AC.yaml` passes `govops lint` (UC-01 complete).
- The IGA platform (e.g., SailPoint, Saviynt, or equivalent) is configured to ingest
  capability catalog exports.
- The GovOps toolchain (IGA Exporter, `govops drift`) is installed.
- Deployed policy artifacts exist in `govops/policies/` for Cedar (payments) and OPA (fraud).

### Step-by-Step Workflow

**Step 1 — Export `GovOps-AC` to IGA-ingestible format.**

The IGA Exporter translates the capability catalog into a CSV format suitable for
IGA platform ingest. Each row represents one capability with its canonical id,
human-readable title, risk-tier, and sensitivity classification.

```
$ govops iga-export \
    --catalog govops/GovOps-AC.yaml \
    --format csv \
    --output govops/exports/acme-capabilities-2026-05-14.csv

govops iga-export v0.9.0
Exporting 4 capabilities to CSV...
Written: govops/exports/acme-capabilities-2026-05-14.csv
```

```csv
# govops/exports/acme-capabilities-2026-05-14.csv
capability_id,title,service,action,resource,risk_tier,data_sensitivity,lifecycle_stage
payments:read:invoice,Read invoice,payments,read,Invoice,medium,pii,runtime
payments:transfer:bank-account,Transfer funds,payments,transfer,BankAccount,critical,confidential,runtime
lending:approve:loan,Approve loan,lending,approve,Loan,critical,confidential,runtime
fraud:flag:transaction,Flag transaction,fraud,flag,Transaction,high,internal,runtime
```

The IGA platform ingests this CSV and creates entitlement records keyed by
`capability_id`. Access reviews and recertification campaigns reference these canonical
ids rather than engine-specific permission strings (e.g., Cedar action names or
OpenFGA relation names).

**Step 2 — Scope an IGA access review by risk-tier and sensitivity.**

The IGA Team configures a quarterly access review scoped to capabilities at
`risk-tier: critical` or `data-sensitivity: confidential | pii`. The
`applicability-groups` defined in `GovOps-AC.yaml` drive this scoping.

```
$ govops iga-export \
    --catalog govops/GovOps-AC.yaml \
    --format csv \
    --filter "risk-tier=critical OR sensitivity IN (confidential, pii)" \
    --output govops/exports/acme-high-risk-review-2026-Q2.csv

govops iga-export v0.9.0
Applying filter: risk-tier=critical OR sensitivity IN (confidential, pii)
Matching capabilities: 3 of 4

Written: govops/exports/acme-high-risk-review-2026-Q2.csv
```

```csv
capability_id,title,risk_tier,data_sensitivity
payments:read:invoice,Read invoice,medium,pii
payments:transfer:bank-account,Transfer funds,critical,confidential
lending:approve:loan,Approve loan,critical,confidential
```

The filter matches three capabilities: `payments:read:invoice` (pii), `payments:transfer:bank-account` (critical + confidential), and `lending:approve:loan` (critical + confidential). `fraud:flag:transaction` is high risk but internal sensitivity only, so it is excluded from this particular campaign.

The IGA platform uses this scoped export to launch the Q2 access review campaign,
targeting only the capabilities that meet the risk threshold.

**Step 3 — Run a recertification campaign referencing canonical capability ids.**

The IGA platform sends recertification requests to capability owners. Each request
references the canonical capability id from `GovOps-AC`, not an engine-specific
permission string:

```
Recertification Request — Q2 2026
Capability: payments:transfer:bank-account
Title: Transfer funds
Risk tier: critical
Data sensitivity: confidential
Current access holders: [alice@acme.com, svc-payments-batch@acme.com]
Certifier: payments-team-lead@acme.com
Action required: Certify or revoke by 2026-06-30
```

The canonical id `payments:transfer:bank-account` is stable across policy engine
changes — if Acme migrates from Cedar to OPA, the capability id does not change, and
the IGA recertification history remains intact.

**Step 4 — Use `govops drift` to detect policy-present/catalog-absent capabilities.**

The IGA Team runs `govops drift` to find capabilities present in deployed policy but
absent from `GovOps-AC`. These represent "shadow entitlements" — permissions that
exist in enforcement but are not governed.

```
$ govops drift \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar.cedar \
    --engine cedar \
    --policy govops/policies/fraud-detection-opa.rego \
    --engine opa

govops drift v0.9.0

Drift Report — Acme Bank (2026-05-14)
──────────────────────────────────────

Cedar engine (govops/policies/payments-cedar.cedar):
  Type B (policy rule with no catalog entry):
    ⚠ action: "batch-transfer", resource: "BankAccount"
        Found in Cedar policy rule: permit(action == "batch-transfer", resource.type == "BankAccount")
        No corresponding capability in GovOps-AC.yaml
        Suggested id: payments:batch-transfer:bank-account
        Recommendation: Add to catalog or remove from policy.

Fraud engine (govops/policies/fraud-detection-opa.rego):
  No drift detected.

Summary: 1 drift finding (Type B). 0 Type A. 0 Type C.
```

The IGA Team escalates the `payments:batch-transfer:bank-account` finding to the
Platform Security Engineer. The engineer determines this is a legitimate capability
that was added to the Cedar policy without going through the catalog authoring process
(UC-01 complete). Resolution: add `payments:batch-transfer:bank-account` to `GovOps-AC.yaml`
and run `govops lint` to validate.

**Step 5 — Capability deprecation and IGA catalog update.**

The payments team decides to deprecate `payments:read:invoice` and replace it with
`payments:read:invoice-summary` (a narrower capability that returns only invoice
headers, not line items, reducing PII exposure). The Platform Security Engineer
updates `GovOps-AC.yaml`:

```yaml
  # Deprecated entry
  - id: payments:read:invoice
    title: Read invoice (deprecated)
    group: g.payments
    action: read
    resource: Invoice
    state: Deprecated
    replaced-by: payments:read:invoice-summary
    data-sensitivity: pii
    risk-tier: medium

  # New entry
  - id: payments:read:invoice-summary
    title: Read invoice summary
    group: g.payments
    action: read
    resource: InvoiceSummary
    documented-requester-classes: [interactive-subject, software-agent]
    data-sensitivity: internal
    risk-tier: low
    lifecycle-stage: runtime
    description: >
      Retrieve invoice header fields only (no line items). Reduced PII exposure
      compared to payments:read:invoice.
```

The IGA Exporter detects the deprecation and generates a notification:

```
$ govops iga-export \
    --catalog govops/GovOps-AC.yaml \
    --format csv \
    --output govops/exports/acme-capabilities-2026-05-28.csv \
    --diff govops/exports/acme-capabilities-2026-05-14.csv

govops iga-export v0.9.0

Capability changes detected since 2026-05-14:
  DEPRECATED: payments:read:invoice → replaced by payments:read:invoice-summary
  ADDED:      payments:read:invoice-summary (risk-tier: low, sensitivity: internal)
  ADDED:      payments:batch-transfer:bank-account (risk-tier: critical, sensitivity: confidential)

IGA ACTION REQUIRED:
  1. Update IGA entitlement records: retire payments:read:invoice, create payments:read:invoice-summary
  2. Migrate active access grants from payments:read:invoice to payments:read:invoice-summary
  3. Re-scope Q2 access review to include payments:batch-transfer:bank-account (risk-tier: critical)
```

The IGA Team processes the notification, updates the IGA platform's entitlement
catalog, and migrates active grants before the deprecation deadline.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| IGA export (full) | CSV | `govops/exports/acme-capabilities-2026-05-14.csv` | Created by IGA Exporter |
| IGA export (scoped) | CSV | `govops/exports/acme-high-risk-review-2026-Q2.csv` | Created by IGA Exporter with filter |
| Updated capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | Modified: deprecation + new entries |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in the CSV exports and drift output
  match the three-segment format, including the newly suggested
  `payments:batch-transfer:bank-account`.
- **P3 (Lint and Drift Output Consistency):** The `govops drift` output references only
  `payments:batch-transfer:bank-account` as a Type B finding — a capability present in
  the Cedar policy but absent from the catalog excerpt shown in UC-01.

### Cross-References

| Item | Reference |
|---|---|
| IGA Exporter tool | Design §10 |
| `applicability-groups` for scoping | Design §6.3 |
| `govops drift` tool | Design §10 |
| Capability deprecation pattern | Design §12 (open question 4) |
| Relationship to IGA entitlement catalogs | Design §4 (non-goals) |
| `#CapabilityCatalog` | Design §6 |

## UC-04: Policy Drift Detection

### Context

The capability catalog expresses governance intent; deployed policy expresses
enforcement reality. Drift between the two is inevitable in a fast-moving engineering
organization. This use case shows how `govops drift` detects three categories of
divergence across two policy engines (Cedar and OPA/Rego), how the engineer decides
how to resolve each finding, and how drift detection integrates into CI/CD to prevent
ungoverned policy changes from reaching production.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank, payments, lending, and fraud domains)
- **Secondary:** Policy Engine Operator (owns Cedar and OPA deployments)

### Preconditions

- UC-03 complete: catalog includes `payments:read:invoice-summary`, `payments:batch-transfer:bank-account`, and deprecated `payments:read:invoice`.
- `govops/GovOps-AC.yaml` passes `govops lint`.
- Deployed policy artifacts exist:
  - `govops/policies/payments-cedar.cedar` (Cedar, payments service)
  - `govops/policies/fraud-detection-opa.rego` (OPA/Rego, fraud detection service)
- The `govops drift` Cedar and OPA analyzer plug-ins are installed.

### Step-by-Step Workflow

**Step 1 — Run `govops drift` against both policy engines.**

```
$ govops drift \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar.cedar --engine cedar \
    --policy govops/policies/fraud-detection-opa.rego --engine opa

govops drift v0.9.0

Drift Report — Acme Bank (2026-05-28)
──────────────────────────────────────

Cedar engine (govops/policies/payments-cedar.cedar):
  [Type A] Catalog entry with no policy rule:
    ⚠ payments:read:invoice-summary
        Capability in GovOps-AC.yaml (state: Active)
        No Cedar policy rule found for action="read", resource.type="InvoiceSummary"
        Recommendation: Add policy rule or mark capability as design-time only.

  [Type C] Context-expectations diverge from deployed policy:
    ⚠ payments:transfer:bank-account
        documented-context-expectations: context.acr == 'urn:mfa'
        Cedar policy rule: permits transfer for service accounts with context.service-account == true
          WITHOUT requiring context.acr == 'urn:mfa'
        Recommendation: Update Cedar policy to enforce MFA for service accounts,
          or update documented-context-expectations to document the exception.

OPA engine (govops/policies/fraud-detection-opa.rego):
  [Type B] Policy rule with no catalog entry:
    ⚠ action: "escalate", resource: "Transaction"
        Found in OPA rule: allow { input.action == "escalate"; input.resource.type == "Transaction" }
        No corresponding capability in GovOps-AC.yaml
        Suggested id: fraud:escalate:transaction
        Recommendation: Add to catalog or remove from policy.

  No Type A or Type C findings for OPA engine.

Unified Drift Summary:
  Type A (catalog entry, no policy rule):  1 finding  [Cedar]
  Type B (policy rule, no catalog entry):  1 finding  [OPA]
  Type C (context-expectations diverge):   1 finding  [Cedar]
  Total: 3 findings across 2 engines.
```

**Step 2 — Resolve the three drift scenarios.**

**Scenario (a) — Type A: `payments:read:invoice-summary` in catalog, no Cedar rule.**

The capability was added to the catalog (UC-03 Step 5) but the Cedar policy has not
yet been updated to enforce it. The engineer has three options:

| Option | Action | When to use |
|---|---|---|
| Update policy | Add Cedar rule for `read` on `InvoiceSummary` | Capability is live and should be enforced |
| Update catalog | Mark capability `lifecycle-stage: design-time` | Capability is planned but not yet deployed |
| Accept exception | Document in `govops/exceptions/drift-exceptions.yaml` | Intentional gap with documented rationale |

Decision: the payments team confirms `payments:read:invoice-summary` is live. The
Policy Engine Operator adds the Cedar rule. After the policy update, re-running
`govops drift` shows no Type A finding for this capability.

**Scenario (b) — Type B: `fraud:escalate:transaction` in OPA policy, not in catalog.**

The fraud team added an escalation capability to the OPA policy without going through
the catalog authoring process. The engineer adds `fraud:escalate:transaction` to `GovOps-AC.yaml`:

```yaml
  - id: fraud:escalate:transaction
    title: Escalate transaction
    group: g.fraud
    action: escalate
    resource: Transaction
    documented-requester-classes: [interactive-subject, software-agent]
    engine-bindings:
      - engine: opa
        policy-artifact: govops/policies/fraud-detection-opa.rego
    data-sensitivity: internal
    risk-tier: high
    lifecycle-stage: runtime
    description: >
      Escalate a flagged transaction to a senior fraud analyst queue (follow-on to flag).
```

After adding the entry, the engineer runs `govops lint` to confirm `action.escalate` and `resource.transaction` resolve from the lexicon.

After adding the entry and running `govops lint` to validate, re-running `govops drift`
shows no Type B finding for this capability.

**Scenario (c) — Type C: `payments:transfer:bank-account` context-expectations diverge.**

The Cedar policy has a service-account bypass that permits transfers without MFA. The drift report confirms the divergence: `documented-context-expectations` require MFA, but deployed policy does not enforce it for service accounts.

Decision: the engineer removes the service-account bypass from the Cedar policy. After the policy update, re-running `govops drift` shows no Type C finding.

**Step 3 — CI/CD integration: drift flagged before production.**

The drift check is integrated into the CI/CD pipeline as a gate (UC-05). When the
Policy Engine Operator submits a pull request that introduces the service-account
bypass, the pipeline runs `govops drift` and fails the build:

```
CI/CD Pipeline — policy-change-pr-447
Step 3: govops drift
  ✗ DRIFT DETECTED — build failed

  [Type C] Context-expectations diverge:
    payments:transfer:bank-account
    documented-context-expectations: context.acr == 'urn:mfa'
    Proposed Cedar policy: permits transfer for service accounts without context.acr == 'urn:mfa'

  Policy change introduces drift. Resolve before merging.
  EXIT 1
```

The pull request is blocked from merging until the drift is resolved.

**Step 4 — Multi-engine drift report composition.**

The unified drift summary at the end of the `govops drift` output composes per-engine
sub-reports into a single view. Each finding is tagged with its engine so the
responsible Policy Engine Operator can be notified:

```
Unified Drift Summary (after resolutions):
  Cedar engine:  0 findings
  OPA engine:    0 findings
  Total: 0 findings. Catalog and deployed policy are aligned.
```

The per-engine sub-reports are also available individually for engine-specific
remediation workflows.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Updated capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | Modified: added `fraud:escalate:transaction` |
| Drift exception log | Convention | `govops/exceptions/drift-exceptions.yaml` | Created if exceptions are accepted |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in the drift report (`payments:read:invoice-summary`,
  `payments:transfer:bank-account`, `fraud:escalate:transaction`) match the three-segment format.
  The suggested id `fraud:escalate:transaction` also matches.
- **P4 (PARC Context Predicate Consistency):** The Type C drift finding for
  `payments:transfer:bank-account` correctly identifies the divergence between the
  capability's `documented-context-expectations` (`context.acr == 'urn:mfa'`) and the
  deployed Cedar policy (which permits without MFA for service accounts). This is
  consistent with the capability excerpt shown in UC-01.
- **P3 (Lint and Drift Output Consistency):** The `govops drift` output references only
  capability ids that appear in the catalog or are suggested as new entries — no
  phantom ids.

### Cross-References

| Item | Reference |
|---|---|
| `govops drift` tool | Design §10 |
| `documented-context-expectations` field | Design §6.2(B) |
| Cedar PDP class | Design §8.1 |
| OPA/Rego PDP class | Design §8.1 |
| Type C drift and context expectations | Design §7.1 |
| CI/CD integration | Design §11 (Phase 4) |
| Type C drift and context expectations | UC-01 `payments:transfer:bank-account` |


## UC-05: Continuous Governance in CI/CD

### Context

Point-in-time catalog reviews are insufficient. This use case wires **`govops lint`** and **`govops drift`** into CI/CD so catalog and policy stay aligned on every change.

### Actors

- **Primary:** Platform Security Engineer
- **Secondary:** Policy Engine Operator, Lending Officer

### Preconditions

- UC-01 through UC-04 complete.
- CI system can run GovOps CLI tools.

### Step-by-Step Workflow

**Step 1 — Pipeline configuration.**

```yaml
# govops/.govops-ci.yaml
pipeline:
  steps:
    - name: lint
      command: govops lint
      args: [--catalog, govops/GovOps-AC.yaml, --lexicon, govops/lexicon.yaml]
    - name: drift
      command: govops drift
      args: [--catalog, govops/GovOps-AC.yaml,
             --policy, govops/policies/payments-cedar.cedar, --engine, cedar,
             --policy, govops/policies/fraud-detection-opa.rego, --engine, opa]
gates:
  lint:
    fail-on-errors: true
  drift:
    fail-on-any-drift: true
```

**Step 2 — Policy pull request fails on drift.**

A Cedar change introduces a service-account bypass on `payments:transfer:bank-account` without updating documented-context-expectations:

```
Step: govops drift
  ✗ Type C: payments:transfer:bank-account — context.acr == urn:mfa not enforced for service accounts
  EXIT 1 — merge blocked
```

**Step 3 — Passing pipeline after fix.**

Engineer removes bypass, re-runs pipeline: lint PASS, drift PASS. `GovOps-AC.yaml` and `policies/` are versioned together so governance state is reproducible at any commit.

### Artifacts Produced or Modified

| Artifact | Path | Notes |
|---|---|---|
| CI config | `govops/.govops-ci.yaml` | Lint + drift gates |

### Correctness Properties Demonstrated

- **P1:** Pipeline output references only canonical capability ids.
- **P4:** Required subsections present.

### Cross-References

| Item | Reference |
|---|---|
| Toolchain | Design §10 |
| Adoption Phase 4 | Design §11 |

---

## UC-06: Automated Fraud Flagging (Fraud System)

### Context

Not every **PARC Principal** is a human. Acme's **Fraud System** is an automated fraud-detection service that evaluates payment transactions and requests **`fraud:flag:transaction`** when model scores exceed a threshold. The capability catalog must describe this **(action, resource)** pair and document that **software agents** are expected requesters — so policy, drift checks, and audits refer to the same capability id whether a human analyst or the Fraud System invokes the PDP.

### Actors

- **Primary:** Fraud System (Acme Bank fraud detection platform — non-human **software-agent** principal)
- **Secondary:** Platform Security Engineer (owns catalog and `fraud-detection-opa.rego`)
- **Secondary:** Policy Engine Operator (maintains OPA policy for automated flags)

### Preconditions

- UC-01 complete: `fraud:flag:transaction` is defined in `GovOps-AC.yaml` with `documented-requester-classes` including `software-agent`.
- `govops/policies/fraud-detection-opa.rego` is deployed and referenced by the capability's `engine-bindings`.
- The Fraud System holds a workload identity (e.g., SPIFFE ID or service account) recognized by the PDP.

### Step-by-Step Workflow

**Step 1 — Catalog entry defines the non-human actor.**

From `GovOps-AC.yaml` (see UC-01):

```yaml
  - id: fraud:flag:transaction
    documented-requester-classes: [software-agent, interactive-subject]
    documented-context-expectations:
      - id: c.risk-score
        expression: "context.fraud.risk-score >= 0.85"
```

Human fraud analysts may also flag transactions interactively; the **same capability id** applies. The catalog does not model the Fraud System as a separate capability — it models the **operation** (flag on Transaction) and who may request it.

**Step 2 — Fraud System sends a PARC-shaped authorization request.**

When the payments pipeline emits a high-risk transaction event, the Fraud System calls the PDP:

```json
{
  "principal": {
    "type": "software-agent",
    "id": "svc-acme-fraud-detection",
    "name": "Acme Fraud System"
  },
  "action": "flag",
  "resource": {
    "type": "Transaction",
    "id": "txn-9f2a8c1d"
  },
  "context": {
    "fraud": {
      "risk-score": 0.91,
      "model-version": "fraud-v3.2",
      "source": "batch-scoring"
    }
  }
}
```

The **Action** and **Resource** type align with `fraud:flag:transaction` in `GovOps-AC`. The **Principal** is explicitly non-human. **Context** carries the model score the policy expects.

**Step 3 — PDP decision and alignment with catalog.**

The OPA policy permits the request when `context.fraud.risk-score >= 0.85` and the principal is the registered Fraud System service account. A denied request (score below threshold) does not exercise the capability — no flag is recorded.

**Step 4 — `govops drift` confirms policy matches catalog expectations.**

```
$ govops drift \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/fraud-detection-opa.rego --engine opa

govops drift v0.9.0

OPA engine (fraud-detection-opa.rego):
  No drift for fraud:flag:transaction — policy permits flag on Transaction for
  software-agent principals when context.fraud.risk-score >= 0.85.
```

If drift Type C appears (catalog expects risk score but policy omits it), the Platform Security Engineer updates policy before the Fraud System runs in production.

**Step 5 — Contrast with human-initiated flag on the same capability.**

A human analyst flagging the same transaction uses the identical capability id and PARC shape, but **Principal** is `interactive-subject` and **Context** may include analyst id and case reference instead of batch model metadata. Governance and compliance trace both flows to **`fraud:flag:transaction`**, not to separate permission strings.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | `fraud:flag:transaction` documents software-agent requesters |
| Deployed policy | OPA/Rego | `govops/policies/fraud-detection-opa.rego` | Evaluates Fraud System PARC requests |

### Correctness Properties Demonstrated

- **P1:** `fraud:flag:transaction` matches the three-segment capability id format.
- **P2:** PARC **Context** (`context.fraud.risk-score >= 0.85`) is consistent with `documented-context-expectations` on the capability excerpt in this section.
- **P4:** All required subsections present.

### Cross-References

| Item | Reference |
|---|---|
| `documented-requester-classes` | Design §6.2(B) |
| PARC request shape | Design §2.2, §3 |
| `fraud:flag:transaction` | Design §8.1 |
| `govops drift` | Design §10 |
| Non-human principals | Design §3 (Principal vs. capability) |

---

## Summary Table

| Use Case | Primary Persona | Toolchain | Design §§ |
|---|---|---|---|
| UC-01: Capability Catalog Authoring | Platform Security Engineer | `govops lint` | §6, §10 |
| UC-02: Compliance Mapping and Audit | Compliance Auditor | — | §8–9 |
| UC-03: IGA Integration | IGA Team | IGA Exporter, `govops drift` | §6, §10 |
| UC-04: Policy Drift Detection | Platform Security Engineer | `govops drift` | §7.1, §10 |
| UC-05: Continuous Governance in CI/CD | Platform Security Engineer | `govops lint`, `govops drift` | §10–11 |
| UC-06: Automated Fraud Flagging | Fraud System (non-human) | `govops drift` | §6.2, §7.1, §10 |

---

*Companion to [enterprise-capability-catalog-design.md](./enterprise-capability-catalog-design.md). Tool outputs in each section are consistent with the catalog excerpt and **catalog state** row for that use case (design §8).*

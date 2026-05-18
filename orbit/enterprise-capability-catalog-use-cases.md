# Enterprise Capability Catalog: Use Cases

**Status:** Draft for discussion
**Audience:** GRC Practitioners, Platform Security Engineers, IGA Teams, Compliance Auditors, Policy Engine Operators
**Companion design:** [Cataloging Enterprise Capabilities with Gemara](./enterprise-capability-catalog-design.md)

---

## Executive Summary

This document illustrates how different enterprise personas interact with the GovOps
enterprise capability catalog, TIGER pillar controls, and associated toolchain in
real-world authorization governance workflows. It is a companion to the design document
*Cataloging Enterprise Capabilities with Gemara*, which specifies the artifact model,
schema, and toolchain. Where the design document answers *what* the system is, this
document answers *how* it is used.

Nine use cases span the full lifecycle of authorization governance at Acme Bank, a
fictional financial institution running a payments platform. The shared domain ensures
that artifact excerpts, tool outputs, and cross-references are internally consistent
across all scenarios. The four canonical capabilities used throughout are
`payments:read:invoice`, `payments:transfer:bank-account`,
`governance:policy-write:governance-policy`, and `iam:assume:role`.

The five personas covered are: **Platform Security Engineer** (UC-01, UC-03, UC-07,
UC-08), **GRC Practitioner** (UC-02, UC-04), **Compliance Auditor** (UC-05), **IGA
Team** (UC-06), and **Policy Engine Operator** (UC-07, UC-08 as secondary actor).

A critical distinction runs through every use case: **TIGER produces five independent
ordinal scores** (one per pillar: Transparency, Integrity, Governance, Events,
Resilience), each on a scale of 1–5. There is no single aggregate TIGER score.
**Policy Coverage** is a separate metric — the weighted fraction of declared
`#AssessmentRequirement`s currently satisfied by deployed policy and runtime evidence —
computed by `govops coverage`. The two metrics are complementary: TIGER scores measure
governance maturity per pillar; Policy Coverage measures how much of the declared
requirement surface is currently satisfied.

The use cases progress from foundational authoring (UC-01, UC-02) through proof
workflows (UC-03), score computation (UC-04), compliance audit (UC-05), IGA integration
(UC-06), drift detection (UC-07), and continuous automation (UC-08). Each section is
self-contained and cross-referenced to the design document so a reader can trace from
scenario to specification. A summary table at the end maps each use case to its primary
persona, TIGER pillars exercised, Policy Coverage relevance, toolchain commands, and
design document sections.

---

## Table of Contents

1. [Persona Summary](#persona-summary)
2. [UC-01: Capability Catalog Authoring](#uc-01-capability-catalog-authoring)
3. [UC-02: TIGER Control Authoring and Policy Coverage](#uc-02-tiger-control-authoring-and-policy-coverage)
4. [UC-03: Provable Claims and Proof Workflow](#uc-03-provable-claims-and-proof-workflow)
5. [UC-04: TIGER Pillar Score Computation and Reporting](#uc-04-tiger-pillar-score-computation-and-reporting)
6. [UC-05: Compliance Mapping and Audit](#uc-05-compliance-mapping-and-audit)
7. [UC-06: IGA Integration](#uc-06-iga-integration)
8. [UC-07: Policy Drift Detection](#uc-07-policy-drift-detection)
9. [UC-08: Continuous TIGER in CI/CD](#uc-08-continuous-tiger-in-cicd)
10. [Summary Table](#summary-table)

---

## Persona Summary

| Persona | Role | Use Cases (primary) | Use Cases (secondary) |
|---|---|---|---|
| Platform Security Engineer | Deploys, configures, and maintains PDPs and the GovOps repository | UC-01, UC-03, UC-07, UC-08 | UC-02, UC-06 |
| GRC Practitioner | Defines and audits authorization controls; owns TIGER pillar catalogs | UC-02, UC-04 | UC-03, UC-05, UC-08 |
| Compliance Auditor | Assesses conformance to NIST 800-53, ISO 27001, SOC 2 | UC-05 | UC-04 |
| IGA Team | Manages access reviews, entitlement lifecycle, and provisioning | UC-06 | UC-07 |
| Policy Engine Operator | Owns a specific PDP and its policy lifecycle | — | UC-07, UC-08 |

---


## UC-01: Capability Catalog Authoring

### Context

Before any TIGER control can be authored or any proof can be run, the enterprise must
have a well-formed capability catalog. This use case shows how a Platform Security
Engineer builds `GovOps-AC.yaml` from scratch: defining canonical vocabulary in the
`#Lexicon`, authoring capability entries in both supported profiles, assigning
applicability groups, and validating the result with `govops lint`. Getting the catalog
right at this stage is the foundation for every downstream metric.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank, payments domain)
- **Secondary:** None

### Preconditions

- The `govops/` repository directory exists and is version-controlled.
- The GovOps toolchain (`govops lint`) is installed and on the `PATH`.
- The engineer has identified the set of (action, resource) pairs the payments service
  exposes as authorizable operations.
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
  - id: action.policy-write
    title: policy-write
    definition: Create or modify a policy artifact in the governance plane.
    synonyms: [write-policy, update-policy]
  - id: action.assume
    title: assume
    definition: Adopt the permissions of a named role for the duration of a session.
    synonyms: [switch-role, impersonate]
  - id: resource.invoice
    title: Invoice
    definition: A demand for payment issued by the payments service.
  - id: resource.bank-account
    title: BankAccount
    definition: A demand-deposit account at a financial institution.
  - id: resource.governance-policy
    title: GovernancePolicy
    definition: A policy artifact in the GovOps repository.
  - id: resource.role
    title: Role
    definition: A named collection of permissions in the IAM control plane.
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
      description: Coarse risk classification used by GovOps metrics.
      values: [low, medium, high, critical]
    - id: lifecycle-stage
      title: Lifecycle Stage
      description: When in the SDLC this capability is exercised.
      values: [design-time, build-time, deploy-time, runtime]
groups:
  - id: g.payments
    title: Payments
    description: Capabilities exposed by the payments domain.
  - id: g.iam
    title: IAM
    description: Capabilities exposed by the IAM control plane.
  - id: g.governance
    title: Governance
    description: Capabilities exposed by the governance plane itself.
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
    data-sensitivity: confidential
    risk-tier: critical
    lifecycle-stage: runtime
    description: >
      Initiate a funds transfer between bank accounts in the payments domain.
      This is the highest-risk capability in the payments service.

  - id: governance:policy-write:governance-policy
    title: Modify governance policy
    group: g.governance
    action: policy-write
    resource: GovernancePolicy
    documented-requester-classes: [interactive-subject]
    documented-context-expectations:
      - id: c.approvals
        text: "Two distinct approvers required in PARC Context"
        expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
        expression-language: natural-language
    risk-tier: critical
    lifecycle-stage: runtime
    description: >
      Edit any policy in the GovOps repository. Requires separation of duties.

  - id: iam:assume:role
    title: Assume IAM role
    group: g.iam
    action: assume
    resource: Role
    documented-requester-classes: [interactive-subject, software-agent]
    data-sensitivity: internal
    risk-tier: high
    lifecycle-stage: runtime
    description: >
      Adopt the permissions of a named IAM role for the duration of a session.
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
  ✓ Lexicon resolved: action.policy-write → governance:policy-write:governance-policy
  ✓ Lexicon resolved: action.assume     → iam:assume:role
  ✓ Lexicon resolved: resource.invoice  → payments:read:invoice
  ✓ Lexicon resolved: resource.bank-account → payments:transfer:bank-account
  ✓ Lexicon resolved: resource.governance-policy → governance:policy-write:governance-policy
  ✓ Lexicon resolved: resource.role     → iam:assume:role
  ✓ Group membership: all capabilities reference defined groups
  ✓ Applicability groups: all values within declared group value sets
  ✓ No engine-binding well-formedness errors

4 capabilities validated. 0 errors. 0 warnings.
```

**Step 5 — Introduce a lexicon resolution failure and observe the lint error.**

The engineer temporarily adds a capability that references an action term not in the
lexicon — `payments:approve:payment-order` — where `approve` has not been defined.

```yaml
  # Intentionally broken entry (not in final catalog)
  - id: payments:approve:payment-order
    title: Approve payment order
    group: g.payments
    action: approve
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
  ✓ Lexicon resolved: action.policy-write → governance:policy-write:governance-policy
  ✓ Lexicon resolved: action.assume     → iam:assume:role
  ✗ Lexicon resolution FAILED: action 'approve' not found in lex.govops.actions-resources
      capability: payments:approve:payment-order
      suggestion: Did you mean 'policy-write'? Or add a new term to lexicon.yaml.
  ✗ Lexicon resolution FAILED: resource 'PaymentOrder' not found in lex.govops.actions-resources
      capability: payments:approve:payment-order
      suggestion: Add resource.payment-order to lexicon.yaml.

5 capabilities checked. 2 errors. 0 warnings.
EXIT 1
```

**Resolution:** The engineer adds `action.approve` and `resource.payment-order` to
`lexicon.yaml`, then re-runs `govops lint` to confirm the error is resolved. The
broken entry is removed from this use case's final catalog.

**Step 6 — Verify applicability-group assignments drive downstream metrics.**

The engineer confirms that the `risk-tier` and `sensitivity` applicability-group
assignments on each capability will be used by `govops coverage` (UC-02) to bucket
capabilities by risk and by the IGA Exporter (UC-06) to scope access reviews. The
assignments are not enforced by `govops lint` beyond checking that values are within
the declared group value sets — their downstream effect is in the metrics layer.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Lexicon | `#Lexicon` | `govops/lexicon.yaml` | Created; defines canonical action verbs and resource type names |
| Capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | Created; 4 capabilities across 3 groups |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All four capability ids (`payments:read:invoice`,
  `payments:transfer:bank-account`, `governance:policy-write:governance-policy`,
  `iam:assume:role`) match `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$`.
- **P6 (Lint Output Consistency):** The `govops lint` passing output references only
  the four capability ids present in the catalog excerpt above; the failing output
  references only `payments:approve:payment-order`, which appears in the broken entry
  shown in the same step.
- **P7 (Section Structure):** This section contains all seven required subsections.

### Cross-References

| Item | Reference |
|---|---|
| Convention-only profile (Profile A) | Design §6.2(A) |
| Schema extension profile (Profile B) | Design §6.2(B) |
| `#EnterpriseCapability` CUE definition | Design §6.2(B) |
| `#Lexicon` artifact | Design §6.4 |
| `#Group` and applicability-groups | Design §6.3 |
| `govops lint` tool | Design §12 |
| Gemara artifact: `#CapabilityCatalog` | ADR-0019 |
| Gemara artifact: `#Lexicon` | ADR-0021 |


---

## UC-02: TIGER Control Authoring and Policy Coverage

### Context

With the capability catalog in place, the GRC Practitioner's next task is to express
what governance requirements those capabilities must satisfy. This use case shows how
to author a TIGER pillar control catalog, link it to `GovOps-AC` via
`#MultiEntryMapping`, and measure **Policy Coverage** — the fraction of declared
`#AssessmentRequirement`s currently satisfied. Policy Coverage is a distinct metric
from the five TIGER pillar scores; improving coverage is a prerequisite for improving
pillar scores, but the two are not the same thing.

### Actors

- **Primary:** GRC Practitioner (Acme Bank, authorization governance)
- **Secondary:** Platform Security Engineer (consulted on provable-claim feasibility)

### Preconditions

- `govops/GovOps-AC.yaml` exists and passes `govops lint` (UC-01 complete).
- `govops/lexicon.yaml` exists.
- The GovOps toolchain (`govops coverage`) is installed.
- The GRC Practitioner has reviewed the capability catalog and identified which
  capabilities require formal governance controls.

### Step-by-Step Workflow

**Step 1 — Author the Integrity pillar control catalog.**

The GRC Practitioner creates `govops/GovOps-Integrity.yaml`. The control
`GOVOPS-INT-01` governs `payments:transfer:bank-account` — the highest-risk capability
in the catalog. It contains two `#AssessmentRequirement`s for the same capability:
one configuration assertion (`GOVOPS-INT-01.01`) and one provable claim
(`GOVOPS-INT-01.02`), illustrating the difference between the two flavors (design §8).

```yaml
# govops/GovOps-Integrity.yaml
title: GovOps Integrity Catalog
metadata:
  id: cat.govops.integrity
  type: ControlCatalog
  gemara-version: "0.x"
  description: >
    Request context and evidence for PARC evaluations over catalog capabilities
    meet documented assurance requirements.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.int
    title: Integrity
    description: Context and evidence preconditions on authorization requests.
controls:
  - id: GOVOPS-INT-01
    title: Sensitive capabilities require strong authentication evidence in context
    objective: >
      No authorization path exists in which a sensitive (action, resource) capability
      is permitted unless the PARC request carries sufficient authentication evidence.
    group: g.int
    state: Active
    capabilities:
      reference-id: ec
      entries:
        - reference-id: payments:transfer:bank-account
    assessment-requirements:
      # Flavor 1: Configuration assertion — checks that a control is PRESENT
      - id: GOVOPS-INT-01.01
        text: >
          For every capability with data-sensitivity in {confidential, pii, regulated}
          OR risk-tier >= high, the deployed policy MUST require an authentication
          context class reference (acr) of urn:mfa or stronger on the PARC Context
          when permitting that action on that resource.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
      # Flavor 2: Provable claim — checks that the control is SUFFICIENT
      - id: GOVOPS-INT-01.02
        text: >
          A symbolic analysis of the deployed policy MUST yield UNSAT for the
          formula: Permit on (action=transfer, resource=BankAccount) AND
          context.acr != urn:mfa.
        applicability: [production]
        state: Active
        flavor: provable-claim
```

The key distinction:
- `GOVOPS-INT-01.01` is a **configuration assertion**: it checks that the MFA
  requirement is *present* in policy. An auditor can verify this by inspecting the
  deployed policy artifact.
- `GOVOPS-INT-01.02` is a **provable claim**: it checks that no execution path
  *circumvents* the MFA requirement. This requires symbolic analysis (UC-03) and
  yields an UNSAT certificate or a counterexample.

**Step 2 — Author the Governance pillar control catalog.**

The GRC Practitioner creates `govops/GovOps-Governance.yaml` to govern
`governance:policy-write:governance-policy`.

```yaml
# govops/GovOps-Governance.yaml  (excerpt)
controls:
  - id: GOVOPS-GV-01
    title: Governance changes require separation of duties
    objective: No single party can unilaterally modify governance policy.
    group: g.gv
    state: Active
    capabilities:
      reference-id: ec
      entries:
        - reference-id: governance:policy-write:governance-policy
    assessment-requirements:
      - id: GOVOPS-GV-01.01
        text: >
          Capabilities with action == policy-write and resource == GovernancePolicy
          MUST require at least two distinct approvers in the request context.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
      - id: GOVOPS-GV-01.02
        text: >
          Symbolic analysis MUST yield UNSAT for the formula: governance policy-write
          succeeds AND distinct(context.approvals) < 2.
        applicability: [production]
        state: Active
        flavor: provable-claim
      - id: GOVOPS-GV-01.03
        text: Emergency overrides MUST expire automatically within a documented bound.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
```

**Step 3 — Run `govops coverage` to measure Policy Coverage.**

The GRC Practitioner runs `govops coverage` against the catalog and the two pillar
catalogs authored so far. Policy Coverage is the weighted fraction of declared
`#AssessmentRequirement`s currently satisfied by deployed policy and runtime evidence.
It is **not** a TIGER pillar score.

```
$ govops coverage \
    --catalog govops/GovOps-AC.yaml \
    --controls govops/GovOps-Integrity.yaml govops/GovOps-Governance.yaml \
    --evidence govops/evidence/ \
    --proofs govops/proofs/

govops coverage v0.9.0

Policy Coverage Report — Acme Bank  (2026-05-14)
─────────────────────────────────────────────────
Overall Policy Coverage:  2 / 5 requirements satisfied  (40%)

By risk tier:
  critical  : 2 / 5 requirements  (40%)   ← all declared requirements are on critical capabilities
  high      : 0 / 0 requirements  (n/a)   ← iam:assume:role has no controls yet
  medium    : 0 / 0 requirements  (n/a)   ← payments:read:invoice has no controls yet
  low       : 0 / 0 requirements  (n/a)

Satisfied requirements:
  ✓ GOVOPS-INT-01.01  payments:transfer:bank-account  [configuration-assertion]  evidence: decision-log
  ✓ GOVOPS-GV-01.01   governance:policy-write:governance-policy  [configuration-assertion]  evidence: policy-inspection

Unsatisfied requirements:
  ✗ GOVOPS-INT-01.02  payments:transfer:bank-account  [provable-claim]  no proof artifact found
  ✗ GOVOPS-GV-01.02   governance:policy-write:governance-policy  [provable-claim]  no proof artifact found
  ✗ GOVOPS-GV-01.03   governance:policy-write:governance-policy  [configuration-assertion]  no evidence of override-expiry mechanism

Coverage gaps (capabilities with no AssessmentRequirement in any pillar):
  ⚠ payments:read:invoice          risk-tier: medium   sensitivity: pii
  ⚠ iam:assume:role                risk-tier: high     sensitivity: internal

NOTE: Policy Coverage (40%) is a separate metric from the TIGER pillar scores.
      Improving coverage is a prerequisite for improving pillar scores, but
      the two are not the same. Run 'govops tiger' to see per-pillar scores.
```

**Step 4 — Identify and remediate a critical coverage gap.**

The output shows that `iam:assume:role` (risk-tier: high) has no `#AssessmentRequirement`
in any pillar catalog. The GRC Practitioner prioritizes this gap because high-risk
capabilities with no governing control represent the highest unmitigated exposure.

Remediation: the GRC Practitioner authors a Resilience pillar requirement for
`iam:assume:role` (since role assumption must remain enforceable during outages):

```yaml
# Addition to govops/GovOps-Resilience.yaml
  - id: GOVOPS-RS-01
    title: Critical capabilities remain enforceable during outages
    # ...
    capabilities:
      reference-id: ec
      entries:
        - reference-id: payments:transfer:bank-account
        - reference-id: iam:assume:role
    assessment-requirements:
      - id: GOVOPS-RS-01.01
        text: Capabilities with risk-tier == critical or high MUST support local PDP evaluation.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
      - id: GOVOPS-RS-01.02
        text: Cached policy material MUST be cryptographically validated before use.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
      - id: GOVOPS-RS-01.03
        text: PDP unreachability MUST result in fail-closed for critical capabilities.
        applicability: [production]
        state: Active
        flavor: configuration-assertion
```

After authoring this control, re-running `govops coverage` removes `iam:assume:role`
from the coverage gaps list. The GRC Practitioner notes that improving Policy Coverage
by adding these requirements does not automatically improve the Resilience pillar score
— the score only improves once the requirements are *satisfied* with evidence or proofs.

**Step 5 — Understand the relationship between Policy Coverage and TIGER pillar scores.**

Policy Coverage answers: *"What fraction of what we've declared are we currently
satisfying?"* TIGER pillar scores answer: *"How mature is our governance posture for
each pillar?"* A team can have 100% Policy Coverage with only trivial controls (low
TIGER scores) or high TIGER scores with low coverage if they have many unsatisfied
provable claims. The two metrics are used together in governance review meetings (UC-04).

> **Note on the Identity pillar:** `GovOps-Identity.yaml` is not authored in this use
> case. The contents of the Identity pillar control catalog are defined by the GovOps
> Initiative Orbit WG and are not prescribed by this document.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Integrity pillar catalog | `#ControlCatalog` | `govops/GovOps-Integrity.yaml` | Created; 1 control, 2 requirements |
| Governance pillar catalog | `#ControlCatalog` | `govops/GovOps-Governance.yaml` | Created; 1 control, 3 requirements |
| Resilience pillar catalog | `#ControlCatalog` | `govops/GovOps-Resilience.yaml` | Created; 1 control, 3 requirements |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in catalog excerpts and tool output
  match the three-segment format.
- **P2 (AssessmentRequirement ID Format):** All requirement ids (`GOVOPS-INT-01.01`,
  `GOVOPS-INT-01.02`, `GOVOPS-GV-01.01`–`GOVOPS-GV-01.03`, `GOVOPS-RS-01.01`–
  `GOVOPS-RS-01.03`) match `^GOVOPS-[A-Z]+-[0-9]+\.[0-9]+$`.
- **P3 (TIGER Score Integrity):** This use case does not show TIGER scores; it shows
  Policy Coverage only, correctly presented as a separate metric.
- **P6 (Lint/Coverage Output Consistency):** The `govops coverage` output references
  only `payments:transfer:bank-account`, `governance:policy-write:governance-policy`,
  `payments:read:invoice`, and `iam:assume:role` — all present in the UC-01 catalog.
- **P8 (Identity Pillar Non-Prescription):** The use case explicitly notes that
  `GovOps-Identity.yaml` contents are defined by the GovOps Initiative Orbit WG.

### Cross-References

| Item | Reference |
|---|---|
| Configuration assertion vs. provable claim | Design §8 |
| `#ControlCatalog` artifact | Design §7 |
| `#MultiEntryMapping` for capability references | Design §6.6 |
| `govops coverage` tool | Design §12 |
| Policy Coverage metric | Design §9 (distinct from TIGER) |
| TIGER pillar catalogs | Design §7.1–7.5 |
| Gemara artifact: `#AssessmentRequirement` | Design §7 |


---

## UC-03: Provable Claims and Proof Workflow

### Context

Configuration assertions tell an auditor that a control is *present*. Provable claims
tell the auditor that the control is *sufficient* — that no execution path in the
deployed policy circumvents it. This use case shows the end-to-end proof workflow: from
authoring an engine-neutral provable claim through running `govops prove`, recording
the result in a `#EvaluationLog`, handling a counterexample, and detecting and
refreshing a stale proof after a policy redeploy.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank, payments domain)
- **Secondary:** GRC Practitioner (reviews proof results; decides on remediation)

### Preconditions

- `govops/GovOps-AC.yaml` passes `govops lint` (UC-01 complete).
- `govops/GovOps-Integrity.yaml` contains `GOVOPS-INT-01.02` as a provable claim
  (UC-02 complete).
- The payments service Cedar policy is deployed at `govops/policies/payments-cedar.cedar`.
- The `govops prove` Cedar analyzer plug-in is installed.
- `govops/proofs/` and `govops/evidence/` directories exist.

### Step-by-Step Workflow

**Step 1 — Review the engine-neutral provable claim.**

The claim `GOVOPS-INT-01.02` is already authored in `GovOps-Integrity.yaml` (UC-02).
It is expressed in engine-neutral terms: it names a capability id, a PARC Context
predicate, and the deployed policy. The claim does not reference Cedar-specific syntax.

```yaml
# From govops/GovOps-Integrity.yaml
- id: GOVOPS-INT-01.02
  text: >
    A symbolic analysis of the deployed policy MUST yield UNSAT for the
    formula: Permit on (action=transfer, resource=BankAccount) AND
    context.acr != urn:mfa.
  applicability: [production]
  state: Active
  flavor: provable-claim
  capability-ref: payments:transfer:bank-account
  parc-context-predicate:
    expression: "context.acr != 'urn:mfa'"
    expression-language: natural-language
```

The claim's negation formula is: *"There exists a PARC request where action=transfer,
resource=BankAccount is Permitted AND context.acr is not urn:mfa."* If the analyzer
finds no such request (UNSAT), the claim holds. If it finds one (SAT), a counterexample
is returned.

**Step 2 — Run `govops prove` against the Cedar policy (UNSAT — claim holds).**

The engineer runs `govops prove` for the Cedar PDP. The tool invokes the Cedar
symbolic analyzer, captures the result, writes the proof artifact to `proofs/`, and
reports the outcome.

```
$ govops prove \
    --claim GOVOPS-INT-01.02 \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar.cedar \
    --engine cedar \
    --output govops/proofs/

govops prove v0.9.0
Claim:    GOVOPS-INT-01.02
Engine:   cedar
Policy:   govops/policies/payments-cedar.cedar
Hash:     sha256:a3f8c2d1e9b047f6a2c5d8e1f3b6a9c2d5e8f1b4a7c0d3e6f9b2c5d8e1f4a7c0

Invoking Cedar symbolic analyzer...
  Formula: ∃ request. Permit(action=transfer, resource=BankAccount) ∧ context.acr ≠ "urn:mfa"
  Result:  UNSAT

  The negation of the claim is unsatisfiable: no authorization path exists in the
  deployed Cedar policy where payments:transfer:bank-account is permitted without
  context.acr == "urn:mfa".

Proof artifact written: govops/proofs/GOVOPS-INT-01.02-2026-05-14.json
Evaluation log updated: govops/evidence/eval-log-acme-2026-05-14.yaml
```

The proof artifact:

```json
{
  "claim-id": "GOVOPS-INT-01.02",
  "capability-id": "payments:transfer:bank-account",
  "formula": "Permit(action=transfer, resource=BankAccount) AND context.acr != 'urn:mfa'",
  "result": "UNSAT",
  "counterexample": null,
  "analyzer": "cedar-symbolic-v1.2",
  "policy-hash": "sha256:a3f8c2d1e9b047f6a2c5d8e1f3b6a9c2d5e8f1b4a7c0d3e6f9b2c5d8e1f4a7c0",
  "generated-at": "2026-05-14T09:15:00Z",
  "signature": "acme-govops-signing-key-2026"
}
```

**Step 3 — Update the `#EvaluationLog` with the proof artifact via `#ArtifactMapping`.**

`govops prove` automatically updates the evaluation log. The log entry records the
proof artifact reference, the deployed policy hash, and the result.

```yaml
# govops/evidence/eval-log-acme-2026-05-14.yaml  (excerpt)
metadata:
  id: log.acme.ec.integrity.2026-05-14
  type: EvaluationLog
  gemara-version: "0.x"
  description: Evaluation of GOVOPS-INT-01 against deployed Cedar policy.
target:
  id: prod.payments.v1.4.2
  name: Payments service (production)
  type: Software
  environment: production
result: Pass
evidence:
  - type: Proof
    artifact:
      reference-id: proofs/GOVOPS-INT-01.02-2026-05-14.json
      hash: sha256:b7d4e2f1a8c5d9e3f6b0c4d7e1f5a9c3d6e0f4b8c2d5e9f3a7b1c4d8e2f6a0b4
    claim-id: GOVOPS-INT-01.02
    result: UNSAT
    policy-hash: sha256:a3f8c2d1e9b047f6a2c5d8e1f3b6a9c2d5e8f1b4a7c0d3e6f9b2c5d8e1f4a7c0
  - type: DecisionLog
    artifact:
      reference-id: evidence/payments-decisions-2026-05-14.json
    window: "2026-05-13T00:00:00Z/2026-05-14T00:00:00Z"
    summary: "All 1,247 Permit decisions on action=transfer included context.acr=urn:mfa"
```

**Step 4 — Simulate a SAT result (counterexample) and remediation.**

The engineer tests a modified policy that accidentally introduces an exception path —
a service account bypass that permits transfers without MFA. Running `govops prove`
against this policy returns SAT with a counterexample.

```
$ govops prove \
    --claim GOVOPS-INT-01.02 \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar-with-bypass.cedar \
    --engine cedar \
    --output govops/proofs/

govops prove v0.9.0
Claim:    GOVOPS-INT-01.02
Engine:   cedar
Policy:   govops/policies/payments-cedar-with-bypass.cedar
Hash:     sha256:f1e2d3c4b5a6978869504132231415161718192021222324252627282930313233

Invoking Cedar symbolic analyzer...
  Formula: ∃ request. Permit(action=transfer, resource=BankAccount) ∧ context.acr ≠ "urn:mfa"
  Result:  SAT

  COUNTEREXAMPLE FOUND — the claim does NOT hold for the deployed policy.

  Counterexample:
    Principal:  { "type": "ServiceAccount", "id": "svc-payments-batch" }
    Action:     "transfer"
    Resource:   { "type": "BankAccount", "id": "acct-12345" }
    Context:    { "acr": "urn:password-only", "service-account": true }

  The Cedar policy contains a rule that permits transfers for service accounts
  without requiring context.acr == "urn:mfa". This violates GOVOPS-INT-01.02.

PROOF FAILED — claim not satisfied. No proof artifact written.
EXIT 1
```

The counterexample is surfaced to the engineer with the specific PARC fields that
satisfy the negation. The PARC Context predicate `context.acr != 'urn:mfa'` is
consistent with the capability's `documented-context-expectations` (which requires
`context.acr == 'urn:mfa'`), confirming the policy gap.

**Remediation:** The engineer removes the service-account bypass rule from the Cedar
policy, re-runs `govops prove`, and confirms UNSAT before merging the policy change.

**Step 5 — Proof workflow for an OpenFGA graph PDP.**

For the `iam:assume:role` capability, the IAM service uses OpenFGA rather than Cedar.
The proof workflow is the same from the GovOps perspective — `govops prove` invokes
the OpenFGA analyzer plug-in instead of the Cedar plug-in.

```
$ govops prove \
    --claim GOVOPS-RS-01.01 \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/iam-openfga-model.json \
    --engine openfga \
    --output govops/proofs/

govops prove v0.9.0
Claim:    GOVOPS-RS-01.01
Engine:   openfga
Policy:   govops/policies/iam-openfga-model.json
Hash:     sha256:c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8

Invoking OpenFGA reachability analyzer...
  Claim: iam:assume:role supports local PDP evaluation (GOVOPS-RS-01.01)
  Check: OpenFGA model includes local evaluation path for 'assume' on 'Role'
  Result: UNSAT (no path exists where local evaluation is unavailable for this tuple)

Proof artifact written: govops/proofs/GOVOPS-RS-01.01-2026-05-14.json
```

The proof artifact shape is identical regardless of engine; only the `analyzer` field
differs. This engine neutrality is a core design property (design §8.1).

**Step 6 — Detect and refresh a stale proof.**

Two weeks later, the payments team redeploys the Cedar policy with a new rule. The
policy hash changes. `govops prove` detects the mismatch when run in the CI/CD
pipeline (UC-08):

```
$ govops prove \
    --claim GOVOPS-INT-01.02 \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar.cedar \
    --engine cedar \
    --output govops/proofs/

govops prove v0.9.0
Claim:    GOVOPS-INT-01.02
Engine:   cedar
Policy:   govops/policies/payments-cedar.cedar
Hash:     sha256:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d7

WARNING: Stale proof detected.
  Existing proof: govops/proofs/GOVOPS-INT-01.02-2026-05-14.json
  Proof policy hash:   sha256:a3f8c2d1e9b047f6a2c5d8e1f3b6a9c2d5e8f1b4a7c0d3e6f9b2c5d8e1f4a7c0
  Current policy hash: sha256:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d7
  The proof is no longer valid for the current policy. Re-running analysis...

Invoking Cedar symbolic analyzer...
  Result: UNSAT

Proof artifact written: govops/proofs/GOVOPS-INT-01.02-2026-05-28.json
Evaluation log updated: govops/evidence/eval-log-acme-2026-05-28.yaml
```

The stale proof is replaced with a fresh one anchored to the new policy hash. The
TIGER Integrity pillar score (which had dropped when the stale proof was detected)
recovers after the refresh (UC-04).

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Cedar proof (UNSAT) | Proof artifact | `govops/proofs/GOVOPS-INT-01.02-2026-05-14.json` | Created by `govops prove` |
| OpenFGA proof (UNSAT) | Proof artifact | `govops/proofs/GOVOPS-RS-01.01-2026-05-14.json` | Created by `govops prove` |
| Evaluation log | `#EvaluationLog` | `govops/evidence/eval-log-acme-2026-05-14.yaml` | Updated with `#ArtifactMapping` entries |
| Refreshed Cedar proof | Proof artifact | `govops/proofs/GOVOPS-INT-01.02-2026-05-28.json` | Replaces stale proof after policy redeploy |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** `payments:transfer:bank-account` and `iam:assume:role`
  both match the three-segment format throughout.
- **P2 (AssessmentRequirement ID Format):** `GOVOPS-INT-01.02` and `GOVOPS-RS-01.01`
  match `^GOVOPS-[A-Z]+-[0-9]+\.[0-9]+$`.
- **P4 (PARC Context Predicate Consistency):** The counterexample PARC Context
  `{ "acr": "urn:password-only" }` is consistent with the capability's
  `documented-context-expectations` (`context.acr == 'urn:mfa'`) — the counterexample
  shows a context that violates the expectation, which is exactly what a SAT result
  means.
- **P5 (Proof Logical Consistency):** The UNSAT result in Step 2 is accompanied by
  the claim whose negation is the formula checked. The SAT result in Step 4 is
  accompanied by a concrete counterexample showing the PARC fields that satisfy the
  negation. No UNSAT result appears with a counterexample; no SAT result appears
  without one.

### Cross-References

| Item | Reference |
|---|---|
| Engine-neutral provable claim | Design §8.1 |
| Recording a proof in Gemara | Design §8.2 |
| `#EvaluationLog` and `#ArtifactMapping` | Design §8.2 |
| `govops prove` tool | Design §12 |
| Cedar PDP class | Design §8.1 (policy-language PDP) |
| OpenFGA PDP class | Design §8.1 (graph/relationship PDP) |
| Stale proof detection | Design §9.1 (policy hash in score computation) |
| TIGER Integrity pillar | Design §7.2 |


---

## UC-04: TIGER Pillar Score Computation and Reporting

### Context

After controls are authored and proofs are run, the GRC Practitioner needs to
understand the organization's governance maturity across the five TIGER pillars. This
use case shows how `govops tiger` computes five independent ordinal scores (one per
pillar, each 1–5), how those scores change when the catalog or proofs change, and how
the scores are used alongside Policy Coverage in a governance review meeting. There is
no single aggregate TIGER score.

### Actors

- **Primary:** GRC Practitioner (Acme Bank, authorization governance)
- **Secondary:** Compliance Auditor (attends governance review meeting)

### Preconditions

- UC-01, UC-02, and UC-03 are complete: the catalog, pillar control catalogs, and
  proof artifacts are in place.
- `govops/evidence/eval-log-acme-2026-05-14.yaml` exists with proof evidence entries.
- The GovOps toolchain (`govops tiger`) is installed.

### Step-by-Step Workflow

**Step 1 — Run `govops tiger` to compute the five per-pillar scores.**

```
$ govops tiger \
    --catalog govops/GovOps-AC.yaml \
    --controls govops/GovOps-Transparency.yaml \
                govops/GovOps-Integrity.yaml \
                govops/GovOps-Governance.yaml \
                govops/GovOps-Events.yaml \
                govops/GovOps-Resilience.yaml \
    --evidence govops/evidence/ \
    --proofs govops/proofs/ \
    --output govops/tiger/

govops tiger v0.9.0

Computing TIGER pillar scores for Acme Bank (2026-05-14)...

  Transparency  : 3 / 5  (2 of 2 requirements satisfied; limited to medium-risk capabilities)
  Integrity     : 4 / 5  (2 of 2 requirements satisfied for payments:transfer:bank-account)
  Governance    : 3 / 5  (1 of 3 requirements satisfied; 2 provable claims pending)
  Events        : 3 / 5  (2 of 3 requirements satisfied; replay not yet verified)
  Resilience    : 3 / 5  (2 of 3 requirements satisfied; fail-closed not yet proven)

Scores written to: govops/tiger/tiger-score.yaml
Evidence written to: govops/tiger/tiger-evidence.json
```

**Step 2 — Review the `tiger-score.yaml` output.**

```yaml
# govops/tiger/tiger-score.yaml
metadata:
  id: tiger.acme.2026-05-14
  generated-at: "2026-05-14T10:00:00Z"
  policy-versions:
    - engine: cedar
      version: "payments-cedar-v2026.05.14-1"
    - engine: openfga
      version: "iam-openfga-v2026.05.14-1"
  catalog-version: "ec.acme:2026.1"
score:
  transparency: 3
  integrity: 4
  governance: 3
  events: 3
  resilience: 3
```

**Maturity level semantics for each score (1–5):**

| Score | Maturity level | Meaning |
|---|---|---|
| 1 | Initial | No controls declared; no evidence; governance is ad hoc |
| 2 | Developing | Some controls declared; configuration assertions only; no proofs |
| 3 | Defined | Controls declared for critical capabilities; some assertions satisfied; proofs in progress |
| 4 | Managed | All critical capabilities governed; most provable claims satisfied with current proofs |
| 5 | Optimizing | All capabilities governed; all provable claims satisfied; continuous proof refresh in CI/CD |

The Integrity pillar scores 4 because both `GOVOPS-INT-01.01` (configuration assertion)
and `GOVOPS-INT-01.02` (provable claim) are satisfied for `payments:transfer:bank-account`.
The Governance pillar scores 3 because `GOVOPS-GV-01.02` and `GOVOPS-GV-01.03` are
not yet satisfied.

> **Note on the Identity pillar:** The Identity pillar score is not shown in this
> example. The scoring criteria and control catalog contents for the Identity pillar
> are defined by the GovOps Initiative Orbit WG and are not prescribed by this
> document. Any Identity pillar score shown in an implementation would be illustrative
> only.

**Step 3 — Review the `tiger-evidence.json` per-pillar inputs.**

```json
{
  "generated-at": "2026-05-14T10:00:00Z",
  "catalog-version": "ec.acme:2026.1",
  "pillars": {
    "transparency": {
      "score": 3,
      "requirements": [
        {
          "id": "GOVOPS-TR-01.01",
          "capability": "payments:transfer:bank-account",
          "status": "satisfied",
          "evidence-type": "DecisionLog",
          "evidence-ref": "evidence/payments-decisions-2026-05-14.json"
        },
        {
          "id": "GOVOPS-TR-01.02",
          "capability": "payments:transfer:bank-account",
          "status": "satisfied",
          "evidence-type": "DecisionLog",
          "evidence-ref": "evidence/payments-decisions-2026-05-14.json"
        }
      ],
      "notes": "Score limited to 3: Transparency controls cover only payments:transfer:bank-account. payments:read:invoice and iam:assume:role have no Transparency controls yet."
    },
    "integrity": {
      "score": 4,
      "requirements": [
        {
          "id": "GOVOPS-INT-01.01",
          "capability": "payments:transfer:bank-account",
          "status": "satisfied",
          "evidence-type": "DecisionLog",
          "evidence-ref": "evidence/payments-decisions-2026-05-14.json"
        },
        {
          "id": "GOVOPS-INT-01.02",
          "capability": "payments:transfer:bank-account",
          "status": "satisfied",
          "evidence-type": "Proof",
          "evidence-ref": "proofs/GOVOPS-INT-01.02-2026-05-14.json",
          "proof-result": "UNSAT",
          "policy-hash": "sha256:a3f8c2d1e9b047f6a2c5d8e1f3b6a9c2d5e8f1b4a7c0d3e6f9b2c5d8e1f4a7c0"
        }
      ],
      "notes": "Score 4: all declared Integrity requirements satisfied. Score would reach 5 when Integrity controls are extended to governance:policy-write:governance-policy."
    },
    "governance": {
      "score": 3,
      "requirements": [
        {
          "id": "GOVOPS-GV-01.01",
          "capability": "governance:policy-write:governance-policy",
          "status": "satisfied",
          "evidence-type": "DecisionLog"
        },
        {
          "id": "GOVOPS-GV-01.02",
          "capability": "governance:policy-write:governance-policy",
          "status": "unsatisfied",
          "evidence-type": "Proof",
          "evidence-ref": null,
          "notes": "No proof artifact found. govops prove not yet run for GOVOPS-GV-01.02."
        },
        {
          "id": "GOVOPS-GV-01.03",
          "capability": "governance:policy-write:governance-policy",
          "status": "unsatisfied",
          "evidence-type": "DecisionLog",
          "notes": "No evidence of override-expiry mechanism in decision logs."
        }
      ],
      "notes": "Score 3: 1 of 3 Governance requirements satisfied. Run govops prove for GOVOPS-GV-01.02 and implement override-expiry to improve."
    },
    "events": {
      "score": 3,
      "requirements": [
        { "id": "GOVOPS-EV-01.01", "status": "satisfied" },
        { "id": "GOVOPS-EV-01.02", "status": "satisfied" },
        { "id": "GOVOPS-EV-01.03", "status": "unsatisfied",
          "notes": "Event stream replay not yet verified for investigation use case." }
      ],
      "notes": "Score 3: replay capability not yet demonstrated."
    },
    "resilience": {
      "score": 3,
      "requirements": [
        { "id": "GOVOPS-RS-01.01", "status": "satisfied",
          "evidence-type": "Proof",
          "evidence-ref": "proofs/GOVOPS-RS-01.01-2026-05-14.json" },
        { "id": "GOVOPS-RS-01.02", "status": "satisfied" },
        { "id": "GOVOPS-RS-01.03", "status": "unsatisfied",
          "notes": "Fail-closed behavior for payments:transfer:bank-account not yet proven." }
      ],
      "notes": "Score 3: fail-closed proof pending."
    }
  }
}
```

**Step 4 — Score decrease when a new capability is added without a control.**

The payments team adds a new capability `payments:refund:payment` to `GovOps-AC.yaml`
(risk-tier: high, data-sensitivity: pii) without authoring any TIGER controls for it.
Running `govops tiger` shows the Transparency and Events pillar scores decrease:

```
$ govops tiger  # after adding payments:refund:payment with no controls

  Transparency  : 2 / 5  ← decreased (new high-risk capability has no Transparency control)
  Integrity     : 3 / 5  ← decreased (new high-risk capability has no Integrity control)
  Governance    : 3 / 5  (unchanged)
  Events        : 2 / 5  ← decreased (new capability has no Events control)
  Resilience    : 3 / 5  (unchanged)

Gap identified in tiger-evidence.json:
  "notes": "Score decreased: payments:refund:payment (risk-tier: high) has no
            AssessmentRequirement in Transparency, Integrity, or Events pillars."
```

The `notes` field in `tiger-evidence.json` identifies the specific gap in catalog
terms, making it actionable.

**Step 5 — Score decrease when a proof is invalidated.**

After the payments Cedar policy is redeployed (new hash), the existing proof for
`GOVOPS-INT-01.02` becomes stale. Before `govops prove` is re-run, `govops tiger`
detects the stale proof and decreases the Integrity score:

```
$ govops tiger  # after policy redeploy, before govops prove refresh

  Integrity     : 3 / 5  ← decreased (GOVOPS-INT-01.02 proof is stale: policy hash mismatch)

  tiger-evidence.json notes:
    "GOVOPS-INT-01.02: proof stale. Proof policy hash sha256:a3f8... does not match
     current policy hash sha256:9e8d.... Re-run govops prove to refresh."
```

After `govops prove` refreshes the proof (UC-03 Step 6), the Integrity score returns
to 4.

**Step 6 — Governance review meeting: TIGER scores + Policy Coverage together.**

At the monthly governance review, the GRC Practitioner presents both metrics:

| Metric | Value | What it measures |
|---|---|---|
| Policy Coverage | 67% (10/15 requirements satisfied) | Fraction of declared requirements currently satisfied |
| Transparency score | 3/5 | Maturity of decision observability governance |
| Integrity score | 4/5 | Maturity of context/evidence precondition governance |
| Governance score | 3/5 | Maturity of governance-plane authority controls |
| Events score | 3/5 | Maturity of decision telemetry governance |
| Resilience score | 3/5 | Maturity of degraded-mode enforcement governance |

**What TIGER does NOT measure:**
- Adversarial robustness against unmodeled attacks (e.g., novel exploit paths not
  captured in the capability catalog)
- Quality of the catalog itself (an under-specified catalog can yield high TIGER scores
  with trivial controls)
- Workforce administration or HR identity lifecycle, except where it surfaces as PARC
  Context evidence in authorization requests
- IGA recertification cadence or entitlement hygiene (those are IGA metrics, UC-06)

The review meeting uses the `notes` fields in `tiger-evidence.json` to drive the
remediation backlog: each unsatisfied requirement is a concrete, actionable item
expressed in catalog terms.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| TIGER score | Derived | `govops/tiger/tiger-score.yaml` | Created/updated by `govops tiger` |
| TIGER evidence | Derived | `govops/tiger/tiger-evidence.json` | Created/updated by `govops tiger` |

### Correctness Properties Demonstrated

- **P2 (AssessmentRequirement ID Format):** All requirement ids in `tiger-evidence.json`
  match `^GOVOPS-[A-Z]+-[0-9]+\.[0-9]+$`.
- **P3 (TIGER Score Integrity):** `tiger-score.yaml` contains exactly five per-pillar
  integer scores (`transparency: 3`, `integrity: 4`, `governance: 3`, `events: 3`,
  `resilience: 3`), all in {1, 2, 3, 4, 5}. No `aggregate` field is present.
- **P8 (Identity Pillar Non-Prescription):** The use case explicitly notes that the
  Identity pillar score is not shown and that scoring criteria are defined by the
  GovOps Initiative Orbit WG.

### Cross-References

| Item | Reference |
|---|---|
| TIGER metric definition | Design §9 |
| Per-pillar score formula | Design §9.1 |
| `tiger-score.yaml` shape | Design §9.4 |
| `tiger-evidence.json` | Design §9.4 |
| `govops tiger` tool | Design §12 |
| TIGER pillar catalogs | Design §7.1–7.5 |
| What TIGER does not measure | Design §9.3 |
| Policy Coverage (distinct metric) | Design §12 (`govops coverage`) |


---

## UC-05: Compliance Mapping and Audit

### Context

Compliance auditors need to trace from external framework controls (NIST 800-53,
ISO 27001) through the GovOps repository to concrete proof artifacts. This use case
shows how a `#MappingDocument` links TIGER controls to two external frameworks, how
an auditor queries the mapping to identify governance gaps, and how the evidence trace
from compliance control to proof artifact is constructed. It also shows how the auditor
distinguishes observable claims (runtime evidence) from symbolic proofs and what each
implies about assurance level.

### Actors

- **Primary:** Compliance Auditor (Acme Bank, internal audit)
- **Secondary:** GRC Practitioner (provides mapping context and evidence)

### Preconditions

- UC-01 through UC-04 are complete: catalog, pillar catalogs, proofs, and TIGER scores
  are in place.
- `govops/tiger/tiger-evidence.json` exists.
- The auditor has access to the GovOps repository (read-only).

### Step-by-Step Workflow

**Step 1 — Review the `#MappingDocument` linking TIGER controls to NIST 800-53 and ISO 27001.**

The GRC Practitioner has authored `govops/mappings/govops-to-nist-iso.yaml` mapping
TIGER Integrity and Governance controls to NIST 800-53 AC-3 and ISO 27001 A.5.15.

```yaml
# govops/mappings/govops-to-nist-iso.yaml
title: GovOps TIGER Controls to NIST 800-53r5 and ISO 27001:2022
metadata:
  id: map.acme.govops.nist-iso
  type: MappingDocument
  gemara-version: "0.x"
  description: >
    Maps GovOps TIGER Integrity and Governance controls to NIST SP 800-53 Rev. 5
    and ISO/IEC 27001:2022 controls.
  author: { id: acme-grc, name: Acme GRC Team, type: Software Assisted }
  mapping-references:
    - id: govops-int
      title: GovOps Integrity Catalog
      version: "0.1"
      url: govops/GovOps-Integrity.yaml
    - id: govops-gv
      title: GovOps Governance Catalog
      version: "0.1"
      url: govops/GovOps-Governance.yaml
    - id: nist80053r5
      title: NIST SP 800-53 Rev. 5
      version: "5.1.1"
    - id: iso27001-2022
      title: ISO/IEC 27001:2022
      version: "2022"
source-reference:
  reference-id: govops-int
  entry-type: Control
target-reference:
  reference-id: nist80053r5
  entry-type: Control
mappings:
  - id: m.int.ac3
    source: GOVOPS-INT-01
    relationship: implements
    targets:
      - entry-id: AC-3
        rationale: >
          Access enforcement on sensitive (action, resource) capabilities.
          GOVOPS-INT-01 requires that high-risk capabilities enforce authentication
          context preconditions on PARC requests.
        confidence-level: High
      - entry-id: IA-2(1)
        rationale: >
          Strong authentication evidence required in PARC Context for privileged
          (action, resource) pairs.
        confidence-level: High
  - id: m.int.iso
    source: GOVOPS-INT-01
    relationship: implements
    targets:
      - entry-id: A.5.15
        rationale: >
          Access control policy enforcement for sensitive capabilities.
        confidence-level: High
  - id: m.gv.ac3
    source: GOVOPS-GV-01
    relationship: implements
    targets:
      - entry-id: AC-3
        rationale: >
          Separation of duties for governance policy changes enforces access control
          on the governance plane itself.
        confidence-level: High
      - entry-id: AC-5
        rationale: >
          Separation of duties: no single party can unilaterally modify governance policy.
        confidence-level: High
  - id: m.gv.iso
    source: GOVOPS-GV-01
    relationship: implements
    targets:
      - entry-id: A.5.15
        rationale: >
          Access control for governance artifacts.
        confidence-level: High
      - entry-id: A.5.3
        rationale: >
          Segregation of duties for governance policy changes.
        confidence-level: High
```

**Step 2 — Auditor query: capabilities at risk-tier >= high lacking a TIGER control mapped to AC-3.**

The auditor runs a query against the mapping document and capability catalog:

```
$ govops coverage \
    --catalog govops/GovOps-AC.yaml \
    --controls govops/GovOps-Integrity.yaml govops/GovOps-Governance.yaml \
    --mappings govops/mappings/govops-to-nist-iso.yaml \
    --query "risk-tier >= high AND no-control-mapped-to AC-3"

govops coverage v0.9.0 — Compliance Query

Query: capabilities with risk-tier >= high that lack a TIGER control mapped to AC-3

Results:
  ⚠ iam:assume:role
      risk-tier: high
      sensitivity: internal
      TIGER controls: GOVOPS-RS-01 (Resilience only)
      AC-3 mapping: NONE — no Integrity or Governance control mapped to AC-3 for this capability

  ✓ payments:transfer:bank-account
      risk-tier: critical
      TIGER controls: GOVOPS-INT-01 (mapped to AC-3 via m.int.ac3)
      AC-3 mapping: PRESENT

  ✓ governance:policy-write:governance-policy
      risk-tier: critical
      TIGER controls: GOVOPS-GV-01 (mapped to AC-3 via m.gv.ac3)
      AC-3 mapping: PRESENT

1 capability at risk-tier >= high lacks an AC-3-mapped TIGER control: iam:assume:role
```

The auditor identifies `iam:assume:role` as a gap and requests that the GRC
Practitioner author an Integrity control for it before the audit closes.

**Step 3 — Evidence trace: from compliance control id to proof artifact.**

The auditor requests evidence for NIST AC-3 compliance for
`payments:transfer:bank-account`. The trace is:

```
Compliance control:  NIST AC-3 (Access Enforcement)
        ↓ via MappingDocument map.acme.govops.nist-iso (mapping m.int.ac3)
TIGER control:       GOVOPS-INT-01 (Sensitive capabilities require strong authentication evidence)
        ↓ assessment-requirements
AssessmentRequirement: GOVOPS-INT-01.01 (configuration assertion)
                       GOVOPS-INT-01.02 (provable claim)
        ↓ via EvaluationLog log.acme.ec.integrity.2026-05-14
EvaluationLog:       govops/evidence/eval-log-acme-2026-05-14.yaml
        ↓ evidence entries (ArtifactMapping)
Proof artifact:      govops/proofs/GOVOPS-INT-01.02-2026-05-14.json
                     result: UNSAT, policy-hash: sha256:a3f8...
Decision log:        govops/evidence/payments-decisions-2026-05-14.json
                     1,247 Permit decisions, all with context.acr=urn:mfa
```

**Step 4 — `tiger-evidence.json` as the auditor-facing trail.**

The auditor relies on `tiger-evidence.json` as the primary evidence trail. The fields
the auditor uses:

| Field | Auditor use |
|---|---|
| `pillars.<pillar>.requirements[].id` | Identifies which `#AssessmentRequirement` was evaluated |
| `pillars.<pillar>.requirements[].status` | Pass/fail for each requirement |
| `pillars.<pillar>.requirements[].evidence-type` | Distinguishes `Proof` from `DecisionLog` |
| `pillars.<pillar>.requirements[].evidence-ref` | Path to the proof artifact or decision log |
| `pillars.<pillar>.requirements[].proof-result` | `UNSAT` or `SAT` for provable claims |
| `pillars.<pillar>.requirements[].policy-hash` | Anchors the proof to a specific policy version |
| `pillars.<pillar>.notes` | Human-readable gap explanation in catalog terms |

**Step 5 — Distinguishing observable claims from symbolic proofs.**

The auditor encounters two satisfied requirements for `payments:transfer:bank-account`:

| Requirement | Evidence type | Assurance level |
|---|---|---|
| `GOVOPS-INT-01.01` | `DecisionLog` (observable claim) | **Observed:** the control was present and exercised in the reporting window. Does not prove the control is sufficient for all possible requests. |
| `GOVOPS-INT-01.02` | `Proof` (symbolic proof, UNSAT) | **Proved:** no authorization path exists in the deployed policy where the capability is permitted without `context.acr == 'urn:mfa'`. Stronger assurance than observation alone. |

The auditor notes that `GOVOPS-INT-01.01` (observable) provides evidence of *presence*;
`GOVOPS-INT-01.02` (symbolic proof) provides evidence of *sufficiency*. Both are
recorded in the evaluation log; the proof artifact is the higher-assurance evidence.
For a SOC 2 Type II audit, the decision log covers the observation period; for a
security assessment, the UNSAT proof is the stronger artifact.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| Compliance mapping | `#MappingDocument` | `govops/mappings/govops-to-nist-iso.yaml` | Created by GRC Practitioner |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in the mapping and query output
  match the three-segment format.
- **P2 (AssessmentRequirement ID Format):** `GOVOPS-INT-01.01`, `GOVOPS-INT-01.02`,
  `GOVOPS-GV-01.01`–`GOVOPS-GV-01.03` all match the format regex.
- **P5 (Proof Logical Consistency):** The UNSAT proof for `GOVOPS-INT-01.02` is
  accompanied by the claim whose negation was checked; no counterexample is shown
  because the result is UNSAT.
- **P6 (Lint/Coverage Output Consistency):** The `govops coverage` query output
  references only `iam:assume:role`, `payments:transfer:bank-account`, and
  `governance:policy-write:governance-policy` — all present in the UC-01 catalog.

### Cross-References

| Item | Reference |
|---|---|
| `#MappingDocument` artifact | Design §11 |
| Compliance framework mappings | Design §11 |
| `govops coverage` compliance query | Design §12 |
| Observable claim vs. symbolic proof | Design §8.3 |
| `tiger-evidence.json` as audit trail | Design §9.4 |
| NIST 800-53 AC-3 | External: NIST SP 800-53 Rev. 5 |
| ISO 27001 A.5.15 | External: ISO/IEC 27001:2022 |


---

## UC-06: IGA Integration

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
- Deployed policy artifacts exist in `govops/policies/` for Cedar and OpenFGA engines.

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
governance:policy-write:governance-policy,Modify governance policy,governance,policy-write,GovernancePolicy,critical,,runtime
iam:assume:role,Assume IAM role,iam,assume,Role,high,internal,runtime
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
governance:policy-write:governance-policy,Modify governance policy,critical,
```

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
    --policy govops/policies/iam-openfga-model.json \
    --engine openfga

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

OpenFGA engine (govops/policies/iam-openfga-model.json):
  No drift detected.

Summary: 1 drift finding (Type B). 0 Type A. 0 Type C.
```

The IGA Team escalates the `payments:batch-transfer:bank-account` finding to the
Platform Security Engineer. The engineer determines this is a legitimate capability
that was added to the Cedar policy without going through the catalog authoring process
(UC-01). Resolution: add `payments:batch-transfer:bank-account` to `GovOps-AC.yaml`
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
- **P6 (Lint/Coverage Output Consistency):** The `govops drift` output references only
  `payments:batch-transfer:bank-account` as a Type B finding — a capability present in
  the Cedar policy but absent from the catalog excerpt shown in UC-01.

### Cross-References

| Item | Reference |
|---|---|
| IGA Exporter tool | Design §12 |
| `applicability-groups` for scoping | Design §6.3 |
| `govops drift` tool | Design §12 |
| Capability deprecation pattern | Design §14 (open question 5) |
| Relationship to IGA entitlement catalogs | Design §4 (non-goals) |
| `#CapabilityCatalog` | Design §6 |


---

## UC-07: Policy Drift Detection

### Context

The capability catalog expresses governance intent; deployed policy expresses
enforcement reality. Drift between the two is inevitable in a fast-moving engineering
organization. This use case shows how `govops drift` detects three categories of
divergence across two policy engines (Cedar and OPA/Rego), how the engineer decides
how to resolve each finding, and how drift detection integrates into CI/CD to prevent
ungoverned policy changes from reaching production.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank, payments and IAM domains)
- **Secondary:** Policy Engine Operator (owns Cedar and OPA deployments)

### Preconditions

- `govops/GovOps-AC.yaml` passes `govops lint` (UC-01 complete).
- Deployed policy artifacts exist:
  - `govops/policies/payments-cedar.cedar` (Cedar, payments service)
  - `govops/policies/iam-opa.rego` (OPA/Rego, IAM service)
- The `govops drift` Cedar and OPA analyzer plug-ins are installed.

### Step-by-Step Workflow

**Step 1 — Run `govops drift` against both policy engines.**

```
$ govops drift \
    --catalog govops/GovOps-AC.yaml \
    --policy govops/policies/payments-cedar.cedar --engine cedar \
    --policy govops/policies/iam-opa.rego --engine opa

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

OPA engine (govops/policies/iam-opa.rego):
  [Type B] Policy rule with no catalog entry:
    ⚠ action: "delegate", resource: "Role"
        Found in OPA rule: allow { input.action == "delegate"; input.resource.type == "Role" }
        No corresponding capability in GovOps-AC.yaml
        Suggested id: iam:delegate:role
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

The capability was added to the catalog (UC-06 Step 5) but the Cedar policy has not
yet been updated to enforce it. The engineer has three options:

| Option | Action | When to use |
|---|---|---|
| Update policy | Add Cedar rule for `read` on `InvoiceSummary` | Capability is live and should be enforced |
| Update catalog | Mark capability `lifecycle-stage: design-time` | Capability is planned but not yet deployed |
| Accept exception | Document in `govops/exceptions/drift-exceptions.yaml` | Intentional gap with documented rationale |

Decision: the payments team confirms `payments:read:invoice-summary` is live. The
Policy Engine Operator adds the Cedar rule. After the policy update, re-running
`govops drift` shows no Type A finding for this capability.

**Scenario (b) — Type B: `iam:delegate:role` in OPA policy, not in catalog.**

The IAM team added a delegation capability to the OPA policy without going through
the catalog authoring process. The engineer adds `iam:delegate:role` to `GovOps-AC.yaml`:

```yaml
  - id: iam:delegate:role
    title: Delegate IAM role
    group: g.iam
    action: delegate
    resource: Role
    documented-requester-classes: [interactive-subject]
    risk-tier: high
    lifecycle-stage: runtime
    description: >
      Temporarily delegate the permissions of a named IAM role to another principal.
```

After adding the entry and running `govops lint` to validate, re-running `govops drift`
shows no Type B finding for this capability.

**Scenario (c) — Type C: `payments:transfer:bank-account` context-expectations diverge.**

The Cedar policy has a service-account bypass that permits transfers without MFA —
the same issue surfaced as a SAT counterexample in UC-03 Step 4. The drift report
confirms the divergence from a different angle: the `documented-context-expectations`
field says MFA is required, but the deployed policy does not enforce it for service
accounts.

Decision: the engineer removes the service-account bypass from the Cedar policy
(consistent with the remediation in UC-03). After the policy update, re-running
`govops drift` shows no Type C finding.

**Step 3 — CI/CD integration: drift flagged before production.**

The drift check is integrated into the CI/CD pipeline as a gate (UC-08). When the
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
| Updated capability catalog | `#CapabilityCatalog` | `govops/GovOps-AC.yaml` | Modified: added `iam:delegate:role` |
| Drift exception log | Convention | `govops/exceptions/drift-exceptions.yaml` | Created if exceptions are accepted |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in the drift report (`payments:read:invoice-summary`,
  `payments:transfer:bank-account`, `iam:delegate:role`) match the three-segment format.
  The suggested id `iam:delegate:role` also matches.
- **P4 (PARC Context Predicate Consistency):** The Type C drift finding for
  `payments:transfer:bank-account` correctly identifies the divergence between the
  capability's `documented-context-expectations` (`context.acr == 'urn:mfa'`) and the
  deployed Cedar policy (which permits without MFA for service accounts). This is
  consistent with the capability excerpt shown in UC-01.
- **P6 (Lint/Coverage Output Consistency):** The `govops drift` output references only
  capability ids that appear in the catalog or are suggested as new entries — no
  phantom ids.

### Cross-References

| Item | Reference |
|---|---|
| `govops drift` tool | Design §12 |
| `documented-context-expectations` field | Design §6.2(B) |
| Cedar PDP class | Design §8.1 |
| OPA/Rego PDP class | Design §8.1 |
| Drift as input to TIGER | Design §9.1 (policy hash) |
| CI/CD integration | Design §13 (Phase 4) |
| Type C drift and provable claims | UC-03 Step 4 (same issue, different detection method) |


---

## UC-08: Continuous TIGER in CI/CD

### Context

Point-in-time governance assessments are insufficient for a fast-moving engineering
organization. This use case shows how the full GovOps toolchain integrates into a
CI/CD pipeline to provide continuous TIGER score maintenance: every policy change
triggers re-evaluation of affected provable claims, drift detection, and TIGER score
updates. Gate conditions prevent ungoverned policy changes from reaching production.

### Actors

- **Primary:** Platform Security Engineer (Acme Bank, CI/CD and GovOps automation)
- **Secondary:** GRC Practitioner (sets gate thresholds; reviews score changes)
- **Secondary:** Policy Engine Operator (submits policy changes via pull requests)

### Preconditions

- UC-01 through UC-07 are complete: catalog, pillar catalogs, proofs, TIGER scores,
  and drift baselines are in place.
- The CI/CD system (e.g., GitHub Actions, GitLab CI) is configured to run the GovOps
  toolchain on policy change pull requests.
- Gate thresholds are configured in `govops/.govops-ci.yaml`.

### Step-by-Step Workflow

**Step 1 — Configure the CI/CD pipeline and gate thresholds.**

```yaml
# govops/.govops-ci.yaml
pipeline:
  steps:
    - name: lint
      command: govops lint
      args: [--catalog, govops/GovOps-AC.yaml, --lexicon, govops/lexicon.yaml]
    - name: coverage
      command: govops coverage
      args: [--catalog, govops/GovOps-AC.yaml,
             --controls, govops/GovOps-Integrity.yaml, govops/GovOps-Governance.yaml,
                         govops/GovOps-Events.yaml, govops/GovOps-Resilience.yaml,
             --evidence, govops/evidence/, --proofs, govops/proofs/]
    - name: drift
      command: govops drift
      args: [--catalog, govops/GovOps-AC.yaml,
             --policy, govops/policies/payments-cedar.cedar, --engine, cedar,
             --policy, govops/policies/iam-opa.rego, --engine, opa]
    - name: prove
      command: govops prove
      args: [--catalog, govops/GovOps-AC.yaml,
             --all-claims, --evidence, govops/evidence/, --proofs, govops/proofs/]
    - name: tiger
      command: govops tiger
      args: [--catalog, govops/GovOps-AC.yaml,
             --controls, govops/GovOps-Transparency.yaml, govops/GovOps-Integrity.yaml,
                         govops/GovOps-Governance.yaml, govops/GovOps-Events.yaml,
                         govops/GovOps-Resilience.yaml,
             --evidence, govops/evidence/, --proofs, govops/proofs/,
             --output, govops/tiger/]

gates:
  lint:
    fail-on-errors: true
    fail-on-warnings: false
  coverage:
    minimum-policy-coverage: 0.70   # 70% overall
    minimum-critical-coverage: 1.00 # 100% for critical capabilities
  drift:
    fail-on-any-drift: true
  tiger:
    minimum-per-pillar:
      transparency: 3
      integrity: 3
      governance: 3
      events: 3
      resilience: 3
    # No aggregate threshold — per-pillar minimums only
```

**Step 2 — Pipeline execution on a policy change pull request.**

The Policy Engine Operator submits PR #512: a Cedar policy update that adds a new
rule for `payments:transfer:bank-account`. The CI/CD pipeline runs the full sequence:

```
CI/CD Pipeline — policy-change-pr-512
Triggered by: PR #512 (payments-cedar.cedar: add batch-transfer rule)
──────────────────────────────────────────────────────────────────────

Step 1: govops lint
  ✓ 5 capabilities validated. 0 errors. 0 warnings.
  Gate: PASS

Step 2: govops coverage
  Policy Coverage: 10 / 15 requirements satisfied (67%)
  Critical coverage: 5 / 5 (100%)
  Gate: PASS (overall 67% >= 70% threshold? NO — FAIL)

  ✗ GATE FAILED: Policy Coverage 67% is below the configured minimum of 70%.
    3 unsatisfied requirements:
      GOVOPS-GV-01.02  governance:policy-write:governance-policy  [provable-claim, no proof]
      GOVOPS-GV-01.03  governance:policy-write:governance-policy  [configuration-assertion, no evidence]
      GOVOPS-EV-01.03  (all capabilities)  [configuration-assertion, replay not verified]
    Recommendation: Satisfy at least 1 additional requirement before merging.
  EXIT 1
```

The build fails at the coverage gate. The GRC Practitioner reviews the failing
requirements and decides to run `govops prove` for `GOVOPS-GV-01.02` before the PR
can merge.

**Step 3 — Policy change triggers re-evaluation of affected provable claims.**

After the GRC Practitioner runs `govops prove` for `GOVOPS-GV-01.02` and the proof
is committed alongside the policy change, the pipeline re-runs:

```
CI/CD Pipeline — policy-change-pr-512 (re-run after proof added)
──────────────────────────────────────────────────────────────────

Step 1: govops lint
  ✓ PASS

Step 2: govops coverage
  Policy Coverage: 11 / 15 requirements satisfied (73%)
  Critical coverage: 5 / 5 (100%)
  Gate: PASS (73% >= 70%)

Step 3: govops drift
  Cedar engine: 0 findings
  OPA engine:   0 findings
  Gate: PASS

Step 4: govops prove
  Re-evaluating affected claims for changed policy: payments-cedar.cedar
  Hash: sha256:7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5

  GOVOPS-INT-01.02  payments:transfer:bank-account  → UNSAT  ✓ (proof refreshed)
  GOVOPS-GV-01.02   governance:policy-write:governance-policy → UNSAT  ✓ (new proof)

  2 proofs refreshed. 0 failures.
  Gate: PASS

Step 5: govops tiger
  transparency: 3  ✓ (>= minimum 3)
  integrity:    4  ✓ (>= minimum 3)
  governance:   4  ✓ (>= minimum 3)  ← improved from 3 (GOVOPS-GV-01.02 now satisfied)
  events:       3  ✓ (>= minimum 3)
  resilience:   3  ✓ (>= minimum 3)
  Gate: PASS

All gates passed. PR #512 is approved for merge.
```

**Step 4 — Proof artifacts and evaluation logs versioned alongside policy artifacts.**

The pipeline commits the updated proof artifacts and evaluation log alongside the
policy change in the same pull request:

```
PR #512 — files changed:
  govops/policies/payments-cedar.cedar          (policy change)
  govops/proofs/GOVOPS-INT-01.02-2026-05-28.json  (refreshed proof)
  govops/proofs/GOVOPS-GV-01.02-2026-05-28.json   (new proof)
  govops/evidence/eval-log-acme-2026-05-28.yaml   (updated evaluation log)
  govops/tiger/tiger-score.yaml                   (updated TIGER scores)
  govops/tiger/tiger-evidence.json                (updated evidence)
```

This ensures that the governance state at any point in time is reproducible: checking
out any commit gives a consistent view of the policy, proofs, evaluation logs, and
TIGER scores.

**Step 5 — Analyzer unavailable: fail-closed vs. degrade gracefully.**

During a pipeline run, the Cedar symbolic analyzer service is temporarily unavailable.
The `govops prove` step handles this according to the capability's risk-tier:

```
Step 4: govops prove
  Invoking Cedar symbolic analyzer for GOVOPS-INT-01.02...
  ERROR: Cedar analyzer service unavailable (connection timeout after 30s)

  Applying degraded-mode policy:
    payments:transfer:bank-account  risk-tier: critical
      → FAIL-CLOSED: proof cannot be refreshed; treating as UNSATISFIED
      → Integrity pillar score will reflect missing proof

    payments:read:invoice-summary  risk-tier: low
      → DEGRADE GRACEFULLY: using last known proof (may be stale)
      → Warning added to tiger-evidence.json

  Gate: FAIL (critical capability proof unavailable → fail-closed)
  EXIT 1

  Recommendation: Retry when Cedar analyzer is available.
                  Do not merge policy changes affecting critical capabilities
                  until proofs are refreshed.
```

The TIGER score reflects the missing proof:

```yaml
# govops/tiger/tiger-score.yaml (degraded run)
score:
  transparency: 3
  integrity: 3   # ← decreased from 4: GOVOPS-INT-01.02 proof unavailable (fail-closed)
  governance: 4
  events: 3
  resilience: 3
```

```json
// tiger-evidence.json (degraded run, integrity pillar excerpt)
{
  "integrity": {
    "score": 3,
    "requirements": [
      {
        "id": "GOVOPS-INT-01.02",
        "capability": "payments:transfer:bank-account",
        "status": "unsatisfied",
        "notes": "Proof unavailable: Cedar analyzer service unreachable. Fail-closed applied for critical capability. Score decreased until proof is refreshed."
      }
    ]
  }
}
```

Once the analyzer recovers, the pipeline re-runs `govops prove`, refreshes the proof,
and the Integrity score returns to 4.

### Artifacts Produced or Modified

| Artifact | Gemara type | Path | Notes |
|---|---|---|---|
| CI/CD config | Convention | `govops/.govops-ci.yaml` | Created; defines pipeline and gate thresholds |
| Refreshed proofs | Proof artifacts | `govops/proofs/GOVOPS-INT-01.02-2026-05-28.json`, `govops/proofs/GOVOPS-GV-01.02-2026-05-28.json` | Created/updated by `govops prove` in pipeline |
| Updated evaluation log | `#EvaluationLog` | `govops/evidence/eval-log-acme-2026-05-28.yaml` | Updated by `govops prove` |
| Updated TIGER scores | Derived | `govops/tiger/tiger-score.yaml`, `govops/tiger/tiger-evidence.json` | Updated by `govops tiger` |

### Correctness Properties Demonstrated

- **P1 (Capability ID Format):** All capability ids in pipeline output match the
  three-segment format.
- **P2 (AssessmentRequirement ID Format):** `GOVOPS-INT-01.02`, `GOVOPS-GV-01.02`,
  `GOVOPS-GV-01.03`, `GOVOPS-EV-01.03` all match the format regex.
- **P3 (TIGER Score Integrity):** Both the normal-run and degraded-run `tiger-score.yaml`
  contain exactly five per-pillar integer scores in {1, 2, 3, 4, 5} with no aggregate
  field. The degraded-run score correctly shows `integrity: 3` (decreased from 4).
- **P5 (Proof Logical Consistency):** The UNSAT results for `GOVOPS-INT-01.02` and
  `GOVOPS-GV-01.02` in Step 3 are accompanied by the claims whose negations were
  checked; no counterexamples are shown because both results are UNSAT.

### Cross-References

| Item | Reference |
|---|---|
| GovOps toolchain commands | Design §12 |
| Continuous TIGER (Phase 4) | Design §13 |
| Proof artifact versioning | Design §8.2 |
| Fail-closed semantics | Design §7.5 (`GOVOPS-RS-01.03`) |
| TIGER score decrease on stale proof | Design §9.1 |
| Per-pillar score formula | Design §9.1 |
| Policy Coverage gate | Design §12 (`govops coverage`) |


---

## Summary Table

| Use Case | Primary Persona | TIGER Pillars Exercised | Policy Coverage Relevance | Toolchain Commands | Design §§ Referenced |
|---|---|---|---|---|---|
| UC-01: Capability Catalog Authoring | Platform Security Engineer | None (foundational) | Establishes the capability surface that coverage is measured against | `govops lint` | §6.2, §6.3, §6.4, §12 |
| UC-02: TIGER Control Authoring and Policy Coverage | GRC Practitioner | Integrity, Governance, Resilience | Directly: shows Policy Coverage metric, gap detection, and prioritization | `govops coverage` | §7, §8, §9, §12 |
| UC-03: Provable Claims and Proof Workflow | Platform Security Engineer | Integrity, Resilience | Satisfying provable claims improves coverage and pillar scores | `govops prove` | §8.1, §8.2, §8.3, §12 |
| UC-04: TIGER Pillar Score Computation and Reporting | GRC Practitioner | All five pillars | Shows Policy Coverage alongside TIGER scores in governance review | `govops tiger` | §9.1–9.4, §12 |
| UC-05: Compliance Mapping and Audit | Compliance Auditor | Integrity, Governance | Coverage gaps surface as compliance risks in audit queries | `govops coverage` (query mode) | §11, §12 |
| UC-06: IGA Integration | IGA Team | None directly | Risk-tier/sensitivity groups (which drive coverage bucketing) scope IGA reviews | IGA Exporter, `govops drift` | §6.3, §12 |
| UC-07: Policy Drift Detection | Platform Security Engineer | Integrity (Type C drift) | Drift findings may indicate ungoverned capabilities (coverage gaps) | `govops drift` | §12, §13 |
| UC-08: Continuous TIGER in CI/CD | Platform Security Engineer | All five pillars | Coverage gate (70% minimum) is a hard pipeline gate | `govops lint`, `govops coverage`, `govops drift`, `govops prove`, `govops tiger` | §12, §13 |

---

## Cross-Reference Table

| Use Case | Design §§ | Gemara Artifact Types | TIGER Pillar Controls | Toolchain Commands |
|---|---|---|---|---|
| UC-01 | §6.2(A), §6.2(B), §6.3, §6.4, §12 | `#CapabilityCatalog`, `#EnterpriseCapability`, `#Lexicon`, `#Group` | None | `govops lint` |
| UC-02 | §7, §7.2, §7.3, §7.5, §8, §9, §12 | `#ControlCatalog`, `#AssessmentRequirement`, `#MultiEntryMapping` | `GOVOPS-INT-01`, `GOVOPS-GV-01`, `GOVOPS-RS-01` | `govops coverage` |
| UC-03 | §8.1, §8.2, §8.3, §12 | `#EvaluationLog`, `#ArtifactMapping`, `#Evidence` (Proof, DecisionLog) | `GOVOPS-INT-01.02`, `GOVOPS-RS-01.01` | `govops prove` |
| UC-04 | §9.1–9.4, §12 | `#EvaluationLog`, `tiger-score.yaml`, `tiger-evidence.json` | All five pillars | `govops tiger` |
| UC-05 | §11, §12 | `#MappingDocument`, `#EntryMapping`, `#EvaluationLog` | `GOVOPS-INT-01`, `GOVOPS-GV-01` | `govops coverage` |
| UC-06 | §6.3, §12 | `#CapabilityCatalog`, IGA export (CSV/SCIM/OSCAL) | None directly | IGA Exporter, `govops drift` |
| UC-07 | §12, §13 | `#CapabilityCatalog`, deployed policy artifacts | `GOVOPS-INT-01` (Type C) | `govops drift` |
| UC-08 | §12, §13 | All artifact types | All five pillars | `govops lint`, `govops coverage`, `govops drift`, `govops prove`, `govops tiger` |

---

*This document is a companion to [Cataloging Enterprise Capabilities with Gemara](./enterprise-capability-catalog-design.md).
All capability ids, assessment requirement ids, TIGER scores, and tool outputs in this
document are internally consistent with the Acme Bank payments scenario established in
design §10.*

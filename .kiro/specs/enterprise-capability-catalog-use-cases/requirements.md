# Requirements Document

## Introduction

This document specifies requirements for a **use-cases document** that illustrates how different enterprise personas interact with the GovOps enterprise capability catalog (`GovOps-AC`) and associated toolchain in real-world authorization governance workflows.

Where the design document specifies *what* the system is, the use-cases document shows *how* it is used — through concrete, narrative scenarios grounded in the Gemara artifact model, the PARC authorization request shape, and the `GovOps-AC` capability catalog.

Each use case must be self-contained enough to be read independently, yet cross-referenced to the design document's sections and artifact types so that a reader can trace from scenario to specification.

---

## Glossary

- **Capability**: An **(action, resource)** pair that is the unit of authorization governance in the catalog. Not a principal, not a PARC tuple.
- **PARC**: Principal, Action, Resource, Context — e.g. an OpenID AuthZEN authorization request envelope used at the PDP boundary.
- **GovOps-AC**: The `#CapabilityCatalog` artifact that inventories the enterprise's authorization surface as **(action, resource)** capabilities.
- **Canonical capabilities (Acme Bank scenario)**: `payments:read:invoice`, `payments:transfer:bank-account`, `lending:approve:loan`, and `fraud:flag:transaction` — used consistently across all use cases.
- **Lending Officer**: A business role responsible for loan approval policy, lending risk controls, and stewardship of lending-related capabilities in `GovOps-AC` (e.g., `lending:approve:loan`).
- **Fraud System**: Acme's automated fraud-detection platform — a **non-human** actor (`software-agent` PARC Principal) that requests **`fraud:flag:transaction`** when transaction risk scores exceed policy thresholds.
- **IGA Team**: An Identity Governance and Administration team responsible for access reviews, entitlement lifecycle, and provisioning.
- **Platform Security Engineer**: An engineer responsible for deploying, configuring, and maintaining policy decision points and the GovOps repository.
- **Compliance Auditor**: An internal or external auditor assessing conformance to frameworks such as NIST 800-53, ISO 27001, or SOC 2.
- **Policy Engine Operator**: An engineer responsible for a specific PDP (Cedar, OPA/Rego, OpenFGA, AuthZEN-conformant, etc.) and its policy lifecycle.
- **GovOps Repository**: The `govops/` directory of Gemara artifacts and deployed policy material.
- **govops lint / drift**: The GovOps toolchain commands described in design §12.
- **IGA Exporter**: The toolchain component that translates the capability catalog into a format suitable for IGA platform ingest.
- **Lexicon**: The `#Lexicon` artifact that defines canonical action verbs and resource type names used across the catalog.

---

## Requirements

### Requirement 1: Persona Coverage

**User Story:** As a reader of the use-cases document, I want each key enterprise persona to be represented by at least one concrete use case, so that I can understand how the catalog serves different organizational roles.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL include at least one use case for each of the following personas: Lending Officer, IGA Team, Platform Security Engineer, Compliance Auditor, Policy Engine Operator, and Fraud System (non-human actor).
2. WHEN a use case is presented, THE Use_Cases_Document SHALL identify the primary persona, the goal the persona is trying to achieve, and the GovOps artifacts or toolchain commands the persona interacts with.
3. THE Use_Cases_Document SHALL include a persona summary table that maps each persona to the use cases in which they appear.
4. WHEN a use case involves multiple personas, THE Use_Cases_Document SHALL identify the primary actor and any secondary actors, and describe how they collaborate.

---

### Requirement 2: Capability Catalog Authoring Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how to author and validate a `GovOps-AC` capability catalog, so that I can understand the end-to-end authoring workflow from lexicon definition through `govops lint`.

#### Acceptance Criteria

1. WHEN the authoring use case is presented, THE Use_Cases_Document SHALL show how action and resource terms are defined in the `#Lexicon` before being referenced in `#EnterpriseCapability` entries.
2. WHEN the authoring use case is presented, THE Use_Cases_Document SHALL show at least one capability entry using the convention-only profile (front-matter in `description`) and at least one using the `#EnterpriseCapability` schema extension.
3. WHEN `govops lint` is run against the catalog, THE Use_Cases_Document SHALL describe what the tool checks (lexicon resolution, group membership, engine-binding well-formedness) and what a passing vs. failing output looks like.
4. IF a capability entry references an action or resource term not defined in the `#Lexicon`, THEN THE Use_Cases_Document SHALL show how `govops lint` surfaces the violation and how the author resolves it.
5. THE Use_Cases_Document SHALL show how `applicability-groups` (sensitivity, risk-tier, lifecycle stage) are assigned to capabilities and how those assignments drive downstream use (IGA scoping, compliance queries).

---

### Requirement 3: Compliance Mapping and Audit Use Case

**User Story:** As a Compliance Auditor, I want a use case showing how enterprise capabilities are mapped to compliance frameworks, so that I can understand how to produce audit evidence from the GovOps repository.

#### Acceptance Criteria

1. WHEN the compliance mapping use case is presented, THE Use_Cases_Document SHALL show how a `#MappingDocument` links capabilities in `GovOps-AC` to entries in at least two external frameworks (e.g., NIST 800-53 AC-3 and ISO 27001 A.5.15).
2. THE Use_Cases_Document SHALL show how an auditor queries the mapping to answer: "Which capabilities at risk-tier >= high lack a mapping to AC-3?"
3. WHEN an auditor requests evidence for a specific compliance control, THE Use_Cases_Document SHALL show how the auditor traces from the compliance control id through the `#MappingDocument` to the capability entry in `GovOps-AC`.
4. THE Use_Cases_Document SHALL show what fields in the capability entry the auditor relies on (risk-tier, data-sensitivity, documented-context-expectations, group).

---

### Requirement 4: IGA Integration Use Case

**User Story:** As an IGA Team member, I want a use case showing how the enterprise capability catalog feeds an IGA platform's entitlement catalog, so that I can understand how access reviews target authoritative capability definitions instead of opaque permission strings.

#### Acceptance Criteria

1. WHEN the IGA integration use case is presented, THE Use_Cases_Document SHALL show how the IGA Exporter translates `GovOps-AC` entries into a format suitable for IGA platform ingest (CSV, SCIM, or OSCAL profile).
2. THE Use_Cases_Document SHALL show how an IGA access review is scoped to capabilities at a specific risk-tier or sensitivity level, using the `applicability-groups` defined in the catalog.
3. WHEN an IGA recertification campaign is run, THE Use_Cases_Document SHALL show how the campaign references canonical capability ids from `GovOps-AC` rather than engine-specific permission strings.
4. IF a capability is deprecated or renamed in `GovOps-AC`, THEN THE Use_Cases_Document SHALL show how the IGA Team is notified and how the IGA catalog is updated to reflect the change.

---

### Requirement 5: Policy Drift Detection Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how `govops drift` detects divergence between the capability catalog and deployed policy artifacts, so that I can maintain alignment between governance intent and enforcement reality.

#### Acceptance Criteria

1. WHEN the drift detection use case is presented, THE Use_Cases_Document SHALL show how `govops drift` compares `GovOps-AC` entries against deployed policy artifacts for at least two policy engines.
2. THE Use_Cases_Document SHALL show three drift scenarios: (a) a capability in the catalog with no corresponding policy rule, (b) a policy rule with no corresponding catalog entry, and (c) a capability whose `documented-context-expectations` diverge from what the deployed policy actually enforces.
3. WHEN drift is detected, THE Use_Cases_Document SHALL show how the engineer decides whether to update the catalog, update the policy, or document the divergence as an accepted exception.
4. THE Use_Cases_Document SHALL show how drift detection integrates into a CI/CD pipeline so that policy deploys that introduce drift are flagged before reaching production.
5. IF a drift report is generated in a multi-engine environment (e.g., Cedar for one service, OPA for another), THEN THE Use_Cases_Document SHALL show how per-engine sub-reports compose into a unified drift view.

---

### Requirement 6: Continuous Governance in CI/CD Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how the GovOps toolchain integrates into a CI/CD pipeline, so that I can understand how catalog and policy alignment is enforced automatically rather than point-in-time.

#### Acceptance Criteria

1. WHEN the CI/CD use case is presented, THE Use_Cases_Document SHALL show the sequence of toolchain commands (`govops lint`, `govops drift`) and the order in which they run in a pipeline.
2. THE Use_Cases_Document SHALL show what pipeline gate conditions cause a build to fail (e.g., lint errors, drift detected).
3. WHEN a policy change is merged, THE Use_Cases_Document SHALL show how the pipeline re-checks catalog alignment automatically.
4. THE Use_Cases_Document SHALL show how the catalog and policy artifacts are versioned together so that the governance state at any point in time is reproducible.

---

### Requirement 7: Correctness Properties of the Use-Cases Document

**User Story:** As a Gemara maintainer or GovOps WG reviewer, I want the use-cases document to demonstrate verifiable correctness properties, so that the scenarios are internally consistent and traceable to the design specification.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL ensure that every capability id referenced in a use case follows the `<service>:<action>:<resource>` format defined in design §6.2(A).
2. THE Use_Cases_Document SHALL ensure that every PARC Context predicate shown in a use case is consistent with the capability's `documented-context-expectations` field in the catalog excerpt shown in the same use case.
3. WHEN a use case shows a `govops lint` or `govops drift` output, THE Use_Cases_Document SHALL ensure the output is consistent with the catalog excerpt shown in the same use case (no phantom capability ids, no missing group references).
4. THE Use_Cases_Document SHALL include a cross-reference table mapping each use case to the design document sections, Gemara artifact types, and toolchain commands it exercises.

---

### Requirement 9: Non-Human Actor (Fraud System) Use Case

**User Story:** As a platform owner, I want a use case showing how an automated **Fraud System** exercises a catalogued capability as a non-human **software-agent** principal, so that readers understand how `GovOps-AC` governs service-to-service authorization rather than only human users.

#### Acceptance Criteria

1. WHEN the Fraud System use case is presented, THE Use_Cases_Document SHALL show **`fraud:flag:transaction`** as the capability id with `documented-requester-classes` including `software-agent`.
2. THE Use_Cases_Document SHALL show a sample **PARC-shaped authorization request** where **Principal** is a non-human actor (e.g., `type: software-agent`, service id `svc-acme-fraud-detection`) and **Action** / **Resource** align with the catalog entry.
3. THE Use_Cases_Document SHALL show **Context** attributes (e.g., fraud risk score) consistent with the capability's `documented-context-expectations` in the same section.
4. THE Use_Cases_Document SHALL contrast automated Fraud System invocation with optional human analyst invocation of the **same** capability id.
5. THE Use_Cases_Document SHALL show how `govops drift` (or equivalent) confirms deployed fraud policy aligns with catalog expectations for `fraud:flag:transaction`.

---

### Requirement 8: Document Structure and Navigation

**User Story:** As a reader of the use-cases document, I want the document to be structured so that I can navigate to a specific persona's use cases without reading the entire document.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL open with an executive summary (≤ 300 words) that describes the document's purpose, the personas covered, and how the use cases relate to the design document.
2. THE Use_Cases_Document SHALL include a table of contents with links to each use case section.
3. WHEN a use case references a design document section or Gemara artifact type, THE Use_Cases_Document SHALL include an inline cross-reference (section number or anchor link).
4. THE Use_Cases_Document SHALL include a summary table at the end mapping each use case to: primary persona, toolchain commands used, and design document sections referenced.
5. THE Use_Cases_Document SHALL use consistent section structure for each use case: Context, Actors, Preconditions, Step-by-Step Workflow, Artifacts Produced or Modified, Correctness Properties Demonstrated, and Cross-References.

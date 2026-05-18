# Requirements Document

## Introduction

This document specifies requirements for a **use-cases document** that illustrates how different enterprise personas interact with the GovOps enterprise capability catalog, TIGER pillar controls, and associated toolchain in real-world authorization governance workflows.

Where the design document specifies *what* the system is, the use-cases document shows *how* it is used — through concrete, narrative scenarios grounded in the Gemara artifact model, the PARC authorization request shape, and the TIGER metric.

Each use case must be self-contained enough to be read independently, yet cross-referenced to the design document's sections, artifact types, and TIGER pillar controls so that a reader can trace from scenario to specification.

---

## Glossary

- **Capability**: An **(action, resource)** pair that is the unit of authorization governance in the catalog. Not a principal, not a PARC tuple.
- **PARC**: Principal, Action, Resource, Context — e.g. an OpenID AuthZEN authorization request envelope used at the PDP boundary.
- **TIGER**: The five governance pillars — Transparency, Identity, Governance, Events, Resilience — and the five independent ordinal scores (one per pillar, each 1–5) that together characterize the maturity of an organization's authorization governance posture. TIGER does **not** produce a single aggregate score.
- **TIGER Pillar Score**: An ordinal score from 1 to 5 assigned to a single TIGER pillar, reflecting the maturity level of governance for that pillar. The five pillar scores are reported independently and are not combined into a weighted average.
- **Policy Coverage**: A separate metric — the weighted fraction of declared `#AssessmentRequirement`s that are currently satisfied by deployed policy and runtime evidence. Policy Coverage is computed by `govops coverage` and is distinct from the TIGER pillar scores.
- **GovOps-AC**: The `#CapabilityCatalog` artifact that inventories the enterprise's authorization surface as **(action, resource)** capabilities.
- **TIGER Pillar Catalog**: A `#ControlCatalog` artifact scoped to one TIGER pillar (e.g., `GovOps-Transparency.yaml`, `GovOps-Governance.yaml`). The contents of `GovOps-Identity.yaml` are defined by the GovOps Initiative Orbit WG and are not prescribed by this document.
- **AssessmentRequirement**: A Gemara artifact that expresses a requirement over a capability — either a configuration assertion or a provable claim.
- **Provable Claim**: An `#AssessmentRequirement` whose satisfaction can be verified by symbolic analysis (SAT/SMT), yielding an UNSAT certificate or a counterexample.
- **GRC Practitioner**: A Governance, Risk, and Compliance professional responsible for defining and auditing authorization controls.
- **IGA Team**: An Identity Governance and Administration team responsible for access reviews, entitlement lifecycle, and provisioning.
- **Platform Security Engineer**: An engineer responsible for deploying, configuring, and maintaining policy decision points and the GovOps repository.
- **Compliance Auditor**: An internal or external auditor assessing conformance to frameworks such as NIST 800-53, ISO 27001, or SOC 2.
- **Policy Engine Operator**: An engineer responsible for a specific PDP (Cedar, OPA/Rego, OpenFGA, AuthZEN-conformant, etc.) and its policy lifecycle.
- **GovOps Repository**: The `govops/` directory of Gemara artifacts, deployed policy material, proofs, evidence, and TIGER score artifacts.
- **govops lint / coverage / drift / prove / tiger**: The GovOps toolchain commands described in design §12.
- **IGA Exporter**: The toolchain component that translates the capability catalog into a format suitable for IGA platform ingest.
- **Lexicon**: The `#Lexicon` artifact that defines canonical action verbs and resource type names used across the catalog.

---

## Requirements

### Requirement 1: Persona Coverage

**User Story:** As a reader of the use-cases document, I want each key enterprise persona to be represented by at least one concrete use case, so that I can understand how the catalog serves different organizational roles.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL include at least one use case for each of the following personas: GRC Practitioner, IGA Team, Platform Security Engineer, Compliance Auditor, and Policy Engine Operator.
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
5. THE Use_Cases_Document SHALL show how `applicability-groups` (sensitivity, risk-tier, lifecycle stage) are assigned to capabilities and how those assignments drive downstream metrics.

---

### Requirement 3: TIGER Control Authoring and Policy Coverage Use Case

**User Story:** As a GRC Practitioner, I want a use case showing how to author TIGER pillar control catalogs and measure Policy Coverage, so that I can understand how requirements are expressed and tracked against the capability catalog.

#### Acceptance Criteria

1. WHEN the TIGER control authoring use case is presented, THE Use_Cases_Document SHALL show how a `#ControlCatalog` entry references capabilities in `GovOps-AC` via `#MultiEntryMapping`.
2. THE Use_Cases_Document SHALL show at least one configuration-assertion `#AssessmentRequirement` and at least one provable-claim `#AssessmentRequirement` for the same capability, illustrating the difference between the two flavors.
3. WHEN `govops coverage` is run, THE Use_Cases_Document SHALL describe what the output reports the **Policy Coverage** metric: which capabilities have no governing control, which controls have no proof, and which risk-tier buckets are under-governed. The use case SHALL make clear that Policy Coverage is a separate metric from the TIGER pillar scores.
4. IF a capability with `risk-tier: critical` has no `#AssessmentRequirement` in any TIGER pillar catalog, THEN THE Use_Cases_Document SHALL show how `govops coverage` flags the gap and what remediation looks like.
5. THE Use_Cases_Document SHALL show how the GRC Practitioner uses Policy Coverage output to prioritize which capabilities to govern next, and how improving coverage may — but does not automatically — improve TIGER pillar scores.
6. THE Use_Cases_Document SHALL NOT prescribe the contents of `GovOps-Identity.yaml`; any examples involving the Identity pillar SHALL note that its control catalog is defined by the GovOps Initiative Orbit WG.

---

### Requirement 4: Provable Claims and Proof Workflow Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how to generate and record a symbolic proof for a provable-claim `#AssessmentRequirement`, so that I can understand the end-to-end proof workflow from claim authoring through `govops prove` to `#EvaluationLog` update.

#### Acceptance Criteria

1. WHEN the proof workflow use case is presented, THE Use_Cases_Document SHALL show a complete provable claim expressed in engine-neutral terms (referencing a capability id, a PARC Context predicate, and the deployed policy).
2. THE Use_Cases_Document SHALL show how `govops prove` invokes an engine-specific analyzer, captures the result (UNSAT certificate or counterexample), and writes the proof artifact to `proofs/`.
3. WHEN a proof is produced, THE Use_Cases_Document SHALL show how the `#EvaluationLog` is updated to reference the proof artifact via `#ArtifactMapping`, including the deployed policy hash.
4. IF the symbolic analyzer returns a satisfying assignment (counterexample), THEN THE Use_Cases_Document SHALL show how the counterexample is surfaced to the engineer and what remediation steps are expected.
5. THE Use_Cases_Document SHALL show how the proof workflow differs across at least one PDP classes (e.g., a policy-language PDP such as Cedar or a graph PDP such as OpenFGA).
6. THE Use_Cases_Document SHALL show how a stale proof (policy hash mismatch) is detected and how the engineer re-runs `govops prove` to refresh it.

---

### Requirement 5: TIGER Pillar Score Computation and Reporting Use Case

**User Story:** As a GRC Practitioner, I want a use case showing how the five TIGER pillar scores are computed and reported, so that I can understand what each score measures, how scores change over time, and how to interpret gaps per pillar.

#### Acceptance Criteria

1. WHEN the TIGER score use case is presented, THE Use_Cases_Document SHALL show that `govops tiger` produces **five independent ordinal scores** — one for each pillar (Transparency, Identity, Governance, Events, Resilience) — each on a scale of 1 to 5. There is no single aggregate TIGER score.
2. THE Use_Cases_Document SHALL show a sample `tiger-score.yaml` output containing the five per-pillar scores and explain what each score level (1–5) signifies for a given pillar in terms of governance maturity.
3. THE Use_Cases_Document SHALL show a sample `tiger-evidence.json` that records the per-pillar inputs used to derive each score, and explain how the `notes` field identifies specific gaps in catalog terms.
4. WHEN a new capability is added to `GovOps-AC` without a corresponding TIGER control, THE Use_Cases_Document SHALL show how the affected pillar score may decrease and how the gap is identified in `tiger-evidence.json`.
5. WHEN a policy re-deploy invalidates an existing UNSAT proof, THE Use_Cases_Document SHALL show how the affected pillar score may decrease until `govops prove` is re-run and the proof is refreshed.
6. THE Use_Cases_Document SHALL show how the TIGER pillar scores are used in a governance review meeting — what questions each score answers and what TIGER does not measure (adversarial robustness, catalog quality, IGA lifecycle). The use case SHALL also show how Policy Coverage is presented alongside TIGER scores to give a complete governance picture.
7. THE Use_Cases_Document SHALL NOT prescribe the scoring criteria or control contents for the Identity pillar; any Identity pillar score shown in an example SHALL be treated as illustrative and SHALL note that the scoring criteria are defined by the GovOps Initiative Orbit WG.

---

### Requirement 6: Compliance Mapping and Audit Use Case

**User Story:** As a Compliance Auditor, I want a use case showing how enterprise capabilities are mapped to compliance frameworks, so that I can understand how to produce audit evidence from the GovOps repository.

#### Acceptance Criteria

1. WHEN the compliance mapping use case is presented, THE Use_Cases_Document SHALL show how a `#MappingDocument` links TIGER controls to entries in at least two external frameworks (e.g., NIST 800-53 AC-3 and ISO 27001 A.5.15).
2. THE Use_Cases_Document SHALL show how an auditor queries the mapping to answer: "Which capabilities at risk-tier >= high lack a TIGER control mapped to AC-3?"
3. WHEN an auditor requests evidence for a specific compliance control, THE Use_Cases_Document SHALL show how the auditor traces from the compliance control id through the `#MappingDocument` to the TIGER `#AssessmentRequirement` to the `#EvaluationLog` and proof artifact.
4. THE Use_Cases_Document SHALL show how the `tiger-evidence.json` file serves as the auditor-facing trail, including what fields the auditor relies on.
5. IF a TIGER requirement is satisfied by runtime evidence (observable claim) rather than a symbolic proof, THEN THE Use_Cases_Document SHALL show how the auditor distinguishes the two evidence types and what each implies about assurance level.

---

### Requirement 7: IGA Integration Use Case

**User Story:** As an IGA Team member, I want a use case showing how the enterprise capability catalog feeds an IGA platform's entitlement catalog, so that I can understand how access reviews target authoritative capability definitions instead of opaque permission strings.

#### Acceptance Criteria

1. WHEN the IGA integration use case is presented, THE Use_Cases_Document SHALL show how the IGA Exporter translates `GovOps-AC` entries into a format suitable for IGA platform ingest (CSV, SCIM, or OSCAL profile).
2. THE Use_Cases_Document SHALL show how an IGA access review is scoped to capabilities at a specific risk-tier or sensitivity level, using the `applicability-groups` defined in the catalog.
3. WHEN an IGA recertification campaign is run, THE Use_Cases_Document SHALL show how the campaign references canonical capability ids from `GovOps-AC` rather than engine-specific permission strings.
4. THE Use_Cases_Document SHALL show how the IGA Team uses `govops drift` to detect capabilities present in deployed policy but absent from the catalog, and how that gap is resolved.
5. IF a capability is deprecated or renamed in `GovOps-AC`, THEN THE Use_Cases_Document SHALL show how the IGA Team is notified and how the IGA catalog is updated to reflect the change.

---

### Requirement 8: Policy Drift Detection Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how `govops drift` detects divergence between the capability catalog and deployed policy artifacts, so that I can maintain alignment between governance intent and enforcement reality.

#### Acceptance Criteria

1. WHEN the drift detection use case is presented, THE Use_Cases_Document SHALL show how `govops drift` compares `GovOps-AC` entries against deployed policy artifacts for at least two policy engines.
2. THE Use_Cases_Document SHALL show three drift scenarios: (a) a capability in the catalog with no corresponding policy rule, (b) a policy rule with no corresponding catalog entry, and (c) a capability whose `documented-context-expectations` diverge from what the deployed policy actually enforces.
3. WHEN drift is detected, THE Use_Cases_Document SHALL show how the engineer decides whether to update the catalog, update the policy, or document the divergence as an accepted exception.
4. THE Use_Cases_Document SHALL show how drift detection integrates into a CI/CD pipeline so that policy deploys that introduce drift are flagged before reaching production.
5. IF a drift report is generated in a multi-engine environment (e.g., Cedar for one service, OPA for another), THEN THE Use_Cases_Document SHALL show how per-engine sub-reports compose into a unified drift view.

---

### Requirement 9: Continuous TIGER in CI/CD Use Case

**User Story:** As a Platform Security Engineer, I want a use case showing how the GovOps toolchain integrates into a CI/CD pipeline for continuous TIGER score maintenance, so that I can understand how governance assurance is automated rather than point-in-time.

#### Acceptance Criteria

1. WHEN the CI/CD use case is presented, THE Use_Cases_Document SHALL show the sequence of toolchain commands (`govops lint`, `govops coverage`, `govops drift`, `govops prove`, `govops tiger`) and the order in which they run in a pipeline.
2. THE Use_Cases_Document SHALL show what pipeline gate conditions cause a build to fail (e.g., lint errors, Policy Coverage below a configured threshold, drift detected, any TIGER pillar score falling below a per-pillar configured minimum).
3. WHEN a policy change is merged, THE Use_Cases_Document SHALL show how the pipeline re-evaluates affected provable claims and updates the TIGER score automatically.
4. THE Use_Cases_Document SHALL show how proof artifacts and evaluation logs are versioned and stored alongside policy artifacts so that the governance state at any point in time is reproducible.
5. IF the symbolic analyzer is unavailable during a pipeline run, THEN THE Use_Cases_Document SHALL show how the pipeline handles the failure (fail-closed vs. degrade gracefully) and how the affected TIGER pillar score reflects the missing proof.

---

### Requirement 10: Correctness Properties of the Use-Cases Document

**User Story:** As a Gemara maintainer or GovOps WG reviewer, I want the use-cases document to demonstrate verifiable correctness properties, so that the scenarios are internally consistent and traceable to the design specification.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL ensure that every capability id referenced in a use case follows the `<service>:<action>:<resource>` format defined in design §6.2(A).
2. THE Use_Cases_Document SHALL ensure that every `#AssessmentRequirement` id referenced in a use case follows the `<PILLAR>-<NN>.<NN>` format used in the TIGER pillar catalogs (e.g., `GOVOPS-INT-01.02`).
3. FOR ALL use cases that include a TIGER score example, THE Use_Cases_Document SHALL ensure that each per-pillar score is an integer in the range [1, 5]. There is no aggregate score to validate; the use case SHALL NOT show a single combined TIGER number.
4. THE Use_Cases_Document SHALL ensure that every PARC Context predicate shown in a use case is consistent with the capability's `documented-context-expectations` field in the catalog excerpt shown in the same use case.
5. FOR ALL use cases that show a proof workflow, THE Use_Cases_Document SHALL ensure that the proof result (UNSAT or SAT with counterexample) is logically consistent with the claim being checked — an UNSAT result means the negation of the claim is unsatisfiable, and a SAT result means a counterexample exists.
6. THE Use_Cases_Document SHALL include a cross-reference table mapping each use case to the design document sections, Gemara artifact types, TIGER pillar controls, and toolchain commands it exercises.
7. WHEN a use case shows a `govops lint` or `govops coverage` output, THE Use_Cases_Document SHALL ensure the output is consistent with the catalog excerpt shown in the same use case (no phantom capability ids, no missing group references).

---

### Requirement 11: Document Structure and Navigation

**User Story:** As a reader of the use-cases document, I want the document to be structured so that I can navigate to a specific persona's use cases or a specific TIGER pillar's scenarios without reading the entire document.

#### Acceptance Criteria

1. THE Use_Cases_Document SHALL open with an executive summary (≤ 500 words) that describes the document's purpose, the personas covered, and how the use cases relate to the design document.
2. THE Use_Cases_Document SHALL include a table of contents with links to each use case section.
3. WHEN a use case references a design document section, Gemara artifact type, or TIGER pillar control, THE Use_Cases_Document SHALL include an inline cross-reference (section number or anchor link).
4. THE Use_Cases_Document SHALL include a summary table at the end mapping each use case to: primary persona, TIGER pillars exercised, Policy Coverage relevance, toolchain commands used, and design document sections referenced.
5. THE Use_Cases_Document SHALL use consistent section structure for each use case: Context, Actors, Preconditions, Step-by-Step Workflow, Artifacts Produced or Modified, Correctness Properties Demonstrated, and Cross-References.

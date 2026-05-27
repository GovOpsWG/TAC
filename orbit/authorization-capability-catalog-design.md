# Authorization Capability Catalog: Design

**Status:** Draft for discussion

**Companion design:** [Authorization Capability Catalog: Use Cases](./authorization-capability-catalog-use-cases.md)

---

## 1. Abstract

This document proposes a design for representing **authorization capabilities** — discrete **(action, resource)** pairs the organization recognizes as units of authorization risk and governance — as **first-class catalog entries in the [Gemara](https://gemara.openssf.org) GRC engineering model**, and for organizing a complete **GovOps repository** of Gemara artifacts around them.

Three ideas combine to form the design:

1. **Capabilities are action–resource pairs**;  At evaluation time, a PDP receives a **PARC-shaped request**. Engine neutrality comes from the fact that every PDP class can consume PARC-shaped requests while the catalog stays a domain-centric inventory of discrete **(action, resource)** entries.
2. **A GovOps repository centered on `GovOps-ACCC`.** A single `#CapabilityCatalog` inventories the enterprise's authorization surface; a `#Lexicon` defines canonical terms; and **`#MappingDocument` files link capabilities to an OSCAL-aligned control layer first** (abstract control objectives or a `#ControlCatalog`), not directly to NIST, ISO, or SOC 2 control numbers. Governance metadata (risk-tier, sensitivity, documented context expectations) lives **on the capability entries**, not in a parallel pillar or scoring framework. 
3. **Policy binaries are not stored in this repository** — they are authored and published separately as versioned artifacts and distributed using software supply-chain trust models (signing, provenance, registries).
4. **Catalog–policy alignment.** `govops lint` validates the catalog; `govops drift` compares the catalog to **published policy releases** (supplied at run time by URI, digest, or local path) so the authorization surface stays enumerable and aligned with enforcement.

Gemara already provides 90% of the substrate: a stable `#CapabilityCatalog` (ADR-0019), `#ControlCatalog` and `#AssessmentRequirement`, mapping primitives that include `Capability` as an `EntryType`, `#Lexicon`, `#EvaluationLog`, `#EnforcementLog`, and `#AuditLog`. The remaining 10% is a small, additive **Authorization Capability Profile** of `#Capability` and a set of conventions for arranging the artifacts.

---

## 2. Background and motivation

### 2.1 What enterprises need to govern

GovOps requires a shared, declarative inventory of **(action, resource) capabilities** — that is:

1. **Finitely enumerable** so it can be reviewed, measured, and analyzed.
2. **Engine neutral** compatible with any AuthZEN-conformant PDP (e.g. OPA, Cedarling, OpenFGA).
3. **Composable** with threats, controls, risks, and policies so a capability becomes a traceable unit of governance linking exposed operations, associated risks, and measurable enforcement outcomes.
4. **GRC-interoperable** via a Trestle/OSCAL compliance layer: Gemara owns capability semantics; [OSCAL Compass compliance-trestle](https://github.com/oscal-compass/compliance-trestle) owns normalization, validation, and projection to specific frameworks (NIST SP 800-53, ISO 27001, SOC 2, FedRAMP, PCI, and internal accreditations). 

### 2.2 Why Gemara

Gemara already supplies:

- A layered model (Vectors → Threats → Controls → Policies → Evaluation/Enforcement/Audit logs) that maps cleanly onto authorization governance.
- A first-class **`#CapabilityCatalog`** artifact (ADR-0019) intended specifically so that capabilities can be referenced uniformly by `id` from threats, controls, and other catalogs, instead of duplicated inline.
- Mapping primitives (`#ArtifactMapping`, `#EntryMapping`, `#MultiEntryMapping`, `#MappingDocument`) that already include `Capability` in `#EntryType`, so capabilities can be the source or target of mappings to/from controls and external frameworks.
- A **`#Lexicon`** artifact for controlled vocabulary, useful for action verbs and resource type names.
- A `#Catalog` base that supports `extends` and `imports` for vendor-specific or industry overlays.

In short, Gemara already has the right *shape*; what is missing is a profile that constrains `#Capability` to carry a well-typed **(action, resource)** capability identity, plus conventions for organizing controls and evidence around that inventory — without conflating the catalog with the PARC request envelope.

---

## 3. Definitions

| Term | Meaning |
|---|---|
| **Capability** | An **(action, resource)** pair: a discrete, nameable operation on a resource type. *What* can be done on *what* is catalogued here, independent of *who* is asking in any given request. A capability is **not** a principal and **not** a context tuple. |
| **Principal** | The actor supplied in a **PARC authorization request** when the PDP is asked whether an operation is permitted. Policy may use principal attributes; the principal does not define the capability. |
| **Action** | The verb half of a capability (e.g., `read`, `create`, `delete`, `transfer`, `assume`). |
| **Resource** | The noun half of a capability: the resource **type** the action applies to (e.g., `Invoice`, `BankAccount`, `Repository`, `PolicyDocument`). The catalog records the *type*; runtime requests identify specific instances. |
| **Context** | Other facts provided to a **PARC request** |
| **PARC** | Principal, Action, Resource, Context — the  **authorization request** shape. Used at the PDP boundary; **not** synonymous with "capability." |
| **Capability id** | Opaque lowercase hexadecimal SHA-256 digest of `<group-slug>|<action-slug>|<resource-slug>` (see §6.2). Does not embed group, action, or resource strings in the catalog id field. |
| **Authorization capability** | A Gemara catalog entry whose **identity** is an **(action, resource)** pair (plus `id`, `title`, `description`, `group` per `#Capability`). Optional fields may document risk, sensitivity, or **expected** policy preconditions; those extensions do not redefine the capability. |
| **Documented context expectation** | Optional metadata on a capability describing **PARC Context** the organization expects policy to enforce (e.g., MFA, approval counts). Used by `govops drift` and compliance mapping; not part of capability identity. |
| **OSCAL-aligned control** | An abstract control objective in a `#ControlCatalog` (`GovOps-ACO`) suitable for import into a Trestle workspace. Primary mapping target for capabilities; framework-specific control numbers are derived downstream. |

---

## 4. Design goals and non-goals

### Goals

1. **Reuse, do not replace.** Build on Gemara's existing `#CapabilityCatalog`, `#ControlCatalog`, and mapping primitives. No fork.
2. **Additive schema only.** Any extension must be expressible as a CUE refinement that still validates as a `#Capability`.
3. **Engine-neutral capabilities and PARC-shaped requests.** Capability rows in the catalog are **(action, resource)** only. Interoperability with PDPs uses **PARC** as the standard **request** envelope at evaluation time. The design MUST NOT privilege any specific policy engine, policy store strategy, or authorization API.
4. **Compliance-interoperable.** Each capability maps to OSCAL-aligned control objectives in Gemara first; framework-specific controls are **downstream renderings** produced by Trestle/OSCAL (catalog import, profiles, mapping collections, SSPs, assessment artifacts)—not hard-coded into the capability model.
5. **Reviewable.** The catalog is the canonical input to IGA access reviews, compliance queries, and drift detection.

### Some Non-goals

- Defining a new policy language, evaluation API, or policy store specification.
- Publshing policy artifacts 
- Modeling individual permission grants to specific principals
- Capturing resource instance hierarchies or organizational topology
- Replacing IGA entitlement catalogs. The two can coexist; the authorization capability catalog can feed an IGA catalog or be derived from one.

---

## 5. Repository layout

A GovOps repository is a directory of **Gemara artifacts** that describe the enterprise authorization surface and its compliance mappings. Policy enforcement rules live **outside** this tree as versioned binaries (see below). A representative tree:

```text
govops/
  lexicon.yaml                      # Lexicon
  metadata.yaml                     # Shared metadata fragments (optional)
  GovOps-ACCC.yaml                   # #CapabilityCatalog (Authorization Capability Profile)
  GovOps-ACO.yaml                    # optional: #ControlCatalog (OSCAL-aligned abstract controls)
  mappings/                         # primary: Capability → abstract controls (#MappingDocument)
  exports/oscal/                    # optional: Trestle workspace input / govops OSCAL emit
  exports/                          # e.g. IGA exporter output (optional)
```

Mapping each top-level item to a Gemara artifact type:

| Path | Gemara artifact | Purpose |
|---|---|---|
| `lexicon.yaml` | `#Lexicon` | Canonical action verbs and resource type names. |
| `metadata.yaml` | shared `#Metadata` includes | Author, version, lexicon reference, applicability groups. |
| `GovOps-ACCC.yaml` | `#CapabilityCatalog` of `#AuthorizationCapability` | Enterprise authorization surface. |
| `GovOps-ACO.yaml` | `#ControlCatalog` (optional) | OSCAL-aligned abstract control objectives (enterprise or GovOps reference). |
| `mappings/` | `#MappingDocument` | **Primary:** capability → abstract control mappings. Framework projections are downstream (Trestle/OSCAL). |
| `exports/oscal/` | convention | Artifacts for Trestle import/transform (catalog fragments, mapping-collection inputs). |
| `exports/` | convention | IGA-ingestible views of the catalog (optional). |

Notes:

- Nothing in the layout is normative. Organizations MAY add other Gemara catalogs (`#ControlCatalog`, `#ThreatCatalog`, `#EvaluationLog`) using the standard Gemara model; those are **not** part of the ACC core layout.

---

## 6. The Authorization Capability Catalog (`GovOps-ACCC`)

### 6.1 Where it sits in Gemara

A `#CapabilityCatalog` is **not** layered. It is **domain content** that crosses Gemara's seven layers, just as ADR-0019 already treats it for system capabilities:

```text
Layer 1   Vectors, Guidance
Layer 2   Threats, Controls
Layer 3   Risks, Policies
Layer 4   Sensitive activities (e.g., grant access, deploy policy)
Layer 5   Evaluation logs
Layer 6   Enforcement logs
Layer 7   Audit logs

  ^   Capability Catalog (cross-cutting domain content)
      - referenced by Threats via #Threat.capabilities
      - referenced by Controls via #MappingDocument
      - referenced by Policies via imports
      - measured by Evaluation/Enforcement logs (coverage, drift, proof)
```

### 6.2 The Authorization Capability Profile

Two equivalent encodings are offered. Authors MAY pick (A) for a zero-schema-change start and migrate to (B) when the schema extension is ratified by Gemara.

#### (A) Convention-only profile (no schema change)

Use the existing `#Capability` fields and encode the structured payload in `id` and `description`:

- `id` MUST be the lowercase hexadecimal SHA-256 digest of the UTF-8 string `<group-slug>|<action-slug>|<resource-slug>` (e.g., `0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3`). Slugs are lowercase identifiers. **Group slug** is derived from the entry's `#Group` `id` by removing an optional `g.` prefix (`g.payments` → `payments`).
- `title` is the human label.
- `description` MUST contain a YAML front-matter block with **`group`**, **`action`**, and **`resource`** slugs (the preimage of `id`). The **(action, resource)** pair is the **capability identity** for PARC and policy; `group` scopes the id. Any other front-matter keys are optional **catalog documentation** and **do not** redefine the capability.

```yaml
- id: 0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3
  title: Transfer funds from bank account
  group: g.payments
  description: |
    ---
    group: payments
    action: transfer
    resource: bank-account
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
      expected-context:
        - id: c.mfa
          text: "Policy MUST require strong authentication evidence in context"
          expression: "context.entra_id_token.amr has value 'mfa'"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical
    ---
    Initiate a funds transfer between bank accounts in the payments domain.
```

This validates against today's stable `#CapabilityCatalog`. Tooling parses the front matter.

#### (B) Schema extension (recommended)

Introduce `#AuthorizationCapability` as an embedding of `#Capability`. The CUE definition (intended for upstream contribution to Gemara, or hosted as a sibling overlay module under the GovOps WG):

```cue
// Schema lifecycle: experimental
@status("experimental")
package gemara

// AuthorizationCapability is a Capability whose identity is the pair
// (action, resource). Optional fields document risk, sensitivity, or
// expected PDP / policy context — they are NOT part of the capability.
#AuthorizationCapability: {
    #Capability

    // id MUST be SHA-256 hex of <group-slug>|<action-slug>|<resource-slug> (§6.2).
    // action + resource together ARE the capability identity.
    action:   string
    resource: string

    // Optional: documented classes of requester often seen in PARC requests
    // for this capability (policy-engine subject kinds).
    // Does NOT define the capability; policy may allow a wider or narrower set.
    "documented-requester-classes"?: [string, ...string] @go(DocumentedRequesterClasses)

    // Optional: documented expectations on PARC Context for policy reviewers.
    // Does NOT define the capability; actual enforcement is in published policy releases.
    "documented-context-expectations"?: [#ContextPredicate, ...#ContextPredicate] @go(DocumentedContextExpectations)

    // data-sensitivity classifies the data exposed by the action.
    // Values SHOULD come from an applicability-group named "sensitivity".
    "data-sensitivity"?: string @go(DataSensitivity)

    // risk-tier is a coarse risk classification used by GovOps metrics.
    "risk-tier"?: "low" | "medium" | "high" | "critical" @go(RiskTier)

    // engine-bindings correlate this (action, resource) capability to
    // engine-specific symbols. Optional; does not change capability identity.
    "engine-bindings"?: [#EngineBinding, ...#EngineBinding] @go(EngineBindings)
}

// ContextPredicate documents an expectation on PARC Context for reviewers
// or linters. Enforcement remains in policy, not in the capability row.
#ContextPredicate: {
    id:   string
    text: string

    // expression is an optional machine-readable form of the predicate.
    // expression-language identifies the dialect.
    expression?:            string
    "expression-language"?: string
}

// EngineBinding correlates this capability with engine-specific symbols and,
// optionally, a published policy release. Policy bytes are NOT stored in the
// GovOps repository; policy-release identifies a versioned binary elsewhere.
#EngineBinding: {
    engine: string

    // action-id is the engine-specific identifier for the action.
    "action-id"?: string

    // resource-type is the engine-specific identifier for the resource type.
    "resource-type"?: string

    // policy-release points at a published policy binary (registry URI, semver,
    // content digest). Used for drift correlation and audit traceability.
    "policy-release"?: {
        uri?:     string
        version?: string
        digest?:  string
    }

    notes?: string
}
```

The **normative** fields that define what is being governed are **`action` and `resource`** only. Everything else is optional documentation, risk metadata, or engine correlation. That separation keeps the catalog a domain-centric **capability registry** and avoids collapsing "capability" into a full PARC tuple.

### 6.3 Catalog organization with `#Group`

`#Group` (post ADR-0020) is the single grouping primitive — a way to **group** related (action, resource) pairs in the catalog:

- `groups`: one entry per logical group (e.g., `payments`, `iam`, `governance`, `release-engineering`).
- Capability `id` values are opaque SHA-256 digests; the entry's `group` field references the Gemara group `id` (e.g., `g.payments`), and **group slug** for hashing is derived from that reference.
- Each `#AuthorizationCapability.group` references a group `id`.

Use `metadata.applicability-groups` for orthogonal classifications that drive metrics:

- **Sensitivity** — `public`, `internal`, `confidential`, `pii`, `regulated`.
- **Risk tier** — `low`, `medium`, `high`, `critical`.
- **Lifecycle stage** — `design-time`, `build-time`, `deploy-time`, `runtime`.
- **Invocation profile** — `interactive-subject`, `software-only`, `mixed` (optional catalog metadata describing how the capability is usually invoked; does not change the (action, resource) identity).

These groups become the dimensions for GovOps metrics: *"What high-impact capabilities exist? Which are weakly governed?"*.

### 6.4 Anchoring vocabulary with `#Lexicon`

Each `action` and `resource` SHOULD resolve to a term in the repository's `#Lexicon` so that synonyms collapse to a single canonical id:

```yaml
title: GovOps Action and Resource Lexicon
metadata:
  id: lex.govops.actions-resources
  type: Lexicon
  gemara-version: "0.x"
  description: Canonical verbs and resource type names for authorization capabilities.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
terms:
  - id: action.transfer
    title: transfer
    definition: Move value or ownership from a source resource to a target resource.
    synonyms: [send, move, remit]
  - id: resource.bank-account
    title: BankAccount
    definition: A demand-deposit account at a financial institution.
```

The `metadata.lexicon` field on the `#CapabilityCatalog` already points at this lexicon document. Tooling can lint the catalog by checking that all action/resource strings resolve to lexicon term ids.

### 6.5 Relationship to Gemara's existing `#Resource`

Gemara already defines `#Resource` (in `entities.cue`) as a runtime entity that is the *target of an evaluation* — the thing an `#EvaluationLog`, `#EnforcementLog`, or `#AuditLog` is *about*. That is **not** the same as the **resource type** half of an **(action, resource)** capability row in the catalog.

The two stay separate:

- `#AuthorizationCapability.resource` is the **resource slug** half of the capability (e.g., `bank-account`), aligned with the lexicon term used in the capability id preimage.
- `#Resource` (entities) is a **runtime instance** being evaluated (e.g., the production payments service running v1.4.2).

A measurement record carries both: the `target` of an evaluation log is a `#Resource`; what was *evaluated about it* is the conformance of its policies to a set of `#AuthorizationCapability` entries (each keyed by **action + resource**).

### 6.6 Threats and risks over capabilities

Gemara's existing `#Threat.capabilities` field already accepts `#MultiEntryMapping`, so threats can target an authorization capability without any additional schema work:

```yaml
threats:
  - id: t.transfer.fraud
    title: Unauthorized funds transfer
    description: An attacker with stale credentials initiates fraudulent transfers.
    group: g.financial
    capabilities:
      - reference-id: ec
        entries:
          - reference-id: 0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3
```

Layer-3 `#Risk` entries and `#Policy` documents reference threats and controls in turn, so a single line of an authorization capability flows through the entire seven-layer model.

---

## 7. Optional Gemara layering and context expectations

The **Authorization Capability Catalog** (`GovOps-ACCC`) is the Phase 1 core. Organizations MAY also maintain standard Gemara `#ControlCatalog`, `#ThreatCatalog`, `#Risk`, and `#Policy` artifacts that reference capability ids (see §6.7). That full layered model is valuable but **not required** for ACC conformance and is **not** split into a separate pillar or scoring silo.

### 7.1 Documented context expectations

High-risk capabilities SHOULD record **documented-context-expectations** on the catalog entry — human-readable (and optionally machine-readable) statements about **PARC Context** the organization expects published policy to enforce (e.g., `context.amr has_value 'urn:mfa'`, minimum approver count). These expectations are **not** part of capability identity; they document governance intent and feed **`govops drift`** Type-C checks (catalog expectation vs. a supplied policy release).

---

## 8. Worked example

A minimal **Acme Bank** payments scenario: catalog authoring, compliance mapping, and drift against a published Cedar policy release.

### 8.1 `GovOps-ACC.yaml` (excerpt)

```yaml
title: Acme Authorization Capability Catalog
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
    - id: risk-tier
      title: Risk Tier
      description: Coarse risk classification for scoping reviews and mappings.
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
  - id: 18420fcf746d2f060e1f24f96bcb820ceca600cad09e1d8a7c5a405db190d1a7
    title: Read invoice
    description: Retrieve an existing Invoice by id.
    group: g.payments
    action: read
    resource: invoice
    documentation:
      typical-requester-classes: [interactive-subject, software-agent, autonomous-actor]
    data-sensitivity: pii
    risk-tier: medium

  - id: 0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3
    title: Transfer funds
    description: Initiate a funds transfer between bank accounts.
    group: g.payments
    action: transfer
    resource: bank-account
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
      expected-context:
        - id: c.mfa
          text: Policy MUST require strong authentication evidence in PARC Context
          expression: "context.acr == 'urn:mfa'"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical

  - id: 3c6e0df2c70f2ecdea45d9e413db49b8c5e94921eb34eb7eb2a6de3ea46919cd
    title: Approve loan
    description: Approve a loan application or facility change.
    group: g.lending
    action: approve
    resource: loan
    documentation:
      typical-requester-classes: [interactive-subject]
      expected-context:
        - id: c.approvals
          text: Two distinct approvers required in PARC Context
          expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical

  - id: ec0e5e80cae8a6933dd1e0ad377b1c92f5c9fa8062d32e547ca52a0ea9ceffa2
    title: Flag transaction
    description: Flag a payment transaction for fraud review.
    group: g.fraud
    action: flag
    resource: transaction
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
    data-sensitivity: internal
    risk-tier: high
```

### 8.2 Primary mapping: capabilities to OSCAL-aligned controls (excerpt)

GovOps **does not** map capabilities directly to NIST control numbers in the ACC core. The first hop is always capability → abstract control objective (Gemara `#ControlCatalog` or equivalent), expressed in OSCAL-compatible terms for Trestle to consume.

```yaml
# mappings/acme-capabilities-to-controls.yaml
title: Acme Capabilities to OSCAL-aligned Controls
metadata:
  id: map.acme.acc.aco
  type: MappingDocument
  mapping-references:
    - id: acc
      title: Acme Authorization Capability Catalog
    - id: aco
      title: Acme OSCAL-aligned Control Objectives
source-reference:
  reference-id: acc
  entry-type: Capability
target-reference:
  reference-id: aco
  entry-type: Control
mappings:
  - id: m.transfer.access-enforcement
    source: 0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3
    relationship: implements
    targets:
      - entry-id: govops.ac-03.access-enforcement
        rationale: Critical transfer capability; MFA documented in context expectations.
      - entry-id: govops.ia-02.strong-authentication
        rationale: Strong authentication evidence expected in PARC Context for transfers.
```

A GovOps-native compliance query becomes: *"Which capabilities at `risk-tier >= high` lack a mapping to `govops.ac-03.access-enforcement`?"* — answered from `GovOps-ACCC` and this mapping document alone.

### 8.3 Framework projection via Trestle/OSCAL (illustrative)

**NIST SP 800-53 Rev. 5**, **ISO 27001**, **SOC 2**, and other accreditations are **downstream**. Trestle imports framework catalogs as OSCAL JSON/YAML, maintains mapping collections between catalogs, resolves profiles, and emits SSPs, assessment plans, and results. A capability traced to `govops.ac-03.access-enforcement` may project to NIST `AC-3` and `IA-2(1)` through an OSCAL mapping collection—without changing the capability id or GovOps semantics when NIST revises.

```text
GovOps-ACCC (capabilities)
    │  #MappingDocument (Gemara)
    ▼
GovOps-ACO / abstract control catalog
    │  Trestle: validate, split/merge, transform
    ▼
OSCAL catalog · profile · mapping-collection · SSP · assessment-results
    │  framework-specific views
    ▼
NIST 800-53 · ISO 27001 · SOC 2 · FedRAMP · PCI · internal controls
```

Example audit trace for a transfer capability:

```text
Capability 0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3
  → MappingDocument m.transfer.access-enforcement
  → govops.ac-03.access-enforcement (GovOps-ACO)
  → (Trestle) OSCAL mapping collection → NIST SP 800-53 AC-3, IA-2(1)
  → (optional) govops drift on published Cedar policy release (UC-04)
```

### 8.4 Drift check (illustrative)

Policy for the payments service is published separately (e.g., `oci://registry.acme.example/authz/payments-policy:2026.1.4` with digest `sha256:…`). Drift is run against that release, not against files in the GovOps tree:

```bash
govops drift \
  --catalog govops/GovOps-ACC.yaml \
  --policy oci://registry.acme.example/authz/payments-policy:2026.1.4 \
  --digest sha256:abc123… \
  --engine cedar
```

A Type-C finding on `0c451a4b7305a117ad7e4c874799c5982100823ef1b71db3490fffc35a63fef3` means the supplied policy release does not match the catalog's `context.acr == 'urn:mfa'` expectation — the gap is expressed in **capability id** terms, not a separate metrics framework.

---

## 9. Compliance architecture: GovOps, Trestle/OSCAL, and frameworks

GovOps and Gemara own the **ontology of authorization capabilities and governance intent**. Trestle/OSCAL own **compliance translation and interoperability**. Specific standards are **projections**, not source-of-truth fields on capability rows.

| Layer | Owner | Role | Artifacts |
|---|---|---|---|
| **Enterprise truth** | Gemara / GovOps | Finite authorization surface; governance metadata on capabilities | `#CapabilityCatalog` (`GovOps-ACCC`), `#Lexicon`, capability → control `#MappingDocument` |
| **Compliance interoperability** | OSCAL Compass / Trestle | Normalize, validate, transform, and compose OSCAL documents; bridge governance to operational compliance tooling | OSCAL catalog, profile, **mapping collection**, component-definition, SSP, assessment-plan, assessment-results, POA&M |
| **Framework renderings** | NIST, ISO, AICPA, regulators, enterprise GRC | Downstream targets that evolve on independent cadences | NIST SP 800-53, ISO/IEC 27001, SOC 2, FedRAMP, PCI, OSPS Baseline, internal control sets |

This matches how Trestle is used in practice: an ensemble for **creating, validating, and governing** OSCAL artifacts in `git`, with **transform tasks** from other formats into OSCAL—not a single-framework converter. Trestle v4 explicitly supports NIST's **Mapping Model** for relationships between catalogs.

### 9.1 Responsibilities

**Gemara/GovOps SHOULD:**

- Treat capabilities as canonical governance objects referenced by opaque ids.
- Map capabilities to **abstract, OSCAL-aligned control objectives** via `#MappingDocument` (and optionally a `#ControlCatalog` such as `GovOps-ACO.yaml`).
- Keep framework semantics **out of** the Authorization Capability Profile (no NIST `AC-3` literals on capability rows).
- Emit or exchange artifacts that Trestle can import (JSON/YAML OSCAL fragments, mapping inputs).

**Trestle/OSCAL SHOULD:**

- Import and validate framework catalogs (e.g., NIST SP 800-53 Rev. 5 via `trestle import`).
- Maintain **mapping collections** from the GovOps abstract catalog to framework catalogs.
- Resolve profiles, assemble SSPs, and produce assessment and posture artifacts for auditors and GRC tools.
- Absorb framework revisions by updating OSCAL-side mappings, not by rewriting GovOps capability semantics.

### 9.2 Why map abstract first

1. **One capability, many frameworks** — the same `0c451a4b…` transfer capability can simultaneously inform SOC 2, PCI, ISO, FedRAMP, and internal objectives through parallel OSCAL mapping collections.
2. **Independent evolution** — NIST, ISO, and SOC 2 releases change downstream catalogs; the enterprise capability inventory stays stable.
3. **Internal control objectives** — enterprises can define `govops.*` controls first, then project outward.
4. **Additive frameworks** — new accreditations are new mapping exercises and Trestle transforms, not Gemara schema redesigns.

### 9.3 Gemara `#MappingDocument` usage

Use Gemara's `#MappingDocument` with `source-reference` pointing at **`#CapabilityCatalog`** (`GovOps-ACCC`) and `entry-type: Capability`. The **primary target** is an OSCAL-aligned `#ControlCatalog` (enterprise `GovOps-ACO` or a GovOps reference catalog)—**not** NIST/ISO/SOC 2 control ids in the ACC core layout.

Secondary mappings (framework catalog ↔ abstract catalog, OSPS Baseline ↔ capabilities) MAY live in the GovOps repository for convenience or in a **Trestle workspace** under `catalogs/`, `profiles/`, and mapping-collection paths. Both are valid; the **normative GovOps boundary** remains capability → abstract control.

Direct capability → NIST-only `#MappingDocument` examples are **illustrative shortcuts** for small deployments; the reference architecture is two-hop.

### 9.4 OSPS Baseline and project scope

For OSS maintainers, OSPS Baseline mappings follow the same pattern: capabilities → abstract controls → OSPS via Trestle/OSCAL (or a maintained Gemara `#MappingDocument` at the OSPS layer). GovOps does not embed OSPS control text in capability entries.

---

## 10. Tooling implications

Phase 1 reference tooling (no Gemara schema changes beyond the optional Authorization Capability Profile):

1. **`govops lint`** — Capability `id` matches SHA-256 of `<group-slug>|<action-slug>|<resource-slug>`, lexicon resolution, group membership, applicability-group value checks, engine-binding well-formedness.
2. **`govops drift`** — Compare catalog **(action, resource)** entries to **published policy releases** per engine (artifacts passed in at run time by path, URI, or digest); report Type A (catalog without policy), Type B (policy without catalog), Type C (context-expectation mismatch).
3. **IGA exporter** — Emit CSV / SCIM from `GovOps-ACCC` for entitlement catalogs and access reviews.
4. **`govops oscal-export` (optional)** — Emit OSCAL-aligned catalog/mapping fragments for import into a Trestle workspace; framework projection is completed with `trestle` commands (import, merge, profile resolve, mapping collection).

Engine-specific drift plug-ins treat policy formats as opaque. A future phase MAY add `govops prove` for symbolic analysis of context expectations (§7.2).

---

## 11. Adoption / migration path

**Phase 0 — Convention-only profile (today).** Use stable `#CapabilityCatalog` with §6.2(A) front-matter; validate with existing Gemara tooling.

**Phase 1 — GovOps overlay and reference mappings.** Publish `#AuthorizationCapability` CUE overlay, reference `GovOps-ACCC` / `GovOps-ACO` templates, capability → abstract-control mappings, Trestle workspace examples for NIST/ISO/SOC 2 projection, `govops lint` and `govops drift`.

**Phase 2 — Upstream into Gemara.** Propose `#AuthorizationCapability` via Gemara ADR when stable.

**Phase 3 — Engine adapters and optional proofs.** Read-only catalog emitters for common PDPs; optional provable-claim workflow.

**Phase 4 — Continuous governance in CI/CD.** Lint and drift gates on every policy change (see use cases UC-05).

---

## 12. Open questions

1. **Granularity of `resource`.** Type vs. pattern in **PARC Context** — should the profile model graph-native hierarchies?
2. **Context predicate normalization.** CEL-like AST vs. natural-language strings on capabilities.
3. **Documented requester classes.** Hints only vs. lint-enforced alignment with policy extracts.
4. **Versioning of capabilities.** `replaced-by` pattern for splits and renames.
5. **Public reference catalogs.** Community process for Kubernetes, GitHub, cloud IAM surfaces.
6. **AuthZEN integration depth.** Guidance for feeding evaluate responses into optional evidence workflows.

---

## 13. Summary

Gemara already provides `#CapabilityCatalog`, `#MappingDocument`, `#Lexicon`, and optional layered controls. GovOps adds:

<!-- add content-->

The *thing governed* is the finite inventory of **(action, resource)** capabilities; *assurance* starts with catalog–policy alignment and **abstract control mappings**, with framework views produced through Trestle/OSCAL and stronger proofs optional later.

---

## 14. References

- Gemara — `capabilitycatalog.cue` (stable), `controlcatalog.cue` (stable), `mapping_inline.cue`, `mappingdocument.cue`, `lexicon.cue`, `entities.cue`, `threatcatalog.cue`, `evaluationlog.cue`.
- Gemara ADRs — 0017 (Base Catalog Type), 0018 (Promote Nested Concepts to Catalogs), 0019 (Promote Capabilities), 0020 (Groups), 0021 (Lexicon).
- OpenID AuthZEN — Authorization API specification using the PARC **request** shape.
- RFC 7519 — JSON Web Token (JWT), commonly used as signed evidence carried in **PARC Context**.
- [OSCAL Compass compliance-trestle](https://github.com/oscal-compass/compliance-trestle) — OSCAL validation, transformation, catalog/profile/mapping-collection workflows, CI/CD compliance pipelines.
- NIST [OSCAL](https://pages.nist.gov/OSCAL/) — standard interchange format; Mapping Model for cross-catalog relationships.
- NIST SP 800-53 r5, ISO/IEC 27001, SOC 2, OSPS Baseline — example **downstream** framework catalogs (imported and mapped via Trestle, not embedded in capability rows).

# Cataloging Enterprise Capabilities with Gemara

**Status:** Draft for discussion
**Audience:** OpenSSF GovOps WG (proposed), Gemara maintainers, GRC and IGA practitioners
**Related:**
- [GovOps WG Proposal](./openssf-wg-proposal.md)
- [Gemara Project](https://gemara.openssf.org) — `#CapabilityCatalog` (stable), ADR-0019 *Promote Capabilities to a First-class Catalog*
- OpenID AuthZEN — PARC (Principal, Action, Resource, Context) authorization **request** shape
- Signed tokens in **PARC Context** — A common integration pattern is that **JSON Web Tokens** ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)) and similar signed artifacts are passed as **evidence** inside the request **Context**; the PDP evaluates their claims when deciding whether an **(action, resource)** capability may be permitted for this invocation.

---

## 1. Abstract

This document proposes a design for representing **enterprise capabilities** — discrete **(action, resource)** pairs the organization recognizes as units of authorization risk and governance — as **first-class catalog entries in the [Gemara](https://gemara.openssf.org) GRC engineering model**, and for organizing a complete **GovOps repository** of Gemara artifacts around them.

In a **capability-based** model, applications and policy engines ask whether a **named operation on a named resource** can be granted; the **capability** is that **(action, resource)** pair, not the requester and not the ambient context. The **PARC Principal**, **Context** (often including **signed JWTs** or other attestations whose claims the PDP interprets), and environmental attributes are **inputs to policy** when a decision is requested — they determine *whether* a capability may be exercised *right now*, not *what* the capability is. OpenID AuthZEN's **PARC** envelope (Principal|Subject, Action, Resource, Context) is the standard shape of that **authorization request** at the PDP boundary; it is not the definition of a capability.

Three ideas combine to form the design:

1. **Capabilities are action–resource pairs**;  At evaluation time, a PDP receives a **PARC-shaped request**. Engine neutrality comes from the fact that every PDP class can consume PARC-shaped requests while the catalog stays a domain-centric inventory of discrete **(action, resource)** entries — separate from person-centric **entitlement** matrices maintained in IAM or IGA systems.
2. **A GovOps repository centered on `GovOps-AC`.** A single `#CapabilityCatalog` inventories the enterprise's authorization surface; a `#Lexicon` defines canonical terms; and `#MappingDocument` files link capabilities to compliance frameworks. Governance metadata (risk-tier, sensitivity, documented context expectations) lives **on the capability entries**, not in a parallel pillar or scoring framework. **Policy binaries are not stored in this repository** — they are authored and published separately as versioned artifacts and distributed using software supply-chain trust models (signing, provenance, registries).
3. **Catalog–policy alignment.** `govops lint` validates the catalog; `govops drift` compares the catalog to **published policy releases** (supplied at run time by URI, digest, or local path) so the authorization surface stays enumerable and aligned with enforcement.

Gemara already provides 90% of the substrate: a stable `#CapabilityCatalog` (ADR-0019), `#ControlCatalog` and `#AssessmentRequirement`, mapping primitives that include `Capability` as an `EntryType`, `#Lexicon`, `#EvaluationLog`, `#EnforcementLog`, and `#AuditLog`. The remaining 10% is a small, additive **Enterprise Capability Profile** of `#Capability` and a set of conventions for arranging the artifacts.

---

## 2. Background and motivation

### 2.1 What enterprises need to govern

Modern enterprises run thousands of services, APIs, and autonomous agents. Each participates in authorization decisions that are ultimately answered as: *given this PARC Principal (requester), this action on this resource, and this context (including token and environmental evidence), does policy permit the operation?*

Today the **authorization surface** — the set of meaningful **(action, resource)** combinations the estate exposes — is described in many incompatible ways:

- IAM systems (AWS, Azure, GCP) each define their own service / action / resource taxonomies.
- IGA platforms model "entitlements" as opaque rows in a directory.
- Application teams hard-code action / resource pairs in policy engines (Cedar, OPA / Rego, OpenFGA, AuthZEN PDPs, Casbin, custom RBAC).
- Compliance teams describe the same surface in prose ("limit access to PII to authorized personnel") with no machine-readable link to enforcement.

The **GovOps WG proposal** (Objective #2, *"Govern Capabilities, Not Just Identities"*) explicitly calls out the need to expand governance from *"who has which role"* to governing **what operations on what resources** exist and how they are controlled. That requires a shared, declarative inventory — a **catalog of (action, resource) capabilities** — that is:

1. **Finitely enumerable** so it can be reviewed, measured, and reasoned about (GovOps Objective #3).
2. **Engine neutral** so policy can be analyzed across Cedar, OPA, OpenFGA, IAM, graph engines, and AuthZEN-conformant PDPs.
3. **Composable** with threats, controls, risks, and policies so a capability becomes traceable from "what the enterprise exposes as authorizable" through "what could go wrong" to "what we measure and enforce."
4. **GRC-aligned** to plug into ISO 27001, SOC 2, NIST 800-53 reporting without a parallel data model.

### 2.2 Capabilities are (action, resource); PARC is how PDPs evaluate requests

A **capability** is an **(action, resource)** pair: a discrete, nameable operation on a resource type in a domain. It is **not** a principal, not a role, and not a context tuple. Risk and governance attach to the pair itself; **Context** (including JWT-derived claims) supplies evidence at evaluation time, not the capability's identity.

The OpenID AuthZEN working group standardizes **PARC** — Principal|Subject, Action, Resource, Context — as the **payload of an authorization request** to a PDP. A conforming PDP accepts a PARC-shaped request and returns Permit / Deny plus optional obligations and reasons. PARC is intentionally orthogonal to the policy store strategy:

- A **policy-language PDP** (Cedar, Rego) evaluates rules over PARC-shaped requests.
- A **relationship/graph PDP** (OpenFGA, Zanzibar) checks reachability given PARC-shaped inputs.
- An **ABAC engine** evaluates attribute predicates over PARC-shaped inputs.
- An **RBAC mapper** resolves the Principal's roles and ultimately answers about **action on resource** for a concrete request.

**Relationship between the catalog and PARC.** The **GovOps-AC** catalog lists **capabilities** — **(action, resource)** — the organization treats as first-class. When an application asks for a decision, it sends a **PARC request** whose **Action** and **Resource** (and usually resource instance identifier) refer to one of those capabilities; **Principal** and **Context** supply the evidence policy uses to decide **whether that capability may be exercised in this invocation** — including, in typical deployments, **signed JWTs** or equivalent tokens whose claims appear in **Context** for the PDP to evaluate. The catalog is therefore a domain-centric **registry of (action, resource) capabilities**; PARC is the wire-level shape of the authorization **question**, not the definition of the capability.

### 2.3 Why Gemara

Gemara already supplies:

- A layered model (Vectors → Threats → Controls → Policies → Evaluation/Enforcement/Audit logs) that maps cleanly onto authorization governance.
- A first-class **`#CapabilityCatalog`** artifact (ADR-0019) intended specifically so that capabilities can be referenced uniformly by `id` from threats, controls, and other catalogs, instead of duplicated inline.
- Mapping primitives (`#ArtifactMapping`, `#EntryMapping`, `#MultiEntryMapping`, `#MappingDocument`) that already include `Capability` in `#EntryType`, so capabilities can be the source or target of mappings to/from controls and external frameworks.
- A **`#Lexicon`** artifact for controlled vocabulary, useful for action verbs and resource type names.
- A `#Catalog` base that supports `extends` and `imports` for vendor-specific or industry overlays.

In short, Gemara already has the right *shape*; what is missing is a profile that constrains `#Capability` to carry a well-typed **(action, resource)** capability identity, plus conventions for organizing controls and evidence around that inventory — without conflating the catalog with the PARC request envelope.

### 2.4 Automated policy conformance

GRC frameworks today express requirements as **assertions about configuration**:

> "Sensitive transactions MUST require multi-factor authentication."

Compliance is then attested by an auditor checking that an MFA toggle is on. That tells us the *control is present*; it does not tell us the *control is sufficient*.

GovOps + Gemara can elevate the standard. With a **(action, resource)** capability catalog and a **published policy release** (versioned binary evaluated out-of-band from the GovOps repo), the same requirement can be expressed as a **claim about the enforced authorization surface**:

> "There exists no satisfiable authorization state in which capability `payments:transfer:bank-account` (the pair **transfer** on **BankAccount**) may be permitted and `context.acr` is not `urn:mfa`."

That claim is stated over **requests** (PARC-shaped evaluations for that action and resource); the **capability** in the catalog remains only **transfer / BankAccount**. 

PDP implementations can provide different solutions to provide automation of conformance checks, with the goal of reducing manual auditor reviews. 

---

## 3. Definitions

| Term | Meaning |
|---|---|
| **Capability** | An **(action, resource)** pair: a discrete, nameable operation on a resource type. *What* can be done on *what* is catalogued here, independent of *who* is asking in any given request. A capability is **not** a principal and **not** a context tuple. |
| **Principal** | The actor supplied in a **PARC authorization request** when the PDP is asked whether an operation is permitted. Policy may use principal attributes; the principal does not define the capability. |
| **Action** | The verb half of a capability (e.g., `read`, `create`, `delete`, `transfer`, `assume`). |
| **Resource** | The noun half of a capability: the resource **type** the action applies to (e.g., `Invoice`, `BankAccount`, `Repository`, `PolicyDocument`). The catalog records the *type*; runtime requests identify specific instances. |
| **Context** | Attributes bundled with a **PARC request** (time, IP, approval counts, device posture, etc.), commonly including **signed JWTs** ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)) or similar tokens whose claims the PDP treats as **evidence**. Context answers *whether* a capability may be exercised in this invocation; it is not part of the capability identity. |
| **PARC** | Principal, Action, Resource, Context — the  **authorization request** shape. Used at the PDP boundary; **not** synonymous with "capability." |
| **Enterprise capability** | A Gemara catalog entry whose **identity** is an **(action, resource)** pair (plus `id`, `title`, `description`, `group` per `#Capability`). Optional fields may document risk, sensitivity, or **expected** policy preconditions; those extensions do not redefine the capability. |
| **Documented context expectation** | Optional metadata on a capability describing **PARC Context** the organization expects policy to enforce (e.g., MFA, approval counts). Used by `govops drift` and compliance mapping; not part of capability identity. |

---

## 4. Design goals and non-goals

### Goals

1. **Reuse, do not replace.** Build on Gemara's existing `#CapabilityCatalog`, `#ControlCatalog`, and mapping primitives. No fork.
2. **Additive schema only.** Any extension must be expressible as a CUE refinement that still validates as a `#Capability`.
3. **Engine-neutral capabilities and PARC-shaped requests.** Capability rows in the catalog are **(action, resource)** only. Interoperability with PDPs uses **PARC** as the standard **request** envelope at evaluation time. The design MUST NOT privilege any specific policy engine, policy store strategy, or authorization API.
4. **GRC-aligned.** Each capability can be mapped to entries in NIST 800-53, ISO 27001, SOC 2 via `#MappingDocument`.
5. **Reviewable.** The catalog is the canonical input to IGA access reviews, compliance queries, and drift detection.

### Non-goals

- Defining a new policy language, evaluation API, or policy store specification.
- Modeling individual permission grants to specific principals (that is a runtime decision/enforcement concern — Gemara Layers 5–7).
- Capturing resource instance hierarchies or organizational topology.
- Replacing IGA entitlement catalogs. The two can coexist; the enterprise capability catalog can feed an IGA catalog or be derived from one.

---

## 5. Repository layout

A GovOps repository is a directory of **Gemara artifacts** that describe the enterprise authorization surface and its compliance mappings. Policy enforcement rules live **outside** this tree as versioned binaries (see below). A representative tree:

```text
govops/
  lexicon.yaml                       # #Lexicon
  metadata.yaml                      # shared metadata fragments (optional)
  GovOps-AC.yaml                     # #CapabilityCatalog (Enterprise Capability Profile)
  mappings/                          # #MappingDocument files (NIST, ISO, SOC 2, OSPS, ...)
  exports/                           # IGA exporter output (CSV, etc.; optional)
```

Mapping each top-level item to a Gemara artifact type:

| Path | Gemara artifact | Purpose |
|---|---|---|
| `lexicon.yaml` | `#Lexicon` | Canonical action verbs and resource type names. |
| `metadata.yaml` | shared `#Metadata` includes | Author, version, lexicon reference, applicability groups. |
| `GovOps-AC.yaml` | `#CapabilityCatalog` of `#EnterpriseCapability` | Enterprise authorization surface. |
| `mappings/` | `#MappingDocument` | Capability-to-framework mappings. |
| `exports/` | convention | IGA-ingestible views of the catalog (optional). |

Notes:

- Nothing in the layout is normative. Organizations MAY add other Gemara catalogs (`#ControlCatalog`, `#ThreatCatalog`, `#EvaluationLog`) using the standard Gemara model; those are **not** part of the ACC core layout.
- **Policy artifacts are out of scope for the repository layout.** Authorization policy is authored in engine-specific tooling, **published as versioned binaries**, and distributed like other software (signed releases, OCI or artifact registries, internal package feeds). `govops drift` accepts policy inputs at **invocation time** (file path, release URI, content digest); it does not require policy bytes to live beside the catalog. Drift plug-ins treat each engine's format as opaque and extract **(action, resource)** tuples for comparison. Capability `engine-bindings` MAY record a **policy-release** reference (URI, version, digest) for correlation without embedding policy source in the GovOps repo.

---

## 6. The Enterprise Capability Catalog (`GovOps-AC`)

### 6.1 Where it sits in Gemara

A `#CapabilityCatalog` is **not** layered. It is **domain content** that crosses Gemara's seven layers, just as ADR-0019 already treats it for system capabilities:

```text
Layer 1   Vectors, Guidance
Layer 2   Threats, Controls          <-- optional Gemara catalogs (not required for ACC)
Layer 3   Risks, Policies
Layer 4   Sensitive activities (e.g., grant access, deploy policy)
Layer 5   Evaluation logs            <-- per-claim proof results
Layer 6   Enforcement logs
Layer 7   Audit logs

  ^   Capability Catalog (cross-cutting domain content)
      - referenced by Threats via #Threat.capabilities
      - referenced by Controls via #MappingDocument
      - referenced by Policies via imports
      - measured by Evaluation/Enforcement logs (coverage, drift, proof)
```

### 6.2 The Enterprise Capability Profile

Two equivalent encodings are offered. Authors MAY pick (A) for a zero-schema-change start and migrate to (B) when the schema extension is ratified by Gemara.

#### (A) Convention-only profile (no schema change)

Use the existing `#Capability` fields and encode the structured payload in `id` and `description`:

- `id` MUST follow the form `<service>:<action>:<resource>` (e.g., `payments:transfer:bank-account`).
- `title` is the human label.
- `description` MUST contain a YAML front-matter block. **Only `action` and `resource` are required** — they are the **capability identity**. Any other keys are optional **catalog documentation** (risk tier, sensitivity, expected context for policy reviewers, typical caller classes in requests, etc.) and **do not** redefine the capability.

```yaml
- id: payments:transfer:bank-account
  title: Transfer funds
  group: g.payments
  description: |
    ---
    action: Transfer
    resource: BankAccount
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
      expected-context:
        - id: c.mfa
          text: "Policy MUST require strong authentication evidence in context"
          expression: "context.acr == 'urn:mfa'"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical
    ---
    Initiate a funds transfer between bank accounts in the payments domain.
```

This validates against today's stable `#CapabilityCatalog`. Tooling parses the front matter.

#### (B) Schema extension (recommended)

Introduce `#EnterpriseCapability` as an embedding of `#Capability`. The CUE definition (intended for upstream contribution to Gemara, or hosted as a sibling overlay module under the GovOps WG):

```cue
// Schema lifecycle: experimental
@status("experimental")
package gemara

// EnterpriseCapability is a Capability whose identity is the pair
// (action, resource). Optional fields document risk, sensitivity, or
// expected PDP / policy context — they are NOT part of the capability.
#EnterpriseCapability: {
    #Capability

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

`#Group` (post ADR-0020) is the single grouping primitive. Use it for **service** or **domain**:

- `groups`: one entry per service or business domain (e.g., `payments`, `iam`, `governance`, `release-engineering`).
- Each `#EnterpriseCapability.group` references a group `id`.

Use `metadata.applicability-groups` for orthogonal classifications that drive metrics:

- **Sensitivity** — `public`, `internal`, `confidential`, `pii`, `regulated`.
- **Risk tier** — `low`, `medium`, `high`, `critical`.
- **Lifecycle stage** — `design-time`, `build-time`, `deploy-time`, `runtime`.
- **Invocation profile** — `interactive-subject`, `software-only`, `mixed` (optional catalog metadata describing how the capability is usually invoked; does not change the (action, resource) identity).

These groups become the dimensions for GovOps metrics: *"What high-impact capabilities exist? Which are weakly governed?"* (GovOps Objective #5).

### 6.4 Anchoring vocabulary with `#Lexicon`

Each `action` and `resource` SHOULD resolve to a term in the repository's `#Lexicon` so that synonyms collapse to a single canonical id:

```yaml
title: GovOps Action and Resource Lexicon
metadata:
  id: lex.govops.actions-resources
  type: Lexicon
  gemara-version: "0.x"
  description: Canonical verbs and resource type names for enterprise capabilities.
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

- `#EnterpriseCapability.resource` is the **resource type** half of the capability (e.g., `BankAccount`).
- `#Resource` (entities) is a **runtime instance** being evaluated (e.g., the production payments service running v1.4.2).

A measurement record carries both: the `target` of an evaluation log is a `#Resource`; what was *evaluated about it* is the conformance of its policies to a set of `#EnterpriseCapability` entries (each keyed by **action + resource**).

### 6.6 Threats and risks over capabilities

Gemara's existing `#Threat.capabilities` field already accepts `#MultiEntryMapping`, so threats can target an enterprise capability without any additional schema work:

```yaml
threats:
  - id: t.transfer.fraud
    title: Unauthorized funds transfer
    description: An attacker with stale credentials initiates fraudulent transfers.
    group: g.financial
    capabilities:
      - reference-id: ec
        entries:
          - reference-id: payments:transfer:bank-account
```

Layer-3 `#Risk` entries and `#Policy` documents reference threats and controls in turn, so a single line of an enterprise capability flows through the entire seven-layer model.

---

## 7. Optional Gemara layering and context expectations

The **Authorization Capability Catalog** (`GovOps-AC`) is the Phase 1 core. Organizations MAY also maintain standard Gemara `#ControlCatalog`, `#ThreatCatalog`, `#Risk`, and `#Policy` artifacts that reference capability ids (see §6.7). That full layered model is valuable but **not required** for ACC conformance and is **not** split into a separate pillar or scoring silo.

### 7.1 Documented context expectations

High-risk capabilities SHOULD record **documented-context-expectations** on the catalog entry — human-readable (and optionally machine-readable) statements about **PARC Context** the organization expects published policy to enforce (e.g., `context.acr == 'urn:mfa'`, minimum approver count). These expectations are **not** part of capability identity; they document governance intent and feed **`govops drift`** Type-C checks (catalog expectation vs. a supplied policy release).

### 7.2 Optional: provable claims (future phase)

Where an organization wants stronger assurance than drift alone, a future GovOps phase MAY attach symbolic proofs to context expectations (UNSAT over a published policy release). That work reuses Gemara `#EvaluationLog` and optional `proofs/` artifacts; it is out of scope for the minimal ACC repository layout in §5.

---

## 8. Worked example

A minimal **Acme Bank** payments scenario: catalog authoring, compliance mapping, and drift against a published Cedar policy release.

### 8.1 `GovOps-AC.yaml` (excerpt)

```yaml
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
  - id: payments:read:invoice
    title: Read invoice
    description: Retrieve an existing Invoice by id.
    group: g.payments
    action: read
    resource: Invoice
    documentation:
      typical-requester-classes: [interactive-subject, software-agent, autonomous-actor]
    data-sensitivity: pii
    risk-tier: medium

  - id: payments:transfer:bank-account
    title: Transfer funds
    description: Initiate a funds transfer between bank accounts.
    group: g.payments
    action: transfer
    resource: BankAccount
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
      expected-context:
        - id: c.mfa
          text: Policy MUST require strong authentication evidence in PARC Context
          expression: "context.acr == 'urn:mfa'"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical

  - id: lending:approve:loan
    title: Approve loan
    description: Approve a loan application or facility change.
    group: g.lending
    action: approve
    resource: Loan
    documentation:
      typical-requester-classes: [interactive-subject]
      expected-context:
        - id: c.approvals
          text: Two distinct approvers required in PARC Context
          expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
          expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical

  - id: fraud:flag:transaction
    title: Flag transaction
    description: Flag a payment transaction for fraud review.
    group: g.fraud
    action: flag
    resource: Transaction
    documentation:
      typical-requester-classes: [interactive-subject, software-agent]
    data-sensitivity: internal
    risk-tier: high
```

### 8.2 Mapping capabilities to NIST (excerpt)

```yaml
# mappings/acme-capabilities-nist.yaml
title: Acme Capabilities to NIST 800-53r5
metadata:
  id: map.acme.ec.nist80053r5
  type: MappingDocument
  mapping-references:
    - id: ec
      title: Acme Enterprise Capability Catalog
    - id: nist80053r5
      title: NIST SP 800-53 Rev. 5
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
        rationale: Access enforcement on critical transfer capability; MFA documented in context expectations.
      - entry-id: IA-2(1)
        rationale: Strong authentication evidence in PARC Context for transfers.
```

A compliance query becomes: *"Which capabilities at `risk-tier >= high` lack a mapping to AC-3?"* — answered from `GovOps-AC` and the mapping document alone.

### 8.3 Drift check (illustrative)

Policy for the payments service is published separately (e.g., `oci://registry.acme.example/authz/payments-policy:2026.1.4` with digest `sha256:…`). Drift is run against that release, not against files in the GovOps tree:

```bash
govops drift \
  --catalog govops/GovOps-AC.yaml \
  --policy oci://registry.acme.example/authz/payments-policy:2026.1.4 \
  --digest sha256:abc123… \
  --engine cedar
```

A Type-C finding on `payments:transfer:bank-account` means the supplied policy release does not match the catalog's `context.acr == 'urn:mfa'` expectation — the gap is expressed in **capability id** terms, not a separate metrics framework.

---

## 9. Mapping to compliance frameworks

Use Gemara's `#MappingDocument` with `source-reference` pointing at the **`#CapabilityCatalog`** (`GovOps-AC`) and `entry-type: Capability`. Map capability ids directly to NIST 800-53, ISO 27001, SOC 2, or OSPS Baseline controls. Optional separate `#ControlCatalog` mappings remain valid Gemara but are not required for ACC.

---

## 10. Tooling implications

Phase 1 reference tooling (no Gemara schema changes beyond the optional EC profile):

1. **`govops lint`** — Lexicon resolution, group membership, applicability-group value checks, engine-binding well-formedness.
2. **`govops drift`** — Compare catalog **(action, resource)** entries to **published policy releases** per engine (artifacts passed in at run time by path, URI, or digest); report Type A (catalog without policy), Type B (policy without catalog), Type C (context-expectation mismatch).
3. **IGA exporter** — Emit CSV / SCIM / OSCAL from `GovOps-AC` for entitlement catalogs and access reviews.

Engine-specific drift plug-ins treat policy formats as opaque. A future phase MAY add `govops prove` for symbolic analysis of context expectations (§7.2).

---

## 11. Adoption / migration path

**Phase 0 — Convention-only profile (today).** Use stable `#CapabilityCatalog` with §6.2(A) front-matter; validate with existing Gemara tooling.

**Phase 1 — GovOps overlay and reference mappings.** Publish `#EnterpriseCapability` CUE overlay, reference `GovOps-AC` templates, NIST/ISO/SOC 2 mapping examples, `govops lint` and `govops drift`.

**Phase 2 — Upstream into Gemara.** Propose `#EnterpriseCapability` via Gemara ADR when stable.

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

- **Enterprise Capability Profile** — capability identity = **(action, resource)**; **PARC** for authorization **requests** only ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) JWTs in **Context** as typical evidence).
- **Minimal GovOps repository** — `GovOps-AC`, lexicon, mappings; governance metadata on capabilities; policy as external versioned binaries.
- **Tooling** — `govops lint`, `govops drift`, IGA export; CI/CD alignment without a parallel scoring silo.

The *thing governed* is the finite inventory of **(action, resource)** capabilities; *assurance* starts with catalog–policy alignment and framework mappings, with stronger proofs optional later.

---

## 14. References

- Gemara — `capabilitycatalog.cue` (stable), `controlcatalog.cue` (stable), `mapping_inline.cue`, `mappingdocument.cue`, `lexicon.cue`, `entities.cue`, `threatcatalog.cue`, `evaluationlog.cue`.
- Gemara ADRs — 0017 (Base Catalog Type), 0018 (Promote Nested Concepts to Catalogs), 0019 (Promote Capabilities), 0020 (Groups), 0021 (Lexicon).
- OpenSSF GovOps WG proposal — `orbit/openssf-wg-proposal.md` (this repository).
- OpenID AuthZEN — Authorization API specification using the PARC **request** shape.
- RFC 7519 — JSON Web Token (JWT), commonly used as signed evidence carried in **PARC Context**.
- NIST SP 800-53 r5, ISO/IEC 27001, SOC 2 — example mapping targets for `#MappingDocument`.

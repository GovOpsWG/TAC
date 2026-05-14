# Cataloging Enterprise Capabilities with Gemara

**Status:** Draft for discussion
**Audience:** OpenSSF GovOps WG (proposed), Gemara maintainers, GRC and IGA practitioners
**Related:**
- [GovOps WG Proposal](./openssf-wg-proposal.md)
- [Gemara Project](https://gemara.openssf.org) — `#CapabilityCatalog` (stable), ADR-0019 *Promote Capabilities to a First-class Catalog*
- OpenID AuthZEN — PARC (Principal, Action, Resource, Context) authorization request shape

---

## 1. Abstract

This document proposes a design for representing **Enterprise Capabilities** — concrete `(Principal-types, Action, Resource, Context)` tuples that an enterprise authorizes — as **first-class catalog entries in the [Gemara](https://gemara.openssf.org) GRC engineering model**, and for organizing a complete **GovOps repository** of Gemara artifacts around them.

Three ideas combine to form the design:

1. **PARC as the canonical capability shape.** Following the OpenID AuthZEN authorization API, a capability is described by its Principal types, Action, Resource, and Context predicates. PARC is engine-neutral: any PDP — policy-language, graph, ABAC, RBAC, or hybrid — produces decisions over PARC-shaped requests, so a PARC-shaped capability declaration travels across engines without lock-in.
2. **A GovOps repository of Gemara artifacts.** A single `#CapabilityCatalog` (`GovOps-AC`) inventories the enterprise's authorization surface, and a small family of `#ControlCatalog` artifacts — one per **TIGER** pillar (Transparency, Identity, Governance, Events, Resilience) — express the requirements those capabilities must satisfy. Mapping documents tie the catalog to external compliance frameworks.
3. **Provable operational claims, not just policy assertions.** Where today's GRC says *"the system MUST require MFA"*, GovOps elevates the standard to *"there exists no satisfiable authorization path where this capability succeeds without MFA, given the deployed policy."* That shift — from *attested presence* to *demonstrated impossibility* — is what the **TIGER metric** measures.

Gemara already provides 90% of the substrate: a stable `#CapabilityCatalog` (ADR-0019), `#ControlCatalog` and `#AssessmentRequirement`, mapping primitives that include `Capability` as an `EntryType`, `#Lexicon`, `#EvaluationLog`, `#EnforcementLog`, and `#AuditLog`. The remaining 10% is a small, additive **Enterprise Capability Profile** of `#Capability` and a set of conventions for arranging the artifacts.

---

## 2. Background and motivation

### 2.1 What enterprises need to govern

Modern enterprises run thousands of services, APIs, and autonomous agents. Each makes — or is the target of — authorization decisions of the form *"Principal P is allowed to perform Action A on Resource R under Context C."*

Today that surface is described in many incompatible ways:

- IAM systems (AWS, Azure, GCP) each define their own service / action / resource taxonomies.
- IGA platforms model "entitlements" as opaque rows in a directory.
- Application teams hard-code action / resource pairs in policy engines (Cedar, OPA / Rego, OpenFGA, AuthZEN PDPs, Casbin, custom RBAC).
- Compliance teams describe the same surface in prose ("limit access to PII to authorized personnel") with no machine-readable link to enforcement.

The **GovOps WG proposal** (Objective #2, *"Govern Capabilities, Not Just Identities"*) explicitly calls out the need to expand governance from *"who has which role"* to *"which actions are possible on which resources under which conditions."* That requires a shared, declarative inventory — a **catalog of enterprise capabilities** — that is:

1. **Finitely enumerable** so it can be reviewed, measured, and reasoned about (GovOps Objective #3).
2. **Engine neutral** so policy can be analyzed across Cedar, OPA, OpenFGA, IAM, graph engines, and AuthZEN-conformant PDPs.
3. **Composable** with threats, controls, risks, and policies so a capability becomes traceable from "what the enterprise authorizes" through "what could go wrong" to "what we measure and enforce."
4. **GRC-aligned** to plug into ISO 27001, SOC 2, NIST 800-53 reporting without a parallel data model.

### 2.2 Why PARC is the right canonical shape

The OpenID AuthZEN working group has standardized **PARC** — Principal, Action, Resource, Context — as the request payload for authorization decisions. A conforming PDP exposes an evaluate endpoint that accepts a PARC-shaped request and returns Permit / Deny plus optional obligations and reasons. PARC is intentionally orthogonal to the policy store strategy:

- A **policy-language PDP** (Cedar, Rego) evaluates rules over PARC inputs.
- A **relationship/graph PDP** (OpenFGA, Zanzibar) checks reachability over PARC inputs.
- An **ABAC engine** evaluates attribute predicates over PARC inputs.
- An **RBAC mapper** resolves the Principal's roles and looks up Action/Resource pairs.

Because every PDP class consumes PARC requests, a PARC-shaped *capability declaration* is the natural enterprise-side counterpart: each catalog entry is the type signature of a class of PARC requests the enterprise authorizes. That makes the catalog the canonical, vendor-neutral inventory of the authorization surface — independent of any one policy store specification.

### 2.3 Why Gemara

Gemara already supplies:

- A layered model (Vectors → Threats → Controls → Policies → Evaluation/Enforcement/Audit logs) that maps cleanly onto authorization governance.
- A first-class **`#CapabilityCatalog`** artifact (ADR-0019) intended specifically so that capabilities can be referenced uniformly by `id` from threats, controls, and other catalogs, instead of duplicated inline.
- Mapping primitives (`#ArtifactMapping`, `#EntryMapping`, `#MultiEntryMapping`, `#MappingDocument`) that already include `Capability` in `#EntryType`, so capabilities can be the source or target of mappings to/from controls and external frameworks.
- A **`#Lexicon`** artifact for controlled vocabulary, useful for action verbs and resource type names.
- A `#Catalog` base that supports `extends` and `imports` for vendor-specific or industry overlays.

In short, Gemara already has the right *shape*; what is missing is a profile that constrains `#Capability` to express PARC semantics, plus conventions for organizing the surrounding artifacts.

### 2.4 The conceptual leap: from policy assertions to provable claims

GRC frameworks today express requirements as **assertions about configuration**:

> "Sensitive transactions MUST require multi-factor authentication."

Compliance is then attested by an auditor checking that an MFA toggle is on. That tells us the *control is present*; it does not tell us the *control is sufficient*.

GovOps + Gemara can elevate the standard. With a PARC-shaped capability catalog and a deployed policy artifact, the same requirement can be expressed as a **claim about the deployed authorization surface**:

> "There exists no satisfiable authorization state in which capability `payments:transfer:bank-account` succeeds and `context.acr` is not `urn:mfa`."

That claim is checkable. Modern policy decision points expose symbolic analyzers (SAT/SMT-based) that can produce a decision proof: either an UNSAT certificate (the requirement holds for every reachable state) or a satisfying assignment (a counterexample showing how the requirement could be violated). The same proof shape works whether the engine is Cedar, Rego (with an SMT backend), or a graph engine reduced to a constraint problem.

The TIGER metric (§9) measures the fraction of declared requirements for which such a proof exists, weighted by capability risk. That is materially stronger than checkbox compliance.

---

## 3. Definitions

| Term | Meaning |
|---|---|
| **Principal** | The actor making an authorization request: human user, service identity, or autonomous agent. Carries attributes such as roles, identity assurance level, and group memberships. |
| **Action** | A verb naming the operation to be performed (e.g., `read`, `create`, `delete`, `transfer`, `assume`). |
| **Resource** | A typed noun naming what the action operates on (e.g., `Invoice`, `BankAccount`, `Repository`, `PolicyDocument`). The catalog records the *type*; runtime decisions resolve to specific instances. |
| **Context** | Environmental attributes that scope the request: time, IP, MFA state, device posture, approval count, geo, etc. |
| **PARC** | The 4-tuple (Principal, Action, Resource, Context) used as the canonical request shape by OpenID AuthZEN-conformant PDPs and as the canonical capability-declaration shape in this design. |
| **Enterprise Capability** | A reviewable unit of authorization: a PARC type signature plus governance metadata (sensitivity, risk tier, owner). |
| **TIGER** | Five governance pillars used by this design: **T**ransparency, **I**dentity, **G**overnance, **E**vents, **R**esilience. Also the name of the derived metric defined in §9. |
| **Provable claim** | A requirement expressed in a form checkable by a policy analyzer, such that the deployed policy either admits a proof of the claim (e.g., UNSAT for a counterexample formula) or yields a counterexample. |

---

## 4. Design goals and non-goals

### Goals

1. **Reuse, do not replace.** Build on Gemara's existing `#CapabilityCatalog`, `#ControlCatalog`, and mapping primitives. No fork.
2. **Additive schema only.** Any extension must be expressible as a CUE refinement that still validates as a `#Capability`.
3. **Engine-neutral via PARC.** Capability declarations describe PARC type signatures and SHOULD NOT privilege any specific policy engine, policy store strategy, or authorization API.
4. **Provable claims first-class.** The design records not only declarative requirements but also the proof artifacts that show the deployed authorization surface satisfies them.
5. **GRC-aligned.** Each capability and each control can be mapped to entries in NIST 800-53, ISO 27001, SOC 2 via `#MappingDocument`.
6. **Reviewable.** The catalog is the canonical input to GovOps access review, coverage, risk, and TIGER metrics.

### Non-goals

- Defining a new policy language, evaluation API, or policy store specification.
- Modeling individual permission grants to specific principals (that is a runtime decision/enforcement concern — Gemara Layers 5–7).
- Capturing resource instance hierarchies or organizational topology.
- Replacing IGA entitlement catalogs. The two can coexist; the enterprise capability catalog can feed an IGA catalog or be derived from one.

---

## 5. Repository layout

A GovOps repository is a directory of Gemara artifacts plus the deployed policy material and runtime evidence that supports it. A representative tree:

```text
govops/
  lexicon.yaml                       # #Lexicon
  metadata.yaml                      # shared metadata fragments
  GovOps-AC.yaml                     # #CapabilityCatalog (Enterprise Capability Profile)
  GovOps-Transparency.yaml           # #ControlCatalog (T)
  GovOps-Identity.yaml               # #ControlCatalog (I)
  GovOps-Governance.yaml             # #ControlCatalog (G)
  GovOps-Events.yaml                 # #ControlCatalog (E)
  GovOps-Resilience.yaml             # #ControlCatalog (R)
  policies/                          # deployed policy artifacts (engine-specific)
  proofs/                            # decision proofs / counterexamples per claim
  evidence/                          # runtime decision logs, attestations
  mappings/                          # #MappingDocument files (NIST, ISO, SOC 2, ...)
  tiger/
    tiger-score.yaml                 # latest derived TIGER score
    tiger-evidence.json              # inputs the score was computed from
```

Mapping each top-level item to a Gemara artifact type:

| Path | Gemara artifact | Purpose |
|---|---|---|
| `lexicon.yaml` | `#Lexicon` | Canonical action verbs and resource type names. |
| `metadata.yaml` | shared `#Metadata` includes | Author, version, lexicon reference, applicability groups. |
| `GovOps-AC.yaml` | `#CapabilityCatalog` of `#EnterpriseCapability` | Enterprise authorization surface. |
| `GovOps-{T,I,G,E,R}.yaml` | `#ControlCatalog` | Pillar-scoped requirements that capabilities must satisfy. |
| `policies/` | engine-specific files | Source of truth for what the deployed PDPs evaluate. |
| `proofs/` | logical certificates referenced from `#EvaluationLog.evidence` | Per-claim UNSAT proofs or counterexamples. |
| `evidence/` | runtime data referenced from `#Evidence` entries | Decision logs, attestations feeding evaluations. |
| `mappings/` | `#MappingDocument` | Capability-to-framework mappings. |
| `tiger/` | derived | Computed metrics, not authored by hand. |

Notes:

- Nothing in the layout is normative. The same artifacts could be split, merged, or re-organized to match an organization's repo conventions. What matters is that each artifact validates against its Gemara schema.
- The `policies/` and `evidence/` directories deliberately do not impose a vendor format. Each policy or log is referenced by `#ArtifactMapping` from the relevant Gemara artifact, so the format is selected per-engine.

---

## 6. The Enterprise Capability Catalog (`GovOps-AC`)

### 6.1 Where it sits in Gemara

A `#CapabilityCatalog` is **not** layered. It is **domain content** that crosses Gemara's seven layers, just as ADR-0019 already treats it for system capabilities:

```text
Layer 1   Vectors, Guidance
Layer 2   Threats, Controls          <-- TIGER pillar control catalogs live here
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
- `description` MUST contain a YAML front-matter block with the structured PARC fields.

```yaml
- id: payments:transfer:bank-account
  title: Transfer funds
  group: g.payments
  description: |
    ---
    principal-types: [User, Service]
    action: Transfer
    resource: BankAccount
    context-predicates:
      - id: c.mfa
        text: "Strong authentication required"
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

// EnterpriseCapability is a Capability whose semantics are a PARC
// (Principal, Action, Resource, Context) type signature describing
// a class of authorization requests the enterprise authorizes.
#EnterpriseCapability: {
    #Capability

    // principal-types enumerates the principal kinds permitted to
    // hold this capability (e.g., "User", "Service", "Agent").
    "principal-types"?: [string, ...string] @go(PrincipalTypes)

    // action is a verb naming the authorizable operation.
    // SHOULD be drawn from the catalog's lexicon for consistency.
    action: string

    // resource is the resource type the action operates on.
    // SHOULD be drawn from the catalog's lexicon.
    resource: string

    // context-predicates enumerates predicates over the request
    // context that MUST hold for the action to be allowed.
    "context-predicates"?: [#ContextPredicate, ...#ContextPredicate] @go(ContextPredicates)

    // data-sensitivity classifies the data exposed by the action.
    // Values SHOULD come from an applicability-group named "sensitivity".
    "data-sensitivity"?: string @go(DataSensitivity)

    // risk-tier is a coarse risk classification used by GovOps metrics.
    "risk-tier"?: "low" | "medium" | "high" | "critical" @go(RiskTier)

    // engine-bindings records non-normative engine-specific identifiers.
    // The catalog itself remains engine-neutral; bindings exist so
    // implementations can correlate this PARC entry to the symbols
    // used by their PDP, IAM service, or graph engine.
    "engine-bindings"?: [#EngineBinding, ...#EngineBinding] @go(EngineBindings)
}

// ContextPredicate captures one predicate over the PARC Context.
#ContextPredicate: {
    id:   string
    text: string

    // expression is an optional machine-readable form of the predicate.
    // expression-language identifies the dialect.
    expression?:            string
    "expression-language"?: string
}

// EngineBinding correlates this PARC entry with engine-specific symbols.
// Listed engines are illustrative; the field is open to any string.
#EngineBinding: {
    engine: string

    // action-id is the engine-specific identifier for the action.
    "action-id"?: string

    // resource-type is the engine-specific identifier for the resource type.
    "resource-type"?: string

    notes?: string
}
```

The top-level fields (`principal-types`, `action`, `resource`, `context-predicates`) ARE the PARC type signature; nothing about the shape is engine-specific. Engine-specific symbols live in the optional `engine-bindings` array, which any number of engines can populate.

### 6.3 Catalog organization with `#Group`

`#Group` (post ADR-0020) is the single grouping primitive. Use it for **service** or **domain**:

- `groups`: one entry per service or business domain (e.g., `payments`, `identity`, `governance`, `release-engineering`).
- Each `#EnterpriseCapability.group` references a group `id`.

Use `metadata.applicability-groups` for orthogonal classifications that drive metrics:

- **Sensitivity** — `public`, `internal`, `confidential`, `pii`, `regulated`.
- **Risk tier** — `low`, `medium`, `high`, `critical`.
- **Lifecycle stage** — `design-time`, `build-time`, `deploy-time`, `runtime`.
- **Audience** — `human-only`, `service-only`, `mixed`, `agent-eligible`.

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

Gemara already defines `#Resource` (in `entities.cue`) as a runtime entity that is the *target of an evaluation* — the thing an `#EvaluationLog`, `#EnforcementLog`, or `#AuditLog` is *about*. That is **not** the same as the PARC `Resource` type used inside an enterprise capability.

The two stay separate:

- `#EnterpriseCapability.resource` is a **type name** drawn from the lexicon (e.g., `BankAccount`).
- `#Resource` (entities) is a **runtime instance** being evaluated (e.g., the production payments service running v1.4.2).

A measurement record carries both: the `target` of an evaluation log is a `#Resource`; what was *evaluated about it* is the conformance of its policies to a set of `#EnterpriseCapability` entries.

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

## 7. The TIGER pillar control catalogs

Each TIGER pillar is a `#ControlCatalog` whose controls reference capabilities in `GovOps-AC` (via `#MultiEntryMapping`) and whose `#AssessmentRequirement`s define what must be true of those capabilities. A single requirement is the unit of evaluation: either it admits a proof, has runtime evidence, or fails.

### 7.1 `GovOps-Transparency.yaml`

Transparency requires that decisions are observable and explainable: a stakeholder can reconstruct why a request was permitted or denied.

```yaml
title: GovOps Transparency Catalog
metadata:
  id: cat.govops.transparency
  type: ControlCatalog
  gemara-version: "0.x"
  description: Decisions over enterprise capabilities must be observable and reconstructable.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.tr
    title: Transparency
    description: Decision observability and explainability.
controls:
  - id: GOVOPS-TR-01
    title: Sensitive actions produce explainable decisions
    objective: Stakeholders can reconstruct how a decision was made for any high-risk capability.
    group: g.tr
    state: Active
    assessment-requirements:
      - id: GOVOPS-TR-01.01
        text: >
          Every capability with risk-tier >= high MUST produce a signed decision record
          containing principal id, action, resource id, deciding policy id, and the
          context attributes the decision relied on.
        applicability: [production]
        state: Active
      - id: GOVOPS-TR-01.02
        text: >
          A reviewer MUST be able to reconstruct, from evidence alone, the chain of
          policies and attributes that produced any recorded Permit decision.
        applicability: [production]
        state: Active
```

Evidence sources: PDP decision logs, AuthZEN evaluate response reasons, append-only event stores, attestations from the policy distribution pipeline.

### 7.2 `GovOps-Identity.yaml`

Identity requires that high-impact capabilities are unreachable under weak identity assurance. The novel framing is **mathematical impossibility, not configuration presence**.

```yaml
title: GovOps Identity Catalog
metadata:
  id: cat.govops.identity
  type: ControlCatalog
  gemara-version: "0.x"
  description: Sensitive capabilities are unreachable without sufficient identity assurance.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.id
    title: Identity
    description: Identity assurance preconditions on capabilities.
controls:
  - id: GOVOPS-ID-01
    title: Sensitive capabilities require strong authentication
    objective: No authorization path exists in which a sensitive capability succeeds without strong authentication.
    group: g.id
    state: Active
    assessment-requirements:
      - id: GOVOPS-ID-01.01
        text: >
          For every capability with data-sensitivity in {confidential, pii, regulated}
          OR risk-tier >= high, the deployed policy MUST require an authentication
          context class reference (acr) of urn:mfa or stronger.
        applicability: [production]
        state: Active
      - id: GOVOPS-ID-01.02
        text: >
          A symbolic analysis of the deployed policy MUST yield UNSAT for the
          formula: capability succeeds AND principal.acr != urn:mfa.
        applicability: [production]
        state: Active
```

Note the difference between `01.01` and `01.02`: the first checks that a configuration is present; the second checks that no execution path circumvents it. Both are recorded; `01.02` is the provable claim.

### 7.3 `GovOps-Governance.yaml`

Governance governs *who can govern* — separation of duties, dual control, time-bounded overrides, change approval.

```yaml
title: GovOps Governance Catalog
metadata:
  id: cat.govops.governance
  type: ControlCatalog
  gemara-version: "0.x"
  description: Authority over the GovOps repository itself is bounded and auditable.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.gv
    title: Governance
    description: Controls over the governance plane itself.
controls:
  - id: GOVOPS-GV-01
    title: Governance changes require separation of duties
    objective: No single principal can unilaterally modify governance policy.
    group: g.gv
    state: Active
    assessment-requirements:
      - id: GOVOPS-GV-01.01
        text: >
          Capabilities with action == policy-write and resource == GovernancePolicy
          MUST require at least two distinct approvers in the request context.
        applicability: [production]
        state: Active
      - id: GOVOPS-GV-01.02
        text: >
          Symbolic analysis MUST yield UNSAT for the formula: governance policy-write
          succeeds AND distinct(context.approvals) < 2.
        applicability: [production]
        state: Active
      - id: GOVOPS-GV-01.03
        text: Emergency overrides MUST expire automatically within a documented bound.
        applicability: [production]
        state: Active
```

### 7.4 `GovOps-Events.yaml`

Events define the decision telemetry the rest of the framework depends on. Without events there is no evidence and no proof artifact to record.

```yaml
title: GovOps Events Catalog
metadata:
  id: cat.govops.events
  type: ControlCatalog
  gemara-version: "0.x"
  description: Decision telemetry is captured in a form that supports replay and proof.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.ev
    title: Events
    description: Decision telemetry and event capture.
controls:
  - id: GOVOPS-EV-01
    title: Governance-significant events are captured
    objective: Decision and policy events relevant to governance are captured durably.
    group: g.ev
    state: Active
    assessment-requirements:
      - id: GOVOPS-EV-01.01
        text: Every authorization decision affecting a catalog capability MUST emit an event.
        applicability: [production]
        state: Active
      - id: GOVOPS-EV-01.02
        text: Policy distribution events (publish, rollback) MUST emit signed records.
        applicability: [production]
        state: Active
      - id: GOVOPS-EV-01.03
        text: Event streams MUST support replay for investigation and re-evaluation.
        applicability: [production]
        state: Active
```

### 7.5 `GovOps-Resilience.yaml`

Resilience requires that governance decisions remain enforceable when the central governance plane is degraded — through embedded PDPs, signed cached policy material, and explicit fail-closed semantics for critical capabilities.

```yaml
title: GovOps Resilience Catalog
metadata:
  id: cat.govops.resilience
  type: ControlCatalog
  gemara-version: "0.x"
  description: Governance decisions remain enforceable during degradation of the governance plane.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.rs
    title: Resilience
    description: Continuity of governance under degradation.
controls:
  - id: GOVOPS-RS-01
    title: Critical capabilities remain enforceable during outages
    objective: Decision-point unavailability does not turn into bypass.
    group: g.rs
    state: Active
    assessment-requirements:
      - id: GOVOPS-RS-01.01
        text: Capabilities with risk-tier == critical MUST support local PDP evaluation.
        applicability: [production]
        state: Active
      - id: GOVOPS-RS-01.02
        text: Cached policy material MUST be cryptographically validated before use.
        applicability: [production]
        state: Active
      - id: GOVOPS-RS-01.03
        text: PDP unreachability MUST result in fail-closed for critical capabilities.
        applicability: [production]
        state: Active
```

---

## 8. From policy assertions to provable operational claims

A central design idea is that `#AssessmentRequirement`s come in two flavors and Gemara already accommodates both:

1. **Configuration assertions.** *"Capability X requires MFA in policy."* Verified by inspecting deployed policy or runtime decision logs.
2. **Provable claims.** *"There exists no satisfiable authorization state in which X succeeds without MFA."* Verified by feeding the deployed policy to a symbolic analyzer and recording the resulting proof or counterexample.

Both kinds of requirement live in the TIGER pillar catalogs. The GovOps repository convention is that the *assessment-requirement text* is engine-neutral (refers to the catalog, the PARC fields, and predicates over context attributes), while the *evidence and proofs* in `evidence/` and `proofs/` are produced by whatever PDP is deployed.

### 8.1 Engine-neutral statement of a provable claim

A provable claim names: a capability id, a property to prove (or refute) about its PARC envelope, and an analyzer attestation. The same statement is meaningful for any PDP class:

| PDP class | How the claim is discharged |
|---|---|
| Policy-language PDP (Cedar, Rego, etc.) | Symbolic analyzer reduces the claim to SAT/SMT and returns UNSAT or a counterexample. |
| Graph / relationship PDP (OpenFGA, Zanzibar) | Reachability check, optionally reduced to constraints over relationship tuples and context. |
| ABAC engine | Constraint-solver over the predicate set defining each rule. |
| RBAC mapper | Enumerate role -> action/resource closures and check disjointness with the claim. |
| Hybrid / AuthZEN-conformant PDP | Vendor-provided analyzer or external proof tool over the underlying representation. |

The catalog does not prescribe the analyzer. It only requires that the proof artifact be produced, signed (when feasible), and referenced from the evaluation log.

### 8.2 Recording a proof in Gemara

The `#EvaluationLog` schema already includes `#Evidence` entries with a `type` field (`#EvidenceType` is open). The repository convention adds two well-known evidence types:

- `DecisionLog` — a runtime decision record correlating to a capability id.
- `Proof` — a signed analyzer output referencing a specific claim id, deployed policy version, and analyzer identity.

A `Proof` evidence entry points (via `#ArtifactMapping`) at a file under `proofs/` containing the analyzer transcript: the formula, the result (UNSAT/SAT), and any counterexample. The hash of the deployed policy artifact is included so the proof is anchored to a specific point-in-time policy state.

### 8.3 What the leap buys

A traditional assertion ("MFA is required") tells an auditor that a control is *present*. A provable claim ("no path succeeds without MFA") tells the auditor that the control is *sufficient* — relative to the deployed policy. The first is necessary; the second is what makes governance defensible against unintended interactions, exception flags, and policy drift.

GovOps does not require every requirement to be provable. Many TIGER requirements (event capture, signed distribution, override expiry) are observably-true rather than provably-true. The repository simply records which is which, and the TIGER metric (§9) weights the two appropriately.

---

## 9. The TIGER metric

TIGER is the **delta between declared governance requirements and provable operational reality** across five pillars: Transparency, Identity, Governance, Events, Resilience.

### 9.1 Per-pillar score

For each pillar P, define:

```text
score(P) =
   sum over each AssessmentRequirement r in pillar P:
       weight(r) * status(r)
   ----------------------------------------------------
   sum over each AssessmentRequirement r in pillar P:
       weight(r)
```

Where:

- `weight(r)` is an authoring-time weight derived from the risk-tier of the capabilities the parent control references (a control governing only `low` capabilities counts less than one governing `critical` capabilities).
- `status(r)` is `1` if the requirement is satisfied with current evidence, `0` if it is not. For provable claims, "satisfied" means the proof is current (its policy hash matches the deployed policy version) and its result is UNSAT for the negation. For observable claims, "satisfied" means the runtime evidence stream confirms the requirement over the reporting window.

### 9.2 Aggregate TIGER

```text
TIGER = w_T * score(T) + w_I * score(I) + w_G * score(G) + w_E * score(E) + w_R * score(R)
```

Default pillar weights (illustrative; organizations may re-weight): `T=20, I=25, G=20, E=15, R=20`. The Identity pillar is weighted slightly higher by default because identity preconditions tend to dominate other pillars in measurable risk reduction; other organizations will prefer different defaults.

### 9.3 What TIGER measures and what it does not

TIGER measures the fraction of *declared* requirements that are *currently satisfied* by *deployed* policy and runtime evidence. It is therefore:

- **Forward-looking** when the catalog grows: adding a new capability without controls reduces TIGER.
- **Drift-sensitive** when policy changes: a re-deploy that invalidates an UNSAT proof reduces TIGER until the proof is regenerated.
- **Engine-neutral**: it does not assume any particular PDP, so a heterogeneous environment composes naturally (per-engine sub-scores, weighted by capability assignment to engines).

TIGER does *not* measure:

- Adversarial robustness against unmodeled attacks.
- Quality of the catalog itself (an under-specified catalog can yield a high TIGER by trivial controls).
- Anything about identity or workforce administration that is not expressed as a capability requirement.

These are appropriate concerns for adjacent metrics (vulnerability disclosure, IGA recertification cadence, etc.), not TIGER.

### 9.4 Reporting

The `tiger/tiger-score.yaml` artifact is the rendered output. A minimal shape:

```yaml
metadata:
  id: tiger.acme.2026-05-13
  generated-at: "2026-05-13T17:30:00Z"
  policy-versions:
    - engine: example-pdp
      version: "v2026.05.13-1"
  catalog-version: "ec.acme:2026.1"
score:
  T: 0.92
  I: 0.81
  G: 0.95
  E: 0.78
  R: 0.88
  aggregate: 0.86
inputs:
  evidence: tiger/tiger-evidence.json
  proofs:   proofs/
```

`tiger-evidence.json` is the per-requirement decision: what was checked, against which evidence/proof, with which result. It is the auditor-facing trail.

---

## 10. Worked example

A minimal but complete example for an "Acme Bank" payments scenario, spanning the catalog, one TIGER pillar control, runtime evidence, a proof, and the resulting TIGER inputs.

### 10.1 `GovOps-AC.yaml` (excerpt)

```yaml
title: Acme Enterprise Capability Catalog
metadata:
  id: cat.acme.ec
  type: CapabilityCatalog
  gemara-version: "0.x"
  version: "2026.1"
  date: "2026-05-01T00:00:00Z"
  description: Authorizable PARC tuples across Acme services.
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
      description: Coarse risk classification used by GovOps metrics.
groups:
  - id: g.payments
    title: Payments
    description: Capabilities exposed by the payments domain.
  - id: g.identity
    title: Identity
    description: Capabilities exposed by the identity control plane.
  - id: g.governance
    title: Governance
    description: Capabilities exposed by the governance plane itself.
capabilities:
  - id: payments:read:invoice
    title: Read invoice
    description: Retrieve an existing Invoice by id.
    group: g.payments
    principal-types: [User, Service, Agent]
    action: read
    resource: Invoice
    data-sensitivity: pii
    risk-tier: medium

  - id: payments:transfer:bank-account
    title: Transfer funds
    description: Initiate a funds transfer between bank accounts.
    group: g.payments
    principal-types: [User, Service]
    action: transfer
    resource: BankAccount
    context-predicates:
      - id: c.mfa
        text: Strong authentication required
        expression: "context.acr == 'urn:mfa'"
        expression-language: natural-language
    data-sensitivity: confidential
    risk-tier: critical

  - id: governance:policy-write:governance-policy
    title: Modify governance policy
    description: Edit any policy in the GovOps repository.
    group: g.governance
    principal-types: [User]
    action: policy-write
    resource: GovernancePolicy
    context-predicates:
      - id: c.approvals
        text: Two distinct approvers required
        expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
        expression-language: natural-language
    risk-tier: critical
```

### 10.2 The Identity pillar control that governs the transfer capability

Already shown as `GOVOPS-ID-01` in §7.2. The control's threats reference the relevant `#Threat` entry (e.g., `t.transfer.fraud`), and the threat's `capabilities` list points back at `payments:transfer:bank-account`.

### 10.3 An evaluation log recording the proof

```yaml
metadata:
  id: log.acme.ec.identity.2026-05-13
  type: EvaluationLog
  gemara-version: "0.x"
  description: Daily evaluation of GOVOPS-ID-01 against deployed policy.
target:
  id: prod.payments.v1.4.2
  name: Payments service (production)
  type: Software
  environment: production
result: Pass
# Per-requirement evaluation entries (illustrative):
#   GOVOPS-ID-01.01 -> evidence: DecisionLog stream confirms acr=urn:mfa for all
#                       observed payments:transfer:bank-account decisions
#                       in the reporting window.
#   GOVOPS-ID-01.02 -> evidence: Proof artifact in proofs/id-01-02-2026-05-13.json,
#                       analyzer attests UNSAT for the negation, signed against
#                       deployed policy hash sha256:...
```

### 10.4 The TIGER score that results

```yaml
metadata:
  id: tiger.acme.2026-05-13
  generated-at: "2026-05-13T17:30:00Z"
  catalog-version: "ec.acme:2026.1"
score:
  T: 0.94
  I: 0.81
  G: 0.95
  E: 0.78
  R: 0.88
  aggregate: 0.87
notes: |
  Identity score reduced because GOVOPS-ID-01.02 holds for payments:transfer:bank-account
  but the corresponding UNSAT proof is missing for governance:policy-write:governance-policy
  (analyzer pending). Will re-run after analyzer integration completes.
```

The note format is intentional: TIGER tells you not only the score but the specific gap, expressed in catalog terms.

---

## 11. Mapping to compliance frameworks

Use Gemara's existing `#MappingDocument` to record how enterprise capabilities and TIGER controls relate to entries in NIST 800-53, ISO 27001, SOC 2, or any other framework. `#EntryType` already includes both `Capability` and `Control`, so mappings work without any schema change:

```yaml
title: Enterprise Capability and TIGER Identity Mapping to NIST 800-53r5
metadata:
  id: map.govops.id.nist80053r5
  type: MappingDocument
  gemara-version: "0.x"
  description: Map GOVOPS-ID controls and EC entries to NIST 800-53r5 controls.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
  mapping-references:
    - id: ec
      title: Acme Enterprise Capability Catalog
      version: "2026.1"
      url: https://acme.example/catalogs/ec.yaml
    - id: govops-id
      title: GovOps Identity Catalog
      version: "0.1"
      url: https://acme.example/catalogs/govops-identity.yaml
    - id: nist80053r5
      title: NIST SP 800-53 Rev. 5
      version: "5.1.1"
source-reference:
  reference-id: govops-id
  entry-type: Control
target-reference:
  reference-id: nist80053r5
  entry-type: Control
mappings:
  - id: m1
    source: GOVOPS-ID-01
    relationship: implements
    targets:
      - entry-id: AC-3
        rationale: "Access enforcement on sensitive capabilities."
        confidence-level: High
      - entry-id: IA-2(1)
        rationale: "MFA required for privileged actions."
```

GovOps Objective #5 ("policy coverage matches risk exposure") becomes a query: *"Which capabilities at risk-tier >= high lack a TIGER control that is mapped (via implements/supports) to AC-3 in NIST or A.5.15 in ISO 27001?"*

---

## 12. Tooling implications

The design implies a small toolchain. None of these tools requires changes to the Gemara schema beyond the optional EC profile.

1. **`govops lint`** — Validates that every `action`/`resource` resolves to a lexicon term, every `group` is defined, every `engine-binding` is well-formed, and every TIGER control references at least one capability.
2. **`govops coverage`** — Emits the GovOps coverage and risk metrics over the catalog plus mapping documents.
3. **`govops drift`** — Compares the catalog against deployed policy artifacts (per engine) and flags actions present in one but not the other.
4. **`govops prove`** — Drives an engine-specific analyzer to produce proof artifacts for each provable claim, signs them, and updates the corresponding `#EvaluationLog`.
5. **`govops tiger`** — Computes the TIGER score from current evaluation logs and writes `tiger/tiger-score.yaml` and `tiger/tiger-evidence.json`.
6. **IGA exporter** — Translates the catalog into a CSV / SCIM / OSCAL profile suitable for ingest by an IGA platform's entitlement catalog, so existing access reviews target authoritative capability definitions instead of opaque permission strings.

The toolchain is engine-pluggable: each `govops prove` plug-in calls a specific analyzer (vendor-provided or open source) for the engines used in the deployment.

---

## 13. Adoption / migration path

**Phase 0 — Convention-only profile (today).** Adopters can model enterprise capabilities right now using the stable `#CapabilityCatalog` and the front-matter convention in §6.2(A). All Gemara tooling validates the catalog as is. TIGER pillar catalogs are valid `#ControlCatalog`s today.

**Phase 1 — GovOps overlay package.** The GovOps WG publishes `#EnterpriseCapability` and `#ContextPredicate` as a CUE module that imports the Gemara package. Adopters opt in by importing the overlay. Reference TIGER catalogs and reference mapping documents to NIST/ISO/SOC 2 are published.

**Phase 2 — Upstream into Gemara.** Once stable in practice, propose the schema extension via a Gemara ADR (in the spirit of ADR-0019) so `#EnterpriseCapability` becomes a native artifact.

**Phase 3 — Engine integrations.** Land per-engine `govops prove` plug-ins and ship reference catalogs for common services (S3, GitHub, Kubernetes, GitLab, Jira), the way MITRE ATT&CK provides reusable vector content.

**Phase 4 — Continuous TIGER.** Wire the toolchain into CI/CD so each policy deploy triggers re-evaluation of relevant claims and updates the TIGER score automatically.

---

## 14. Open questions

1. **Granularity of `resource`.** Is `resource` always a *type* (e.g., `Invoice`), or can it sometimes be a *resource pattern* (e.g., `Invoice in tenant=X`)? This proposal treats patterns as `context-predicates`, but graph-style engines admit hierarchies natively — should the profile model them?
2. **Context predicate normalization.** Storing predicate `expression` as a string is simple but opaque. A normalized AST (CEL-like, engine-independent) would enable cross-engine analysis but adds scope.
3. **Identity of the principal.** The profile records `principal-types`, not identities. Is that sufficient for the GovOps metrics, or do we need a "principal capability" inverse view?
4. **Inheritance and roles.** Should the profile express role to capability mappings, or is that the IGA layer's concern? Current proposal: out of scope for this catalog.
5. **Versioning of capabilities.** When an enterprise renames or splits a capability, what is the recommended `replaced-by` pattern? `#Control` already has one — should `#EnterpriseCapability` mirror it?
6. **TIGER weighting.** Default pillar weights and per-control risk weights need community calibration. A weight registry maintained by the WG seems plausible.
7. **Public reference catalogs.** Who curates "the" reference catalog for, say, GitHub or Kubernetes capabilities? A community process modeled on Vector / Threat catalogs is plausible but needs sponsorship.
8. **AuthZEN integration depth.** The design uses PARC as an abstraction; should the WG also publish a guidance document showing how an AuthZEN-conformant evaluate response feeds the `Evidence` pipeline?

---

## 15. Summary

Gemara already provides most of what is needed to govern enterprise capabilities: a stable `#CapabilityCatalog`, a stable `#ControlCatalog` and `#AssessmentRequirement`, mapping primitives that include `Capability` as an `EntryType`, a `#Lexicon`, and the layered Threat/Control/Policy/Log machinery. The remaining work is:

- **A PARC-shaped Enterprise Capability Profile** of `#Capability`, engine-neutral by construction.
- **A repository convention** that pairs the capability catalog with five TIGER pillar `#ControlCatalog`s.
- **A discipline of provable claims** alongside conventional configuration assertions, recorded in `#EvaluationLog` evidence entries.
- **A TIGER metric** that measures the fraction of declared requirements that are demonstrably true of the deployed authorization surface.

Together they make "GovOps" a measurable, vendor-neutral practice rather than a slogan: the *thing being governed* (the PARC capability surface) is catalogued, the *requirements over it* are organized into TIGER pillars, and the *assurance* that those requirements hold is either runtime-observed or symbolically proved — with the proof artifact preserved alongside the catalog as a first-class governance asset.

---

## 16. References

- Gemara — `capabilitycatalog.cue` (stable), `controlcatalog.cue` (stable), `mapping_inline.cue`, `mappingdocument.cue`, `lexicon.cue`, `entities.cue`, `threatcatalog.cue`, `evaluationlog.cue`.
- Gemara ADRs — 0017 (Base Catalog Type), 0018 (Promote Nested Concepts to Catalogs), 0019 (Promote Capabilities), 0020 (Groups), 0021 (Lexicon).
- OpenSSF GovOps WG proposal — `orbit/openssf-wg-proposal.md` (this repository).
- OpenID AuthZEN — Authorization API specification using the PARC request shape.
- NIST SP 800-53 r5, ISO/IEC 27001, SOC 2 — example mapping targets for `#MappingDocument`.

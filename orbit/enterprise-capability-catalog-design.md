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

1. **Capabilities are action–resource pairs; PARC is the request envelope.** The catalog inventories **(action, resource)**. At evaluation time, a PDP receives a **PARC-shaped request** (PARC Principal — the requester — plus action, resource instance, and context, often carrying **signed JWT evidence** in Context for policy to consume). Engine neutrality comes from the fact that every PDP class can consume PARC-shaped requests while the catalog stays a domain-centric inventory of discrete **(action, resource)** entries — separate from person-centric **entitlement** matrices maintained in IAM or IGA systems.
2. **A GovOps repository of Gemara artifacts.** A single `#CapabilityCatalog` (`GovOps-AC`) inventories the enterprise's authorization surface as those **(action, resource)** capabilities, and a small family of `#ControlCatalog` artifacts — one per **TIGER** pillar (Transparency, **Identity**, Governance, Events, Resilience) — express the requirements those capabilities must satisfy. Mapping documents tie the catalog to external compliance frameworks.
3. **Provable operational claims, not just policy assertions.** Where today's GRC says *"the system MUST require MFA"*, GovOps elevates the standard to *"there exists no satisfiable authorization path where this capability succeeds without MFA, given the deployed policy."* That shift — from *attested presence* to *demonstrated impossibility* — is what the **TIGER metric** measures.

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

**Capability (this document).** A **capability** is an **(action, resource)** pair: a discrete, nameable operation on a resource type in a domain. It is **not** a principal, not a role, and not a context tuple. Risk and governance attach to the pair itself; **Context** (including JWT-derived claims) supplies evidence at evaluation time, not the capability's identity.

**PARC (AuthZEN).** The OpenID AuthZEN working group standardizes **PARC** — Principal, Action, Resource, Context — as the **payload of an authorization request** to a PDP. A conforming PDP accepts a PARC-shaped request and returns Permit / Deny plus optional obligations and reasons. PARC is intentionally orthogonal to the policy store strategy:

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

### 2.4 The conceptual leap: from policy assertions to provable claims

GRC frameworks today express requirements as **assertions about configuration**:

> "Sensitive transactions MUST require multi-factor authentication."

Compliance is then attested by an auditor checking that an MFA toggle is on. That tells us the *control is present*; it does not tell us the *control is sufficient*.

GovOps + Gemara can elevate the standard. With a **(action, resource)** capability catalog and a deployed policy artifact, the same requirement can be expressed as a **claim about the deployed authorization surface**:

> "There exists no satisfiable authorization state in which capability `payments:transfer:bank-account` (the pair **transfer** on **BankAccount**) may be permitted and `context.acr` is not `urn:mfa`."

That claim is stated over **requests** (PARC-shaped evaluations for that action and resource); the **capability** in the catalog remains only **transfer / BankAccount**.

That claim is checkable. Modern policy decision points expose symbolic analyzers (SAT/SMT-based) that can produce a decision proof: either an UNSAT certificate (the requirement holds for every reachable state) or a satisfying assignment (a counterexample showing how the requirement could be violated). The same proof shape works whether the engine is Cedar, Rego (with an SMT backend), or a graph engine reduced to a constraint problem.

The TIGER metric (§9) measures the fraction of declared requirements for which such a proof exists, weighted by capability risk. That is materially stronger than checkbox compliance.

---

## 3. Definitions

| Term | Meaning |
|---|---|
| **Capability** | An **(action, resource)** pair: a discrete, nameable operation on a resource type. *What* can be done on *what* is catalogued here, independent of *who* is asking in any given request. A capability is **not** a principal and **not** a context tuple. |
| **Principal** | The actor supplied in a **PARC authorization request** when the PDP is asked whether an operation is permitted. Policy may use principal attributes; the principal does not define the capability. |
| **Action** | The verb half of a capability (e.g., `read`, `create`, `delete`, `transfer`, `assume`). |
| **Resource** | The noun half of a capability: the resource **type** the action applies to (e.g., `Invoice`, `BankAccount`, `Repository`, `PolicyDocument`). The catalog records the *type*; runtime requests identify specific instances. |
| **Context** | Attributes bundled with a **PARC request** (time, IP, approval counts, device posture, etc.), commonly including **signed JWTs** ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)) or similar tokens whose claims the PDP treats as **evidence**. Context answers *whether* a capability may be exercised in this invocation; it is not part of the capability identity. |
| **PARC** | Principal, Action, Resource, Context — the OpenID AuthZEN **authorization request** shape. Used at the PDP boundary; **not** synonymous with "capability." |
| **Enterprise capability** | A Gemara catalog entry whose **identity** is an **(action, resource)** pair (plus `id`, `title`, `description`, `group` per `#Capability`). Optional fields may document risk, sensitivity, or **expected** policy preconditions; those extensions do not redefine the capability. |
| **TIGER** | Five governance pillars: **T**ransparency, **I**dentity (management proficiency for human, software, and organizational principals), **G**overnance, **E**vents, **R**esilience. Also the name of the derived metric defined in §9. |
| **Identity proficiency score** | An integer **1–5** rating of how well the enterprise manages a class of **principal** (human, software, or organizational) that appears in **PARC** authorization requests. Recorded separately from pass/fail control satisfaction; see §7.2 and §9.1. |
| **Provable claim** | A requirement expressed in a form checkable by a policy analyzer, such that the deployed policy either admits a proof of the claim (e.g., UNSAT for a counterexample formula) or yields a counterexample. |

---

## 4. Design goals and non-goals

### Goals

1. **Reuse, do not replace.** Build on Gemara's existing `#CapabilityCatalog`, `#ControlCatalog`, and mapping primitives. No fork.
2. **Additive schema only.** Any extension must be expressible as a CUE refinement that still validates as a `#Capability`.
3. **Engine-neutral capabilities and PARC-shaped requests.** Capability rows in the catalog are **(action, resource)** only. Interoperability with PDPs uses **PARC** as the standard **request** envelope at evaluation time. The design MUST NOT privilege any specific policy engine, policy store strategy, or authorization API.
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
  GovOps-Identity.yaml               # #ControlCatalog (I — identity management proficiency, 1–5)
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
    identity-proficiency.yaml        # latest 1–5 scores: human, software, organizational
```

Mapping each top-level item to a Gemara artifact type:

| Path | Gemara artifact | Purpose |
|---|---|---|
| `lexicon.yaml` | `#Lexicon` | Canonical action verbs and resource type names. |
| `metadata.yaml` | shared `#Metadata` includes | Author, version, lexicon reference, applicability groups. |
| `GovOps-AC.yaml` | `#CapabilityCatalog` of `#EnterpriseCapability` | Enterprise authorization surface. |
| `GovOps-{T,I,G,E,R}.yaml` | `#ControlCatalog` | Pillar-scoped requirements; **Identity** (`GovOps-Identity.yaml`) additionally defines 1–5 proficiency for human, software, and organizational principals. |
| `tiger/identity-proficiency.yaml` | convention | Latest **1–5** identity class scores and normalized Identity pillar input to TIGER. |
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
    // Does NOT define the capability; actual enforcement is in deployed policy.
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

// EngineBinding correlates this capability with engine-specific symbols.
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
          containing the PARC Principal identifier, action, resource instance identifier,
          deciding policy id, and the context attributes the decision relied on.
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

The **Identity** pillar measures **how proficiently the enterprise manages identities** that appear as **PARC Principals** in authorization requests — across three classes:

| Class | What is managed | Examples of mature practice (illustrative, not exhaustive) |
|---|---|---|
| **Human** | Workforce and interactive users | Identity Governance and Administration (IGA), **Agama**-based (or equivalent) identity orchestration, widely adopted multi-factor authentication (MFA), Privileged Access Management (PAM) |
| **Software** | Services, agents, and workloads | [SPIFFE](https://spiffe.io/) identities for cloud and platform agents, asymmetric authentication (mTLS, signed service credentials), short-lived workload credentials, secrets rotation |
| **Organizational** | Third parties, business units, and federated org boundaries | Supplier/customer federation with contract-scoped trust, business-unit identity namespaces, partner onboarding with lifecycle and offboarding, org-level attestations in **Context** |

This pillar is **orthogonal to the capability catalog**: it does not redefine **(action, resource)** entries in `GovOps-AC`. Instead it scores whether the **identity plane** that supplies **Principals** and **Context** (including **JWT** claims) is fit for the authorization surface the catalog describes. Weak identity management undermines every capability regardless of policy quality.

#### Proficiency scale (1–5)

Each class receives an integer score from **1** (ad hoc / immature) to **5** (optimized, measured, continuously improved). Scores are **authored or assessed** per reporting period and recorded in `tiger/identity-proficiency.yaml` (and referenced from `#EvaluationLog` evidence). The **Identity pillar score** used in aggregate TIGER (§9) is the **mean of the three class scores, normalized to 0–1** (`mean / 5`).

| Score | Label | Human (examples) | Software (examples) | Organizational (examples) |
|:---:|---|---|---|---|
| **1** | Ad hoc | Manual account provisioning; no central IGA; MFA rare | Shared service accounts; long-lived passwords in config | Informal partner access; no BU separation |
| **2** | Basic | Partial IGA for some apps; MFA on VPN only | Some service accounts in vault; inconsistent naming | Ad hoc federation for largest partners |
| **3** | Defined | IGA with periodic access reviews; MFA on critical apps | SPIFFE or equivalent in some clusters; mTLS on key APIs | Documented third-party onboarding; BU labels in directory |
| **4** | Managed | IGA + orchestration (e.g., Agama-style flows); MFA widely adopted; PAM for admins | SPIFFE widely used; asymmetric auth default for east-west | Federated org identities with lifecycle; BU-scoped policies |
| **5** | Optimized | Continuous access review; step-up and risk-based auth integrated with PDP Context | Automated rotation; zero long-lived secrets; SVIDs everywhere applicable | Measured partner risk tiers; automated deprovisioning; org claims in JWT |

```yaml
title: GovOps Identity Catalog
metadata:
  id: cat.govops.identity
  type: ControlCatalog
  gemara-version: "0.x"
  description: >
    Assess enterprise proficiency (1–5) managing human, software, and organizational
    identities that supply PARC Principals and Context for authorization.
  author: { id: govops-wg, name: GovOps WG, type: Software Assisted }
groups:
  - id: g.id-human
    title: Human identity
    description: Workforce and interactive user identity management.
  - id: g.id-software
    title: Software identity
    description: Service, agent, and workload identity management.
  - id: g.id-org
    title: Organizational identity
    description: Third-party, business-unit, and federated organizational identity.
controls:
  - id: GOVOPS-ID-HUMAN
    title: Human identity management proficiency
    objective: >
      The enterprise can assign, review, and revoke human access with assurance
      appropriate to the authorization surface catalogued in GovOps-AC.
    group: g.id-human
    state: Active
    assessment-requirements:
      - id: GOVOPS-ID-HUMAN.01
        text: >
          The enterprise MUST maintain a documented 1–5 proficiency score for human
          identity management, evaluated against capabilities such as IGA coverage,
          identity orchestration (e.g., Agama-based or equivalent flows), MFA adoption,
          and PAM for privileged users. Score MUST be recorded each reporting period.
        applicability: [enterprise]
        state: Active
      - id: GOVOPS-ID-HUMAN.02
        text: >
          Human identity scores below 3 MUST trigger a documented remediation plan
          linked to high-risk (action, resource) capabilities in GovOps-AC.
        applicability: [enterprise]
        state: Active

  - id: GOVOPS-ID-SOFTWARE
    title: Software identity management proficiency
    objective: >
      Workloads and agents present cryptographically strong, attributable identities
      to PDPs (e.g., via SPIFFE, asymmetric authentication).
    group: g.id-software
    state: Active
    assessment-requirements:
      - id: GOVOPS-ID-SOFTWARE.01
        text: >
          The enterprise MUST maintain a documented 1–5 proficiency score for software
          identity management, evaluated against practices such as SPIFFE (or equivalent)
          workload identities, asymmetric authentication for service-to-service calls,
          elimination of long-lived shared secrets, and alignment between service
          identity and entries in GovOps-AC engine-bindings.
        applicability: [enterprise]
        state: Active

  - id: GOVOPS-ID-ORG
    title: Organizational identity management proficiency
    objective: >
      Third parties and business units are represented with lifecycle-managed,
      policy-relevant organizational identity in authorization Context.
    group: g.id-org
    state: Active
    assessment-requirements:
      - id: GOVOPS-ID-ORG.01
        text: >
          The enterprise MUST maintain a documented 1–5 proficiency score for
          organizational identity (third parties, business units, federated orgs),
          evaluated against federation hygiene, contractual scope, and whether
          organizational attributes are available to policy via PARC Context.
        applicability: [enterprise]
        state: Active
```

Example proficiency artifact (not a Gemara catalog type today; convention for tooling):

```yaml
# tiger/identity-proficiency.yaml
metadata:
  id: identity-proficiency.acme.2026-05-13
  reporting-period: "2026-Q2"
  assessed-by: acme-security-architecture
scores:
  human: 4
  software: 3
  organizational: 3
  pillar-mean: 3.33
  pillar-normalized: 0.67   # mean / 5 — feeds TIGER aggregate (§9.2)
rationale:
  human: >
    Enterprise IGA covers 90% of workforce apps; Agama orchestration for customer
  software: >
    SPIFFE in Kubernetes production; legacy VMs still use shared keys (remediation tracked).
  organizational: >
    Top-20 suppliers federated; BU claims in internal JWT; partner tiering in progress.
```

High-risk **(action, resource)** capabilities in `GovOps-AC` still require **PARC Context** evidence (e.g., `context.acr`, JWT claims) in **deployed policy** — that is enforced by controls in other pillars (Transparency, Governance) and by policy analysis in §8. The Identity pillar answers a prior question: *are the Principals and organizational context trustworthy enough to support those policies?*

### 7.3 `GovOps-Governance.yaml`

Governance governs **authority over governance artifacts** — separation of duties, dual control, time-bounded overrides, change approval.

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
    objective: No single party can unilaterally modify governance policy.
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

Both kinds of requirement live in the TIGER pillar catalogs. The GovOps repository convention is that the *assessment-requirement text* is engine-neutral (refers to catalog **(action, resource)** capability ids, **PARC Context** in authorization requests, and deployed policy), while the *evidence and proofs* in `evidence/` and `proofs/` are produced by whatever PDP is deployed.

### 8.1 Engine-neutral statement of a provable claim

A provable claim names: a capability id (an **(action, resource)** pair), a property to prove (or refute) about **PARC-shaped authorization requests** or policy closure for that pair, and an analyzer attestation. The same statement is meaningful for any PDP class:

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

TIGER is the **delta between declared governance requirements and provable operational reality** across five pillars: Transparency, **Identity**, Governance, Events, Resilience.

### 9.1 Per-pillar score

**Transparency, Governance, Events, Resilience** — pass/fail over `#AssessmentRequirement` entries:

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

**Identity** — proficiency on a **1–5** scale (§7.2), then normalized for TIGER:

```text
score(Identity) = (score_human + score_software + score_org) / (3 * 5)
```

Where each of `score_human`, `score_software`, and `score_org` is an integer from **1** to **5** recorded in `tiger/identity-proficiency.yaml`. The denominator **15** is the maximum achievable sum (three classes × 5). Example: human=4, software=3, organizational=3 → `score(Identity) = 10/15 ≈ 0.67`.

Pass/fail requirements in `GovOps-Identity.yaml` (e.g., that scores are documented, remediation plans exist when human score is below 3) use the same `status(r)` formula as other pillars and may be tracked separately in `tiger-evidence.json`.

### 9.2 Aggregate TIGER

```text
TIGER = w_T * score(T) + w_I * score(I) + w_G * score(G) + w_E * score(E) + w_R * score(R)
```

Default pillar weights (illustrative; organizations may re-weight): `T=20, I=25, G=20, E=15, R=20`. The **Identity** pillar is weighted slightly higher by default because weak human, software, or organizational identity management undermines authorization policy for the entire **(action, resource)** surface; other organizations will prefer different defaults.

### 9.3 What TIGER measures and what it does not

TIGER measures the fraction of *declared* requirements that are *currently satisfied* by *deployed* policy and runtime evidence. It is therefore:

- **Forward-looking** when the catalog grows: adding a new capability without controls reduces TIGER.
- **Drift-sensitive** when policy changes: a re-deploy that invalidates an UNSAT proof reduces TIGER until the proof is regenerated.
- **Engine-neutral**: it does not assume any particular PDP, so a heterogeneous environment composes naturally (per-engine sub-scores, weighted by capability assignment to engines).

TIGER does *not* measure:

- Adversarial robustness against unmodeled attacks.
- Quality of the catalog itself (an under-specified catalog can yield a high TIGER by trivial controls).
- Detailed HR processes beyond what is needed to score **human** identity proficiency (1–5); the Identity pillar scores management maturity, not every HR policy.

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
      description: Coarse risk classification used by GovOps metrics.
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

  - id: governance:policy-write:governance-policy
    title: Modify governance policy
    description: Edit any policy in the GovOps repository.
    group: g.governance
    action: policy-write
    resource: GovernancePolicy
    documentation:
      typical-requester-classes: [interactive-subject]
      expected-context:
        - id: c.approvals
          text: Two distinct approvers required in PARC Context
          expression: "size(context.approvals) >= 2 && distinct(context.approvals)"
          expression-language: natural-language
    risk-tier: critical
```

### 10.2 Identity proficiency for Acme (illustrative)

Acme records quarterly identity proficiency alongside the capability catalog. The **transfer** capability (`payments:transfer:bank-account`) depends on trustworthy **human** principals (MFA in JWT **Context**) and **software** principals (payment processors using workload identity):

```yaml
# tiger/identity-proficiency.yaml (excerpt — see §7.2 for full shape)
metadata:
  id: identity-proficiency.acme.2026-05-13
  reporting-period: "2026-Q2"
scores:
  human: 4
  software: 3
  organizational: 3
  pillar-normalized: 0.67
rationale:
  human: IGA + Agama orchestration for customer flows; MFA enterprise-wide; PAM for ops admins.
  software: SPIFFE in K8s payments namespace; legacy batch jobs still on shared keys (remediation Q3).
  organizational: Federated B2B partners; BU claim in internal tokens; supplier tiering pilot.
```

### 10.3 An evaluation log recording policy proof (Transparency / provable claims)

Separate from identity proficiency, Acme evaluates whether deployed policy enforces MFA **Context** on the transfer capability:

```yaml
metadata:
  id: log.acme.ec.policy.2026-05-13
  type: EvaluationLog
  gemara-version: "0.x"
  description: Policy analysis for payments:transfer:bank-account (Transparency pillar).
target:
  id: prod.payments.v1.4.2
  name: Payments service (production)
  type: Software
  environment: production
result: Pass
# Illustrative:
#   Proof: UNSAT for Permit(transfer, BankAccount) AND context.acr != urn:mfa
```

### 10.4 The TIGER score that results

```yaml
metadata:
  id: tiger.acme.2026-05-13
  generated-at: "2026-05-13T17:30:00Z"
  catalog-version: "ec.acme:2026.1"
score:
  T: 0.94
  I: 0.67
  G: 0.95
  E: 0.78
  R: 0.88
  aggregate: 0.84
identity-proficiency:
  human: 4
  software: 3
  organizational: 3
notes: |
  Identity pillar (I=0.67) driven by identity-proficiency.yaml mean 3.33/5.
  Software identity remediation (shared keys on batch) is tracked against GOVOPS-ID-SOFTWARE.01.
  Policy proof for transfer MFA Context satisfied (Transparency); catalog capability unchanged.
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
  description: Map GOVOPS-ID identity proficiency controls and EC entries to NIST 800-53r5.
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
    source: GOVOPS-ID-HUMAN
    relationship: implements
    targets:
      - entry-id: AC-2
        rationale: "Account management and IGA proficiency for human principals."
        confidence-level: High
      - entry-id: IA-2(1)
        rationale: "MFA adoption scored in human identity proficiency."
  - id: m2
    source: GOVOPS-ID-SOFTWARE
    relationship: implements
    targets:
      - entry-id: IA-5
        rationale: "Authenticator management for software/workload identities (SPIFFE, asymmetric auth)."
  - id: m3
    source: GOVOPS-ID-ORG
    relationship: supports
    targets:
      - entry-id: AC-20
        rationale: "Third-party and organizational identity federation proficiency."
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

1. **Granularity of `resource`.** Is `resource` always a *type* (e.g., `Invoice`), or can it sometimes be a *resource pattern* (e.g., `Invoice in tenant=X`)? This proposal treats patterns as **PARC Context** or policy constraints, but graph-style engines admit hierarchies natively — should the profile model them?
2. **Context predicate normalization.** Storing predicate `expression` as a string is simple but opaque. A normalized AST (CEL-like, engine-independent) would enable cross-engine analysis but adds scope.
3. **Documented requester classes.** Optional `documented-requester-classes` can drift from deployed policy. Should linters treat them as non-authoritative hints only, or require a `#MappingDocument` when they diverge from policy extracts?
4. **Inheritance and roles.** Should the profile express role to capability mappings, or is that the IGA layer's concern? Current proposal: out of scope for this catalog.
5. **Versioning of capabilities.** When an enterprise renames or splits a capability, what is the recommended `replaced-by` pattern? `#Control` already has one — should `#EnterpriseCapability` mirror it?
6. **TIGER weighting.** Default pillar weights and per-control risk weights need community calibration. A weight registry maintained by the WG seems plausible.
7. **Identity proficiency rubric.** Should the 1–5 scale be standardized in a separate Gemara artifact or remain a GovOps convention in `identity-proficiency.yaml`? Community calibration of level descriptors per industry sector is an open question.
8. **Public reference catalogs.** Who curates "the" reference catalog for, say, GitHub or Kubernetes capabilities? A community process modeled on Vector / Threat catalogs is plausible but needs sponsorship.
9. **AuthZEN integration depth.** The design uses PARC as an abstraction; should the WG also publish a guidance document showing how an AuthZEN-conformant evaluate response feeds the `Evidence` pipeline?

---

## 15. Summary

Gemara already provides most of what is needed to govern enterprise capabilities: a stable `#CapabilityCatalog`, a stable `#ControlCatalog` and `#AssessmentRequirement`, mapping primitives that include `Capability` as an `EntryType`, a `#Lexicon`, and the layered Threat/Control/Policy/Log machinery. The remaining work is:

- **An Enterprise Capability Profile** of `#Capability` whose **identity** is **(action, resource)** — engine-neutral, with **PARC** reserved for **authorization requests** at the PDP boundary (OpenID AuthZEN), not conflated with the capability tuple. **Context** commonly carries **signed JWTs** as evidence for policy evaluation ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)).
- **A repository convention** that pairs the capability catalog with five TIGER pillar `#ControlCatalog`s, including **`GovOps-Identity.yaml`** with **1–5 proficiency** scoring for human, software, and organizational identity management.
- **A discipline of provable claims** alongside conventional configuration assertions, recorded in `#EvaluationLog` evidence entries.
- **A TIGER metric** that measures the fraction of declared requirements that are demonstrably true of the deployed authorization surface.

Together they make "GovOps" a measurable, vendor-neutral practice rather than a slogan: the *thing being governed* is the inventory of **(action, resource)** capabilities; the *requirements over them* are organized into TIGER pillars; and the *assurance* that those requirements hold is either runtime-observed or symbolically proved over **PARC-shaped requests** and deployed policy — with the proof artifact preserved alongside the catalog as a first-class governance asset.

---

## 16. References

- Gemara — `capabilitycatalog.cue` (stable), `controlcatalog.cue` (stable), `mapping_inline.cue`, `mappingdocument.cue`, `lexicon.cue`, `entities.cue`, `threatcatalog.cue`, `evaluationlog.cue`.
- Gemara ADRs — 0017 (Base Catalog Type), 0018 (Promote Nested Concepts to Catalogs), 0019 (Promote Capabilities), 0020 (Groups), 0021 (Lexicon).
- OpenSSF GovOps WG proposal — `orbit/openssf-wg-proposal.md` (this repository).
- OpenID AuthZEN — Authorization API specification using the PARC **request** shape.
- RFC 7519 — JSON Web Token (JWT), commonly used as signed evidence carried in **PARC Context**.
- NIST SP 800-53 r5, ISO/IEC 27001, SOC 2 — example mapping targets for `#MappingDocument`.

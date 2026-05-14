# Proposal: GovOps Initiative — New ORBIT Technical Initiative

**Status:** Draft for TSC consideration
**Proposed lifecycle stage:** Sandbox (initial)
**Proposed initiative lead:** Michael Schwartz (Gluu) — co-leads welcome
**TAC sponsor:** TBD (TSC will identify alongside the existing ORBIT TAC sponsor)
**Charter section to amend:** §3 *Technical Initiatives*, *Active Technical Initiatives* table

---

## 1. One-page summary

The **GovOps Initiative** develops interoperable, vendor-neutral Gemara artifacts and tooling for the **continuous, measurable governance of software authorization**. GovOps takes a **capability-based** approach: the unit of governance is the **(action, resource) capability**, not the identity that requests it. The discipline GovOps formalizes is treating *whether a given action on a given resource is allowed under specified context and supporting evidence* as a first-class, machine-readable governance domain.

GovOps is structured as a multi-phase initiative. **Phase 1 is the Authorization Capability Catalog (ACC)** — a Gemara-based way to inventory the authorization surface of an enterprise or open-source project as a finite set of **(action, resource) capabilities**, each carrying the context and evidence predicates that must hold for the action to be allowed. Subsequent phases will extend the same artifact substrate into adjacent governance concerns (see §4.2).

In one sentence: GovOps is to **authorization governance** what OSPS Baseline is to **project security baselines** — a maintainer-friendly catalog plus an enterprise-grade overlay, both expressed as Gemara schema, both consumable by ORBIT tooling.

The Phase 1 (ACC) deliverables are all expressed as existing Gemara artifact types so they require no Gemara schema changes to begin:

1. An **Enterprise Capability Profile** of `#Capability` — a Gemara `#CapabilityCatalog` whose entries are (action, resource) capabilities, each carrying the context and evidence predicates required for the action to be allowed.
2. A small family of **TIGER pillar `#ControlCatalog` templates** — Transparency, Integrity, Governance, Events, Resilience — that express requirements over a capability catalog. (Each pillar names a property of capabilities and their request context, not a property of any requester.)
3. **`#MappingDocument` artifacts** linking ACC content to OSPS Baseline, NIST 800-53, ISO 27001, and SOC 2.

A working design document for Phase 1 already exists in this repository: [`enterprise-capability-catalog-design.md`](./enterprise-capability-catalog-design.md). It is the technical basis for this proposal.

GovOps Phase 1 is engine-neutral by construction. The catalog itself is organized around the **(action, resource) capability** — that is the unit governance acts on. At runtime, decisions over the catalog are *requested* in the **PARC** (Principal, Action, Resource, Context) envelope standardized by OpenID AuthZEN, which every PDP class — policy-language, graph, ABAC, RBAC, hybrid — already speaks. PARC describes how a decision is *asked for*; the catalog describes what is *governed*. The catalog therefore travels across implementations without privileging any one vendor.

---

## 2. Why this belongs in ORBIT

ORBIT's mission per CHARTER.md §1.a is *"to develop and maintain interoperable resources related to the identification and presentation of security-relevant data."* Authorization decisions are among the most security-relevant data an enterprise or open-source project produces.

Today the authorization surface is most often catalogued indirectly — through entitlement rows, role assignments, group memberships, or hard-coded engine-specific policy strings — with no standard structure for *identifying* the surface itself (what actions on what resources can be requested) or *presenting* the conditions under which those actions are allowed. The result is that governance asks the wrong question first ("who can do what?") and only later struggles back to the question that actually scales ("is this action on this resource allowed under this context and evidence?").

GovOps inverts the order. It catalogues the authorization surface as **capabilities** — (action, resource) pairs — and then governs the conditions over those capabilities, independent of who is requesting them. Specifically, GovOps:

- Makes the authorization surface **identifiable** as a finite, addressable set of capability ids.
- Makes it **presentable** as Gemara YAML/JSON validatable against a stable schema.
- Makes the requirements over it **interoperable** by expressing them as Gemara `#ControlCatalog` entries with `#MappingDocument` ties to common compliance frameworks.

The work is naturally cross-cutting across the four existing TIs (see §5 below), satisfies CHARTER §3.c.i interoperability, and produces durable artifacts in the same style as OSPS Baseline and Security Insights.

---

## 3. Mission and scope

### 3.1 Mission

Develop and maintain interoperable, engine-neutral catalog artifacts, templates, and tooling that let an enterprise or open-source project govern its authorization surface as a first-class, machine-readable, continuously measurable domain — using existing Gemara schema and ORBIT-friendly conventions.

The initiative is structured as a multi-phase effort. **Phase 1 (year 1)** delivers the Authorization Capability Catalog. Subsequent phases extend the same substrate into adjacent governance concerns (see §4.2 *Future-phase candidates*); each future phase enters scope only after TSC review, so the initiative grows incrementally and never outruns demonstrated demand.

### 3.2 Phase 1 — In scope (year 1)

- Definition and stewardship of an **Enterprise Capability Profile** (EC Profile) of Gemara's stable `#Capability` — first as a convention-only profile against today's schema, then (subject to Gemara TI agreement) upstreamed as `#EnterpriseCapability`.
- Reference content: a **TIGER pillar template pack** (five `#ControlCatalog` files) and at least one **OSS-project-sized template** suitable for the typical maintainer.
- Reference mapping documents to OSPS Baseline, NIST 800-53, ISO 27001, SOC 2.
- Validation tooling: `govops lint`, `govops coverage`, `govops drift`. These are thin wrappers over Gemara SDK calls.
- Conformance criteria for ACC catalogs (what makes a catalog "ACC-conformant").
- Liaison with adjacent groups (OpenID AuthZEN for PARC, Gemara for schema feedback, OSPS for project-grade templates).

### 3.3 Out of scope (every phase)

- Defining a new policy language, evaluation API, or policy store specification.
- Building production authorization engines, IGA systems, or runtime enforcement.
- Mandating any one PDP, IAM platform, or policy language.
- Modeling individual permission grants or principal identities — those remain in IGA / IAM / runtime systems.
- Normative work that would duplicate or override Gemara, OSPS Baseline, or Security Insights deliverables.

### 3.4 Why this scope is non-overlapping with existing TIs

| Existing TI | What it does | What GovOps Initiative does | Overlap |
|---|---|---|---|
| Gemara | Defines the schema. | Uses the schema; contributes a profile candidate back. | None — GovOps is a *consumer and contributor*, not a competitor. |
| OSPS Baseline | Defines a baseline of project security controls. | Defines a complementary authorization surface inventory and TIGER controls; can be referenced from OSPS where relevant. | None — different control domain. |
| Security Insights Spec | Standardizes machine-readable project security info. | Optionally references SI for project metadata; can be referenced from SI for "this project's authorization surface". | Adjacent, not overlapping. |
| ORBIT Launchpad | Connects tooling maintainers to end users. | Becomes a Launchpad-aligned adoption vehicle for enterprise authorization tooling. | Complementary. |

---

## 4. Deliverables and roadmap

### 4.1 Phase 1 — Authorization Capability Catalog (year 1)

| # | Deliverable | Type | Target month |
|---|---|---|---|
| D1 | EC Profile v0.1 (convention-only over existing `#Capability`) | Gemara YAML + spec | M3 |
| D2 | TIGER pillar template pack v0.1 (five `#ControlCatalog` files) | Gemara YAML | M4 |
| D3 | Reference mapping: ACC ↔ OSPS Baseline | `#MappingDocument` | M5 |
| D4 | Reference mapping: ACC ↔ NIST 800-53r5 | `#MappingDocument` | M6 |
| D5 | OSS-project template (small, maintainer-grade) | Gemara YAML | M6 |
| D6 | `govops lint` and `govops coverage` reference tools | Apache-2.0 code | M7 |
| D7 | Reference mapping: ACC ↔ ISO 27001 / SOC 2 | `#MappingDocument` | M9 |
| D8 | EC Profile v0.2 — proposal to upstream `#EnterpriseCapability` to Gemara as an ADR | ADR draft + CUE | M10 |
| D9 | Conformance criteria document for ACC | Spec | M11 |
| D10 | Phase 1 report and Phase 2 scope proposal | Markdown | M12 |

All Gemara artifacts are CC-BY-4.0 (per ORBIT charter §7.b.iv). All code is Apache-2.0.

### 4.2 Future-phase candidates (year 2+)

Each future phase enters scope only by explicit TSC review per CHARTER §3.c. The list below is illustrative, not committed; it shows the trajectory the initiative anticipates so reviewers can judge fit and longevity.

- **Phase 2 — Provable claims and TIGER metric formalization.** Operationalize the *provable operational claims* discipline outlined in the design document: standardize how a Gemara `#EvaluationLog` carries a `Proof` evidence type, define a per-pillar score function, and ship a reference TIGER score generator. Aligns with GovOps's "make risk measurable and comparable" objective.
- **Phase 3 — Reference adapters for common engines.** Engine-neutral remains the rule, but practitioners need integration glue. Phase 3 ships read-only adapters that emit ACC-conformant catalogs from common authorization estates (cloud IAM, OPA/Rego corpora, OpenFGA stores, Cedar policies, AuthZEN-conformant PDPs). All adapters are optional and treat their input formats as opaque.
- **Phase 4 — Sector and industry overlays.** Reusable ACC overlays for high-leverage sectors (financial services, healthcare, public sector) maintained by domain-aligned sub-teams.
- **Phase 5 — Conformance and assurance program.** A community process for declaring "ACC-conformant" catalogs and for reviewing TIGER score reports; aligned with OSPS Baseline's self-attestation model.

The TSC may add, reorder, or drop any of these. The proposer commits only to the Phase 1 scope in §4.1.

---

## 5. Interoperability with existing Technical Initiatives

CHARTER §3.c requires interoperability with **no less than two** other Technical Initiatives. The GovOps Initiative is interoperable with **all four**, as described below. For each, the section quotes the form of interop the charter recognizes (§3.c.i: "acceptance of outputs from one project as inputs to the core feature set of another"). Phase 1 (ACC) deliverables are the concrete artifacts that drive interop today; later phases extend the same pattern.

### 5.1 Gemara (Lead: Jennifer Power, Red Hat)

**Form of interop:** GovOps *consumes Gemara output as core input.* Every GovOps artifact validates against existing Gemara schemas (`#CapabilityCatalog`, `#ControlCatalog`, `#MappingDocument`, `#Lexicon`, `#EvaluationLog`, `#AuditLog`). GovOps also *produces input* for Gemara: a real-world test corpus and a candidate `#EnterpriseCapability` ADR (mirroring ADR-0019's pattern).

**What Gemara gains:**

- The first major *external* adopter of `#CapabilityCatalog` (ADR-0019) beyond the existing system-capability use, validating the design under a more granular workload.
- Concrete test data exercising `#MultiEntryMapping`, `#MappingDocument` with `Capability` as an `EntryType`, `#Lexicon`, and `applicability-groups`.
- A candidate ADR (`#EnterpriseCapability`) carrying the EC Profile upstream when ready, in the same spirit as ADR-0018 → ADR-0019.
- A reference set of mapping documents to NIST 800-53 / ISO 27001 / SOC 2 that other Gemara users can reuse.

### 5.2 OSPS Baseline (Lead: Ben Cotton, Kusari)

**Form of interop:** GovOps *produces a `#MappingDocument` whose source-reference is OSPS Baseline*, allowing OSPS-Baseline-conformant projects to declare authorization-surface coverage through GovOps capability ids. GovOps *also consumes* OSPS Baseline groupings to align category names where the two overlap.

**What OSPS Baseline gains:**

- A complementary authorization-specific overlay for projects whose security posture is partly defined by *what actions are allowed on what resources and under what conditions* (release minting, branch protection, dependency upgrade approvals, secret access). These are CRA-relevant concerns under-represented in today's baseline.
- An OSS-project-sized GovOps template (D5) sized for the typical maintainer — explicitly designed not to add baseline conformance burden.
- A path for OSPS-Baseline-conformant projects to extend their assertions into authorization without leaving the Gemara family of artifacts.
- A reusable mapping document linking GovOps content back to OSPS Baseline controls where relevant.

### 5.3 Security Insights Specification (Lead: Eddie Knight, Sonatype — TSC Chair)

**Form of interop:** GovOps *produces output consumable by Security Insights*: a project's published `GovOps-AC.yaml` (or equivalent capability catalog) is a structured artifact a Security Insights manifest can reference, declaring "this project's authorization surface lives at `<URL>`." GovOps *consumes* Security Insights data when available (project metadata, ownership, contact) to populate `metadata.author` and `#RACI` fields.

**What Security Insights gains:**

- A new structured, machine-readable category of security-relevant data for SI consumers (LFX Insights, Best Practices Badge, downstream tools) to render and aggregate.
- Continuity with the work Eddie co-authored in Gemara ADR-0019 — GovOps is the natural downstream adoption story for that ADR, broadening the value of both.
- A second-axis lens (authorization, alongside posture) that strengthens the Security Insights value proposition without expanding its spec.

### 5.4 ORBIT Launchpad (Lead: Nicole Bates)

**Form of interop:** GovOps *consumes Launchpad's matchmaking function*: the initiative's enterprise contributors and OSS authz tooling maintainers (PDP and engine projects) connect through Launchpad. GovOps *produces* an adoption-pattern artifact suitable for Launchpad's mutually-beneficial-alignment objective: an enterprise can publish its GovOps catalog as a reusable pattern, and OSS authz tooling can declare GovOps-compatibility.

**What Launchpad gains:**

- A concrete adoption flow that exercises Launchpad's mission: enterprise practitioners donate templates and learnings; OSS authz tooling maintainers gain a standard target their tools can plug into.
- A success-story candidate for Launchpad to feature in 2026, aligned with the SIG's CRA-driven motivation.
- A worked example of ORBIT outputs (Gemara schemas, OSPS Baseline, Security Insights) composing into a higher-level workflow.

---

## 6. Why each TSC voting member should support this

A direct answer to the charter §3.c question: *why is this initiative worth a yes vote?*

- **For the Gemara lead:** Vote yes if you want the schema you stewarded (especially `#CapabilityCatalog` per ADR-0019) exercised at scale by a real adopter that contributes test data, a candidate ADR, and reference mappings back to your repository.
- **For the Security Insights lead:** Vote yes if you want the ADR-0019 work you co-authored to acquire a downstream adoption story, and a new category of structured security data for SI consumers to render.
- **For the OSPS Baseline lead:** Vote yes if you want a maintainer-friendly authorization overlay that complements OSPS Baseline without competing with it, addresses CRA-relevant project-authorization concerns (release minting, secret access, approver SoD), and ships a Baseline ↔ ACC mapping you do not have to write.
- **For the ORBIT Launchpad lead:** Vote yes if you want an enterprise-to-OSS adoption pattern that exercises Launchpad's matchmaking mission, gives Launchpad a 2026 success story, and demonstrates ORBIT outputs composing into a higher-level workflow.

The 2/3 threshold per CHARTER §3.c is met if any three of the four leads vote yes.

---

## 7. Relationship to the proposed GovOps WG

A separate proposal exists to form a top-level **OpenSSF GovOps Working Group** (see [`openssf-wg-proposal.md`](./openssf-wg-proposal.md)) for framework-level work on authorization governance — definitions, operating model, metrics model documents, and reference architectures.

The two efforts share a name and a mission, with a clear separation of concerns:

- **GovOps Initiative (this proposal — ORBIT Technical Initiative):** produces the *artifacts, schema profiles, templates, and tooling* — capability catalogs, TIGER pillar control catalog templates, mapping documents, conformance criteria. Outputs are Gemara YAML and Apache-2.0 code.
- **GovOps WG (separately proposed):** produces *framework, metrics model, and operating-model documents* that reference the GovOps Initiative's artifacts. Outputs are predominantly informative documents.

If the GovOps WG proposal is approved, the GovOps Initiative serves as its artifact-producing partner under ORBIT — exactly the pattern OpenSSF already uses for OSPS Baseline (artifacts under ORBIT) alongside the Best Practices WG (framework). If the WG proposal is paused, deferred, or absorbed, the GovOps Initiative continues independently under ORBIT and can be consumed directly by other governance bodies (CNCF GRC, TAG-Security, IGA platforms, GRC vendors).

The proposer's intent is that this ORBIT initiative is the *primary, durable home* for GovOps work. The WG, if approved, complements it; either way, the artifact stream lives here.

---

## 8. Lead, contributors, and governance

### 8.1 Proposed lead

- **Michael Schwartz** (Gluu) — proposer, co-author of the design document, co-proponent of the GovOps WG. Per CHARTER §3.b, the lead is not a lead on any other ORBIT TI.

### 8.2 Anticipated contributors

This proposal builds on the contributor base of the GovOps WG proposal in [`openssf-wg-proposal.md`](./openssf-wg-proposal.md), several of whom have committed to authorship under the GovOps Initiative specifically. The initial contributor list includes IGA practitioners, GRC platform engineers, identity vendors, and OSS authz tooling maintainers. The full list will be confirmed at initiative kickoff per ORBIT custom.

### 8.3 Lead succession

Per CHARTER §3.b.ii, the lead may nominate a replacement at any time. We will document a deputy within the first three months to ensure continuity.

### 8.4 Meeting cadence

- GovOps Initiative working session: bi-weekly, alternating with GovOps WG meetings (when approved) to allow shared participation without scheduling conflicts.
- TSC reporting: GovOps Initiative lead attends ORBIT TSC routines per CHARTER §3.b.i.

### 8.5 Communication channels

- GitHub: a new `ossf/orbit-govops` repository (or similar) created at TSC approval.
- Slack: existing `#wg-orbit` plus a dedicated `#orbit-govops` channel.
- Public meeting notes and recordings.

### 8.6 Licensing and IP

- Documentation under CC-BY-4.0 (CHARTER §7.b.iv).
- Code under Apache-2.0 (CHARTER §7.b.i).
- Data under CDLA-Permissive 1.0 (CHARTER §7.b.v).
- All inbound contributions DCO-signed (CHARTER §7.b.ii).

---

## 9. Asks of the TSC

The proposer requests that the ORBIT TSC, by 2/3 majority per CHARTER §3.c:

1. **Approve** the **GovOps Initiative** as a new Sandbox-level Technical Initiative within the ORBIT WG, with Phase 1 scope as defined in §3.2 and §4.1 (Authorization Capability Catalog).
2. **Confirm** Michael Schwartz (Gluu) as initiative lead, per CHARTER §3.b.
3. **Adopt** the charter amendment in §10 below, adding the GovOps Initiative to the *Active Technical Initiatives* table.

---

## 10. Proposed charter amendment

Add the following row to CHARTER.md §3 *Active Technical Initiatives* table:

```markdown
| GovOps Initiative | Michael Schwartz (@nynymike) |
```

No other charter sections require amendment. The amendment is ratified by 2/3 TSC majority per CHARTER §3.c.

Per CHARTER §3.c, future-phase additions (§4.2) that materially expand initiative scope beyond Phase 1 will be brought back to the TSC as separate update proposals; this proposal does not pre-authorize them.

---

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Scope creep — multi-phase framing invites unbounded expansion | §3.3 fixes a hard "out of scope" list that applies to *every* phase. §4.2 future phases enter scope only by explicit TSC review (§10). The proposer commits only to Phase 1 in §4.1. |
| Scope creep into policy-engine specification | §3.3 explicitly excludes new policy languages and engines. The PARC abstraction is reused, not invented. |
| Perceived overlap with Gemara | The initiative is a *consumer and contributor*, not a competitor. The §5.1 interop description and the planned Gemara ADR contribution make the relationship explicit. The Gemara TI lead is a designated approver. |
| Perceived overlap with OSPS Baseline | The initiative's control domain (authorization-surface governance) is distinct from OSPS Baseline's project-security-baseline domain. The §5.2 interop is bilateral mapping, not redefinition. |
| Naming overlap with the proposed GovOps WG | §7 sets a clear scope split (artifacts vs. framework) that mirrors OSPS Baseline / Best Practices WG. The two are explicitly designed to compose; neither blocks the other. |
| Maintainer burden on small OSS projects | D5 (OSS-project template) is sized for solo maintainers and explicitly minimizes added conformance burden, in alignment with ORBIT Launchpad goals. |
| TIGER acronym ambiguity | The five-letter TIGER expansion (Transparency, Integrity, Governance, Events, Resilience) is used in this proposal. Final naming is open and can be re-decided at kickoff; the I was specifically chosen as **Integrity** rather than *Identity* to keep the pillars capability-centric — the I governs integrity of the request context and its supporting evidence, not who is making the request. |

---

## 12. References

- ORBIT WG Charter — https://github.com/ossf/wg-orbit/blob/main/CHARTER.md
- ORBIT WG README — https://github.com/ossf/wg-orbit
- OSPS Baseline — https://baseline.openssf.org
- Gemara — https://gemara.openssf.org
- Gemara ADR-0019 *Promote Capabilities to a First-class Catalog* — https://gemara.openssf.org/adrs/0019-promote-capabilities
- Security Insights — http://security-insights.openssf.org/
- GovOps Phase 1 (ACC) technical design — [`enterprise-capability-catalog-design.md`](./enterprise-capability-catalog-design.md) (this repository)
- GovOps WG proposal (separate, top-level WG) — [`openssf-wg-proposal.md`](./openssf-wg-proposal.md) (this repository)
- ORBIT Launchpad SIG — [`orbit-wg-launchpad.md`](./orbit-wg-launchpad.md) (this repository)
- OpenID AuthZEN — PARC authorization request shape

---

## Appendix A. Charter conformance checklist

Mapping this proposal to CHARTER §3.c requirements:

| Charter requirement | Where addressed |
|---|---|
| §3.c — Topically relevant | §2 *Why this belongs in ORBIT* |
| §3.c — Interoperable with no less than two other TIs | §5 — interop with **all four** TIs documented |
| §3.c — Acceptance of outputs as inputs (§3.c.i) | §5.1 (Gemara), §5.3 (SI) — explicit two-way artifact flows |
| §3.c — Provides output accepted as input (§3.c.ii) | §5.1 (test corpus, ADR), §5.2 (mapping doc), §5.3 (SI artifact) |
| §3.c — Proposal includes charter update | §10 — amendment text provided |
| §3.b — Identified lead, not a lead on another ORBIT TI | §8.1 — Michael Schwartz |
| §3.c — 2/3 TSC approval required | §9 — explicit ask |
| §7 — Licensing | §8.6 — CC-BY-4.0 / Apache-2.0 / CDLA-Permissive 1.0 / DCO |

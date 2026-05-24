# Proposal: GovOps Initiative — New ORBIT Technical Initiative

**Status:** Draft for TSC consideration

**Proposed lifecycle stage:** Sandbox (initial)

**Proposed initiative lead:** Michael Schwartz (Gluu) — Rohit Khare (Independent)

**TAC sponsor:** TBD (TSC will identify alongside the existing ORBIT TAC sponsor)

**Charter section to amend:** §3 *Technical Initiatives*, *Active Technical Initiatives* table

**Related:**
- GovOps Phase 1 technical design — [Authorization Capability Catalog Design](./authorization-capability-catalog-design.md)
- GovOps use cases (companion) — [Authorization Capability Catalog Use Cases](./authorization-capability-catalog-use-cases.md)
- OpenID AuthZEN Authorization API - [version 1.0](https://openid.net/specs/authorization-api-1_0.html) which defines PARC (Principal, Action, Resource, Context) **authorization request** shape

---

## 1. One-page summary

The **GovOps Initiative** develops interoperable, vendor-neutral Gemara artifacts and tooling for **governance of the software authorization surface**. GovOps takes a **capability-based** approach: the **unit of governance in the catalog** is the **(action, resource)** pair — the capability — not the identity of the requester. In one sentence: GovOps is to **authorization governance** what OSPS Baseline is to **project security baselines** — a maintainer-friendly catalog plus an enterprise-grade overlay, both expressed as Gemara schema, both consumable by ORBIT tooling.

GovOps is structured as a multi-phase initiative. **Phase 1 is the Authorization Capability Catalog (ACC)** — a Gemara-based inventory of **(action, resource)** capabilities plus tooling to keep the catalog valid and aligned with deployed policy.

Phase 1 deliverables use existing Gemara artifact types (no schema changes required to begin):

1. An **Enterprise Capability Profile** of `#Capability` — `GovOps-AC.yaml` whose entries identify capabilities by **`action` and `resource` only**, with optional risk-tier, sensitivity, and documented context expectations (see design document).
2. A **GovOps repository convention** — `govops/` with `GovOps-AC.yaml`, `#Lexicon`, `mappings/` and `exports/`. (design §5).
3. **`#MappingDocument` artifacts** linking capability ids to OSPS Baseline, NIST 800-53, ISO 27001, and SOC 2.
4. **Reference tooling** — `govops lint`, `govops drift`, and the **IGA exporter** (design §10).

A technical design and five persona-driven use cases exist in this repository: [`authorization-capability-catalog-design.md`](./authorization-capability-catalog-design.md) and [`authorization-capability-catalog-use-cases.md`](./authorization-capability-catalog-use-cases.md).

GovOps Phase 1 is authorization engine-neutral. The catalog describes what is *governed*; at runtime, PDPs evaluate **PARC-shaped requests** (e.g. an OpenID AuthZEN Authorization API request). 

---

## 2. Why this belongs in ORBIT

ORBIT's mission per CHARTER.md §1.a is *"to develop and maintain interoperable resources related to the identification and presentation of security-relevant data."* Authorization decisions are among the most security-relevant data an enterprise or open-source project produces.

Today the authorization surface is catalogued indirectly — entitlements, roles, engine-specific policy strings — with no standard way to **inventory** what **(action, resource)** pairs exist or how they are governed. GovOps catalogues  surface  **capabilities** and keeps them aligned with enforcement via drift detection.

Specifically, GovOps:

- Makes the authorization surface **identifiable** as a finite set of capability ids.
- Makes it **presentable** as Gemara YAML/JSON validatable against stable schema.
- Makes compliance and IGA workflows **interoperable** via `#MappingDocument` links to common frameworks.

The work is cross-cutting across the four existing TIs (§5), satisfies CHARTER §3.c.i interoperability, and follows the OSPS Baseline / Security Insights artifact style.

---

## 3. Mission and scope

### 3.1 Mission

Develop and maintain interoperable, engine-neutral **capability catalog** artifacts, templates, and tooling so enterprises and OSS projects can govern their authorization surface as a first-class, machine-readable domain — using existing Gemara schema.

**Phase 1 (year 1)** delivers the Authorization Capability Catalog. Later phases (§4.2) add optional proofs and engine adapters after TSC review.

### 3.2 Phase 1 — In scope (year 1)

- **Enterprise Capability Profile** (EC Profile) — `action` + `resource` capability identity; convention-only first, then candidate `#EnterpriseCapability` ADR to Gemara.
- **GovOps repository convention** — `GovOps-AC.yaml`, lexicon, mappings, policies (design §5).
- Reference **mapping documents** to OSPS Baseline, NIST 800-53, ISO 27001, SOC 2.
- **OSS-project-sized template** for typical maintainers.
- **`govops lint`** and **`govops drift`** reference tools; **IGA exporter** for entitlement catalogs.
- **Conformance criteria** for ACC catalogs and GovOps repositories.
- Liaison with OpenID AuthZEN (PARC), Gemara (schema), OSPS (templates).

### 3.3 Out of scope (every phase)

- New policy languages, evaluation APIs, or policy store specifications.
- Production PDPs, IGA systems, or runtime enforcement products.
- Mapping any single PDP or policy language.
- Modeling per-request permission grants 
- Duplicating Gemara, OSPS Baseline, or Security Insights normative specs.

### 3.4 Why this scope is non-overlapping with existing TIs

| Existing TI | What it does | What GovOps Initiative does | Overlap |
|---|---|---|---|
| Gemara | Defines the schema. | Consumes schema; contributes EC Profile ADR. | None — consumer/contributor. |
| OSPS Baseline | Project security baseline controls. | Authorization-surface inventory + mappings. | None — different domain. |
| Security Insights Spec | Machine-readable project security info. | Publishable `GovOps-AC.yaml` reference from SI. | Adjacent. |
| ORBIT Launchpad | Tooling matchmaking. | Adoption pattern for authz tooling + catalogs. | Complementary. |

---

## 4. Deliverables and roadmap

### 4.1 Phase 1 — Authorization Capability Catalog (year 1)

| # | Deliverable | Type | Target month |
|---|---|---|---|
| D1 | EC Profile v0.1 (convention-only over existing `#Capability`) | Gemara YAML + spec | M3 |
| D2 | Reference `GovOps-AC` + lexicon template | Gemara YAML | M4 |
| D3 | Reference mapping: ACC ↔ OSPS Baseline | `#MappingDocument` | M5 |
| D4 | Reference mapping: ACC ↔ NIST 800-53r5 | `#MappingDocument` | M6 |
| D5 | OSS-project template (maintainer-grade) | Gemara YAML | M6 |
| D6 | `govops lint` and `govops drift` reference tools | Apache-2.0 code | M7 |
| D7 | Reference mapping: ACC ↔ ISO 27001 / SOC 2 | `#MappingDocument` | M9 |
| D8 | IGA exporter reference implementation | Apache-2.0 code | M9 |
| D9 | EC Profile v0.2 — `#EnterpriseCapability` ADR draft to Gemara | ADR + CUE | M10 |
| D10 | Conformance criteria document for ACC | Spec | M11 |
| D11 | Phase 1 report and Phase 2 scope proposal | Markdown | M12 |

All Gemara artifacts are CC-BY-4.0 (per ORBIT charter §7.b.iv). All code is Apache-2.0.

### 4.2 Future-phase candidates (year 2+)

Each phase requires explicit TSC review per CHARTER §3.c.

The proposer commits only to Phase 1 in §4.1.

---

## 5. Interoperability with existing Technical Initiatives

CHARTER §3.c requires interoperability with **no less than two** other TIs. GovOps interoperates with **all four**.

### 5.1 Gemara (Lead: Jennifer Power, Red Hat)

**Form of interop:** GovOps consumes `#CapabilityCatalog`, `#MappingDocument`, `#Lexicon`. Produces test corpus (Acme scenario, design §8; five use cases) and candidate `#EnterpriseCapability` ADR.

**What Gemara gains:** First major external adopter of `#CapabilityCatalog` (ADR-0019) at enterprise granularity; reference mappings to NIST/ISO/SOC 2.

### 5.2 OSPS Baseline (Lead: Ben Cotton, Kusari)

**Form of interop:** `#MappingDocument` from OSPS Baseline to GovOps capability ids; OSS template sized for maintainers.

**What OSPS Baseline gains:** Authorization-surface overlay for CRA-relevant project controls without leaving Gemara artifacts.

### 5.3 Security Insights Specification (Lead: Eddie Knight, Sonatype — TSC Chair)

**Form of interop:** Published `GovOps-AC.yaml` referenceable from Security Insights manifests.

**What Security Insights gains:** Structured authorization-surface data for SI consumers; downstream story for ADR-0019.

### 5.4 ORBIT Launchpad (Lead: Nicole Bates, Microsoft)

**Form of interop:** Enterprise catalog patterns + OSS authz tooling compatibility via Launchpad matchmaking.

**What Launchpad gains:** Concrete 2026 adoption flow composing ORBIT artifacts.

---

## 6. Why each TSC voting member should support this

- **Gemara lead:** Exercises `#CapabilityCatalog` at scale; test data and ADR contribution back.
- **Security Insights lead:** ADR-0019 adoption story; new SI data category.
- **OSPS Baseline lead:** Complementary authorization overlay + mapping without competing with Baseline.
- **Launchpad lead:** Enterprise-to-OSS pattern and composed ORBIT workflow.

The 2/3 threshold per CHARTER §3.c is met if any three of the four leads vote yes.

---

## 7. Relationship to the proposed GovOps WG

A separate **OpenSSF GovOps Working Group** proposal ([`openssf-wg-proposal.md`](./openssf-wg-proposal.md)) may produce framework and operating-model documents.

- **GovOps Initiative (ORBIT TI):** Gemara artifacts, EC Profile, templates, `govops lint` / `govops drift`, mappings, conformance criteria.
- **GovOps WG (if approved):** Informative framework documents referencing those artifacts.

Same pattern as OSPS Baseline (ORBIT) + Best Practices WG.

---

## 8. Lead, contributors, and governance

### 8.1 Proposed leads

- **Michael Schwartz** (Gluu) and **Rohit Khare** (Independent) — proposers, co-authors of design and use-case documents. 

### 8.2 Anticipated contributors

IGA practitioners, GRC engineers, authorization vendors, OSS authz maintainers. Currently there are 
85 members in the [GovOps Group on Linkedin](https://www.linkedin.com/groups/17478011/). 

### 8.3 Lead succession

Deputy documented within three months per CHARTER §3.b.ii.

### 8.4 Meeting cadence

- Bi-weekly GovOps Initiative session.
- TSC reporting per CHARTER §3.b.i.

### 8.5 Communication channels

- GitHub: `ossf/orbit-govops` (or similar) at TSC approval.
- Slack: `#wg-orbit`, `#orbit-govops`.
- Public notes and recordings.

### 8.6 Licensing and IP

- Documentation: CC-BY-4.0.
- Code: Apache-2.0.
- Data: CDLA-Permissive 1.0.
- Contributions: DCO-signed.

---

## 9. Asks of the TSC

The proposer requests that the ORBIT TSC, by 2/3 majority per CHARTER §3.c:

1. **Approve** the **GovOps Initiative** as a Sandbox Technical Initiative with Phase 1 scope in §3.2 and §4.1.
2. **Confirm** Michael Schwartz (Gluu) and Rohit Khare as initiative leads per CHARTER §3.b.
3. **Adopt** the charter amendment in §10.

---

## 10. Proposed charter amendment

Add to CHARTER.md §3 *Active Technical Initiatives*:

```markdown
| GovOps Initiative | Michael Schwartz (@nynymike), Rohit Khare (@rohitkhare) |
```

Ratified by 2/3 TSC majority. Future phases (§4.2) require separate TSC proposals.

---

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Scope creep | §3.3 out-of-scope list; Phase 1 only in §4.1; later phases need TSC review. |
| Policy-engine specification creep | §3.3 excludes new languages/APIs; PARC is request envelope only. |
| Overlap with Gemara | Consumer/contributor; Gemara lead as approver. |
| Overlap with OSPS Baseline | Bilateral mapping, distinct domain. |
| Overlap with GovOps WG | §7 scope split (artifacts vs. framework). |
| Maintainer burden | D5 OSS template minimizes conformance cost. |

---

## 12. References

- ORBIT WG Charter — https://github.com/ossf/wg-orbit/blob/main/CHARTER.md
- OSPS Baseline — https://baseline.openssf.org
- Gemara — https://gemara.openssf.org
- Gemara ADR-0019 — https://gemara.openssf.org/adrs/0019-promote-capabilities
- Security Insights — http://security-insights.openssf.org/
- [Authorization Capability Catalog: Design](./authorization-capability-catalog-design.md)
- [Authorization Capability Catalog: Use Cases](./authorization-capability-catalog-use-cases.md)
- [OpenID AuthZEN Authorization API 1.0](https://openid.net/specs/authorization-api-1_0.html)

---

## Appendix A. Charter conformance checklist

| Charter requirement | Where addressed |
|---|---|
| §3.c — Topically relevant | §2 |
| §3.c — Interoperable with ≥2 TIs | §5 (all four) |
| §3.c.i — Outputs as inputs | §5.1, §5.3 |
| §3.c.ii — Provides accepted outputs | §5.1, §5.2 |
| Charter update | §10 |
| §3.b — Lead identified | §8.1 |
| §3.c — 2/3 approval | §9 |
| §7 — Licensing | §8.6 |

# SPEC B17 — Agent profile library

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [B13, A5, A1] · downstream: [B18] · adrs: [0013, 0019, 0032, 0022, 0030] · views: [6.8, 6.2, 6.4]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B17 is the curated profile/template library. Agent "templates" layer via the **`CapabilitySet` overlay model** (ADR 0013/0032: add-if-not-there / replace-if-there) with per-Agent overrides; no new template/override mechanism is introduced — the `CapabilitySet` overlay model already covers this. Egress allowlists ride this model via `egressTargets[]` in `CapabilitySet`; L7 egress enforcement is provided by Envoy (A3 install, A6 sandbox) per D-08. Any default egress a profile declares is a **minimum baseline (a floor)**, extensible per agent — never a closed ceiling/allowlist.

The platform models every unit of agent access — MCP servers, A2A peers, RAG stores, egress
targets, skills, OPA policy refs, LLM providers — as `CapabilitySet` CRDs that compose via a
Helm-style overlay (§6.8). Without a curated starting library, every team would hand-author
CapabilitySets from scratch, diverge on naming and policy refs, and re-derive the same common
shapes (a knowledge-base reader, a code-gen agent, a support agent). B17 is that curated
starting library.

B17 ships an **initial set of named `CapabilitySet` profiles** that bundle Canon-approved
capabilities into reusable, layerable building blocks, plus the documented method for deriving
new profiles using the §6.8 overlay semantics. The profiles are the raw material that B18
(recommended compositions) assembles into complete `Agent` CRDs, and that tenant-onboarding
(§6.9, ADR 0037) offers as starter seeds. B17 ships **CapabilitySet definitions and authoring
guidance only** — not `Agent` CRDs, not the capability CRDs themselves, and not new reconciler
logic.

## 2. Scope

### 2.1 In scope
- An initial library of named `CapabilitySet` CRDs: at minimum a **knowledge-base agent**
  profile, a **code-gen agent** profile, a **customer-support agent** profile, and a
  **minimal RAG agent** profile (§6.8, §14.2 B17).
- Each profile references only Canon-approved capability CRDs (`MCPServer`, `A2APeer`,
  `RAGStore`, `EgressTarget`, `Skill`, `llmProviders`, `opaPolicyRefs`) by name; the
  Knowledge-Base profiles reference the `platform-knowledge-base` `RAGStore` (§6.4, ADR 0022).
- Layering metadata so profiles compose predictably under the §6.8 add-if-not-there /
  replace-if-there overlay model (ADR 0032), including the convention of restating full lists.
- Documentation: how to derive a new profile, the overlay/override resolution rules,
  validation expectations, and the recommended layering order.
- Git/GitOps packaging so profiles reconcile through ArgoCD → kopf operator (B13) like any
  capability CRD.

### 2.2 Out of scope
- The capability CRD reconciler and overlay resolver mechanics — **B13** (kopf operator) and
  **ADR 0032** overlay semantics.
- The capability CRDs themselves (`MCPServer`, `RAGStore`, etc. instances) — registered by
  their owning components (A17 MCP services, B4-provisioned RAG backends, etc.).
- Complete `Agent` CRDs that consume these profiles — **B18** (recommended agent compositions).
- The Headlamp capability-inspector unified view — **B5** (cross-cutting Headlamp plugins).
- OPA policy content referenced by `opaPolicyRefs` — **B16** (initial OPA policy library content)
  / **B3** (framework).
- Tenant onboarding coupling — intentionally decoupled per §6.9 / ADR 0037.

## 3. Context & Dependencies

Upstream consumed:
- **B13** (kopf operator) — reconciles `CapabilitySet` (and the referenced capability CRDs)
  into LiteLLM; B17 profiles are reconciled by it. What is consumed: the `CapabilitySet` CRD
  schema and reconcile behavior.
- **A5** (ARK) — defines the `Agent` CRD whose `capabilitySetRefs[]` reference these profiles;
  B17 designs profiles to slot into that field.
- **A1** (LiteLLM) — the gateway into which capabilities resolve; profiles must reference
  only capabilities reachable through LiteLLM.

ADR decisions honored:
- **ADR 0013** — capability CRD model + `CapabilitySet` layering; B17 only uses the field set
  fixed by this ADR and §6.8.
- **ADR 0032** — CapabilitySet overlay resolution semantics; B17 authors profiles to the
  add-if-not-there / replace-if-there model; finer mechanics are deferred to ADR 0032 design.
- **ADR 0019** — Deep Agents is the opinionated default used by the profile library; profiles
  are SDK-agnostic but documented against `deep-agents`/`langgraph`.
- **ADR 0022** — Knowledge Base is a separate `platform-knowledge-base` `RAGStore` primitive;
  KB profiles include it by reference, never embed it.
- **ADR 0030** — CRD/API versioning; profiles ship under a pinned `CapabilitySet` API version.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B17 authors **instances** of the existing `CapabilitySet` CRD (owner B13); it defines **no new
CRD**. Fields used are exactly those in the interface contract §1.4:
`mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`,
`opaPolicyRefs[]`. Profiles are consumed via `Agent.capabilitySetRefs[]` and `Agent.overrides`
(interface contract §1.2). API version per ADR 0030 (`v1alpha1`/`v1beta1`/`v1` as B13 ships).

Initial profile set (names are deliverable artifacts of this piece, not Canon kinds —
`[PROPOSED — not in source]` as specific identifiers; the *categories* are source-stated in
§6.8 / §14.2 B17):
- `knowledge-base-agent` `[PROPOSED — not in source]` — references `platform-knowledge-base`
  `RAGStore` (Canon) + read-oriented `opaPolicyRefs`.
- `code-gen-agent` `[PROPOSED — not in source]` — references code-oriented `MCPServer`(s) from
  the ADR 0020 initial set (e.g. GitHub) + `egressTargets[]` + `llmProviders[]`.
- `customer-support-agent` `[PROPOSED — not in source]` — mirrors the §6.8 worked example shape
  (support `MCPServer`s, `egressTargets`, `opaPolicyRefs` including rate-limit policy).
- `minimal-rag-agent` `[PROPOSED — not in source]` — a thin RAG-only profile.

### 4.2 APIs / SDK surfaces
N/A — B17 ships declarative CRD artifacts + docs; it exposes no API or SDK surface. (Agents that
consume profiles reach capabilities through the Platform SDK `rag.*`/`memory.*` surfaces owned
by B6 — not part of B17.)

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
B17 emits none directly. When B13 reconciles a B17 profile add/update/delete it emits
`platform.capability.changed` under `platform.capability.*` (§6.8, ADR 0013 update) — that
emission is owned by B13, not B17.

### 4.4 Data schemas / connection-secret contracts
N/A — profiles reference capability CRDs by name only; they hold no data backends and write no
connection secrets. (Backends behind referenced `RAGStore`s use the ADR 0044 connection-secret
contract, owned by B4.)

## 5. OSS-vs-Custom Decision
**Build-new (content/configuration), no new code.** B17 is a curated set of `CapabilitySet`
YAML artifacts plus documentation. It wraps no OSS project directly; it depends on the
`CapabilitySet` CRD (B13) and the overlay model (ADR 0013 / 0032). Rationale: the layering
mechanics already exist; the value B17 adds is curated, Canon-conformant content and the
derivation method.

## 6. Functional Requirements
- REQ-B17-01: The library SHALL provide at least four initial `CapabilitySet` profiles covering
  knowledge-base agent, code-gen agent, customer-support agent, and minimal RAG agent (§14.2 B17).
- REQ-B17-02: Every profile SHALL reference only Canon-approved capability primitives by name
  (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `llmProviders`, `opaPolicyRefs`)
  and SHALL NOT introduce capability fields outside the §1.4 `CapabilitySet` schema.
- REQ-B17-03: Knowledge-base-oriented profiles SHALL include the Knowledge Base by reference to
  the `platform-knowledge-base` `RAGStore` and SHALL NOT embed or duplicate its content (ADR 0022).
- REQ-B17-04: Each profile SHALL declare its capability lists as **full lists** so that under the
  §6.8 replace-if-there overlay a consuming layer's value fully and predictably replaces an
  earlier layer's (no reliance on list concatenation).
- REQ-B17-05: The library SHALL ship documentation describing how to derive a new profile,
  including the layering order, add-if-not-there / replace-if-there resolution, and per-Agent
  `overrides` behavior (§6.8, ADR 0032).
- REQ-B17-06: Profiles SHALL be packaged for GitOps so they reconcile through ArgoCD → B13 kopf
  operator with no manual gateway configuration (§6.8).
- REQ-B17-07: Profiles SHALL pin the `CapabilitySet` API version per ADR 0030 and SHALL document
  the per-component versioning owner (B13) for that CRD.
- REQ-B17-08: The library SHALL be consumable unchanged by B18 such that a `recommended`
  `Agent` CRD can reference a B17 profile via `capabilitySetRefs[]` and resolve deterministically.
- REQ-B17-09: Each profile SHALL document its intended SDK default as `deep-agents` (with
  `langgraph` supported), per ADR 0019, without binding the profile to an SDK in CRD fields.

## 7. Non-Functional Requirements
- Security / policy: profiles MUST reference `opaPolicyRefs` consistent with the RBAC-as-floor /
  OPA-as-restrictor model (ADR 0018); referencing a capability does not by itself grant access —
  enforcement stays at LiteLLM/OPA/Envoy (§6.8).
- Multi-tenancy (§6.9): profiles are namespaced `CapabilitySet`s; cross-namespace reference is
  allowed only when explicitly published with an OPA-checked policy (§6.9). Default namespace-private.
- Observability (§6.5): profile changes are visible via the `platform.capability.changed` event
  (emitted by B13) and the Headlamp capability inspector (B5).
- Scale: the library is small/static; no runtime scaling concern. Growth is by additive profiles.
- Versioning (ADR 0030): additive profile additions are backward-compatible; changing a profile's
  resolved capability set is treated as a meaningful change and documented.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable** — GitOps-packaged `CapabilitySet` YAML.
- Per-product docs (10.5): **Applicable** — profile catalog + derivation guide.
- Runbook (10.7): **N/A — no runtime service; reconcile failures are diagnosed via B13 runbook.**
- Alerts: **N/A — no runtime component emits metrics here.**
- Grafana dashboard (Crossplane XR): **N/A — no per-component metrics surface.**
- Headlamp plugin: **N/A — the capability inspector view is owned by B5.**
- OPA/Rego integration: **Applicable (by reference)** — profiles carry `opaPolicyRefs`; content owned by B16/B3.
- Audit emission (ADR 0034): **N/A — emission happens in B13 on reconcile, not in B17 artifacts.**
- Knative trigger flow: **N/A — no trigger flow originates here.**
- HolmesGPT toolset: **N/A — B17 contributes no toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — Chainsaw for CRD apply/resolve; PyTest for overlay-resolution assertions; Playwright N/A (no UI).
- Tutorials & how-tos: **Applicable** — "derive a new profile" how-to (content in C3 but seeded here).

## 9. Acceptance Criteria
- AC-B17-01 (REQ-B17-01): Applying the library yields ≥4 named `CapabilitySet` resources for the
  four required categories.
- AC-B17-02 (REQ-B17-02): A schema/lint check confirms no profile uses a field outside the §1.4
  `CapabilitySet` schema and every referenced capability name resolves to a Canon kind.
- AC-B17-03 (REQ-B17-03): The knowledge-base profile's `ragStores[]` contains
  `platform-knowledge-base` and no inlined KB content.
- AC-B17-04 (REQ-B17-04): A two-layer overlay test (`baseline` then a B17 profile) resolves to the
  profile's full list for each overlapping field, matching the §6.8 replace semantics exactly.
- AC-B17-05 (REQ-B17-05): The derivation doc exists and a reviewer can follow it to produce a
  valid fifth profile that passes AC-B17-02.
- AC-B17-06 (REQ-B17-06): Profiles reconcile via ArgoCD → B13 with zero manual LiteLLM admin calls.
- AC-B17-07 (REQ-B17-07): Each profile manifest carries an explicit `CapabilitySet` apiVersion.
- AC-B17-08 (REQ-B17-08): A B18 `Agent` CRD referencing a B17 profile in `capabilitySetRefs[]`
  resolves to the expected capability set (verified jointly in B18 tests).
- AC-B17-09 (REQ-B17-09): Each profile's doc states `deep-agents` default / `langgraph` supported
  and the profile CRD contains no `sdk` field.

## 10. Risks & Open Questions
- R1 (med): Profile **names** are not Canon. `[PROPOSED — not in source]` for all four identifiers.
  Reconcile with B18 so both pieces use identical names. Reconciliation note: B17 owns the names;
  B18 consumes them verbatim.
- R2 (med): Recursive includes (a `CapabilitySet` referencing another) are explicitly design-time
  in §6.8 / deferred to ADR 0032. **Open question:** does the library rely on recursive includes
  or only flat profiles + Agent-level layering? Default assumption: flat profiles, no recursion.
- R3 (low): The specific `MCPServer`/`EgressTarget` instances a code-gen or support profile should
  reference depend on the ADR 0020 initial MCP set and A17 registration timing. Open question:
  pin references to A17-shipped names or ship with placeholder refs gated behind their existence.
- R4 (low): Whether per-Agent `overrides` may grant capabilities absent from all referenced sets
  is design-time (§6.8). Profiles assume they may not, pending ADR 0032.

## 11. References
- architecture-overview.md §6.8 Capability registries and approved primitives (~L632–718;
  layering semantics ~L679–714; capability-change event ~L718).
- architecture-overview.md §6.2 Agent runtime architecture (~L247–292; SDK default L284).
- architecture-overview.md §6.4 Knowledge Base as a separate primitive (~L349–368).
- architecture-overview.md §14.2 Workstream B, B17 row (~L1711); B13 row (~L1707); B18 (~L1712).
- ADR 0013 (capability CRD model + CapabilitySet layering), ADR 0032 (overlay semantics),
  ADR 0019 (LangGraph + Deep Agents default), ADR 0022 (Knowledge Base primitive),
  ADR 0018 (RBAC-as-floor / OPA-as-restrictor), ADR 0030 (CRD/API versioning).
- _meta/interface-contract.md §1.2 (`Agent`), §1.4 (`CapabilitySet`), §2 (CloudEvent taxonomy).
- _meta/glossary.md (CapabilitySet, Knowledge Base, Approved capability, platform.capability.changed).

# SPEC B18 — Recommended agent compositions

> kind: COMPONENT · workstream: B · tier: T2
> upstream: [B17, A5, A1, B7] · downstream: [] · adrs: [0013, 0019, 0032, 0022, 0030] · views: [6.2, 6.8, 6.4]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

The audit-adapter interface and `audit_events` schema are **frozen** (D-05): a freeze-gate MUST pass before any component emits audit events. Emission by any B-component is gated on this freeze-gate.

B17 ships a library of layerable `CapabilitySet` profiles, but a profile is not a runnable
agent. Teams still need worked, end-to-end examples showing how to assemble an actual Platform
Agent from those profiles. B18 is that set of reference implementations: **complete `Agent`
CRDs** that reference B17 profiles via `capabilitySetRefs[]`, pick an SDK, bind a sandbox
template and model, and resolve deterministically under the §6.8 overlay model.

B18 is copy-and-modify material. It does not introduce new capability primitives, new CRDs, or
reconciler logic; it demonstrates the intended composition pattern (§6.2 agent runtime, §6.8
layering) so a developer can lift a known-good `Agent` CRD and adapt it rather than authoring
one blind.

## 2. Scope

### 2.1 In scope
- A set of complete, reference `Agent` CRDs (one per common shape: a knowledge-base/chat agent,
  a code-gen agent, a customer-support agent, a minimal-RAG agent) that reference B17 profiles
  (§14.2 B18, §6.2).
- Each composition demonstrates `capabilitySetRefs[]` ordering, per-Agent `overrides`, an `sdk`
  value (`deep-agents` default, `langgraph` supported — ADR 0019), and a `sandboxTemplateRef` /
  `modelRef` binding using only Canon `Agent` fields (interface-contract §1.2).
- Documentation showing the resolved capability view for each composition and how to adapt it.
- GitOps packaging so the examples can be applied/reconciled (ARK) for validation.

### 2.2 Out of scope
- The `CapabilitySet` profiles themselves — **B17** (agent profile library).
- The agent SDK harness images / runtime — **B7** (LangGraph + Deep Agents).
- The `Agent` CRD definition and ARK reconcile behavior — **A5** (ARK).
- Overlay resolution mechanics — **ADR 0032** / **B13**.
- The Headlamp resolved-capability view — **B5**.
- OPA policy content referenced via profiles — **B16** / **B3**.

## 3. Context & Dependencies

Upstream consumed:
- **B17** — the `CapabilitySet` profiles referenced by `capabilitySetRefs[]`; names consumed verbatim.
- **A5** (ARK) — defines/reconciles the `Agent` CRD these compositions are instances of.
- **A1** (LiteLLM) — gateway through which the composed capabilities resolve at runtime.
- **B7** — supplies the supported `sdk` values (`langgraph`, `deep-agents`) the compositions set.

ADR decisions honored:
- **ADR 0013** — capability CRD model + CapabilitySet layering; compositions use only the §1.2/§1.4 field sets.
- **ADR 0019** — `deep-agents` is the opinionated default; compositions default to it, `langgraph` documented.
- **ADR 0032** — overlay resolution; compositions rely on add-if-not-there / replace-if-there only.
- **ADR 0022** — Knowledge Base composition references `platform-knowledge-base` by reference (via the B17 KB profile).
- **ADR 0030** — `Agent` CRD API version pinned per the ARK-owned versioning lifecycle.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B18 authors **instances** of the existing `Agent` CRD (owner A5); no new CRD. Fields used are
exactly interface-contract §1.2: `capabilitySetRefs[]`, `overrides`, `sdk`, `image`,
`sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes`. `sdk` ∈ {`langgraph`,
`deep-agents`}. Profile names referenced in `capabilitySetRefs[]` are B17 artifacts
(`[PROPOSED — not in source]` as identifiers; reused verbatim from B17). Any composition that
`exposes` an A2A/MCP interface declares a version (e.g. `myAgent.v1`) per interface-contract §3.3.

### 4.2 APIs / SDK surfaces
N/A — B18 ships declarative `Agent` CRD artifacts + docs. The agents, once running, call into the
platform via the Platform SDK (`memory.*`/`rag.*`, owned by B6) and author logic via the agent
SDK (B7); neither surface is defined by B18.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
B18 emits none directly. A reconciled composition's lifecycle (created/started/…) flows under
`platform.lifecycle.*` emitted by ARK/sandbox; capability resolution changes flow under
`platform.capability.*` emitted by B13. Both emissions are owned upstream, not by B18.

### 4.4 Data schemas / connection-secret contracts
N/A — compositions reference capabilities and (optionally) `memoryRefs[]` by name; they own no
backend and write no connection secret. Backends behind referenced stores use the ADR 0044
connection-secret contract owned by B4.

## 5. OSS-vs-Custom Decision
**Build-new (content/configuration), no new code.** B18 is curated `Agent` CRD YAML plus docs;
it wraps no OSS project. It composes B17 profiles over the ARK `Agent` CRD (A5) and the B7 SDKs.
Rationale: the runtime and CRD already exist; B18's value is vetted, Canon-conformant reference
compositions users copy.

## 6. Functional Requirements
- REQ-B18-01: B18 SHALL provide a complete reference `Agent` CRD for each of the four common
  shapes that map to the B17 profile set (knowledge-base/chat, code-gen, customer-support,
  minimal RAG) (§14.2 B18).
- REQ-B18-02: Every composition SHALL reference its capabilities only through B17 profiles in
  `capabilitySetRefs[]` (plus per-Agent `overrides`), and SHALL NOT inline capability lists that
  bypass the profile library.
- REQ-B18-03: Each composition SHALL use only Canon `Agent` fields (interface-contract §1.2) and
  SHALL set `sdk` to `deep-agents` (default) or `langgraph` (ADR 0019).
- REQ-B18-04: Each composition's `capabilitySetRefs[]` ordering plus `overrides` SHALL resolve
  deterministically under the §6.8 add-if-not-there / replace-if-there model, and the resolved
  capability set SHALL be documented alongside the CRD.
- REQ-B18-05: Compositions SHALL be GitOps-packaged so they reconcile through ArgoCD → ARK with
  no manual configuration.
- REQ-B18-06: Each composition SHALL pin the `Agent` CRD API version per ADR 0030 and name ARK as
  the versioning owner; any `exposes` interface SHALL declare a version (interface-contract §3.3).
- REQ-B18-07: A composition that needs the Knowledge Base SHALL obtain it via the B17 KB profile
  (which references `platform-knowledge-base`), never by embedding KB content (ADR 0022).

## 7. Non-Functional Requirements
- Security / policy: composing a capability does not grant it — enforcement stays at LiteLLM/OPA/
  Envoy (§6.8, ADR 0018). Compositions inherit `opaPolicyRefs` from their B17 profiles.
- Multi-tenancy (§6.9): compositions are namespaced `Agent`s; any cross-namespace profile
  reference is valid only when explicitly published with an OPA-checked policy. Default namespace-private.
- Observability (§6.5): a composed agent's runs are traceable via `platform.lifecycle.*`
  (ARK/sandbox) and Langfuse/Tempo; B18 adds no new signal.
- Scale: static reference content; no runtime scaling concern.
- Versioning (ADR 0030): adding compositions is backward-compatible; changing a composition's
  resolved capabilities is a documented, meaningful change.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable** — GitOps-packaged `Agent` CRD YAML.
- Per-product docs (10.5): **Applicable** — composition catalog + resolved-capability views + adapt guide.
- Runbook (10.7): **N/A — no runtime service of its own; agent-run failures use ARK/sandbox runbooks.**
- Alerts: **N/A — no component metrics surface here.**
- Grafana dashboard (Crossplane XR): **N/A — no per-component metrics.**
- Headlamp plugin: **N/A — resolved-capability view owned by B5.**
- OPA/Rego integration: **Applicable (by reference)** — via inherited `opaPolicyRefs`; content owned by B16/B3.
- Audit emission (ADR 0034): **N/A — emission is from ARK/B13 on reconcile, not B18 artifacts.**
- Knative trigger flow: **N/A — unless a composition sets `triggers`; the trigger plumbing is B8, not B18.**
- HolmesGPT toolset: **N/A — B18 contributes no toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — Chainsaw for Agent apply/resolve; PyTest for resolved-capability assertions; Playwright N/A (no UI).
- Tutorials & how-tos: **Applicable** — "assemble an agent from profiles" how-to (content in C2/C3, seeded here).

## 9. Acceptance Criteria
- AC-B18-01 (REQ-B18-01): Applying B18 yields four complete, schema-valid `Agent` resources for the four shapes.
- AC-B18-02 (REQ-B18-02): Each composition's capabilities trace to a B17 `capabilitySetRefs[]` entry; a lint check finds no profile-bypassing inline capability lists.
- AC-B18-03 (REQ-B18-03): A schema check confirms each `Agent` uses only §1.2 fields and `sdk` ∈ {`deep-agents`, `langgraph`}.
- AC-B18-04 (REQ-B18-04): A resolution test computes each composition's capability set from its refs+overrides and matches the documented resolved view exactly (§6.8 semantics).
- AC-B18-05 (REQ-B18-05): Compositions reconcile via ArgoCD → ARK with zero manual config.
- AC-B18-06 (REQ-B18-06): Each manifest carries an explicit `Agent` apiVersion; any `exposes` interface declares a version string.
- AC-B18-07 (REQ-B18-07): The KB composition's resolved `ragStores[]` contains `platform-knowledge-base` (via the B17 KB profile) and no inlined KB content.

## 10. Risks & Open Questions
- R1 (med): Composition correctness depends on B17 profile **names** being final. `[PROPOSED — not
  in source]`. Reconciliation note: B17 owns names; B18 imports them verbatim — must land/rebase after B17.
- R2 (med): Full resolution requires ADR 0032 overlay mechanics + B13; if B18 authors before they
  land, resolved-view tests run against a fake resolver. Open question: gate B18 e2e on B13/ADR 0032.
- R3 (low): Concrete `modelRef` / `sandboxTemplateRef` / `image` values depend on what A5/A6 and the
  LLM-provider set ship. Open question: pin to shipped names or use documented placeholders.
- R4 (low): Whether a composition should set `triggers`/`exposes` in the reference examples or leave
  them out for clarity is a content choice; default is minimal examples with one A2A-exposing variant.

## 11. References
- architecture-overview.md §6.2 Agent runtime architecture (~L247–292; SDK default L284; interaction surfaces L286–292).
- architecture-overview.md §6.8 Capability registries + layering (~L632–718; overlay worked example L687–714).
- architecture-overview.md §6.4 Knowledge Base as a separate primitive (~L349–368).
- architecture-overview.md §14.2 Workstream B, B18 row (~L1712); B17 row (~L1711); B7 row (~L1701).
- ADR 0013 (capability CRD model + layering), ADR 0019 (LangGraph + Deep Agents default),
  ADR 0032 (overlay semantics), ADR 0022 (Knowledge Base primitive), ADR 0030 (CRD/API versioning).
- _meta/interface-contract.md §1.2 (`Agent`), §1.4 (`CapabilitySet`), §3.2 (agent SDK), §3.3 (exposed-interface versioning), §2 (CloudEvent taxonomy).
- _meta/glossary.md (Platform Agent, CapabilitySet, Knowledge Base, A2A, MCP).

# SPEC V6-08 — Capability registries and approved primitives [PROPOSED]

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0013, 0032, 0031, 0030, 0018] · views: [6.8]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's capability-registry slice: the
architectural commitment that every unit of agent access — an `MCPServer`, `A2APeer`, `RAGStore`,
`EgressTarget`, `Skill`, plus `llmProviders[]` and `opaPolicyRefs[]` — is a first-class
namespaced Kubernetes CRD reconciled from Git, **never** gateway-internal configuration (§6.8,
ADR 0013). A `CapabilitySet` bundles and layers those references; an `Agent` composes from a
library of CapabilitySets plus per-Agent `overrides`.

The slice is realized by several components (the kopf operator B13 that reconciles the CRDs, the
OpenAPI→MCP converter A12, the MCP-services integration A17, the agent profile library B17). This
SPEC does **not** build any of them; it fixes the invariants, the spanning interfaces, the overlay
resolution semantics, and the enforcement and notification model that each participating component
must honor so the registry presents one coherent governance surface.

## 2. Scope

### 2.1 In scope
- The invariant that approved capabilities are CRDs reconciled into LiteLLM / Envoy / OPA, and that
  unapproved capabilities are unreachable from a Platform Agent.
- The spanning `CapabilitySet` layering contract (Helm-style add-if-not-there / replace-if-there;
  per-Agent `overrides` apply last) as the architectural commitment (§6.8).
- The `platform.capability.changed` CloudEvent notification model and its best-effort status.
- The enforcement triad (LiteLLM, OPA, Envoy) as the authoritative perimeter, independent of
  notification delivery.

### 2.2 Out of scope (and where it lives instead)
- The kopf reconciler implementation and LiteLLM admin-API wiring — **component B13**.
- Detailed CapabilitySet **overlay resolution semantics** (recursive includes, missing-ref
  validation, namespace-boundary rules, override-may-grant question) — **ADR 0032** + design specs.
- `SyntheticMCPServer` synthesis from OpenAPI specs — **component A12**.
- Concrete approved MCP-service set (GitHub, Google Drive, Context7, etc.) — **component A17 / ADR 0020**.
- The curated CapabilitySet profile library — **component B17**.
- Per-event-type CloudEvent schemas under `platform.capability.*` — **component B12 registry**.

## 3. Context & Dependencies

Realizing components and what each contributes to the view:
- **B13** (custom Python kopf operator) — reconciles all capability CRDs into LiteLLM + Envoy;
  emits `platform.capability.changed`. The registry's reconciler of record.
- **A12** (OpenAPI→MCP converter) — produces `SyntheticMCPServer` XRs that back-link to an
  `MCPServer`, extending the registry from OpenAPI specs.
- **A17** (initial MCP services integration) — registers the v1.0 approved MCP-service set as
  `MCPServer` CRDs (ADR 0020).
- **B17** (agent profile library) — ships reusable `CapabilitySet` profiles that consume the
  overlay contract.

ADR decisions honored:
- **ADR 0013** — capabilities are first-class CRDs reconciled by B13; CapabilitySet bundles +
  layers; existence/shape is the architectural commitment, overlay detail deferred to 0032.
- **ADR 0032** — owns the detailed overlay resolution semantics this view defers to.
- **ADR 0031** — `platform.capability.changed` lives under the `platform.capability.*` namespace.
- **ADR 0030** — capability CRDs follow Kubernetes API versioning, owned by B13's reconciler.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor governs who may create/bind sets.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Spanning CRDs (Canon, owner B13 unless noted; all namespaced):
- `MCPServer` — `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility`.
- `A2APeer` — `endpoint`, `direction` (internal/external), `auth`, `tags`.
- `RAGStore` — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`.
- `EgressTarget` — `fqdn`, `port`, `scheme`, `allowedMethods`.
- `Skill` — `gitRef`, `versionPin`, `schemaRef`.
- `CapabilitySet` — `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`.
- `Agent` (owner A5) — references CapabilitySets via `capabilitySetRefs[]` plus `overrides`.
- `SyntheticMCPServer` (XR, owner B4, produced by A12) — `openApiSpecRef`, `authConfigRef`, `mcpServerRef` (back-link).

Versioning: all follow ADR 0030 (`v1alpha1`/`v1beta1`/`v1`; breaking → new `vN` group + conversion
webhook; `vN-1` deprecated ≥1 minor release).

### 4.2 APIs / SDK surfaces
- Platform SDK (B6) consumes `platform.capability.changed` and surfaces a callback / next-turn
  notification; exposes a poll fallback (`refresh_capabilities()` per ADR 0013) for pull harnesses.
- LiteLLM gateway registration API — Platform Agents register A2A/MCP exposings dynamically at
  startup, OPA-gated (§6.9; detailed in V6-09).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: `platform.capability.changed` (the specific capability-change event, ADR 0013) under the
  `platform.capability.*` namespace, on any MCPServer / A2APeer / RAGStore / EgressTarget / Skill /
  CapabilitySet add / update / delete.
- Consumed: by the Platform SDK (callback/next-turn) and any subscriber tagging observability.

### 4.4 Data schemas / connection-secret contracts
N/A — this view introduces no substrate-backed primitive; MCP-service secrets are handled via ESO
(§A17), not a connection-secret XRD.

## 5. OSS-vs-Custom Decision
N/A — VIEW. Realizing components carry their own OSS-vs-custom decisions (B13 custom kopf operator
per ADR 0006; A12/A17/B17 per their specs).

## 6. Functional Requirements

- **REQ-V6-08-01** (invariant): Every approved capability MUST exist as one of the Canon capability
  CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`), reconciled from Git; no
  capability is configured directly on LiteLLM or Envoy.
- **REQ-V6-08-02** (invariant): A Platform Agent MUST be unable to reach any capability not present
  in its resolved CapabilitySet state; unapproved capabilities are unreachable at the perimeter.
- **REQ-V6-08-03** (constraint): CapabilitySets referenced by an `Agent` MUST resolve one at a time
  in declared order with field-level add-if-not-there / replace-if-there semantics; per-Agent
  `overrides` apply last with the same semantics. Lists are replaced wholesale, not concatenated.
- **REQ-V6-08-04** (constraint): On any capability-CRD add/update/delete, the reconciler MUST emit a
  `platform.capability.changed` CloudEvent under `platform.capability.*` carrying the affected
  Agent / CapabilitySet identifiers.
- **REQ-V6-08-05** (constraint): The capability-change notification is **best-effort** UX only;
  enforcement MUST remain authoritative at LiteLLM (gateway), OPA (policy), and Envoy (egress). An
  agent that missed a notification MUST NOT be able to exceed its current capabilities.
- **REQ-V6-08-06** (constraint): The platform MUST be able to render a **resolved per-Agent view**
  of all applied capabilities regardless of which CapabilitySet contributed each, sourced from
  Kubernetes CRDs and LiteLLM applied state (Headlamp plugin, §6.8).
- **REQ-V6-08-07** (constraint): Capability CRDs MUST follow ADR 0030 versioning; their lifecycle
  owner is the B13 reconciler.
- **REQ-V6-08-08** (constraint): Cross-namespace CapabilitySet reference is allowed only when
  explicitly published with an OPA-checked policy; default is namespace-private.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** all capability CRDs are namespaced; the namespace is the tenancy
  boundary. Cross-tenant sharing is opt-in and OPA-checked (REQ-V6-08-08). See V6-09.
- **Security:** the enforcement triad is defense-in-depth; notification is not a security control.
  RBAC-as-floor / OPA-as-restrictor (ADR 0018) governs create/bind.
- **Observability (§6.5):** capability changes are observable via the CloudEvent and tagged in
  observability labels; the resolved-view requirement is load-bearing for operability.
- **Versioning (ADR 0030):** per-component, per the B13 reconciler.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. The view is realized by components; cross-cutting deliverables (Helm, runbook, OPA
integration, audit emission, Headlamp plugin, tests) belong to B13/A12/A17/B17, verified end-to-end
in the PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-08-01** (→REQ-01/02): A capability reachable by an Agent at runtime corresponds to a
  capability CRD in Git; a capability with no CRD is unreachable through LiteLLM and Envoy.
- **AC-V6-08-02** (→REQ-03): For two ordered CapabilitySets plus overrides, the resolved capability
  set matches the §6.8 add/replace/override model (the worked example in §6.8 resolves identically).
- **AC-V6-08-03** (→REQ-04): Any capability-CRD mutation produces exactly one
  `platform.capability.changed` CloudEvent under `platform.capability.*`.
- **AC-V6-08-04** (→REQ-05): An agent that did not receive a notification and attempts a removed
  capability is denied at the perimeter (LiteLLM/OPA/Envoy), not granted.
- **AC-V6-08-05** (→REQ-06): The resolved per-Agent capability view is renderable from CRDs +
  LiteLLM applied state.
- **AC-V6-08-06** (→REQ-08): A CapabilitySet referenced cross-namespace without an OPA-checked
  publication is rejected.

## 10. Risks & Open Questions
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review; it coins no new names.
- **(med)** Whether per-Agent `overrides` may grant capabilities absent from any referenced set is
  **deferred to ADR 0032** — AC-V6-08-02 tests only the add/replace/override mechanics, not grant
  scope. Open question routed to 0032.
- **(low)** Recursive CapabilitySet inclusion (a set referencing another set) is deferred to
  ADR 0032; this view assumes flat references for its ACs.
- **(low)** Per-event-type schema of `platform.capability.changed` (payload fields beyond the
  affected identifiers) is owned by the B12 registry; not fixed here.

## 11. References
- architecture-overview.md §6.8 (capability registries, ~L632–718); §6.9 (visibility/cross-tenant, ~L744–762).
- ADR 0013 (capability CRD model + layering + notification), ADR 0032 (overlay semantics),
  ADR 0031 (CloudEvent taxonomy), ADR 0030 (versioning), ADR 0018 (RBAC-floor/OPA-restrictor).
- Realizing components: B13, A12, A17, B17. Related views: V6-09 (tenancy), V6-12 (CRD inventory),
  V6-13 (versioning).

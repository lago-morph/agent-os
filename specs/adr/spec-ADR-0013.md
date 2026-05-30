# SPEC ADR-0013 — Capability CRD model and CapabilitySet layering [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [B13] · downstream: [A1, A6, A7, A9, B6, B16, B17, A22] · adrs: [0013] · views: [6.8, 6.12]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0013 is a settled T0 decision: every approved capability — `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill` — is a namespaced CRD reconciled from Git by the kopf operator (B13), and `CapabilitySet` is a namespaced, layerable CRD bundling those references plus `llmProviders[]` and `opaPolicyRefs[]`. An `Agent` composes capabilities via `capabilitySetRefs[]` with optional per-Agent `overrides`. The architectural commitment is the **existence and shape** of these CRDs as the single source of truth that LiteLLM, Envoy, OPA, RBAC, Headlamp, audit tagging, and observability all read from. This SPEC states the constraints, interfaces, and conformance that honoring the decision imposes. It does not re-argue capabilities-as-CRDs vs gateway-internal config, and it does not define overlay resolution (ADR 0032).

The problem the decision solves: governance, reuse, audit, and policy must all work off one declarative source of truth rather than scattered gateway/Envoy/admin-UI configuration.

## 2. Scope
### 2.1 In scope
- The obligation that the six capability CRDs + `CapabilitySet` exist as namespaced CRDs reconciled by B13 into LiteLLM and adjacent components.
- The obligation that capabilities are the single source of truth for LiteLLM registry, Envoy allowlists, OPA decisions, Headlamp resolved-view, audit tagging, observability labels.
- The `Agent.capabilitySetRefs[]` + `overrides` composition contract.
- Governance hooks: Gatekeeper admission on all capability CRDs; OPA gating of dynamic registration / virtual-key issuance against `capability_set_refs`; RBAC scoping of who may create/bind sets.
- The capability-change notification model: `platform.capability.changed` CloudEvent fanout (primary) + SDK `refresh_capabilities()` poll fallback; best-effort, not a correctness guarantee.
- The Headlamp resolved-per-Agent-view obligation.
- Default namespace-private; cross-namespace reference only by explicit OPA-checked publication.

### 2.2 Out of scope (and where it lives instead)
- Overlay resolution semantics (recursive inclusion, missing-ref validation, namespace-boundary rules, whether overrides may grant) — **ADR 0032** + design specs (architecture-backlog §1.1).
- kopf operator implementation — component **B13** SPEC.
- LiteLLM gateway internals — component **A1**.
- Per-CRD field schemas beyond the source-stated set — owning component design specs.
- `VirtualKey` / `BudgetPolicy` detail — B13 SPEC (shape only fixed in interface-contract).

## 3. Context & Dependencies
Upstream consumed: **B13** (kopf operator) reconciles the CRDs into LiteLLM/adjacent state. Downstream consumers: **A1** LiteLLM (registry source of truth), **A6** Envoy (egress allowlists from `EgressTarget`), **A7** OPA/Gatekeeper (admission + runtime gating), **A9/A22** Headlamp (resolved view + graphical editor), **B6** SDK (capability-change callback), **B16** OPA content, **B17** agent profile library (ships as CapabilitySets).

ADR decisions honored:
- **ADR 0013** (this) — capability CRDs + CapabilitySet exist; B13 reconciles; Agent composes via refs + overrides.
- **ADR 0006** — kopf operator (B13) is the reconciler into LiteLLM.
- **ADR 0018** — RBAC-floor / OPA-restrictor applies uniformly to capability create/bind.
- **ADR 0030** — capability CRD versioning per-component, B13-owned.
- **ADR 0031** — `platform.capability.changed` rides the closed taxonomy under `platform.capability.*`.
- **ADR 0032** — overlay resolution deferred there.
- **ADR 0019** — the platform SDK surfaces the capability-change callback in both supported SDKs.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
kopf-operator-reconciled (B13), all namespaced:
- `MCPServer` — `endpoint`, `authMode`, `credentialsRef`, `tags`, `scopes`, `visibility`.
- `A2APeer` — `endpoint`, `direction`, `auth`, `tags`.
- `RAGStore` — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`.
- `EgressTarget` — `fqdn`, `port`, `scheme`, `allowedMethods`.
- `Skill` — `gitRef`, `versionPin`, `schemaRef`.
- `CapabilitySet` — `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`. Layerable; overlay semantics ADR 0032.
- `VirtualKey` — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`.
- `BudgetPolicy` — `scope`, `period`, `limits`, `thresholdActions[]`.
Consuming `Agent` field: `capabilitySetRefs[]`, `overrides` (ARK-owned, A5).
Versioning: `v1alpha1`/`v1beta1`/`v1`; breaking change = new `vN` group + conversion webhook, ≥1 minor deprecation; B13 owns the lifecycle.

### 4.2 APIs / SDK surfaces
- Platform SDK (B6) subscribes to `platform.capability.changed` and surfaces a callback / interrupt / next-turn notification to agent code.
- Platform SDK `refresh_capabilities()` poll API — always-available fallback; worst-case staleness one turn. `[PROPOSED — not in source]` exact signature (SDK method signatures not specified in source).
- B13 reconciles CRDs into the LiteLLM admin API (HTTP, URL-path versioned per ADR 0030).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emits `platform.capability.changed` under the `platform.capability.*` namespace on any capability/CapabilitySet/Agent change, carrying affected Agent / CapabilitySet identifiers. No new top-level namespace introduced (closed set).
- Per-event-type schema lives in B12 registry.

### 4.4 Data schemas / connection-secret contracts
- `MCPServer.credentialsRef` / `RAGStore` ingestion reference ESO-managed secrets; this ADR introduces no new connection-secret shape.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: capability CRDs are custom platform CRDs reconciled by the custom Python **kopf** operator B13 into upstream **LiteLLM**; build-new for the CRDs, config/wrap for LiteLLM. Rejected alternative — capabilities as gateway-internal/Envoy/admin-UI config — is not built.)

## 6. Functional Requirements
- REQ-ADR-0013-01: `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, and `CapabilitySet` MUST exist as namespaced CRDs reconciled by B13; no capability may be configured directly on the gateway.
- REQ-ADR-0013-02: A `CapabilitySet` MUST bundle references to capabilities plus `llmProviders[]` and `opaPolicyRefs[]`, and MUST be layerable.
- REQ-ADR-0013-03: An `Agent` MUST compose capabilities via `capabilitySetRefs[]` and MAY carry per-Agent `overrides`; capabilities MUST NOT be restated per-agent outside this mechanism.
- REQ-ADR-0013-04: The capability CRD set MUST be the single source of truth for LiteLLM registry contents, Envoy egress allowlists, OPA decisions, RBAC scoping, Headlamp views, audit tagging, and observability labels.
- REQ-ADR-0013-05: Gatekeeper MUST admit all capability CRDs; OPA MUST gate dynamic agent registration and virtual-key issuance against `capability_set_refs` JWT claims; RBAC MUST scope who may create or bind sets (RBAC-floor / OPA-restrictor per ADR 0018).
- REQ-ADR-0013-06: Every capability and CapabilitySet MUST default to namespace-private; a cross-namespace reference MUST require explicit OPA-checked publication.
- REQ-ADR-0013-07: kopf reconciliation MUST emit a `platform.capability.changed` CloudEvent (under `platform.capability.*`) carrying the affected Agent / CapabilitySet identifiers on any capability/set/Agent change.
- REQ-ADR-0013-08: The platform SDK MUST surface that change to agent code via a callback/interrupt/next-turn notification AND MUST expose a `refresh_capabilities()` poll fallback; notification is best-effort and MUST NOT be the enforcement mechanism.
- REQ-ADR-0013-09: Enforcement MUST remain at LiteLLM / OPA / Envoy regardless of notification delivery (notification is a UX/staleness affordance, not a correctness guarantee).
- REQ-ADR-0013-10: Headlamp MUST render the resolved per-Agent capability view from the CRDs plus LiteLLM applied state.
- REQ-ADR-0013-11: Capability CRD versioning MUST follow ADR 0030 with B13 as owner.

## 7. Non-Functional Requirements
- Security/tenancy: namespace-private default + OPA-checked publication is the cross-tenant sharing boundary (ADR 0016); RBAC-floor / OPA-restrictor (ADR 0018) governs create/bind.
- Observability (§6.5): capability changes are observable via `platform.capability.*` events; resolved views are load-bearing for operability.
- Versioning (ADR 0030): per-component conversion-webhook-backed; B13-owned.
- Reversibility: a capability change is reverted by reverting the Git source; enforcement points re-reconcile.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 standard set is owned by the enforcing components (B13, A1, A6, A7, A22, B6, B17).

## 9. Acceptance Criteria
- AC-ADR-0013-01: Honored when `kubectl api-resources` shows all six capability CRDs + `CapabilitySet` reconciled by B13, and a gateway-only capability config has no effect. (REQ-01)
- AC-ADR-0013-02: Honored when a `CapabilitySet` carrying `llmProviders[]` + `opaPolicyRefs[]` is layered onto another and admitted. (REQ-02)
- AC-ADR-0013-03: Honored when an `Agent` resolves its capabilities solely from `capabilitySetRefs[]` + `overrides`. (REQ-03)
- AC-ADR-0013-04: Honored when LiteLLM registry, Envoy allowlist, an OPA decision, the Headlamp view, and an audit tag for one Agent all derive from the same CapabilitySet. (REQ-04)
- AC-ADR-0013-05: Honored when Gatekeeper denies a malformed capability CRD, OPA denies a registration whose `capability_set_refs` claim is absent, and RBAC denies an unauthorized bind. (REQ-05)
- AC-ADR-0013-06: Honored when a namespace-B Agent cannot reference a namespace-A CapabilitySet until an OPA-checked publication exists. (REQ-06)
- AC-ADR-0013-07: Honored when editing a referenced CapabilitySet emits exactly one `platform.capability.changed` event naming the affected Agent/set. (REQ-07)
- AC-ADR-0013-08: Honored when an agent receives the change via callback, and a non-subscribing runtime observes it within one turn via `refresh_capabilities()`. (REQ-08)
- AC-ADR-0013-09: Honored when, with notification suppressed, a removed capability is still denied at LiteLLM/OPA/Envoy. (REQ-09)
- AC-ADR-0013-10: Honored when Headlamp renders the resolved per-Agent capability view matching LiteLLM applied state. (REQ-10)
- AC-ADR-0013-11: Honored when a breaking capability CRD change without a new `vN` group + conversion webhook fails CI. (REQ-11)

## 10. Risks & Open Questions
- OQ-1 (high): Overlay resolution semantics (recursive inclusion, missing/deleted refs, namespace-boundary cross-references, whether `overrides` may grant capabilities absent from any set) are deferred to ADR 0032; ACs touching layering are partial until it lands. `[PROPOSED]`
- R-1 (med): Best-effort notification can leave an agent believing it has access it lost; mitigated by hard enforcement at LiteLLM/OPA/Envoy (REQ-09) and one-turn poll staleness bound.
- R-2 (low): Headlamp resolved-view correctness depends on reconciled LiteLLM applied state; staleness surfaces as a UI lag, not an enforcement gap.

## 11. References
- ADR 0013 (`adr/0013-capability-crd-and-capabilityset-layering.md`) — the decision enforced here.
- architecture-overview.md §6.8 (capability registries), §6.9 (tenancy), §6.12 (CRD inventory).
- Enforcing components: B13 (kopf operator, owner), A1 (LiteLLM), A6 (Envoy), A7 (OPA/Gatekeeper), A9/A22 (Headlamp resolved view + editor), B6 (SDK callback), B16 (OPA content), B17 (profile library).
- Related ADRs: 0006, 0016, 0018, 0019, 0030, 0031, 0032.

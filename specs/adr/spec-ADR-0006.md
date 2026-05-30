# SPEC ADR-0006 — Python kopf operator for LiteLLM reconciliation [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [A1] · downstream: [A17;B16;B17] · adrs: [0006] · views: [6.8;6.12;6.13]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0006 fixes a custom Python kopf operator (component B13) as the CRD-driven reconciler that keeps LiteLLM in sync with the capability surface. This SPEC states what honoring that decision requires: B13 reconciles exactly the LiteLLM-facing CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) into LiteLLM's admin API and OPA data; the controller-framework invariant holds (every controller is either Crossplane for cloud-shaped or kopf for app-API — no bespoke frameworks); and the operator ships as a subchart of the LiteLLM Helm chart so gateway and operator version-pin together. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- B13 kopf operator as the only reconciler for the eight LiteLLM-facing capability CRDs.
- The controller-framework invariant: Crossplane (cloud-shaped) **or** kopf (app-API), no bespoke frameworks.
- Packaging as a subchart of the LiteLLM Helm chart (single ArgoCD release, automatic gateway↔operator pin).
- HTTP admin API versioned under `/v1/...`; capability-reconcile audit emission.

### 2.2 Out of scope (and where it lives instead)
- LiteLLM gateway install + admin API — component A1 SPEC.
- Capability CRD shape + CapabilitySet existence — ADR 0013; overlay resolution — ADR 0032.
- Cloud-shaped XR reconciliation (`AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, `GrafanaDashboard`) — Crossplane v2 (B4), explicitly NOT B13.
- OPA decision engine — ADR 0002; secret wiring detail (ESO/Secrets) — B13/A17 design.

## 3. Context & Dependencies
Upstream consumed: A1 (LiteLLM admin API the operator reconciles into). Downstream consumers: A17 (initial MCP services), B16 (OPA content), B17 (agent profiles) build on the reconciled capability surface.
ADR decisions honored: **0006** — B13 is the kopf reconciler for LiteLLM-facing CRDs, subcharted with LiteLLM; **0013** — capability CRD existence/shape; **0002** — capabilities are OPA-gated; **0021** — Crossplane owns cloud-shaped XRs (the other side of the split); **0030** — `/v1/...` URL-path API versioning + per-component CRD versioning ownership.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B13-reconciled, all namespaced, owner = B13:
- `MCPServer` — `endpoint`, `authMode`, `credentialsRef`, `tags`, `scopes`, `visibility`.
- `A2APeer` — `endpoint`, `direction`, `auth`, `tags`.
- `RAGStore` — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`.
- `EgressTarget` — `fqdn`, `port`, `scheme`, `allowedMethods`.
- `Skill` — `gitRef`, `versionPin`, `schemaRef`.
- `CapabilitySet` — `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`.
- `VirtualKey` — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`.
- `BudgetPolicy` — `scope`, `period`, `limits`, `thresholdActions[]`.
Versioning per ADR 0030; B13 owns the lifecycle. The kopf operator MUST NOT reconcile Crossplane XRs.

### 4.2 APIs / SDK surfaces
B13 exposes an HTTP admin API versioned under `/v1/...` (URL-path versioning, deprecated versions reachable ≥1 release after replacement). It reconciles into LiteLLM's admin API and into OPA data where applicable.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Capability-registry changes (add/update/delete of the CRDs above) emit under `platform.capability.*` (incl. `platform.capability.changed` per ADR 0013). Reconcile/audit events route through the audit pipeline under `platform.audit.*`. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
B13 implements its own secret wiring (via ESO / Kubernetes Secrets) for LiteLLM-specific CRDs — it does NOT use Crossplane connection secrets and does not participate in Crossplane Compositions.

## 5. OSS-vs-Custom Decision
Build-new (custom): a **Python kopf** operator (B13), packaged as a subchart of the **LiteLLM** Helm chart. Mode: build-new on kopf. Rationale per ADR 0006: no native Crossplane provider for LiteLLM; the team has no Go experience, so a Go Crossplane provider was rejected; kopf achieves the same reconciliation and keeps Python as the default language. Trade-off accepted: lose Crossplane's connection-secret handling/package model/Composition integration for the LiteLLM-facing surface (bounded to those CRDs).

## 6. Functional Requirements
- REQ-ADR-0006-01: B13 MUST reconcile exactly the eight LiteLLM-facing CRDs into LiteLLM's admin API and OPA data where applicable.
- REQ-ADR-0006-02: B13 MUST NOT reconcile cloud-shaped XRs; those remain Crossplane v2 (B4).
- REQ-ADR-0006-03: Every platform controller MUST be either Crossplane (cloud-shaped) or kopf (app-API) — no bespoke controller frameworks.
- REQ-ADR-0006-04: B13 MUST ship as a subchart of the LiteLLM Helm chart (single ArgoCD release, automatic gateway↔operator version pin).
- REQ-ADR-0006-05: B13's admin API MUST be URL-path versioned under `/v1/...`.
- REQ-ADR-0006-06: B13 MUST emit capability-reconcile events to the audit pipeline and capability-change events under `platform.capability.*`.
- REQ-ADR-0006-07: B13 MUST implement its own secret wiring (ESO/Secrets) and not participate in Crossplane Compositions.

## 7. Non-Functional Requirements
- Security: capabilities OPA-gated; secrets via ESO/Secrets; admission-gated CRDs (ADR 0002).
- Versioning: per-component CRD versioning owned by B13; URL-path API versioning with ≥1-release deprecation window (ADR 0030).
- Lifecycle: subchart binding gives automatic gateway↔operator pin and a shared upgrade path.
- Standard custom-component deliverables apply (HolmesGPT toolset, OPA contribution, audit, observability, Knative trigger flow, tests, docs).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1/§14.2 deliverable set is owned by component B13; conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0006-01: Applying each of the eight CRDs results in the corresponding LiteLLM admin-API state; B13 is the reconciler of record. (REQ-01)
- AC-ADR-0006-02: No XR (`AgentEnvironment`/`MemoryStore`/`SyntheticMCPServer`/`GrafanaDashboard`) is reconciled by B13. (REQ-02)
- AC-ADR-0006-03: A repository scan finds no controller framework other than Crossplane or kopf for platform controllers. (REQ-03)
- AC-ADR-0006-04: A single ArgoCD-applied LiteLLM chart deploys both gateway and operator at pinned, matching versions. (REQ-04)
- AC-ADR-0006-05: The admin API is reachable at `/v1/...`; a breaking change without a new URL-path version fails CI. (REQ-05)
- AC-ADR-0006-06: A capability add/update/delete emits a `platform.capability.*` event and an audit record. (REQ-06)
- AC-ADR-0006-07: B13 reads no Crossplane connection secret and appears in no Composition. (REQ-07)

## 10. Risks & Open Questions
- Loss of Crossplane secret/package/Composition integration for LiteLLM CRDs (blast radius: med) — bounded to LiteLLM-facing CRDs; B13 implements equivalents per ADR 0006.
- kopf maturity/operational depth vs a Go provider (low) — accepted to keep Python default; mitigated by subchart binding.
- Open question: exact ESO vs in-chart Secret split for `credentialsRef` (low) — settle in B13/A17 design.

## 11. References
- ADR 0006 (`adr/0006-python-kopf-litellm-operator.md`) — the decision.
- Enforcing components: B13 (kopf operator, owner), A1 (LiteLLM gateway + admin API), B4 (Crossplane — other side of the split), A7 (OPA gating), A18 (audit), B12 (capability events).
- architecture-overview.md §6.1, §6.6, §6.8, §6.12, §6.13, §9, §11; architecture-backlog.md §2.5, §6.
- Related: ADR 0001, 0002, 0013, 0021, 0030, 0031.

# SPEC ADR-0001 — ARK as the agent operator [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A6] · downstream: [A5;B7;B9;B10;B16;B17;B18;B20] · adrs: [0001] · views: [6.2;6.12;6.13]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0001 fixes ARK as the Kubernetes-native operator for Platform Agent lifecycle. This SPEC states what honoring that decision requires: ARK must own and reconcile the canonical ARK CRD set (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`), agents must be declared and reconciled only through these CRDs (no bespoke agent controller), and the decision's stated mitigations (version pinning, namespace isolation, OPA admission, Headlamp visibility) must be in force. The decision is settled; this SPEC captures the obligations it imposes on the platform and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- ARK as component A5: install, version pin, and ownership of the seven ARK CRD reconcilers.
- The constraint that Platform Agents are declared as `Agent` CRDs reconciled into pods running inside `Sandbox` instances.
- Conformance hooks: OPA/Gatekeeper admission on ARK CRDs, namespace-scoped isolation, Headlamp cross-tenant visibility plugin, version-pinned compatibility against the Platform SDK.

### 2.2 Out of scope (and where it lives instead)
- ARK install mechanics, Helm values, full deliverable set — component A5 SPEC.
- Sandbox runtime and Envoy egress — component A6 (ADR 0003).
- `Memory` binding backend (Letta) — A10 (ADR 0005); access modes — ADR 0025.
- Per-tenant RBAC detail — deferred to design (architecture-backlog.md §1.2).
- Agent SDK surface — B7 (ADR 0019); Platform SDK — B6.

## 3. Context & Dependencies
Upstream consumed: A6 (`Sandbox`/`SandboxTemplate` that ARK schedules agent pods into). Downstream consumers: B7, B9, B10, B16, B17, B18, B20 build on the ARK CRD surface.
ADR decisions honored: **0001** — ARK is the agent operator and owns the seven CRDs; **0002** — ARK CRDs are admission-gated by Gatekeeper; **0005** — Letta reached only via ARK `Memory`; **0030** — ARK CRD versioning is per-component, owned by A5; **0019** — `Agent.sdk` accepts `langgraph`/`deep-agents`.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
ARK-reconciled, all namespaced, owner = A5:
- `Agent` — `capabilitySetRefs[]`, `overrides`, `sdk`, `image`, `sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes`.
- `AgentRun` — `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state`.
- `Team` — `members[]`, `coordinationStrategy`.
- `Tool` — defers to ARK (fields not specified in source).
- `Memory` — `memoryStoreRef` (access mode on `MemoryStore`, ADR 0025).
- `Evaluation` — `agentRef`, `datasetRef`, `evaluators[]`.
- `Query` — defers to ARK (fields not specified in source).
Versioning: `v1alpha1`/`v1beta1`/`v1`; breaking change = new `vN` group + conversion webhook, ≥1 minor deprecation window; A5 owns the lifecycle.

### 4.2 APIs / SDK surfaces
Platform SDK (B6) ships a version-pinned compatibility matrix against ARK versions. No new API surface introduced by this ADR.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Agent / `AgentRun` / `Sandbox` lifecycle under `platform.lifecycle.*`. Per-event-type names deferred to B12 registry.

### 4.4 Data schemas / connection-secret contracts
N/A — ARK introduces no substrate primitive; `Memory` binding state lives in Postgres via Letta (ADR 0005).

## 5. OSS-vs-Custom Decision
Upstream project: **ARK** (technical-preview), adopted as-is and version-pinned (A5). Mode: install + config + pin (not fork). Rationale per ADR 0001: broader CRD set and stronger CLI/SDK/dashboard surface than kagent; kagent rejected.

## 6. Functional Requirements
- REQ-ADR-0001-01: Every Platform Agent MUST be declared as an ARK `Agent` CRD; no other agent-controller path exists.
- REQ-ADR-0001-02: ARK MUST own and reconcile exactly the seven CRDs `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`.
- REQ-ADR-0001-03: `Agent` reconciliation MUST schedule agent pods inside an A6-managed `Sandbox` (via `sandboxTemplateRef`).
- REQ-ADR-0001-04: The installed ARK version MUST be pinned and recorded in the Platform SDK compatibility matrix.
- REQ-ADR-0001-05: ARK CRDs MUST be subject to Gatekeeper admission and namespace-scoped isolation.
- REQ-ADR-0001-06: ARK CRD versioning MUST follow ADR 0030 with A5 as the owner.
- REQ-ADR-0001-07: A Headlamp plugin MUST provide cross-tenant visibility into ARK resources (OSS-preview mitigation).

## 7. Non-Functional Requirements
- Security/tenancy: namespace isolation is the tenancy boundary (no ARK-native per-tenant RBAC in v1.0); OPA-as-restrictor over RBAC floor (ADR 0018).
- Observability: ARK lifecycle emitted under `platform.lifecycle.*`; trace_id propagated to `AgentRun`.
- Versioning: per-component, conversion-webhook-backed (ADR 0030).
- Resilience: technical-preview risk mitigated by pinning + planned upstream contribution.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. Enforcement of the §14.1 deliverable set is owned by component A5; conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0001-01: Applying an agent manifest that bypasses the `Agent` CRD (raw Deployment) is rejected/unreconciled — no agent pod runs outside ARK control. (REQ-01)
- AC-ADR-0001-02: `kubectl api-resources` shows all seven ARK CRDs reconciled by ARK and no others claiming them. (REQ-02)
- AC-ADR-0001-03: A reconciled `Agent` produces a pod whose owning `Sandbox` matches `sandboxTemplateRef`. (REQ-03)
- AC-ADR-0001-04: The pinned ARK version in the install equals the version in the SDK compatibility matrix; CI fails on drift. (REQ-04)
- AC-ADR-0001-05: A Gatekeeper constraint denies a non-conformant `Agent`; cross-namespace agent access is blocked. (REQ-05)
- AC-ADR-0001-06: A breaking ARK CRD change without a new `vN` group + conversion webhook fails CI. (REQ-06)
- AC-ADR-0001-07: Headlamp surfaces ARK resources across namespaces for an operator identity. (REQ-07)

## 10. Risks & Open Questions
- ARK is technical-preview (blast radius: high) — mitigated by version pinning + upstream contribution; reconciled per ADR 0001 consequences.
- Per-tenant RBAC depth deferred (med) — `[PROPOSED]` resolution tracked in architecture-backlog.md §1.2, not this ADR.
- Open question: does Gatekeeper admission for ARK CRDs ship with A5 or A7 bundle? (low) — ordering note, settle in A5/A7 plans.

## 11. References
- ADR 0001 (`adr/0001-ark-as-agent-operator.md`) — the decision.
- Enforcing components: A5 (ARK install + CRD reconcilers, owner), A6 (Sandbox), A7 (Gatekeeper admission), A9 (Headlamp visibility), B6 (SDK compat matrix).
- architecture-overview.md §6.2, §6.12, §6.13, §9, §14.1; architecture-backlog.md §2.1.
- Related: ADR 0002, 0005, 0018, 0019, 0030.

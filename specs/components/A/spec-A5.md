# SPEC A5 — ARK agent operator

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [A6] · downstream: [B16, B7, B9, B10, B17, B18, B20] · adrs: [0001] · views: [6.2, 6.12, 6.13]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

ARK is the platform's Kubernetes-native agent operator (component A5; ADR 0001). It supplies and reconciles the CRDs that anchor the declarative Platform Agent model — `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query` (interface-contract §1.2; §6.12) — turning Git-declared, ArgoCD-synced resources into running pods inside `agent-sandbox`-managed Sandboxes (§6.2). It is the control-plane half of the agent runtime: it owns no data-plane traffic itself (LLM/MCP/A2A go through LiteLLM; egress goes through Envoy; sandboxing is owned by A6) but it is the reconciler that binds an `Agent` to its CapabilitySet references, memory bindings, model reference, SDK harness, and sandbox template, and drives execution through `AgentRun`.

The problem ARK solves is declarative-governance-consistent agent lifecycle: every Platform Agent must be a CRD in Git, reconciled into a sandboxed pod whose every outbound call is externally enforced (LiteLLM, OPA, Envoy). ARK is the operator that makes the `Agent` CRD real while leaving all enforcement perimeters external, so the platform's governance posture holds regardless of harness internals (§6.2).

ARK is upstream-technical-preview; A5 is the install + configure + operate package that pins a tested version, layers namespace isolation + OPA admission + a Headlamp plugin for cross-tenant visibility, and ships the standard Workstream A deliverables (ADR 0001 consequences; §9; §14.1).

## 2. Scope

### 2.1 In scope

- ARK install (Helm values / manifests) pinned to a tested upstream version (ADR 0001; §9 mitigation).
- Registration and lifecycle ownership of the seven ARK-reconciled CRDs: `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query` (interface-contract §1.2), including their CRD versioning lifecycle per ADR 0030 (A5 is the owning component for these reconcilers).
- Reconciliation of `Agent` into sandboxed pods by handing off to the agent-sandbox controller (A6) via `sandboxTemplateRef` — A5 requests the Sandbox; A6 owns Sandbox lifecycle.
- Wiring `Agent` consumption of: `capabilitySetRefs[]`, `memoryRefs[]` (→ `Memory` → `MemoryStore`), `modelRef` (→ LiteLLM model), `sdk` (`langgraph` | `deep-agents`, ADR 0019), `image`, `triggers`, `exposes` (A2A/MCP declarations subject to LiteLLM dynamic-registration OPA gating).
- `AgentRun` creation/observation path for executions initiated by Knative event adapters (B8) and Argo Workflows steps (A3) — A5 reconciles the run; the creators are upstream.
- Audit emission for ARK lifecycle events (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query` create/update/delete/state-transition) via the audit adapter library (ADR 0034; §6.6 audit points).
- ARK OTel trace emission into the observability stack (§6.5), correlated by `traceId`/`trace_id` (ADR 0015).
- Knative trigger-flow design for ARK lifecycle CloudEvents under `platform.lifecycle.*` (ADR 0031; §6.7).
- OPA/Gatekeeper admission integration for all seven CRDs (§6.6 OPA decision points).
- ARK Headlamp plugin for cross-tenant agent visibility (ADR 0001; §9).
- HolmesGPT ARK toolset contribution (§6.10; ADR 0012).
- Three-layer tests (Chainsaw / Playwright / PyTest), per-product docs (10.5), operator runbook (10.7), alerts, Grafana dashboard (Crossplane `GrafanaDashboard` XR), tutorials & how-tos (§14.1).

### 2.2 Out of scope (and where it lives instead)

- Sandbox runtime, warm pool, gVisor/Kata isolation, Sandbox lifecycle/audit — **A6** (agent-sandbox + Envoy); ADR 0003. A5 references `SandboxTemplate`/`Sandbox` but does not reconcile them.
- LiteLLM gateway, model routing, MCP/A2A brokering, dynamic-registration enforcement — **A1** / **B2** callbacks.
- Capability CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) reconciliation into LiteLLM — **B13** kopf operator (ADR 0013).
- Letta memory backend itself and `MemoryStore` XR composition — **A10** (Letta) / **B4** (Crossplane) / **B11** (adapter). A5 owns only the `Memory` CRD binding.
- Platform SDK (`memory.*`, `rag.*`) and agent SDK harness images — **B6** / **B7**.
- Agent profile library, recommended compositions — **B17** / **B18**.
- Persistent-volume mapping into agents — **B20**.
- `Tool` and `Query` detailed field sets — **deferred to ARK upstream** (interface-contract §6; not invented here).
- CapabilitySet overlay-resolution semantics — **ADR 0032** + B13.

## 3. Context & Dependencies

**Upstream consumed (HARD):**

- **A6** (agent-sandbox + Envoy egress proxy) — A5 reconciles `Agent` into pods that run inside A6-managed Sandboxes via `sandboxTemplateRef`; the `SandboxTemplate`/`Sandbox` CRDs (interface-contract §1.3) and warm-pool semantics are A6's. ARK cannot place a workload without A6.

**Upstream consumed (continuous / non-blocking):** B14 test framework, B22 threat-model standards (waves.md fan-out edges); A13 (Tempo+Mimir) and A2 (Langfuse) for OTel/trace destinations; A18 audit adapter library (lands W1; A5 links against it as it becomes available — bootstrap-tolerant per the audit-adapter library contract).

**Downstream consumers:** B16 (OPA content targeting ARK CRDs), B7 (agent SDK harness keyed to `Agent.sdk`), B9 (`agent-platform` CLI drives ARK resources), B10 (Coach consumes ARK traces/lifecycle), B17/B18 (profiles/compositions are `Agent` CRDs), B20 (PV mapping into agent pods).

**ADR decisions honored:**

- **ADR 0001** — ARK is THE agent operator; A5 owns the seven CRDs' reconcilers and their versioning lifecycle; pin a tested version; multi-tenancy via namespace isolation + OPA admission + Headlamp plugin (not ARK-native per-tenant RBAC); Letta consumed only through the `Memory` CRD; HolmesGPT gets an ARK toolset.
- **ADR 0030 / §6.13** — per-component CRD versioning; A5 owns conversion-webhook + deprecation-window lifecycle for its seven CRDs.
- **ADR 0019** — `Agent.sdk` accepts `langgraph` and `deep-agents` in v1.0; the value set grows by enrollment without a schema break.
- **ADR 0031 / §6.7** — ARK lifecycle events fall under `platform.lifecycle.*` (closed top-level set; per-event-type names deferred to B12 registry).
- **ADR 0034** — all ARK audit emission goes through the audit adapter library to the audit endpoint; ARK never writes audit records directly.
- **ADR 0002 / §6.6** — Gatekeeper admission gates all seven ARK CRDs; OPA decision points apply.
- **ADR 0025** — `Memory` references a `MemoryStore` whose `accessMode` governs access; A5 enforces nothing about access mode itself beyond honoring the binding.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

Owner: ARK install = A5 (interface-contract §1.2). All namespaced (glossary: no cluster-scoped platform CRDs in v1.0). Fields below are **source-stated only**; anything beyond is flagged.

| CRD | Scope | Source-stated key fields |
|---|---|---|
| `Agent` | namespaced | `capabilitySetRefs[]`, `overrides`, `sdk` (`langgraph`/`deep-agents`), `image`, `sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes` (A2A/MCP) |
| `AgentRun` | namespaced | `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state` |
| `Team` | namespaced | `members[]`, `coordinationStrategy` |
| `Tool` | namespaced | defers to ARK — field set not specified in source |
| `Memory` | namespaced | `memoryStoreRef` (access mode lives on `MemoryStore`, ADR 0025) |
| `Evaluation` | namespaced | `agentRef`, `datasetRef`, `evaluators[]` |
| `Query` | namespaced | defers to ARK — field set not specified in source |

- API versions follow `v1alpha1` → `v1beta1` → `v1` (ADR 0030). `[PROPOSED — not in source]` the exact starting version per CRD is not stated; A5 selects the upstream ARK-shipped version and pins it.
- `[PROPOSED — not in source]` ARK status/conditions sub-resource field names are not enumerated in Canon; A5 reuses ARK upstream status shapes and documents them, tagging any platform-added condition.

### 4.2 APIs / SDK surfaces

- A5 exposes **no new platform HTTP API surface of its own**; the declarative surface is the seven CRDs (§6.2: "Kubernetes CRDs … declarative configuration consumed by ARK"). 
- Platform SDK `memory.*` / `rag.*` (B6) terminate at LiteLLM/Letta, not at ARK; ARK only resolves the `Memory` binding. The Platform SDK ships a version-pinned compatibility matrix against ARK (ADR 0001 consequence; §6.13).
- `[PROPOSED — not in source]` ARK's own admin/reconciler HTTP/gRPC surface (if any) is not specified in Canon; if exposed it follows URL-path `/v1/...` versioning per interface-contract §3.3.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **Emitted:** ARK `Agent`/`AgentRun`/`Sandbox-handoff`/`MemoryStore`-related lifecycle events under **`platform.lifecycle.*`** (created/started/paused/resumed/completed/failed/deleted — interface-contract §2). Audit-relevant ARK actions also emit under **`platform.audit.*`** via the audit adapter.
- **Consumed:** ARK reconciles `AgentRun` resources created by Knative event adapters (B8) and Argo Workflow steps (A3); the triggering CloudEvents themselves are consumed by B8 adapters which then create the `AgentRun` (ARK does not subscribe to the broker directly — §6.7, ADR 0001 consequence).
- Per-event-type **names and schemas** under `platform.lifecycle.*` are **deferred to B12's schema registry** (interface-contract §2, §6). Each event carries `specversion` + `schemaVersion` (ADR 0030/0031). `[PROPOSED — not in source]` concrete event-type names (e.g. an `agentrun` completed type) are not in Canon and must be registered in B12, not coined here.

### 4.4 Data schemas / connection-secret contracts

- ARK persists no platform-of-record data of its own beyond Kubernetes resource state; agent state/checkpoints live in Postgres via Letta/Memory (§6.3), not in ARK.
- N/A — no substrate connection-secret is produced by A5 (connection-secret contract per ADR 0041 applies to B4 substrate XRDs, not to ARK).

## 5. OSS-vs-Custom Decision

- **Upstream project:** ARK (the agent operator), selected over kagent (ADR 0001; architecture-backlog §2.1).
- **Mode:** **config + wrap**, not fork. Install unmodified at a pinned tested version (Helm); wrap with platform deliverables: namespace-isolation posture, Gatekeeper admission policies (B16 targets), audit-adapter linkage, OTel config, Headlamp plugin, HolmesGPT toolset. Upstream contributions planned rather than maintaining a fork (§9 mitigation).
- **Rationale:** ARK's broader CRD set and stronger CLI/SDK/dashboard surface align with the declarative-governance posture (ADR 0001). Technical-preview risk is mitigated by version pinning + external enforcement perimeters.
- **`[PROPOSED — not in source]`** exact upstream ARK version/chart coordinates are not stated in Canon; selected and pinned at install time and recorded in the runbook.

## 6. Functional Requirements

- REQ-A5-01: A5 installs ARK via Helm values/manifests in Git at a single pinned, tested upstream version; the pin is recorded and ArgoCD-synced.
- REQ-A5-02: A5 registers and owns the reconcilers for exactly the seven ARK CRDs `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`; all are namespaced.
- REQ-A5-03: Reconciling an `Agent` resolves `sandboxTemplateRef` and requests a Sandbox from the agent-sandbox controller (A6); A5 does not itself reconcile `Sandbox`/`SandboxTemplate`.
- REQ-A5-04: An `Agent` with `sdk: langgraph` or `sdk: deep-agents` reconciles successfully; an `sdk` value outside the v1.0-enrolled set is rejected at admission (Gatekeeper) rather than silently defaulted.
- REQ-A5-05: An `Agent`'s `capabilitySetRefs[]`, `memoryRefs[]`, `modelRef`, `image`, `triggers`, and `exposes` are honored as the declarative binding; `memoryRefs[]` resolve through `Memory` → `memoryStoreRef` to a `MemoryStore` whose `accessMode` governs access (A5 does not override it).
- REQ-A5-06: A5 reconciles `AgentRun` resources created by Knative event adapters (B8) and Argo Workflow steps (A3), driving execution and surfacing `state`.
- REQ-A5-07: Every audit-relevant ARK lifecycle action (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query` create/update/delete/state-transition) emits a structured audit event through the audit adapter library to the audit endpoint; ARK writes no audit record directly to Postgres/S3/OpenSearch.
- REQ-A5-08: ARK lifecycle CloudEvents are emitted under the `platform.lifecycle.*` namespace only, each carrying `specversion` + `schemaVersion`; A5 introduces no new top-level CloudEvent namespace.
- REQ-A5-09: A5 emits ARK OTel traces into the observability stack, propagating/recording the `traceId` carried on `AgentRun` for Tempo↔Langfuse correlation (ADR 0015).
- REQ-A5-10: All seven ARK CRDs are subject to Gatekeeper admission and the OPA decision points enumerated in §6.6; admission/deny is itself audited.
- REQ-A5-11: A5 ships an ARK Headlamp plugin providing cross-tenant agent visibility, integrated with the A9/A22 framework.
- REQ-A5-12: A5 contributes an ARK HolmesGPT toolset (agent/run lifecycle queries and ARK-state inspection) per §6.10.
- REQ-A5-13: A5 owns the CRD versioning lifecycle (new `vN` group + conversion webhooks, ≥1-minor deprecation window) for its seven CRDs per ADR 0030.
- REQ-A5-14: A5 designs and documents the Knative trigger flow it participates in: which `platform.lifecycle.*` events it emits and that `AgentRun`-creating Triggers (AlertManager→HolmesGPT path included via B8) expect.
- REQ-A5-15: A5 delivers per-product docs, an operator runbook, component-failure alert rules, and a Grafana dashboard delivered as a `GrafanaDashboard` Crossplane XR.

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** isolation is namespace-based (ADR 0001/0016); ARK has no ARK-native per-tenant RBAC, so tenant boundaries are enforced by namespace + Gatekeeper admission + OPA + the Headlamp visibility plugin. Cross-tenant agent visibility is admin-only through the plugin.
- **Observability (§6.5):** ARK emits OTel traces/metrics; `traceId` on `AgentRun` correlates ARK spans with LiteLLM/Langfuse (ADR 0015). Component-failure alerts required (reconcile errors, stuck `AgentRun` state).
- **Scale:** reconcile throughput must keep up with the warm-pool-backed agent placement model (A6); long-running agents' checkpoints persist in Postgres via Letta, not in ARK memory.
- **Versioning (ADR 0030):** per-component; A5-owned; conversion webhooks on breaking CRD changes; `vN-1` deprecated ≥1 minor release. Platform SDK keeps a pinned compatibility matrix against ARK (ADR 0001).
- **Bootstrap tolerance:** ARK lands W1; if A18 audit endpoint or full OPA decision points are not yet wired, A5 degrades to local audit buffering / admission-only gating and is rewired as those land (consistent with the platform's phased posture).

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm values / manifests in Git | Applicable — pinned ARK chart |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable |
| Backup / restore | N/A — ARK holds no system-of-record state (agent state in Postgres via A10/A18); CRDs reconciled from Git |
| Alert rules | Applicable — reconcile failures, stuck `AgentRun`, webhook errors |
| Grafana dashboard (Crossplane `GrafanaDashboard` XR) | Applicable |
| Headlamp plugin | Applicable — cross-tenant agent visibility (ADR 0001) |
| OPA / Rego integration | Applicable — admission for all seven CRDs; targets contributed to B16 |
| Audit emission (ADR 0034) | Applicable — all seven CRD lifecycle actions |
| Knative trigger flow | Applicable — `platform.lifecycle.*` emit; `AgentRun`-creation Triggers (via B8) |
| HolmesGPT toolset | Applicable — ARK toolset (§6.10) |
| 3-layer tests (Chainsaw/Playwright/PyTest) | Applicable — Chainsaw for CRD reconcile, Playwright for Headlamp plugin, PyTest for toolset/adapter logic |
| Tutorials & how-tos | Applicable |

## 9. Acceptance Criteria

- AC-A5-01 (REQ-A5-01): A clean ArgoCD sync installs ARK at the pinned version; `kubectl get crd` shows the seven ARK CRDs registered. 
- AC-A5-02 (REQ-A5-02): All seven CRDs are namespaced (`kubectl api-resources --namespaced=true` lists them; none appear cluster-scoped).
- AC-A5-03 (REQ-A5-03): Applying an `Agent` with a valid `sandboxTemplateRef` results in a Sandbox request to A6 and a running agent pod; no `Sandbox` CRD is reconciled by ARK.
- AC-A5-04 (REQ-A5-04): `Agent` with `sdk: deep-agents` and with `sdk: langgraph` both reconcile; `Agent` with an unknown `sdk` value is denied at admission with a policy reason.
- AC-A5-05 (REQ-A5-05): An `Agent` referencing a `Memory` whose `MemoryStore.accessMode` is `private` cannot be made to read another agent's store via any `Agent` field; the binding resolves and access mode is unchanged by A5.
- AC-A5-06 (REQ-A5-06): An `AgentRun` created by a simulated B8 adapter and by an Argo Workflow step is both reconciled and reaches a terminal `state`.
- AC-A5-07 (REQ-A5-07): For each of create/update/delete on each CRD, exactly one structured audit event is observed at the audit endpoint (or buffered when endpoint absent); no direct DB/S3/OpenSearch write originates from ARK.
- AC-A5-08 (REQ-A5-08): Every CloudEvent ARK emits has `type` under `platform.lifecycle.*` and carries non-empty `specversion` and `schemaVersion`; no event uses another top-level namespace.
- AC-A5-09 (REQ-A5-09): A traced `AgentRun` produces ARK spans in Tempo sharing the run's `traceId`, joinable to the Langfuse trace.
- AC-A5-10 (REQ-A5-10): Submitting an OPA-violating CRD of each of the seven kinds is denied at admission, and each denial emits an audit event.
- AC-A5-11 (REQ-A5-11): The ARK Headlamp plugin loads in the A9 framework and lists `Agent`/`AgentRun` across namespaces for an admin identity, hidden for a non-admin.
- AC-A5-12 (REQ-A5-12): The ARK HolmesGPT toolset answers an agent/run-lifecycle query against live ARK state.
- AC-A5-13 (REQ-A5-13): A simulated breaking change to one CRD ships as a new `vN` group with a working conversion webhook and `vN-1` still served.
- AC-A5-14 (REQ-A5-14): The trigger-flow doc enumerates emitted `platform.lifecycle.*` events and the `AgentRun`-creation Trigger contract, and a wired AlertManager→HolmesGPT path (via B8) creates an `AgentRun`.
- AC-A5-15 (REQ-A5-15): Docs, runbook, alert rules, and a `GrafanaDashboard` XR exist and the dashboard renders ARK reconcile/error metrics.

## 10. Risks & Open Questions

- R-A5-1 (med): ARK is upstream technical-preview — API churn could break pinned assumptions. Mitigation: version pin + external enforcement perimeters + planned upstream contributions (ADR 0001; §9).
- R-A5-2 (med): `Tool` and `Query` field sets are not specified in source. Mitigation: defer to ARK upstream; do not invent (interface-contract §6). Open question: do platform OPA admission policies for `Tool`/`Query` need fields not yet exposed? — resolve with B16 once upstream shape is pinned.
- R-A5-3 (low): per-event-type `platform.lifecycle.*` names are deferred to B12; A5 must not coin them. `[PROPOSED]` flag carried until B12 registers them.
- R-A5-4 (med): no ARK-native per-tenant RBAC — over-broad reconciler permissions are a tenancy risk. Mitigation: namespace isolation + Gatekeeper + OPA-as-restrictor (ADR 0018); flagged into B22 threat model (compromised-tenant, capability-escape classes).
- R-A5-5 (low): A18 audit endpoint and full OPA points land after A5 (W1). Bootstrap degradation path must be tested. Reconciliation note: phased rewire mirrors the HolmesGPT trajectory (ADR 0012).
- Open question (low): does ARK expose its own admin HTTP surface requiring `/v1/...` versioning (§3.3)? `[PROPOSED — not in source]`; confirm at install.

## 11. References

- architecture-overview.md §6.2 (agent runtime, lines 247–292), §6.5 (observability, 370+), §6.6 (audit/OPA points, 511–542), §6.7 (eventing, 553+), §6.9 (multi-tenancy, 720+), §6.12 (CRD inventory, 942+), §6.13 (versioning, 981+), §9 (OSS limitations), §14.1 (Workstream A deliverables, 1647–1690).
- ADR 0001 (ARK as agent operator); ADR 0030 (CRD/API versioning); ADR 0031 (CloudEvent taxonomy); ADR 0034 (audit adapter); ADR 0002 (OPA/Gatekeeper); ADR 0019 (LangGraph/Deep Agents SDK); ADR 0025 (memory access modes); ADR 0015 (Tempo+Langfuse correlation).
- interface-contract §1.1–1.2 (versioning + ARK CRDs), §2 (CloudEvent taxonomy), §3.1 (Platform SDK), §6 (gaps left to component design).
- glossary (ARK, Platform Agent, CRD inventory). Related pieces: A6, A10, A1, B13, B6, B7, B16, B8, A18.

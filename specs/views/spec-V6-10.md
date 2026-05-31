# SPEC V6-10 — Platform self-management with HolmesGPT [PROPOSED]

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0012, 0017, 0022, 0038, 0039, 0018, 0031] · views: [6.10]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's self-management slice: the
architectural commitment that **HolmesGPT is itself a Platform Agent** (an `Agent` CRD), administered
like any other agent, with broad **read-only** access to platform state and **policy-gated** write
access for remediation (§6.10, ADR 0012). It lands **very early** — before the full observability
stack is wired — and is progressively rewired through the platform layers (LiteLLM, audit adapter,
OPA decision points) and grows toolsets as each component lands.

The slice is realized by several components — HolmesGPT itself in its early phase (A14), the Coach
Component for self-improvement (B10), and the Interactive Access Agent that shares the Knowledge Base
(A16). This SPEC does **not** build any of them; it fixes the invariants (HolmesGPT is an ordinary
Platform Agent subject to the same gateway, audit, and policy perimeter), the spanning three-state
OPA permission model, the toolset-contribution convention, the A2A interface it exposes, and the
phased read-only-first deployment trajectory each participating component must honor.

## 2. Scope

### 2.1 In scope
- The invariant that HolmesGPT is a Platform Agent (`Agent` CRD), with its LLM calls through LiteLLM,
  audit through the audit adapter, and tool authorization under OPA — same perimeter as any agent.
- The invariant that the **Knowledge Base (`platform-knowledge-base` RAGStore) is a separate
  primitive HolmesGPT *uses*, not part of HolmesGPT** — any agent including it in its CapabilitySet
  (e.g. the Interactive Access Agent) has the same access.
- The spanning **three-state OPA permission model** (allowed / upon-approval / denied) for
  agent-or-admin-proposed platform actions, with conditions, routed to the generalized approval
  system on `upon-approval`.
- The **toolset-contribution convention** — each component contributes a HolmesGPT toolset as it
  lands; these contribution edges are non-blocking.
- HolmesGPT's exposed **A2A interface** and the phased read-only-first deployment trajectory.

### 2.2 Out of scope (and where it lives instead)
- HolmesGPT install + early-phase ServiceAccount / IAM wiring — **component A14**.
- The Coach Component (self-retrospection, PR/suggestion-card authoring) — **component B10**.
- The Interactive Access Agent build — **component A16**.
- The generalized approval system (`Approval` CRD, Argo Workflow, OPA elevation) — **component B19** /
  ADR 0017; this view only references the `upon-approval` hand-off point.
- The Knowledge Base RAGStore as a primitive — **V6-04 / ADR 0022**; the indexing pipeline — **C8**.
- The policy simulator and Headlamp policy/CRD editors — **ADR 0038 / 0039**; this view references
  them as the authoring surface, not their build.
- Per-toolset method surfaces and per-event-type schemas — owning components / **B12 registry**.

## 3. Context & Dependencies

Realizing components and what each contributes to the view:
- **A14** (HolmesGPT, early phase) — the self-management Platform Agent; lands very early on the raw
  cluster (read-only SA + read-only AWS IAM, no secret-store access), then rewired through the
  platform layers as they land.
- **B10** (Coach Component) — the self-improvement Platform Agent; runs on schedule, observes
  traces/audit, proposes prompt/skill changes via PR or suggestion card.
- **A16** (Interactive Access Agent) — the general-purpose chat-facing Platform Agent that shares the
  same Knowledge Base RAGStore.

ADR decisions honored:
- **ADR 0012** — HolmesGPT as a first-class Platform Agent; three-state OPA permission model; phased
  read-only-first trajectory; toolset contribution.
- **ADR 0017** — `upon-approval` actions route to the generalized approval system.
- **ADR 0022** — the Knowledge Base is a separate primitive.
- **ADR 0038 / 0039** — policy simulator and Headlamp policy/CRD editors as the authoring surface for
  the OPA bundles that gate HolmesGPT.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor; write scope is OPA-restricted over an RBAC floor.
- **ADR 0031** — actions/decisions flow under `platform.policy.*`, `platform.approval.*`,
  `platform.audit.*`.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Spanning CRDs (Canon; all namespaced):
- `Agent` (owner A5) — HolmesGPT, the Coach Component, and the Interactive Access Agent are each an
  `Agent`. Relevant fields: `capabilitySetRefs[]` (includes the Knowledge Base RAGStore),
  `exposes` (HolmesGPT exposes an A2A interface), `modelRef`, `triggers`.
- `RAGStore` (owner B13) — `platform-knowledge-base`, included via CapabilitySet, shared between
  HolmesGPT and the Interactive Access Agent.
- `Approval` (owner B19) — the `upon-approval` hand-off target; `requestingAgent`, `actionType`,
  `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`.

OPA bundles are policy artifacts, **not** platform CRDs, but share the same authoring discipline
(schema-aware validation, diff-against-Git preview, PR-then-ArgoCD apply).

### 4.2 APIs / SDK surfaces
- **HolmesGPT A2A interface** — other Platform Agents hand off troubleshooting questions; calls go
  through LiteLLM and are subject to the same auth + audit as any agent call.
- **Toolset-contribution convention** — each component contributes a toolset (LiteLLM, ARK,
  observability [Tempo + Mimir + Loki + Langfuse], OPA read-only, Sandbox, Eventing, …) as it lands;
  these are non-blocking fan-out edges (waves.md).
- **Three-state OPA decision API** — `allowed` / `upon-approval` / `denied` with attached conditions
  (e.g. "only during business hours", "only for objects matching this label selector").

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031; single owner per namespace, QN-03)
- Consumed: alerts are delivered **via the bus** under `platform.observability.*` (owner A13) as one
  of the two initial v1.0 trigger flows (§6.7) — bus-mediated, no direct AlertManager → HolmesGPT
  trigger. HolmesGPT pulls metrics from Mimir **directly** (data stream, not a bus event).
- Emitted: remediation decisions and policy outcomes under `platform.policy.*` (owner A7); approval
  requests under `platform.approval.*` (owner B19); all actions audited under `platform.audit.*`
  (owner A18). When HolmesGPT detects a security-relevant event it also emits under
  `platform.security.*` (owner A7).

### 4.4 Data schemas / connection-secret contracts
N/A — this view introduces no substrate-backed primitive. HolmesGPT's early-phase AWS reads use a
read-only IAM role with **no Secrets Manager / secret-store access**; later phases use the standard
identity-federation path (V6-11).

## 5. OSS-vs-Custom Decision
N/A — VIEW. HolmesGPT is the upstream Robusta HolmesGPT project configured as a Platform Agent
(A14 / ADR 0012); the Coach Component (B10) and Interactive Access Agent (A16) carry their own
OSS-vs-custom decisions.

## 6. Functional Requirements

- **REQ-V6-10-01** (invariant): HolmesGPT MUST be declared as an `Agent` CRD and administered like
  any other Platform Agent; there is no privileged out-of-band administration path.
- **REQ-V6-10-02** (invariant): Once the corresponding platform layers have landed, HolmesGPT's LLM
  calls MUST route through LiteLLM, its audit emission through the audit adapter / endpoint, and its
  tool authorization through OPA decision points — the same perimeter as any agent.
- **REQ-V6-10-03** (constraint): HolmesGPT's default access MUST be **read-only**; write actions
  (auto-PRs, suggestion cards, autonomous remediations) MUST be policy-controlled and MUST expand
  only incrementally as trust in each remediation pattern is established.
- **REQ-V6-10-04** (constraint): Any platform action a Platform Agent or admin proposes MUST resolve
  through the **three-state OPA model** to exactly one of `allowed`, `upon-approval`, or `denied`,
  with any attached conditions honored; `denied` rejections MUST be audited.
- **REQ-V6-10-05** (constraint): An `upon-approval` decision MUST hold the action pending human
  approval through the generalized approval system (ADR 0017 / B19), not execute it.
- **REQ-V6-10-06** (invariant): The Knowledge Base (`platform-knowledge-base` RAGStore) MUST be a
  separate primitive included via CapabilitySet; any agent including it (e.g. the Interactive Access
  Agent) gets the same access. The Knowledge Base MUST NOT be built into HolmesGPT.
- **REQ-V6-10-07** (constraint): HolmesGPT MUST expose an **A2A interface** reachable through LiteLLM,
  subject to the same auth and audit as any other agent call.
- **REQ-V6-10-08** (constraint): Each platform component MUST be able to contribute a HolmesGPT
  toolset as it lands; toolset-contribution edges are non-blocking and MUST NOT set build-wave depth.
- **REQ-V6-10-09** (constraint): In its first deployment phase HolmesGPT MUST run with a read-only
  ServiceAccount, a read-only AWS IAM role, and **no secret-store access**, then be progressively
  rewired through the platform layers without losing the read-only default.
- **REQ-V6-10-10** (constraint): HolmesGPT's read access to the OPA policies that gate it is
  intentional (loophole analysis); the accepted self-referential risk MUST be documented, not
  silently relied upon.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** HolmesGPT operates with broad cross-namespace **read** access for
  diagnostics; **write** scope remains namespace/label-scoped under OPA conditions. Tenant-scoped
  audit boundaries (V6-09) still apply to its emissions.
- **Security:** read-only-first is the security posture; the three-state model + approval system is
  the write-elevation control; the self-referential-risk trade-off is explicitly accepted (§6.10).
- **Observability (§6.5):** HolmesGPT is both a consumer (observability toolset) and a subject (its
  actions audited like any agent). Trace correlation via `trace_id` (ADR 0015) applies.
- **Versioning (ADR 0030):** the `Agent`/`RAGStore`/`Approval` CRDs follow their owners' versioning.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. Realized by components; HolmesGPT toolset, OPA/Rego integration, audit emission, Knative
trigger flow (bus-mediated alert consumption under `platform.observability.*`, owner A13 — no direct AlertManager → HolmesGPT wire), Headlamp suggestion cards belong to A14/B10/A16 and the
policy/approval owners, verified end-to-end in PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-10-01** (→REQ-01/02): HolmesGPT exists as an `Agent` CRD; after the relevant layers land,
  its LLM traffic is observable through LiteLLM, its audit records flow through the audit adapter, and
  its tool calls hit OPA — with no out-of-band admin path.
- **AC-V6-10-02** (→REQ-03/04): A proposed write action resolves to one of allowed / upon-approval /
  denied; a denied action is rejected and an audit record is emitted.
- **AC-V6-10-03** (→REQ-05): An `upon-approval` action creates an `Approval` and does not execute
  until a human decision is recorded.
- **AC-V6-10-04** (→REQ-06): The Interactive Access Agent and HolmesGPT, both including the Knowledge
  Base in their CapabilitySet, resolve to the same `platform-knowledge-base` RAGStore; removing the
  Knowledge Base from HolmesGPT's CapabilitySet removes its access without rebuilding HolmesGPT.
- **AC-V6-10-05** (→REQ-07): A peer Platform Agent can hand off a question to HolmesGPT over A2A
  through LiteLLM, and the call is audited.
- **AC-V6-10-06** (→REQ-09): A first-phase HolmesGPT deployment with a read-only SA + read-only AWS
  IAM and no secret-store binding can perform diagnostics but cannot write or read secrets.

## 10. Risks & Open Questions
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review; it coins no new names.
- **(med)** Self-red-team: HolmesGPT's read access to its own gating OPA policies is a deliberate
  self-referential risk — accepted in §6.10 because discovery beats latent vulnerability; mitigated
  by the read-only default (REQ-03) and the three-state write gate (REQ-04). Flagged, not eliminated.
- **(med)** The boundary between "narrow remediation executes autonomously" (`allowed`) and
  `upon-approval` is policy-design-time (B16/OPA bundles); this view fixes only the three-state
  contract, not the per-action mapping.
- **(low)** Toolset method surfaces per contributing component are deferred to those components; this
  view fixes only the contribution convention and its non-blocking nature.
- **(low)** Whether the Coach Component shares HolmesGPT's toolsets or maintains its own is a B10
  design choice; not fixed here.

## 11. References
- architecture-overview.md §6.10 (HolmesGPT self-management, ~L774–835): three-state model
  (~L821–827), phased trajectory (~L829–835); §6.7 (bus-mediated alert trigger flow under `platform.observability.*`, owner A13 — no direct AlertManager → HolmesGPT wire).
- ADR 0012 (HolmesGPT first-class agent + three-state + phasing), ADR 0017 (approval system),
  ADR 0022 (Knowledge Base primitive), ADR 0038 (policy simulator), ADR 0039 (Headlamp editors),
  ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0031 (CloudEvent taxonomy).
- Realizing components: A14, B10, A16 (+ B19 approval, A7/B16 policy). Related views: V6-04
  (Knowledge Base), V6-06 (security/policy), V6-11 (identity federation), V6-12 (CRD inventory).

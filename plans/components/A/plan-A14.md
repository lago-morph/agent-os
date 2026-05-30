# PLAN A14 — HolmesGPT (early-phase platform self-management agent)

> spec: SPEC-A14 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: L
> upstream-pieces: [] · downstream-pieces: [A16, B10]

## 1. Implementation Strategy

Ship HolmesGPT in two clearly separated states and a documented migration between them. **Phase 1** lands on the raw Kubernetes cluster in the foundation wave — plain Helm install + read-only ServiceAccount + read-only AWS IAM + no secret-store access — so it is useful for diagnostics during the platform build itself (ADR 0012; waves.md). **Phase N** progressively rewires HolmesGPT through the platform layers as they land: redeclare it as an `Agent` CRD (A5), move LLM calls behind LiteLLM (A1), audit through the adapter/endpoint (A18), tool authorization under the central OPA decision point (A7), A2A through the gateway, upon-approval actions through B19. Read-only is the invariant default at every step. Critical path for v1.0 value: phase-1 deploy → AlertManager trigger flow → KB access → three-state OPA gating → rewiring runbook. Build with fakes for every later component so phase-1 ships without them.

## 2. Ordered Task List

- TASK-01: Phase-1 install — pin HolmesGPT version; Helm + read-only cluster ServiceAccount + read-only AWS IAM role; assert no secret-store access — produces: raw-cluster diagnostic deployment — depends-on: []
- TASK-02: Wire AlertManager → HolmesGPT Knative trigger flow (CloudEvent source + Trigger filter for diagnostic-relevant alerts → dispatch) — produces: initial trigger flow — depends-on: [TASK-01]
- TASK-03: Include the Knowledge Base RAGStore in the CapabilitySet; reach it via `rag.*` (fake SDK/LiteLLM until landed) — produces: KB-grounded diagnostics — depends-on: [TASK-01]
- TASK-04: Redeclare HolmesGPT as an `Agent` CRD with A2A `exposes` once A5 is available — produces: ARK-managed HolmesGPT — depends-on: [TASK-01]
- TASK-05: Implement three-state OPA gating (allowed/upon-approval/denied) at one central decision point; route upon-approval to B19 — produces: action-gating path — depends-on: [TASK-04]
- TASK-06: Implement write surface — auto-PR + Headlamp suggestion cards; gate autonomous remediation on OPA-graduated patterns — produces: v1.0 write surface — depends-on: [TASK-05]
- TASK-07: Implement `LogLevel` diagnostics-toggle request through the OPA-gated path — produces: dynamic toggle driver — depends-on: [TASK-05]
- TASK-08: Build the toolset-registration/aggregation surface (LiteLLM/ARK/observability/OPA/sandbox/eventing) + policy-simulator skill — produces: toolset framework — depends-on: [TASK-04]
- TASK-09: Author the **rewiring runbook** — LLM→LiteLLM, audit→adapter, tools→OPA, A2A→gateway; each step independently shippable + reversible; read-only default preserved — produces: phased-migration procedure — depends-on: [TASK-04, TASK-05]
- TASK-10: Audit emission wiring (`platform.audit.*` actions; `platform.policy.*` decisions) post-rewire — produces: audit/policy events — depends-on: [TASK-09]
- TASK-11: 3-layer tests, docs, alerts, `GrafanaDashboard` XR, suggestion-card plugin, tutorials/how-tos — produces: full deliverable set — depends-on: [TASK-02, TASK-06, TASK-07, TASK-08, TASK-10]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- **None** — A14 is foundation-wave (W0); phase-1 has no internal dependencies (waves.md, CSV upstream empty).

### 3.2 Downstream pieces blocked on this
- **A16** (Interactive Access Agent — shares the KB), **B10** (Coach — consumes HolmesGPT + observability).

### 3.3 Continuous (non-blocking) inputs
- Dotted toolset-contribution edges `A13 -.-> A14`, `A2 -.-> A14` (observability toolsets, non-blocking per waves.md). Rewiring inputs acquired over time: A5, A1, A18, A7, B19, B8 (AlertManager adapter), `LogLevel` (ADR 0035). B14 test framework, B22 threat model.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 and TASK-03 run concurrently (both phase-1).
- After TASK-04: TASK-05 and TASK-08 run concurrently.
- After TASK-05: TASK-06, TASK-07 concurrent; TASK-09 follows.
- TASK-11 fans in.

## 5. Test Strategy

| AC | Layer | Notes / fixtures |
|---|---|---|
| AC-A14-02 (Agent CRD), AC-A14-08 (trigger flow) | Chainsaw | `Agent` reconcile (fake A5 early), AlertManager CloudEvent → Trigger → dispatch (fake B8 source) |
| AC-A14-05 (suggestion cards), AC-A14-13 (plugin) | Playwright | Headlamp suggestion-card UI + B19 approval workflow stub |
| AC-A14-01/03/04/06/07/09/10/11/12 | PyTest | RBAC/IAM read-only assertions, A2A handoff (fake LiteLLM), three-state decision logic, KB `rag.*` (fake), rewiring-step verification, `LogLevel` gating, audit/policy emission, simulator skill, toolset registration |

Fakes for not-yet-landed upstreams: A5 ARK, A1 LiteLLM + B2 callbacks, A18 audit endpoint, A7 OPA decision point + B16 bundle, B19 Approval, B8 AlertManager adapter, B6 SDK `rag.*`, KB RAGStore.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0` (foundation; no upstream specs required)
### 6.2 PR — `piece/A14-holmesgpt-early-phase` → base `wave/0`; carries SPEC-A14 + PLAN-A14
### 6.3 Merge order — independent of W0 siblings; the rewiring tasks land incrementally in later waves as their target layers ship, but the spec/plan land in W0.

## 7. Effort Estimate

- S: TASK-01, TASK-03, TASK-07. M: TASK-02, TASK-04, TASK-05, TASK-06, TASK-08, TASK-10. L: TASK-09 (rewiring runbook spans many components), TASK-11.
- Rollup: **L** (matches CSV).
- Critical path (v1.0 value): TASK-01 → 02 → (03) → 04 → 05 → 09 → 11.

## 8. Rollback / Reversibility

Phase-1 rollback: remove the Helm release, SA, and IAM role — no durable state lost (HolmesGPT holds no system of record). Each rewiring step is independently reversible: a failed rewire reverts to the prior connectivity (e.g. LLM calls fall back ahead of LiteLLM only in a controlled bootstrap window) while keeping read-only default. Reverting A14 entirely breaks downstream diagnostics handoffs (A16 shares only the KB, so it survives; B10 Coach loses a data source). The highest-risk reversal is a rewiring step that inadvertently widens write scope — gate every write-scope change behind the policy simulator and the three-state OPA model before promotion.

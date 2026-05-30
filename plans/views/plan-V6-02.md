# PLAN V6-02 — Agent runtime architecture `[PROPOSED]`

> spec: SPEC-V6-02 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map follows the wave layering: **A6 (agent-sandbox + Envoy egress)** lands first in W0 as the isolation + egress perimeter; **A5 (ARK)** lands in W1 (depends on A6) as the agent operator reconciling `Agent`→pod; **B6 (Platform SDK)** lands in W2 as the harness↔platform contract surface; **B7 (agent SDK: LangGraph + Deep Agents)** lands in W3 on top of B6/A6/A5. The view holds when the reconcile-chain, sandbox-isolation, two-path-egress, SDK-surface-only, and external-enforcement invariants all hold end-to-end. Verification traces one agent from `Agent` CRD apply to a sandboxed pod whose only outbound paths are LiteLLM and Envoy, plus negative tests (bypass attempts, non-allowlisted egress, unsupported-harness governance).

## 2. Ordered Task List
- **TASK-01:** Confirm A6 provides gVisor/Kata isolation, warm pool, hibernation, and the FQDN-allowlisted Envoy egress proxy — produces: isolation+egress conformance checklist — depends-on: []
- **TASK-02:** Confirm A5 reconciles `Agent`→`Sandbox`→pod and owns `Agent`/`AgentRun`/`Team`/`Tool`/`Memory`/`Evaluation`/`Query` lifecycle — produces: reconcile-chain checklist — depends-on: [TASK-01]
- **TASK-03:** Confirm B6 exposes only `memory.*`/`rag.*`/OTel/A2A-registration as the harness contract, terminating at LiteLLM — produces: SDK-surface checklist — depends-on: [TASK-02]
- **TASK-04:** Confirm B7 sets `Agent.sdk` ∈ {`langgraph`,`deep-agents`} and preserves additive-enrollment shape — produces: SDK-enrollment checklist — depends-on: [TASK-02, TASK-03]
- **TASK-05:** Define end-to-end view verification: reconcile chain, isolation, two-path egress, SDK-only access, unsupported-harness governance — produces: view acceptance suite mapping (AC-01..07) — depends-on: [TASK-03, TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A6 — sandbox isolation + Envoy egress (consumed: gVisor/Kata, FQDN allowlist).
- A5 — ARK reconcile chain (consumed: `Agent`/`Sandbox` reconciliation, lifecycle CRDs).
- B6 — Platform SDK surface (consumed: `memory.*`/`rag.*`/OTel/A2A helpers).
- B7 — agent SDK (consumed: `Agent.sdk` value set, harness shape).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; B17, B18, B20, B21 build on the runtime per CSV but bind to component specs).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat model (sandbox-escape, capability-escape, A2A-lateral-movement patterns shape REQ-02/03/07).

## 4. Parallelizable Subtasks
- TASK-01 is the root. TASK-03 and TASK-04 partially overlap once TASK-02 is green (TASK-04 also needs TASK-03 for the SDK-surface dependency). Fan-out is shallow here because the runtime chain is mostly serial (A6→A5→B6→B7).

## 5. Test Strategy
- **Chainsaw (operator/CRD):** `Agent`→pod reconcile via ARK (AC-01); `Sandbox` lifecycle owned by agent-sandbox not ARK (AC-02); `sdk` value admission for langgraph/deep-agents + additive enrollment (AC-05).
- **Playwright (UI/e2e):** none primary (runtime is non-UI); optional Headlamp inspection of sandbox/agent state.
- **PyTest (logic):** non-allowlisted egress blocked at Envoy / LLM-MCP-A2A via LiteLLM (AC-03); SDK-only access to memory/RAG/model/A2A (AC-04); skill served from Git and absent from UI (AC-06); unsupported-harness still governed (AC-07).
- Fixtures/fakes: fake `EgressTarget` set; stub OPA allow/deny for egress + registration; fake LiteLLM endpoint; minimal conformant non-supported harness image.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-02-agent-runtime-architecture` → base `wave/auth`; carries spec-V6-02.md + plan-V6-02.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 M. Rollup: S (authoring). Critical path: TASK-01 → TASK-02 → TASK-03 → TASK-04 → TASK-05.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A5/A6/B6/B7) carry the enforceable change. No downstream code breaks from reverting the view itself.

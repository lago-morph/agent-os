# PLAN A16 — Interactive Access Agent

> spec: SPEC-A16 · kind: COMPONENT · tier: T1
> wave: W2 · estimate: M
> upstream-pieces: [A1, A14] · downstream-pieces: []

## 1. Implementation Strategy

A16 is built by **assembling platform primitives**, not by installing a new product: an `Agent` CRD (general-purpose, chat-facing) + a CapabilitySet that includes the `platform-knowledge-base` RAGStore, wired as LibreChat's default endpoint. The work is the agent definition, the KB-access path via `rag.*`/LiteLLM, the upload→`Memory` wiring (with A8 owning the LibreChat side), and the claim-driven endpoint visibility. The guiding constraint is that A16 must be a working default endpoint as early as the surrounding waves allow (A1@W0, ARK/KB/B13@W1), so LibreChat reaches LLMs/KB through the governed path from the start. Critical path: Agent+CapabilitySet definition → LiteLLM call path + KB access → LibreChat default-endpoint wiring → upload/memory + OPA gating → deliverables. Build with fakes for A8, B13, B6, and the KB RAGStore until they land.

## 2. Ordered Task List

- TASK-01: Author the A16 `Agent` CRD (general-purpose, chat-facing) with `sdk` and a CapabilitySet including `platform-knowledge-base` — produces: agent definition — depends-on: []
- TASK-02: Wire A16's LLM/RAG/MCP/A2A traffic through LiteLLM (A1); confirm OPA/audit/Langfuse callbacks fire on its call path — produces: governed call path — depends-on: [TASK-01]
- TASK-03: Implement KB access via SDK `rag.*` through LiteLLM (fake B6/RAGStore until landed) — produces: KB-grounded answering — depends-on: [TASK-02]
- TASK-04: Wire A16 as the default LibreChat endpoint (coordination with A8); claim-driven visibility in the endpoint picker — produces: default-endpoint behavior — depends-on: [TASK-01]
- TASK-05: Wire LibreChat uploads → `Memory` (accessMode-governed) exposed to A16 (A8 owns the LibreChat upload side) — produces: conversation/upload memory — depends-on: [TASK-04]
- TASK-06: Author OPA integration — CapabilitySet admission scoping, LibreChat agent-pick gating, runtime LiteLLM gating; contribute targets to B16 — produces: policy gating — depends-on: [TASK-02, TASK-04]
- TASK-07: Audit + lifecycle emission (`platform.audit.*` via callbacks/adapter, `platform.lifecycle.*` via ARK runs) — produces: observability events — depends-on: [TASK-02]
- TASK-08: HolmesGPT toolset contribution (KB-query / conversation-health) — produces: toolset — depends-on: [TASK-03]
- TASK-09: 3-layer tests, docs, runbook, alerts, `GrafanaDashboard` XR, tutorials/how-tos — produces: full deliverable set — depends-on: [TASK-03, TASK-05, TASK-06, TASK-07, TASK-08]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- **A1** (LiteLLM) — the governed call path for all A16 traffic.
- **A14** (HolmesGPT) — `[PROPOSED]` CSV edge; reconciled in spec §3 as a **shared-KB / A2A-handoff peer**, not a hard call dependency. Treated as soft for build ordering.

### 3.2 Downstream pieces blocked on this
- None in CSV.

### 3.3 Continuous (non-blocking) inputs
- Effective (not CSV edges): A5 (ARK `Agent`), A8 (LibreChat default-endpoint), B13 (CapabilitySet→LiteLLM), B4 (KB RAGStore), B6 (SDK `rag.*`) — consumed as they land. B14 test framework, B22 threat model.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 and TASK-04 run concurrently.
- After TASK-02/04: TASK-03, TASK-05, TASK-06, TASK-07 run largely concurrently.
- TASK-08 follows TASK-03; TASK-09 fans in.

## 5. Test Strategy

| AC | Layer | Notes / fixtures |
|---|---|---|
| AC-A16-01 (admit), AC-A16-06/08 (sandbox/admission) | Chainsaw | `Agent`/CapabilitySet reconcile + admission; fake A5/B13 |
| AC-A16-02/09 (default endpoint), AC-A16-03/05 (KB answer, upload) | Playwright | LibreChat e2e against A16 (fake/real A8); claim-based visibility |
| AC-A16-04/07/10 | PyTest | egress lockdown (Envoy `EgressTarget`), audit/lifecycle namespace+version, `rag.*`/memory logic, deliverable presence |

Fakes for not-yet-landed upstreams: A8 LibreChat (or stub endpoint picker), B13 CapabilitySet reconcile, B6 SDK `rag.*`/`memory.*`, KB `RAGStore`, A18 audit sink, A7/B16 OPA bundle, A14 A2A peer.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/2` (rolls up A1@W0, A14@W0, and W1 band incl. A5/B13/B4/A10)
### 6.2 PR — `piece/A16-interactive-access-agent` → base `wave/2`; carries SPEC-A16 + PLAN-A16
### 6.3 Merge order — independent of W2 siblings (A17, A22, B6, B9, …); `wave/2` rolls up to main.

## 7. Effort Estimate

- S: TASK-01, TASK-08. M: TASK-02, TASK-03, TASK-04, TASK-05, TASK-06, TASK-07, TASK-09.
- Rollup: **M** (matches CSV).
- Critical path: TASK-01 → 02 → 03 → 09 (with TASK-04/05 on the LibreChat-wiring branch joining at 09).

## 8. Rollback / Reversibility

A16 is a declarative `Agent` + CapabilitySet, so rollback is reverting those manifests; no durable state is lost (conversation/upload memory lives in Letta/Postgres, LibreChat state in shared Postgres). The high-impact reversal is removing A16 while it is LibreChat's default endpoint — LibreChat would have no default agent to surface, breaking the day-one chat path; A8 must fall back to another endpoint or A16 must be reverted together with its default-endpoint wiring. No data destruction in any case.

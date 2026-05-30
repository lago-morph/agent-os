# PLAN B10 — Coach Component (self-retrospection)

> spec: SPEC-B10 · kind: COMPONENT · tier: T1
> wave: W3 · estimate: M
> upstream-pieces: [B6, A5, A2, A14] · downstream-pieces: []

## 1. Implementation Strategy
Build the Coach as a **first-class Platform Agent** (`Agent` CRD + `CapabilitySet`) authored on
the supported agent SDK (B7/ADR 0019), not as new infrastructure. The core is **retrospection
logic** that reads Langfuse traces (A2), audit data, and `Evaluation` results, then emits
**proposals as auto-PRs and/or Headlamp suggestion cards** — reusing HolmesGPT's propose-don't-
mutate pattern (A14/ADR 0012). Skill/prompt changes route through the generalized approval system
(B19) as `Approval` requests. Land after Langfuse/ARK/LiteLLM/observability are up (the §14.2
ordering); OPA gating is optional by design-time decision, so wire read-mostly first and leave the
three-state OPA hook as an additive seam.

## 2. Ordered Task List
- **TASK-01:** Specify the Coach's retrospection inputs + proposal contract (what it reads from
  Langfuse/audit/`Evaluation`; PR-vs-card output shape). — produces: design note — depends-on: []
- **TASK-02:** Author the Coach `Agent` CRD + `CapabilitySet` (Langfuse/observability read,
  `platform-knowledge-base` RAGStore, `llmProviders[]`, `sdk` ∈ {langgraph,deep-agents}). — produces:
  manifests — depends-on: [TASK-01]
- **TASK-03:** Implement the scheduled trigger (decide `Agent.triggers` scheduled vs. scheduled CI
  job — R1). — produces: trigger wiring — depends-on: [TASK-02]
- **TASK-04:** Implement retrospection logic (Langfuse read, audit/`Evaluation` ingestion, candidate
  ranking). — produces: Coach agent code — depends-on: [TASK-01, TASK-02]
- **TASK-05:** Implement proposal emission — auto-PR against the agent/prompt/`Skill` repo +
  suggestion-card payload (bind to B5/B19-ui schema). — produces: proposal emitter — depends-on:
  [TASK-04]
- **TASK-06:** Wire skill-change proposals to B19 as `Approval` (requestingAgent=Coach,
  evidenceRefs[]). — produces: approval integration — depends-on: [TASK-05]
- **TASK-07:** Wire audit emission (every run + proposal) + `platform.evaluation.*`/
  `platform.lifecycle.*` events + OTel spans. — produces: observability wiring — depends-on: [TASK-04]
- **TASK-08:** 3-layer tests + docs/runbook/dashboard XR. — produces: tests + docs — depends-on:
  [TASK-05, TASK-06, TASK-07]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A2 (Langfuse)** — trace/eval corpus read API.
- **A5 (ARK)** — reconciles the Coach `Agent`/`AgentRun`.
- **B6 (Platform SDK)** — `memory.*`/`rag.*`/OTel/A2A helpers the harness uses.
- **A14 (HolmesGPT)** — propose-via-PR/suggestion-card pattern reused; optional A2A handoff target.
- (Implicit) **B7** — the agent SDK the Coach runs on; **B19** — the approval system proposals route to.

### 3.2 Downstream pieces blocked on this
- None (`piece-index.csv` lists no downstream).

### 3.3 Continuous (non-blocking) inputs
- **B14** (test framework — three-layer tests), **B22** (security threat model standards),
  **B12** (CloudEvent schemas for `platform.evaluation.*`/`platform.lifecycle.*`).

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03 (trigger) and TASK-04 (logic) run concurrently.
- After TASK-04: TASK-05 (emission) and TASK-07 (observability) run concurrently; TASK-06 follows
  TASK-05.

## 5. Test Strategy
- **Chainsaw:** apply Coach `Agent` → expect scheduled `AgentRun` + audit + `platform.evaluation.*`/
  `platform.lifecycle.*` events; assert **no** direct `Agent`/`Skill` mutation (AC-B10-01,02,05,07,09,11).
- **PyTest:** retrospection logic over fixtures seeding Langfuse traces + audit + `Evaluation`
  results; proposal-generation; approval-request construction (AC-B10-03,04,06,12).
- **Playwright:** suggestion-card flow end-to-end (exercises B5/B19-ui) (AC-B10-05).
- **Fakes:** fake Langfuse API + fake B19 `Approval` sink + fake suggestion-card endpoint until B5/
  B19-ui land; fake Knowledge Base RAGStore (AC-B10-08).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/3` (contains upstream specs A2/A5/B6/A14, B7, B19-core).
### 6.2 PR — `piece/B10-coach-component` → base `wave/3`; carries spec-B10 + plan-B10.
### 6.3 Merge order — independent of W3 siblings (B7, B14, B19-core); wave/3 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 M · TASK-05 M · TASK-06 S · TASK-07 S · TASK-08 M.
- Rollup: **M** (matches CSV). Critical path: TASK-01 → 02 → 04 → 05 → 06 → 08.

## 8. Rollback / Reversibility
Fully reversible: remove the Coach `Agent`/`CapabilitySet` manifests (ArgoCD prunes the agent).
Since the Coach only **proposes** (never mutates), backing it out leaves no half-applied state —
open suggestion PRs/cards simply stop being created; nothing downstream breaks (no downstream
pieces). Any in-flight `Approval` requests resolve or expire via B19 independently.

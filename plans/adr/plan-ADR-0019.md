# PLAN ADR-0019 — LangGraph SDK + Deep Agents default [PROPOSED]

> spec: SPEC-ADR-0019 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B7, B6] · downstream-pieces: [A5, B17, B18, B21, C2, C3, C4]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0019 is enforced primarily by **B7** (the multi-SDK harness shipping LangGraph + Deep Agents, preserving the enrollment shape) and **B6** (the platform SDK binding both behind one API contract with a two-column compatibility matrix). **A5** enforces that ARK reconciles `Agent` regardless of `sdk` value. The default-path obligation is enforced by **B17/B18** (profiles/compositions on Deep Agents) and **C2/C3/C4** (tutorials/how-tos/reference). The third-party-harness contract is documented but the enforcement perimeter (LiteLLM/OPA/Envoy/CRDs) stays external. Conformance is tested by running both SDKs under ARK, asserting `Agent.sdk` admits exactly the two values and rejects others, portable agent code across both SDKs, and the two-column-and-growing matrix.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B7 ships LangGraph + Deep Agents and preserves the multi-SDK harness shape — produces: B7 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify `Agent.sdk` admits {langgraph, deep-agents} and enrollment is additive (no `vN` migration) — produces: A5/CRD conformance note — depends-on: [TASK-01]
- TASK-04: Verify B6 exposes `memory.*`/`rag.*`/OTel/A2A behind one API contract over both SDKs (portability) — produces: SDK-binding trace — depends-on: [TASK-01]
- TASK-05: Verify default-path coverage (tutorials/how-tos/build-first-agent/profiles/compositions on Deep Agents; reference covers both) — produces: docs coverage audit — depends-on: [TASK-01]
- TASK-06: Verify eval/red-team/build pipeline exercise both SDKs; two-column matrix — produces: pipeline trace — depends-on: [TASK-02]
- TASK-07: Verify documented (unsupported) third-party harness contract + external enforcement — produces: harness-contract trace — depends-on: [TASK-02]
- TASK-08: Verify capability-change (ADR 0013) + overlay (ADR 0032) behave identically across both SDKs — produces: cross-ADR trace — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B7 (agent SDK/harness), B6 (platform SDK).
### 3.2 Downstream pieces blocked on this
- A5 (`sdk` field), B17 (profiles), B18 (compositions), B21 (dev env), C2/C3/C4 (docs).
### 3.3 Continuous (non-blocking) inputs
- ADR 0013 (capability notification), ADR 0032 (overlay), ADR 0015 (correlated tracing), B22 threat model.

## 4. Parallelizable Subtasks
TASK-03, TASK-04, TASK-05 fan out once TASK-01 lands. TASK-06/07 follow TASK-02; TASK-08 follows TASK-04.

## 5. Test Strategy
- AC-01 → Chainsaw+PyTest: both a LangGraph and a Deep Agents agent run under ARK and pass tests; no third SDK enrolled.
- AC-02 → Chainsaw: `Agent.sdk` admits the two values, rejects unenrolled; adding a value needs no `vN` migration.
- AC-03 → PyTest: harness hosts an added SDK by config without redesign.
- AC-04 → PyTest doc-lint: sampled tutorials/how-tos/profiles/compositions use Deep Agents; reference covers both.
- AC-05 → PyTest: same agent code on both SDKs using `memory.*`/`rag.*`/OTel/A2A.
- AC-06 → PyTest: eval/red-team/build run both variants; matrix has two columns.
- AC-07 → doc-lint: third-party harness contract doc exists, marked unsupported.
- AC-08 → Chainsaw: compliant third-party image still subject to LiteLLM/OPA/Envoy; non-compliant bounded by perimeter.
- AC-09 → PyTest: capability change + overlay behave identically on both SDKs.
- Fixtures/fakes: stub ARK `Agent` reconcile, LiteLLM/Envoy perimeter for not-yet-landed A5/A1/A6.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0019-langchain-deep-agents-single-sdk` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
M overall (T0, docs + SDK + pipeline fan-out). Per-task: TASK-01 S, 02 M, 03 S, 04 M, 05 M, 06 S, 07 S, 08 S. Critical path: TASK-01 → 02 → 06.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, B7/B6 lose the two-SDK conformance contract and the docs/profile/compositions lose their Deep Agents default anchor; enrolling additional SDKs remains an additive harness change regardless. No runtime artifact is deleted.

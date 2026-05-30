# PLAN ADR-0022 — Knowledge Base as a separate primitive [PROPOSED]

> spec: SPEC-ADR-0022 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A11, B6] · downstream-pieces: [A14, A16, B10, C8, B13]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0022 is enforced by the **`RAGStore` definition** (one instance named `platform-knowledge-base`, reconciled by **B13** into the LiteLLM admin API), by the **consumer CapabilitySets** of A14/A16/B10 (membership gates access), and by the **SDK `rag.*` path** (B6) being the only access route. **C8** owns the ingestion that populates it; **A11** OpenSearch backs it. Conformance is tested by asserting singularity + naming of the store, that CapabilitySet membership is necessary and sufficient for access, that retrieval is traced/OPA-gated on the LiteLLM path, and that LibreChat carries no direct binding.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify a single `RAGStore` named `platform-knowledge-base` is declared and no agent embeds the corpus — produces: store-definition conformance — depends-on: [TASK-01]
- TASK-03: Verify A14/A16/B10 CapabilitySets include the store and access requires membership — produces: CapabilitySet audit — depends-on: [TASK-01]
- TASK-04: Define conformance test for `rag.*` LiteLLM-mediated path (trace + OPA gate) — produces: PyTest+Chainsaw test — depends-on: [TASK-02]
- TASK-05: Verify A16 is default LibreChat endpoint and LibreChat has no direct store binding — produces: A8/A16 trace — depends-on: [TASK-03]
- TASK-06: Verify store `backend` = OpenSearch; no separate substrate — produces: substrate-reuse check — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A11 (OpenSearch substrate); B6 (`rag.*` surface); B13 (RAGStore reconciler).
### 3.2 Downstream pieces blocked on this
- C8 (ingestion pipeline), A16/A14/B10 (consumers).
### 3.3 Continuous (non-blocking) inputs
- ADR 0024 companion project (vendor docs); B22 threat model (access-excess detection).

## 4. Parallelizable Subtasks
TASK-02 and TASK-03 fan out independently after TASK-01. TASK-04/05/06 follow their respective predecessors.

## 5. Test Strategy
- AC-01 → Chainsaw (assert one RAGStore by name) + PyTest (no-embed lint over agent images).
- AC-02 → Chainsaw/PyTest: with/without CapabilitySet membership → access assertion.
- AC-03 → PyTest assert trace emitted on LiteLLM path + OPA decision recorded.
- AC-04 → PyTest/Playwright: A16 CapabilitySet lists store; LibreChat config has no direct binding.
- AC-05 → Chainsaw assert `backend` resolves to OpenSearch.
- Fixtures/fakes: stub OpenSearch index + LiteLLM gateway for not-yet-landed A11/A1.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0022-knowledge-base-primitive` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: 01 S, 02 S, 03 S, 04 M, 05 S, 06 S. Critical path: TASK-01 → 02 → 04.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, C8 and the consuming agents lose the singular-named-store contract and the rag.*-only access guarantee; no runtime artifact is deleted.

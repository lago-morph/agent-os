# PLAN ADR-0005 — Letta as the memory backend [PROPOSED]

> spec: SPEC-ADR-0005 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A11] · downstream-pieces: [B11]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0005 is enforced by component A10 (Letta install behind ARK's `Memory` CRD, Postgres-persisted), with provisioning by B4 (`MemoryStore` XR), insulation by B11 (adapter), and the only agent-facing path by B6 (SDK `memory.*`). Conformance is proven by: (a) a "no direct Letta path" reachability test; (b) Postgres-as-system-of-record durability + OpenSearch reindex tests; (c) a `MemoryStore`-only provisioning check; (d) an SDK version-pin drift check. No new build work belongs to this ADR — it is a constraint map over A10/B11/B4/B6.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify "only via Memory/SDK" reachability test (no direct Letta path) — produces: e2e test (A10/B6) — depends-on: [TASK-01]
- TASK-03: Specify Postgres-system-of-record durability + OpenSearch reindex test — produces: PyTest + Chainsaw (A10/A11) — depends-on: [TASK-01]
- TASK-04: Specify `MemoryStore`-XR-only provisioning + admission-gate check — produces: Chainsaw (B4/A7) — depends-on: [TASK-01]
- TASK-05: Specify B11 backend-swap insulation + SDK version-pin drift check — produces: PyTest + CI gate (B11/B6) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- A11 — OpenSearch retrieval tier Letta indexes land in.
### 3.2 Downstream blocked on this
- B11 (memory backend adapter).
### 3.3 Continuous (non-blocking) inputs
- B4 `MemoryStore` Composition; B6 SDK + compat matrix; ADR 0025 access-mode semantics; B14 test framework; B15 CI.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/05 → PyTest (reachability via `memory.*` only; B11 backend-swap insulation).
- AC-02/03 → PyTest + Chainsaw (Postgres durability across restart; OpenSearch reindex from primary).
- AC-04/07 → Chainsaw (`MemoryStore`-only provisioning; admission deny).
- AC-06 → PyTest (Letta version-pin drift in CI).
Fixtures: fake memory backend behind B11 until A10 lands; stub `MemoryStore` Composition until B4.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0005-letta-memory` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Backing out means re-opening the memory-backend choice (Mem0/Zep/Cognee/LangMem) and re-implementing B11. Because the SDK `memory.*` contract insulates agents, agent code is unaffected; reversibility is moderate. Downstream breakage limited to B11 and `MemoryStore` `backendType`.

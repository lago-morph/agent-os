# PLAN V6-03 — Memory and data architecture `[PROPOSED]`

> spec: SPEC-V6-03 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map follows the wave layering: **A11 (OpenSearch)** lands in W0 as the retrieval-optimization store; **B4 (Crossplane Compositions)** lands in W1 (foundation band) owning the substrate XRDs and the uniform connection-secret contract; **A10 (Letta)** lands in W1 (depends on A11) as the memory service; **B11 (memory backend adapter)** lands in W1 (depends on A10) adapting `memory.*` to Letta. The view holds when the storage-role, reproducibility, access-mode, SDK-only-access, and substrate-abstraction invariants all hold end-to-end. Verification proves Postgres-as-system-of-record by dropping/rebuilding OpenSearch, exercises the three access modes, and confirms identical connection-secret shape across kind and AWS with admission-validated substrate selection.

## 2. Ordered Task List
- **TASK-01:** Confirm A11 OpenSearch serves vectors/hybrid + advisory audit fanout and is never a system of record — produces: retrieval-role checklist — depends-on: []
- **TASK-02:** Confirm B4 substrate XRs (Crossplane v2, namespace-scoped: `Postgres`/`SearchIndex`/`ObjectStore`/`MongoDocStore`) write the uniform connection secret with substrate-agnostic status and label-driven admission-validated selection — produces: substrate-contract checklist — depends-on: []
- **TASK-03:** Confirm A10 Letta persists state to Postgres and indexes to OpenSearch/object-storage per storage roles — produces: memory-service checklist — depends-on: [TASK-01, TASK-02]
- **TASK-04:** Confirm B11 adapts `memory.*` to Letta and `MemoryStore.accessMode` (private/namespace-shared/RBAC-OPA) is enforced — produces: access-mode checklist — depends-on: [TASK-03]
- **TASK-05:** Define end-to-end view verification: drop/rebuild reproducibility, access-mode enforcement, connection-secret parity, admission rejection of unmatched substrate — produces: view acceptance suite mapping (AC-01..09) — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A11 — OpenSearch retrieval store (consumed: vector/hybrid/advisory-audit indexing).
- B4 — substrate XRDs + Compositions (consumed: connection-secret contract, admission selection).
- A10 — Letta memory service (consumed: state persistence to Postgres + indexing).
- B11 — memory adapter (consumed: `memory.*`→Letta binding, access-mode enforcement).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; A18 audit, A21 tenant onboarding, B20 PV access consume B4 XRDs per CSV but bind to component specs).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat model (tenant-data exfiltration, cross-namespace memory sharing patterns shape REQ-03).

## 4. Parallelizable Subtasks
- TASK-01 and TASK-02 run concurrently (independent roots: OpenSearch role vs substrate contract). Fan-out group: {TASK-01, TASK-02}. TASK-03 joins both; TASK-04 follows TASK-03.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** substrate XRD claim on kind + AWS yields identical connection-secret shape (AC-05); XR status exposes only ready/endpoint/version (AC-06); admission rejects claim with no matching Composition (AC-07); all four substrate XRDs resolve (AC-08).
- **Playwright (UI/e2e):** optional Headlamp inspection of `MemoryStore` access mode.
- **PyTest (logic):** drop/rebuild OpenSearch reproducibility for vectors + audit (AC-01, AC-02); access-mode visibility matrix (AC-03); SDK-only memory/RAG access (AC-04); kind no-archive caveat documented (AC-09).
- Fixtures/fakes: kind + AWS substrate label fixtures; fake object-storage documents for re-embedding; stub OPA for RBAC-OPA access mode; fake audit endpoint.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-03-memory-and-data-architecture` → base `wave/auth`; carries spec-V6-03.md + plan-V6-03.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 S · TASK-05 M. Rollup: S (authoring). Critical path: {TASK-01, TASK-02} → TASK-03 → TASK-04 → TASK-05.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A10/A11/B4/B11) carry the enforceable change. No downstream code breaks from reverting the view itself.

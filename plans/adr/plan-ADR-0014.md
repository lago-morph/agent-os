# PLAN ADR-0014 — Postgres primary; OpenSearch retrieval-only [PROPOSED]

> spec: SPEC-ADR-0014 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A11, B4] · downstream-pieces: [A10, A18, B11, C8, F2]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0014 is a platform invariant enforced by every stateful component's write path plus the rebuild-path contract. Primary enforcers: **B4** (provides `XPostgres`/`XSearchIndex` Compositions writing the uniform connection-secret), **A18** (audit pipeline keeps Postgres+S3 as SoR, OpenSearch advisory), **A11** (OpenSearch operated as the derived retrieval layer), with **A10** (Letta), **C8** (KB indexing), and **F2** (DR testing) each demonstrating a documented rebuild path. Conformance is tested by a system-of-record inventory (no OpenSearch-only data), a rebuild/reindex drill from primaries, a DR drill via Postgres restore + reindex, and a write-path audit asserting every stateful component writes a primary first.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Build the SoR inventory + per-index rebuild-source catalogue — produces: index/rebuild catalogue — depends-on: [TASK-01]
- TASK-03: Verify audit pipeline keeps Postgres+S3 SoR / OpenSearch advisory (ADR 0034 cross-check) — produces: A18 conformance note — depends-on: [TASK-01]
- TASK-04: Verify uniform connection-secret parity across kind/AWS `XPostgres` Compositions — produces: B4 secret-parity note — depends-on: [TASK-01]
- TASK-05: Define reindex-from-primaries conformance drill — produces: PyTest+Chainsaw set — depends-on: [TASK-02]
- TASK-06: Define DR drill (Postgres restore + reindex, no OpenSearch backup) — produces: F2 trace — depends-on: [TASK-02]
- TASK-07: Define write-path audit (every stateful component writes a primary first) — produces: review/CI lint — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A11 (OpenSearch as the derived layer), B4 (`XPostgres`/`XSearchIndex` primaries).
### 3.2 Downstream pieces blocked on this
- A10 (Letta derived indexes), A18 (audit SoR), C8 (KB indexing), F2 (DR testing).
### 3.3 Continuous (non-blocking) inputs
- ADR 0034 (audit pipeline), ADR 0041 (substrate abstraction), B14 test framework.

## 4. Parallelizable Subtasks
TASK-03, TASK-04 fan out once TASK-01 lands. TASK-05/06/07 fan out once TASK-02 lands.

## 5. Test Strategy
- AC-01/AC-02 → PyTest doc/inventory lint: zero OpenSearch-only indexes; every index has a rebuild source.
- AC-03 → review/CI: OpenSearch-only write fails.
- AC-04 → Chainsaw: wipe OpenSearch, reindex from primaries, assert no loss.
- AC-05 → Chainsaw (F2 drill): Postgres restore + reindex, no OpenSearch backup.
- AC-06 → PyTest: OpenSearch down → audit ingestion still succeeds; index rebuilds from S3/Postgres.
- AC-07 → PyTest: same secret keys bind against kind and AWS Compositions.
- AC-08 → Chainsaw: KB/Letta/test-result indexes each rebuild from primary.
- Fixtures/fakes: stub S3 (kind), CloudNativePG + OpenSearch fakes for not-yet-landed substrates.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0014-postgres-primary-opensearch-retrieval` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 M, 03 S, 04 S, 05 M, 06 M, 07 S. Critical path: TASK-01 → 02 → 05/06.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the platform loses the rebuild-path invariant that DR (F2), the audit pipeline (A18), KB (C8), and Letta (A10) all rest on; OpenSearch could silently become a system of record. No runtime artifact is deleted.

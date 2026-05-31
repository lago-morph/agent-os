# PLAN ADR-0009 — OpenSearch as the search and vector store [PROPOSED]

> spec: SPEC-ADR-0009 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A10;A18]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0009 is enforced by component A11 (OpenSearch install as the advisory retrieval tier, adapter-only writes, OSS-Security-plugin SSO), with dual-mode provisioning realized by B4 (`SearchIndex` Composition) and the system-of-record boundary held by A18 (audit) and ADR 0014/0034. Conformance is proven by: (a) reindex-from-primary tests for every index class; (b) an adapter-only-write scan; (c) a dual-mode uniform-connection-secret check; (d) an "OpenSearch-down, audit still succeeds" resilience test; (e) an OSS-Security-plugin/no-oauth2-proxy assertion. No new product build belongs to this ADR — it is a constraint map over A11/B4/A18.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify reindex-from-primary tests (audit/vectors/checkpoints) proving advisory-only — produces: Chainsaw + PyTest (A11/A18) — depends-on: [TASK-01]
- TASK-03: Specify the adapter-only-write scan (no direct write endpoint) — produces: CI scan + e2e (A11) — depends-on: [TASK-01]
- TASK-04: Specify dual-mode `SearchIndex` uniform-connection-secret check (kind vs AWS) — produces: Chainsaw (B4) — depends-on: [TASK-01]
- TASK-05: Specify resilience (OpenSearch-down → audit still ingests) + OSS-Security-plugin/no-oauth2-proxy + tri-modal `rag.*` checks — produces: e2e + PyTest (A11/A18/B6) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- None — A11 is a foundation component.
### 3.2 Downstream blocked on this
- A10 (Letta indexes), A18 (audit fanout).
### 3.3 Continuous (non-blocking) inputs
- B4 `SearchIndex` Composition; B6 `rag.*` API; ADR 0034 audit pipeline; ADR 0011 test-results streamer; B14 test framework.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/04 → Chainsaw + PyTest (reindex-from-primary; OpenSearch-down audit resilience).
- AC-02 → CI scan + e2e (adapter-only writes).
- AC-03 → Chainsaw (dual-mode uniform connection secret).
- AC-05/06 → PyTest (OSS-Security-plugin SSO, no oauth2-proxy; tri-modal `rag.*`).
Fixtures: stub audit/RAG/test-result adapters until A18/B6 land; fake `SearchIndex` Compositions for kind/AWS.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0009-opensearch` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-02.

## 8. Rollback / Reversibility
Backing out means re-opening the search-store choice (Elasticsearch). Because OpenSearch is advisory and all data is reproducible from primary sources, migration risk is bounded; reverting re-points `rag.*`, the Letta index adapter, the audit fanout, and the test-result streamer. Moderate-to-high reversibility; no data loss.

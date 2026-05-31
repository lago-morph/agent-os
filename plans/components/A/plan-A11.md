# PLAN A11 — OpenSearch

> spec: SPEC-A11 · kind: COMPONENT · tier: T0
> wave: W0 · estimate: M
> upstream-pieces: [] · downstream-pieces: [A10, A18]

## 1. Implementation Strategy

Install OpenSearch in both modes — in-cluster on kind and AWS-managed on AWS — fronted by the `SearchIndex` XRD (Composition code owned by B4, jointly contracted with A11). Stand up vector + hybrid + BM25 search from the single backend, wire OIDC/SAML through the OSS Security plugin against Keycloak (no oauth2-proxy), bring up OpenSearch Dashboards, and — most importantly — establish the **advisory-only / reproducibility** posture: every index has a documented, tested reindex path from a primary, and OpenSearch being down never blocks the audit system of record. Because A11 is W0 foundation it lands early so Letta (A10), the audit pipeline (A18), the KB, and the test-result streamer have a retrieval tier. Where B4's `SearchIndex` Composition is not yet final, A11 develops against a local in-cluster install plus a stub connection-secret with the agreed key shape, then validates against the real Composition on both substrates.

## 2. Ordered Task List

- **TASK-01:** In-cluster OpenSearch install (kind) via Helm — produces: dev search tier — depends-on: []
- **TASK-02:** Contract the `SearchIndex` field + connection-secret shape with B4; AWS-managed path — produces: dual-mode contract — depends-on: [TASK-01]
- **TASK-03:** Vector + BM25 + hybrid search enablement — produces: retrieval capabilities — depends-on: [TASK-01]
- **TASK-04:** OSS Security plugin OIDC/SAML against Keycloak (excluded from oauth2-proxy) — produces: auth — depends-on: [TASK-01]
- **TASK-05:** OpenSearch Dashboards with Keycloak SSO — produces: operator UI — depends-on: [TASK-04]
- **TASK-06:** Advisory-only posture: reindex paths per index class + "OpenSearch-down ⇒ audit still succeeds" behavior — produces: reproducibility guarantee — depends-on: [TASK-03]
- **TASK-07:** Uniform connection-secret + substrate-agnostic status (`ready`/`endpoint`/`version`) verification — produces: substrate-invisible contract — depends-on: [TASK-02]
- **TASK-08:** Substrate-claim admission test (no-matching-Composition reject) — produces: admission guard validation — depends-on: [TASK-02]
- **TASK-09:** Audit emission of operator actions via adapter (stub until A18) — produces: audit hooks — depends-on: [TASK-01]
- **TASK-10:** DR-drill reindex runbook (F2-aligned) — produces: reindex runbook — depends-on: [TASK-06]
- **TASK-11:** §14.1 deliverables (docs, alerts, dashboard XR, Headlamp integration, HolmesGPT toolset, tutorials) — produces: deliverable set — depends-on: [TASK-09, TASK-10]
- **TASK-12:** 3-layer test suite mapping all ACs (two-substrate runs) — produces: tests — depends-on: [TASK-11]

Critical path: TASK-01 → 02 → 07 → 06 → 12 (the dual-mode + advisory-only contract is what A10/A18 depend on).

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
None at the wave level (W0 foundation). At runtime A11 sits behind the B4-authored `SearchIndex` XRD, but A11 is foundation and develops against a local install + agreed secret shape until B4's Composition lands.

### 3.2 Downstream pieces blocked on this
A10 (Letta derived indexes), A18 (audit pipeline advisory index host). Cross-cutting: B6 SDK `rag.*`, C8 KB indexing, B14 test-results, A14 HolmesGPT, A23 Kargo (connection secret).

### 3.3 Continuous (non-blocking) inputs
- B4 (`SearchIndex` Composition) — jointly contracted; A11 uses a local install + stub secret first.
- A18 (audit pipeline) — stub audit endpoint until landed; A18 owns the audit *indexer* + index schema, A11 hosts the cluster.
- Primaries (`Postgres`, `ObjectStore` via B4) — needed to *exercise* reindex, not to stand up search.
- B14 test framework, B22 threat model — continuous inputs (F2 DR drill aligns with reindex runbook).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-03 (search) ∥ TASK-04 (auth) ∥ TASK-09 (audit) ∥ TASK-02 (dual-mode contract).
- After TASK-02: TASK-07 (secret/status) ∥ TASK-08 (admission).
- After TASK-03: TASK-06 (reproducibility) → TASK-10 (DR runbook).
- TASK-05 (Dashboards) after TASK-04.

## 5. Test Strategy

| AC | Layer | Fixtures / fakes |
|---|---|---|
| AC-A11-01,08,09,11 | Chainsaw | kind + AWS-managed targets; `SearchIndex` claim (real or stub Composition); mislabeled-env cluster; Gatekeeper |
| AC-A11-02,03,04,05,10,12 | PyTest | sample vector/text corpus; Keycloak OIDC fixture; stub audit endpoint; OpenSearch-down injector; reindex-from-primary fixture |
| AC-A11-06,07 | Playwright / PyTest | Keycloak session; Dashboards |

Fakes for not-yet-landed upstreams: B4 `SearchIndex` Composition (local install + stub secret), stub audit endpoint (A18), primary-store fixtures (object storage / Postgres) to exercise reindex.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A11-opensearch` → base `wave/0`; carries spec-A11.md + plan-A11.md.
### 6.3 Merge order — W0 siblings independent; A11 ↔ B4 coordinate the `SearchIndex` contract; A11 ↔ A18 coordinate index-schema vs cluster-ownership; `wave/0` rolls up to main.

## 7. Effort Estimate

- S: TASK-05, TASK-08, TASK-09. M: TASK-01, TASK-02, TASK-03, TASK-04, TASK-06, TASK-07, TASK-10, TASK-11. L: TASK-12 (two-substrate runs).
- Rollup: **M** (matches piece-index). Critical path ≈ TASK-01→02→07→06→12.

## 8. Rollback / Reversibility

Back out by reverting the OpenSearch install / `SearchIndex` claim. Because OpenSearch is **advisory only and never a system of record** (ADR 0014), rollback loses **no durable platform data** — every index is rebuildable from a primary (object storage / S3 / Postgres) by reindex. **Reverting A11 degrades retrieval** (RAG/KB search, Letta-derived retrieval, audit/test-result query and dashboards) but does **not** break the audit system of record or any primary store. This makes A11 the most reversible of the five T0 pieces: downstream consumers (A10, A18) lose their advisory index and dashboards but their primary write paths continue. Restore is reprovision-then-reindex, not a backup/restore loop.

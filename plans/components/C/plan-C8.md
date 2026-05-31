# PLAN C8 — Knowledge Base RAG indexing pipeline  [PROPOSED]

> spec: SPEC-C8 · kind: COMPONENT · tier: T1
> wave: authoring-parallel (consumer) · estimate: M
> upstream-pieces: [C1, C6, C7] · downstream-pieces: [F3, F6]

## 1. Implementation Strategy
Build a custom ingestion pipeline that converts the authored Markdown corpus (C1 output incl.
C2–C7 and Workstream A docs/runbooks) into entries in the `platform-knowledge-base` `RAGStore`
backed by OpenSearch, treating Git as primary and the index as a derived, fully rebuildable
artifact (ADR 0014). The two load-bearing contracts are specified first: (1) the **trigger
contract** — re-index on a contributor's major/minor commit flag, never on patch — delivered via
the GitHub Actions docs path and/or a Knative flow; and (2) the **shared indexing conventions**
(chunking/metadata/store mapping) that the vendor-doc companion project (ADR 0024) must conform to,
published as the integration contract that F3 later validates. Build against fakes for not-yet-
landed upstreams (a stub `RAGStore`/B13 + a local OpenSearch fixture) and bind to the B13 `RAGStore`
fields (`indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`) without inventing CRD shape.

## 2. Ordered Task List
- **TASK-01:** Specify the trigger contract — major/minor flag location on a commit + patch-exclusion semantics (§6.4) — produces: trigger contract doc — depends-on: [].
- **TASK-02:** Specify + publish the shared indexing conventions (chunking, metadata schema, store mapping) as the companion-project contract (ADR 0024) — produces: conventions spec — depends-on: [].
- **TASK-03:** Implement the corpus→index pipeline (parse Markdown, chunk, embed/index, write to `platform-knowledge-base`); idempotent (ADR 0014) — produces: indexer — depends-on: [TASK-02].
- **TASK-04:** Bind to the B13-reconciled `RAGStore` (`backend`=OpenSearch, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`=C8) — produces: RAGStore binding — depends-on: [TASK-03].
- **TASK-05:** Implement the rebuild-from-primary path (full reindex from Git, on-demand) (ADR 0014) — produces: rebuild path — depends-on: [TASK-03].
- **TASK-06:** Wire the trigger delivery — GitHub Actions docs step (ADR 0010/0033) and/or Knative flow (ADR 0031) — produces: trigger wiring — depends-on: [TASK-01, TASK-03].
- **TASK-07:** Emit audit (adapter, ADR 0034) + result/lag events under one Canon `platform.*` namespace (ADR 0031); expose ingestion-lag metric for D2/runbook — produces: audit + event + metric wiring — depends-on: [TASK-03].
- **TASK-08:** Consume any object-store/search-index via substrate XRD connection-secret (ADR 0044) — produces: substrate wiring — depends-on: [TASK-03].
- **TASK-09:** Pipeline runbook (ingestion lag) + three-layer tests; joint F3 conventions-conformance check — produces: runbook + tests — depends-on: [TASK-04, TASK-05, TASK-06, TASK-07].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **C1** — re-indexable Markdown corpus output + conventions. **C6, C7** — runbook/maintainer content co-owning the corpus shape (so re-index is preprocessing-free).
- Soft/foundation (fake until landed): **A11** OpenSearch (index backend, ADR 0009); **B13** (`RAGStore` reconciled into LiteLLM, ADR 0013).
### 3.2 Downstream pieces blocked on this
- **F3** (companion-project handoff — validates shared conventions), **F6** (final pipeline exercise). Indirectly: every Platform Agent including `platform-knowledge-base` (HolmesGPT, Interactive Access Agent, Coach) reading via `rag.*`; D2 KB-effectiveness dashboard consumes C8's metrics.
### 3.3 Continuous (non-blocking) inputs
- C2–C5 + Workstream A content enlarging the corpus; B12 schema registry for any emitted event types; B14 test framework; B22 threat-model standards.

## 4. Parallelizable Subtasks
- TASK-01 and TASK-02 run concurrently up front (independent contracts).
- After TASK-03: TASK-04, TASK-05, TASK-06, TASK-07, TASK-08 fan out concurrently.
- TASK-09 joins all for tests + the F3 conformance check.

## 5. Test Strategy
- **Chainsaw:** AC-C8-05 (`RAGStore` binding/`ingestionPipelineRef` reconcile path), AC-C8-10 (trigger reconcile end-to-end if Knative-flow path chosen).
- **PyTest:** AC-C8-01 (content queryable after flagged commit), AC-C8-02/-03 (major/minor triggers, patch does not), AC-C8-04 (rebuild-from-primary reproduces content), AC-C8-07 (no vendor-doc acquisition — negative test), AC-C8-08 (audit + single-namespace event), AC-C8-09 (idempotent re-index, no duplicates), AC-C8-11 (connection-secret shape).
- **Playwright:** N/A unless a portal-side major/minor flag UI exists (design-time).
- **Joint:** AC-C8-06 with **F3** (authored-path doc vs companion-conventions doc land in a consistent shape).
- **Fixtures/fakes:** stub `RAGStore` standing in for B13; local OpenSearch fixture for A11; a fake `rag.*` reader to assert queryability; sample corpus with major/minor/patch-flagged commits.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/<C-content-wave>` (contains spec-C1/C6/C7).
### 6.2 PR — `piece/C8-kb-indexing-pipeline` → base of the C-content wave; carries spec-C8.md + plan-C8.md.
### 6.3 Merge order — merge **after** C1/C6/C7 (consumes their corpus); ahead of F3/F6 which validate it; wave rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 S · TASK-05 S · TASK-06 M · TASK-07 M · TASK-08 S · TASK-09 M.
- Rollup: **M**. Critical path: TASK-02 → TASK-03 → TASK-04/06 → TASK-09.

## 8. Rollback / Reversibility
Back out by removing the indexing pipeline deploy/trigger wiring; the `platform-knowledge-base`
index becomes stale but, being a derived artifact (ADR 0014), is fully rebuildable from the Git
corpus once the pipeline is restored — no source data is lost. Downstream impact: agents reading
via `rag.*` see no new authored content after rollback; the vendor-doc companion project's
integration point (F3) has no authored-side conventions to conform to; D2 KB-effectiveness metrics
go dark. The `RAGStore` CRD (B13) and OpenSearch (A11) are unaffected — only the ingestion path is
removed. Reversible; idempotent re-index on re-enable converges the index.

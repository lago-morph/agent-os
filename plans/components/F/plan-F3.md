# PLAN F3 — Vendor documentation companion-project handoff

> spec: SPEC-F3 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes C8; runs at end of v1.0) · estimate: S
> upstream-pieces: [C8, B4] · downstream-pieces: []

## 1. Implementation Strategy

F3 verifies and documents a seam ADR 0024 deliberately left open: the companion project produces vendor docs; the platform's C8 pipeline indexes them into the `platform-knowledge-base` RAGStore. The approach is verify-then-document: wire a real (or faithfully stubbed) vendor-doc source through the existing C8 trigger/ingest path, confirm retrieval via `rag.*`, confirm the major/minor-vs-patch trigger semantics, then write the platform-vs-companion boundary handoff doc. Build nothing — a code-review gate confirms no acquisition/scanning logic leaked into the platform. Critical path: confirm C8 trigger/ingest contract → end-to-end ingest+retrieve of a vendor doc → boundary doc + handoff sign-off.

## 2. Ordered Task List

- **TASK-01:** Confirm the C8 ingestion-pipeline trigger/ingest contract + version (URL-path-versioned, Canon §3.3); identify the external-content-source registration shape on `RAGStore` — produces: integration-point contract note — depends-on: [].
- **TASK-02:** Wire a vendor-doc source (real companion output or faithful acquisition-stub targeting the published contract) and index it into `platform-knowledge-base` via C8 — produces: ingested vendor doc — depends-on: [TASK-01].
- **TASK-03:** Verify retrieval — a Platform Agent retrieves a known vendor-doc fact via `rag.*` through its CapabilitySet — produces: retrieval verification — depends-on: [TASK-02].
- **TASK-04:** Verify trigger semantics — major/minor triggers indexing; patch does not (§14.3 C8) — produces: trigger-semantics record — depends-on: [TASK-02].
- **TASK-05:** Boundary/handoff document — platform-owned (RAGStore, ingest pipeline, trigger interface + version) vs companion-owned (acquisition, version-diff scan, curation); ADR 0024 boundary — produces: handoff doc — depends-on: [TASK-01].
- **TASK-06:** No-leak gate — code review confirming no acquisition/version-scanning logic was added to the platform — produces: review record — depends-on: [TASK-02].
- **TASK-07:** Runbook (verify/repair the companion→Knowledge-Base integration) feeding F6 + how-tos (register a companion vendor-doc source; the handoff boundary) — depends-on: [TASK-05].
- **TASK-08:** 3-layer tests (PyTest ingest+retrieve fixture vendor doc; Chainsaw RAGStore content-source registration) mapping ACs — depends-on: [TASK-03, TASK-04].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **C8** — Knowledge Base RAG indexing pipeline + the major/minor trigger model. Consumed: the trigger/ingest contract F3 verifies the companion targets.
- **B4** — `SearchIndex`/`ObjectStore` substrate backing the `platform-knowledge-base` RAGStore.

### 3.2 Downstream blocked on this
- None (terminal handoff). F6 references the integration runbook.

### 3.3 Continuous (non-blocking) inputs
- The **separate companion project** (ADR 0024) — ideally a real producer; faithful stub if not ready. **A10/A11** RAGStore backends. **D2** RAG-effectiveness sizing. **B14** harness.

## 4. Parallelizable Subtasks
- TASK-05 (boundary doc) parallels TASK-02..04 once the contract (TASK-01) is known.
- TASK-03 and TASK-04 (retrieval vs trigger semantics) are independent after TASK-02.
- TASK-07 (runbook/how-tos) and TASK-08 (tests) parallelize after verification.

## 5. Test Strategy
- **PyTest (logic):** AC-F3-01 (vendor doc searchable post-index), AC-F3-02 (major/minor triggers, patch does not), AC-F3-03 (retrieval via `rag.*`), AC-F3-05 (e2e source→index→retrieve recorded).
- **Chainsaw (operator/CRD):** RAGStore `contentSourceRefs[]`/`ingestionPipelineRef` registration for the vendor source.
- **Playwright:** minimal/optional — retrieval check via LibreChat if used; otherwise N/A.
- **Fixtures/fakes:** acquisition-stub emitting a known vendor doc against the published C8 trigger/ingest contract (R1 — companion may be absent); a major / minor / patch trigger fixture; known-fact retrieval probe.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (C8, B4 merged to main).
### 6.2 PR — `piece/F3-vendor-docs-handoff` → base `wave/F`; carries SPEC-F3 + PLAN-F3.
### 6.3 Merge order — independent of F-siblings; F6 references the integration runbook post-merge.

## 7. Effort Estimate
- Contract confirmation + wiring (TASK-01,02): S.
- Verification + boundary doc + gate (TASK-03..06): S.
- Runbook + tests (TASK-07,08): S.
- **Rollup: S** (matches CSV). **Critical path:** TASK-01 → 02 → 03/04 → 05.

## 8. Rollback / Reversibility
Back out by removing the vendor-doc content source registration from the RAGStore (de-register the companion source); the Knowledge Base reverts to platform-authored content only. Non-destructive — already-indexed vendor docs can be re-indexed; the handoff doc and tests persist as artifacts. **No downstream break** (F3 is terminal); reverting only un-verifies the seam, re-opening the handoff gate.

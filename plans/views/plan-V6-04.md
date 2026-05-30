# PLAN V6-04 — The Knowledge Base as a separate primitive `[PROPOSED]`

> spec: SPEC-V6-04 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map: **A11 (OpenSearch)** lands in W0 as the retrieval backend the `platform-knowledge-base` `RAGStore` indexes into; **C8 (Knowledge Base RAG indexing pipeline)** is a consumer-wave piece (real-build after A/B) that ingests the content sources and applies the major/minor re-indexing rule. The view holds when the singleton-primitive, CapabilitySet-inclusion, content-source, human-judgment-re-indexing, and SDK-`rag.*`-access invariants all hold. Verification confirms exactly one `platform-knowledge-base` `RAGStore` reachable by multiple agents through the same `rag.*` path, with re-indexing triggered only on major/minor release flags. The `RAGStore`/`CapabilitySet` CRDs are reconciled by B13 (gateway slice, V6-01); access is via B6's `rag.*` and the Interactive Access Agent (A16).

## 2. Ordered Task List
- **TASK-01:** Confirm the Knowledge Base is the single `RAGStore` named `platform-knowledge-base`, independent of any agent and included only via CapabilitySet `ragStores[]` — produces: singleton-primitive checklist — depends-on: []
- **TASK-02:** Confirm C8 ingests the three content-source classes (MkDocs portal, per-product docs/runbooks, pinned vendor docs) into the KB index — produces: content-source checklist — depends-on: [TASK-01]
- **TASK-03:** Confirm the human-judgment re-indexing rule (major/minor → re-index; patch → no) and the vendor-doc companion-project ownership boundary (ADR 0024) — produces: re-indexing-rule checklist — depends-on: [TASK-02]
- **TASK-04:** Confirm the access path: agents via `rag.*` over LiteLLM; LibreChat via the Interactive Access Agent (A16); SDK-API vs mount is per-agent design-time — produces: access-path checklist — depends-on: [TASK-01]
- **TASK-05:** Define end-to-end view verification: singleton, multi-agent shared access, content coverage, re-indexing trigger, governed access — produces: view acceptance suite mapping (AC-01..07) — depends-on: [TASK-03, TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A11 — OpenSearch (consumed: KB vector/hybrid index backend).
- C8 — KB RAG indexing pipeline (consumed: ingestion + re-indexing rule).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; A16, HolmesGPT, Coach include the KB via CapabilitySet but bind to component specs).
### 3.3 Continuous (non-blocking) inputs
- B13 (`RAGStore`/`CapabilitySet` reconciliation, gateway slice); B6 (`rag.*` surface); A16 (Interactive Access Agent access path); B14 test framework; ADR 0024 companion-project handoff (F3).

## 4. Parallelizable Subtasks
- TASK-02/03 (content + re-indexing) and TASK-04 (access path) run concurrently once TASK-01 is green. Fan-out group: {TASK-02→TASK-03, TASK-04}.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** exactly one `platform-knowledge-base` `RAGStore` exists and is referenced only by CapabilitySets (AC-01); CapabilitySet inclusion gates reachability (AC-06).
- **Playwright (UI/e2e):** LibreChat → Interactive Access Agent → KB query path; KB absent from agent image builds (AC-02).
- **PyTest (logic):** content-source coverage across the three classes (AC-03); patch flag → no re-index, major/minor flag → re-index (AC-04); vendor-doc re-index driven by companion project (AC-05); SDK-API and mount both governed (AC-07).
- Fixtures/fakes: fake content sources with release-flag commits; stub vendor-doc companion signal; fake `rag.*` over LiteLLM.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-04-knowledge-base-primitive` → base `wave/auth`; carries spec-V6-04.md + plan-V6-04.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 M. Rollup: S (authoring). Critical path: TASK-01 → TASK-02 → TASK-03 → TASK-05.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (C8, A11) and the consuming pieces (A16, B6, B13) carry the enforceable change. No downstream code breaks from reverting the view itself.

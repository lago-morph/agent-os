# PLAN C6 — Cross-cutting / integrated runbooks  [PROPOSED]

> spec: SPEC-C6 · kind: COMPONENT · tier: T1
> wave: authoring-parallel (consumer) · estimate: M
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy
Author the six cross-cutting runbooks (§10.7) as Markdown-in-repo in the C1 portal's reserved
cross-cutting runbook section, one page per symptom, each structured as diagnostic flow →
remediation → escalation. Because the flows traverse many components that land across waves,
author against the documented behavior of each component (its spec/per-product runbook) and the
existing observability signals, writing **partial** runbooks early and finalizing as components
land (§10 doc-timing); F6 later verifies each has been exercised. Co-develop the page template /
front-matter with C8 so the corpus is re-indexable into `platform-knowledge-base` without
preprocessing, and validate cross-references and builds through C1's GitHub Actions docs check.

## 2. Ordered Task List
- **TASK-01:** Agree the runbook page template + front-matter with C8 (RAG-ingestible shape) — produces: runbook template — depends-on: [].
- **TASK-02:** Author "agent runs slow end-to-end" runbook (trace_id Langfuse+Tempo → Mimir → gateway/sandbox) — produces: runbook page — depends-on: [TASK-01].
- **TASK-03:** Author "cost spike traced back through virtual keys" runbook (`VirtualKey`/`BudgetPolicy`, budget-exceeded flow) — produces: runbook page — depends-on: [TASK-01].
- **TASK-04:** Author "audit volume drop" runbook (Postgres+S3 authoritative, OpenSearch advisory, ADR 0034) — produces: runbook page — depends-on: [TASK-01].
- **TASK-05:** Author "eval failures spiking" runbook (`Evaluation`/`platform.evaluation.*`, D1 eval trend) — produces: runbook page — depends-on: [TASK-01].
- **TASK-06:** Author "agent stuck mid-run" runbook (`AgentRun` state, sandbox, LogLevel raise/restore) — produces: runbook page — depends-on: [TASK-01].
- **TASK-07:** Author "tenant-specific failure" runbook (tenant-scoped, §6.9 isolation, redacted examples) — produces: runbook page — depends-on: [TASK-01].
- **TASK-08:** Add per-product runbook cross-reference hand-offs (Workstream A targets) — produces: cross-links — depends-on: [TASK-02..TASK-07].
- **TASK-09:** Validate builds/links via C1 docs check + joint C8 re-index + KB-retrieval smoke — produces: passing checks + retrieval verification — depends-on: [TASK-08].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **C1** — portal + reserved §10.7 runbook section + Markdown/MkDocs conventions and docs CI check.
### 3.2 Downstream pieces blocked on this
- **C8** — re-indexes the runbook Markdown (and validates KB retrieval jointly). **F6** consumes C6 for final compilation/exercise verification.
### 3.3 Continuous (non-blocking) inputs
- Workstream A per-product runbook authors (hand-off targets); the components each flow traverses (LiteLLM, ARK, agent-sandbox, Langfuse, Tempo, Mimir, audit pipeline, Kargo); D1 dashboards; B14 test framework for the link/build checks.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02 through TASK-07 (the six runbooks) run fully concurrently — independent fan-out.
- TASK-08 joins all six; TASK-09 is the final serial validation gate.

## 5. Test Strategy
- **Chainsaw:** N/A — C6 owns no CRD.
- **Playwright:** AC-C6-04 (all six render in the cross-cutting section, separate from per-product), portal nav/search of the section.
- **PyTest / link-lint (via C1 check):** AC-C6-01, -02, -03, -05, -07, -08, -09 (page existence, required sections, named signals, cross-reference validity, ADR-0034 wording, LogLevel step, Canon-name references).
- **Joint with C8:** AC-C6-06, -10 (re-index without preprocessing; KB retrieval returns the right runbook).
- **Fixtures/fakes:** a stub `platform-knowledge-base` index (C8 fake) for the retrieval smoke test before C8 lands; sample per-product runbook stubs as cross-reference targets.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/<C1-wave>` (contains spec-C1, the portal skeleton).
### 6.2 PR — `piece/C6-cross-cutting-runbooks` → base of the C-content wave; carries spec-C6.md + plan-C6.md.
### 6.3 Merge order — independent of C-content siblings (C2–C5, C7, C9); C8 should merge after to exercise re-index; wave rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 M · TASK-07 M · TASK-08 S · TASK-09 S.
- Rollup: **M**. Critical path: TASK-01 → (any runbook) → TASK-08 → TASK-09.

## 8. Rollback / Reversibility
Back out by reverting the runbook PR; the cross-cutting runbook section returns to the empty C1
slot. Downstream impact: C8 has fewer authored docs to index (KB cross-cutting coverage drops to
nothing); HolmesGPT/operators lose the cross-cutting diagnostic content but per-product runbooks
(Workstream A) and dashboards (D1) are unaffected. Reversible at any time; no runtime state.

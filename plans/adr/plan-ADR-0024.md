# PLAN ADR-0024 — Vendor documentation acquisition is a separate companion project [PROPOSED]

> spec: SPEC-ADR-0024 · kind: ADR · tier: T2
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [C8, F3]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0024 is a scoping decision enforced primarily by **C8** (Knowledge Base RAG indexing pipeline — defines the indexing conventions the companion project must match and populates v1.0 from authored docs) and **F3** (companion-project handoff — owns the integration point and cross-reference). The core obligation to verify is *absence of a gate*: no v1.0 component lists the vendor-doc pipeline as a hard dependency, and the platform installs/operates with it absent. The consumption surface (`platform-knowledge-base` RAGStore + `rag.*` API) is the only architecture-owned contract; the companion project is an external producer. Conformance is mostly doc/dependency-lint plus a query test proving authored-only coverage works.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing piece (C8 conventions/population, F3 integration point, dependency-absence checks) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify no v1.0 component declares the vendor-doc pipeline a hard dependency — produces: dependency-absence audit — depends-on: [TASK-01]
- TASK-03: Verify the only architecture-documented vendor-doc surface is the RAGStore + `rag.*` API — produces: surface audit — depends-on: [TASK-01]
- TASK-04: Verify C8 ships the Knowledge Base populated from authored docs/runbooks and indexing conventions are documented — produces: C8 conformance check — depends-on: [TASK-01]
- TASK-05: Verify F3 owns a current cross-reference and licensing/freshness stays companion-side — produces: F3 conformance check — depends-on: [TASK-01]
- TASK-06: Verify a hypothetical contract change routes as a new ADR against ADR 0022 — produces: governance check — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None within v1.0 architecture scope (the companion project is external).
### 3.2 Downstream pieces blocked on this
- C8 (indexing conventions / population), F3 (integration point).
### 3.3 Continuous (non-blocking) inputs
- ADR 0022 (Knowledge Base primitive), ADR 0009 (OpenSearch substrate); B22 threat model for crawl/redistribution trust boundaries.

## 4. Parallelizable Subtasks
TASK-02 through TASK-06 all fan out independently once TASK-01 lands; none depend on each other.

## 5. Test Strategy
- AC-01 → PyTest dependency-lint (no hard dep on vendor pipeline) + Chainsaw install-without-companion smoke.
- AC-02 → PyTest doc-lint: only RAGStore + `rag.*` documented as vendor-doc surface.
- AC-03 → PyTest/Playwright: RAG query answered from C8 authored content in a clean v1.0 install.
- AC-04 → PyTest: vendor content (if present) appears under same indexing conventions; re-index on simulated upstream release.
- AC-05 → doc-lint over F3 cross-reference currency.
- AC-06 → governance-lint: contract-change routed as a new ADR against 0022.
- Fixtures/fakes: stub companion-project producer writing into the named RAGStore for AC-04.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0024-vendor-doc-separate-project` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 S, 04 S, 05 S, 06 S. Critical path: TASK-01 → 04.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, C8 loses the "match conventions / no-gate" contract and F3 loses the integration-point ownership; no runtime artifact is deleted.

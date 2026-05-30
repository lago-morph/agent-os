# PLAN ADR-0008 — Material for MkDocs as the documentation portal [PROPOSED]

> spec: SPEC-ADR-0008 · kind: ADR · tier: T2
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [C1;C2;C3;C4;C5;C6;C7;C8;C9]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0008 is enforced by component C1 (Material for MkDocs portal infra: docs-as-code, per-version builds, contribution workflow) with re-indexing realized by C8 into `platform-knowledge-base`. Conformance is proven by: (a) a co-location + PR-review check (docs in the component repo, reviewed with code); (b) a per-version static build check; (c) a flagged-commit re-index gate (major/minor triggers, patch does not); (d) a vendor-doc version-pin serve check. No new product build belongs to this ADR — it is a constraint map over C1/C8.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify docs-as-code co-location + same-PR review check — produces: CI lint/structure check (C1) — depends-on: [TASK-01]
- TASK-03: Specify per-version static-build + vendor-doc version-pin serve check — produces: build check (C1) — depends-on: [TASK-01]
- TASK-04: Specify the flagged-commit re-index gate into `platform-knowledge-base` (major/minor yes, patch no) — produces: pipeline test (C8) — depends-on: [TASK-01]
- TASK-05: Specify the scope-boundary assertion (no Backstage catalog/plugin surface) — produces: review checklist (C1) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- None — C1 is consumer-tier authoring infrastructure.
### 3.2 Downstream blocked on this
- C2, C3, C4, C5, C6, C7, C8, C9.
### 3.3 Continuous (non-blocking) inputs
- ADR 0022 Knowledge Base primitive; ADR 0009 OpenSearch retrieval tier; ADR 0024 vendor-doc separation; B14 test framework.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/05 → PyTest/CI lint (co-location + same-PR review; no-catalog scope boundary).
- AC-02/04 → build check (per-version static build; vendor-doc version-pin serve).
- AC-03 → PyTest pipeline test (flagged-commit re-index gate).
Fixtures: sample component repo with co-located docs; stub C8 indexer + fake `platform-knowledge-base` until C8 lands.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0008-mkdocs-portal` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-04.

## 8. Rollback / Reversibility
Backing out means re-opening the portal choice (Backstage/Confluence). Because docs are plain Markdown in Git, the content survives a portal swap; reversibility is high. Downstream breakage limited to C1's build/serve and the C8 re-index source format.

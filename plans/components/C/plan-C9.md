# PLAN C9 — Docs-on-docs  [PROPOSED]

> spec: SPEC-C9 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumer) · estimate: S
> upstream-pieces: [C1, C8] · downstream-pieces: []

## 1. Implementation Strategy
Author a concise docs-on-docs section into the C1 portal's §10.8 slot covering the five §10.8
topics: PR-based contribution workflow, MkDocs conventions, the Diataxis/section map, how docs get
indexed into `platform-knowledge-base` (C8's major/minor trigger), and how to query the KB from
LibreChat (Interactive Access Agent) and HolmesGPT. It narrates behavior owned elsewhere (C1
workflow, C8 indexing), so the work is largely cross-linking and explanation; validate via C1's
docs build/link check.

## 2. Ordered Task List
- **TASK-01:** Author contribution-workflow + MkDocs-conventions narrative (§10.8, ADR 0008) — produces: section pages — depends-on: [].
- **TASK-02:** Author the Diataxis/section map (§10.1–§10.7 locations) — produces: map page — depends-on: [].
- **TASK-03:** Author "how docs get indexed" referencing C8 major/minor trigger + patch-exclusion (§6.4) — produces: indexing page — depends-on: [].
- **TASK-04:** Author "how to query the KB" from LibreChat + HolmesGPT (`rag.*`, ADR 0022/0012) — produces: query page — depends-on: [].
- **TASK-05:** Cross-link + validate build/links via C1 docs check; confirm C8 re-index — produces: passing checks — depends-on: [TASK-01, TASK-02, TASK-03, TASK-04].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **C1** — portal + §10.8 slot + contribution-workflow conventions C9 narrates. **C8** — the indexing behavior/trigger C9 explains.
### 3.2 Downstream pieces blocked on this
- None (leaf, T2).
### 3.3 Continuous (non-blocking) inputs
- All other C pieces (C2–C7) define the quadrant/section map; B14 test framework for link/build checks.

## 4. Parallelizable Subtasks
- TASK-01 through TASK-04 are independent and run concurrently; TASK-05 joins for validation.

## 5. Test Strategy
- **Chainsaw:** N/A — no CRD.
- **Playwright:** AC-C9-05 (renders in §10.8 slot), nav/search of the section.
- **PyTest / link-lint (via C1 check):** AC-C9-01/-02/-03/-04 (workflow + conventions present, section map covers §10.1–§10.7, indexing-trigger explanation, KB-query explanation), link validity.
- **Fixtures/fakes:** C8 stub for the re-index confirmation if C8 not yet landed.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/<C-content-wave>` (contains spec-C1, spec-C8).
### 6.2 PR — `piece/C9-docs-on-docs` → base of the C-content wave; carries spec-C9.md + plan-C9.md.
### 6.3 Merge order — merge after C1 and C8; independent of other C siblings; wave rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S.
- Rollup: **S**. Critical path: any of TASK-01..04 → TASK-05.

## 8. Rollback / Reversibility
Revert the C9 PR; the §10.8 slot returns empty. No downstream breaks (leaf); no runtime state.
Contributors lose the consolidated docs-on-docs entry point but the underlying C1 workflow and C8
indexing are unaffected.

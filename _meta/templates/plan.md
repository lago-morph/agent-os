# PLAN <PIECE_ID> — <Title>

> spec: SPEC-<PIECE_ID> · kind: COMPONENT|VIEW|ADR · tier: <T0|T1|T2>
> wave: <W0..W4 | authoring-parallel> · estimate: <S|M|L|XL>
> upstream-pieces: [<ids>] · downstream-pieces: [<ids>]

## 1. Implementation Strategy
One paragraph: the approach to building this piece.
(VIEW: realization map — which components realize this view, in what order.
 ADR: enforcement/verification — which components enforce this decision, how
 conformance is tested.)

## 2. Ordered Task List
`TASK-NN: <description> — produces: <artifact> — depends-on: [TASK refs]`.
Ordered so the critical path is visible.

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed
### 3.2 Downstream pieces blocked on this — ids
### 3.3 Continuous (non-blocking) inputs — e.g. B14 test framework, B22 threat model

## 4. Parallelizable Subtasks
Which TASK-NN can run concurrently; the independent fan-out groups.

## 5. Test Strategy
Map ACs (from SPEC §9) to layers: Chainsaw (operator/CRD), Playwright (UI/e2e),
PyTest (logic). Note fixtures/fakes needed for not-yet-landed upstreams.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/<N>` (contains the upstream specs)
### 6.2 PR — `piece/<ID>-<slug>` → base `wave/<N>`; carries spec + plan for this piece
### 6.3 Merge order — within-wave siblings independent; wave rolls up to main

## 7. Effort Estimate
Per-task S/M/L; rollup; critical path within the piece.

## 8. Rollback / Reversibility
How to back this out; what downstream breaks if reverted.

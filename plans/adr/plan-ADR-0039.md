# PLAN ADR-0039 — Headlamp graphical editors for platform CRDs `[PROPOSED]`

> spec: SPEC-ADR-0039 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A9; A20; B5] · downstream-pieces: [A22; A21; A23; B5]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0039 is enforced at **A9** (framework + shared widgets: reference
pickers, diff viewer, simulator panel), at **B5** (the cross-cutting + overlay-aware editors), and at
**A22** (the graphical-editor framework surface for platform CRDs). The PR-only invariant is the load-
bearing rule, verified by a no-cluster-write test; the diff-against-Git, inline-simulator (ADR 0038),
and effective-resolved-set (ADR 0032) behaviours are each verified per editor. The initial editor set
is verified for coverage; schema-version tracking (ADR 0030) is verified by a rebuild test.

## 2. Ordered Task List
- TASK-01: Verify an editor renders a CRD from OpenAPI schema with enum dropdowns + inline validation + reference picker — produces: Playwright editor test — depends-on: []
- TASK-02: Verify diff-against-current-Git (not cluster) preview — produces: Playwright diff test — depends-on: [TASK-01]
- TASK-03: Verify inline simulator across all layers; failure shown alongside diff — produces: Playwright simulator-inline test — depends-on: [TASK-01]
- TASK-04: Verify overlay-aware editor shows effective resolved set per ADR 0032 — produces: Playwright overlay test — depends-on: [TASK-01]
- TASK-05: Verify submission produces a Git PR with no direct cluster write — produces: integration PR-only test — depends-on: [TASK-02]
- TASK-06: Verify initial editor-set coverage; non-set CRDs show read views + capability inspector — produces: coverage check — depends-on: [TASK-01]
- TASK-07: Verify editor rebuilt against a new served schema version renders new fields; older stored versions readable — produces: schema-version test — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A9 — Headlamp framework + shared widgets. A20 — policy simulator aggregator (inline). B5 — cross-cutting plugin work (owns the editors).
### 3.2 Downstream pieces blocked on this
- A22 (editor framework surface), A21 (`TenantOnboarding` editor consumer), A23 (Kargo authoring upstream), B5 (the editors).
### 3.3 Continuous (non-blocking) inputs
- Each CRD's owning component (OpenAPI schemas); ADR 0032 overlay resolution; ADR 0038 simulator; B14 test framework.

## 4. Parallelizable Subtasks
After TASK-01: TASK-02, TASK-03, TASK-04, TASK-06, TASK-07 fan out. TASK-05 follows TASK-02.

## 5. Test Strategy
- AC-01 → Playwright schema-render + reference picker (TASK-01); fake CRD-schema fixtures until owners land.
- AC-02 → Playwright diff-against-Git (TASK-02).
- AC-03 → Playwright inline-simulator (TASK-03); fake A20 aggregator fixture until A20 lands.
- AC-04 → Playwright overlay effective-resolved-set (TASK-04); reuse ADR 0032 resolution fixture.
- AC-05 → integration PR-only / no-cluster-write (TASK-05).
- AC-06 → coverage check over initial editor set (TASK-06).
- AC-07 → schema-version rebuild test (TASK-07).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0039-headlamp-editors` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 M · TASK-05 S · TASK-06 S · TASK-07 S. Rollup M. Critical path: TASK-01 → TASK-04 (overlay-aware editor is the heaviest).

## 8. Rollback / Reversibility
Reverting backs out the graphical editors; authoring falls back to raw YAML + PR (re-exposing the
destructive-edit / overlay-surprise / late-validation risks the ADR mitigates). GitOps and ArgoCD apply
are unaffected (editors only ever produced PRs). No data migration — the artifacts are React plugins
and tests.

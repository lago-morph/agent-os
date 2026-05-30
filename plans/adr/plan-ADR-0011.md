# PLAN ADR-0011 — Three-layer testing with CLI orchestration [PROPOSED]

> spec: SPEC-ADR-0011 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [B14, B9] · downstream-pieces: [D3, B15, A11, A13, A18]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0011 is enforced primarily by **B14** (the `agent-platform` test framework: runner orchestration, OpenSearch/OTel result emission, audit emission, `--debug`/`--trace` toggle wiring, stress-probe harness) and **B9** (CLI subcommand surface). The "every component lists its tests" obligation is enforced by the cross-cutting checklist in each component SPEC §8. Conformance is tested by running the CLI across the three environments, asserting the result-routing split (unit vs non-unit), proving graceful degradation, and proving toggle restore on abnormal exit. **D3** verifies the dashboard/result-view surfaces exist over the emitted metrics/indexes.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B14 SPEC carries CLI orchestration + result-split + toggle + stress-probe obligations — produces: B14 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify component-SPEC §8 checklist enumerates the 3-layer tests obligation platform-wide — produces: cross-component audit — depends-on: [TASK-01]
- TASK-04: Define conformance tests for result routing (unit CI-only; non-unit OpenSearch+OTel) and audit emission — produces: Chainsaw+PyTest test set — depends-on: [TASK-02]
- TASK-05: Define conformance test for graceful degradation (OpenSearch/OTel down) — produces: PyTest test — depends-on: [TASK-02]
- TASK-06: Define conformance test for toggle scope + restore-on-crash — produces: PyTest+Chainsaw test — depends-on: [TASK-02]
- TASK-07: Verify D3 dashboards/result views map to emitted metrics/indexes — produces: D3 trace — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B14 — the test framework implementing every CLI obligation. B9 — the CLI host surface.
### 3.2 Downstream pieces blocked on this
- D3 (dashboards), B15 (CI/CD reference pipeline).
### 3.3 Continuous (non-blocking) inputs
- A11 (OpenSearch), A13 (Tempo+Mimir), A18 (audit adapter) — soft deps; B22 threat model.

## 4. Parallelizable Subtasks
TASK-04, TASK-05, TASK-06 fan out independently once TASK-02 lands. TASK-03 runs parallel to TASK-02.

## 5. Test Strategy
- AC-01 → Chainsaw (CLI invocation across envs) + PyTest (manifest dispatch).
- AC-02 → PyTest doc-lint over component specs §8.
- AC-03/AC-04 → PyTest (assert OpenSearch/OTel emission counts) + Chainsaw for non-unit cluster path.
- AC-05 → PyTest assert audit record per run.
- AC-06 → PyTest with OpenSearch/OTel fakes down → exit-code assertion.
- AC-07 → PyTest+Chainsaw: kill mid-run, assert `LogLevel` restored.
- AC-08 → Chainsaw concurrent-invocation probe; assert no load-tool binary.
- Fixtures/fakes: stub OpenSearch + OTel collector + audit endpoint for not-yet-landed A11/A13/A18.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0011-three-layer-testing` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 S, 04 M, 05 S, 06 M, 07 S. Critical path: TASK-01 → 02 → 04 → 07.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, B14/D3 lose their conformance contract for result routing and the toggle restore guarantee; no runtime artifact is deleted.

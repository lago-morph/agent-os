# PLAN ADR-0035 — Dynamic log-level and trace-granularity toggle with staged restart for non-reloadable services `[PROPOSED]`

> spec: SPEC-ADR-0035 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A1; A6; A13] · downstream-pieces: [A18; A14; B6; B9; B13; A22]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0035 is enforced at the **component-review gate** (every component
must declare a toggle pattern), at **B6** (the shared in-process reload implementation every
in-process component links, preserving "no impact when off"), and at **A1** (LiteLLM as the
rolling-restart exemplar carrying the staged-restart guardrails). HolmesGPT (A14) and the Test CLI
(B9) are verified to drive the same `LogLevel` surface; A18 records every level change as audit
(ADR 0034); A22 surfaces the effective level in Headlamp. Trace-granularity-as-conditional-span-creation
(ADR 0015) is verified distinctly from sampling.

## 2. Ordered Task List
- TASK-01: Encode "every component declares a toggle pattern" as a component-review gate — produces: review-gate check — depends-on: []
- TASK-02: Verify in-process toggle: `LogLevel` change → in-place re-read, no restart, returns to no-impact-when-off — produces: PyTest/Chainsaw in-process test — depends-on: []
- TASK-03: Verify rolling-restart guardrails on LiteLLM (replicas≥2, readiness-before-SIGTERM, preStop, raised grace) drain streaming without drop; reject single-replica — produces: Chainsaw/integration test — depends-on: []
- TASK-04: Verify trace granularity creates previously-uncreated spans, not sampling — produces: PyTest tracing test — depends-on: [TASK-02]
- TASK-05: Verify one mechanism / two callers: HolmesGPT + Test CLI drive same surface; CLI re-asserts default on teardown; HolmesGPT write is policy-gated — produces: integration test — depends-on: [TASK-02]
- TASK-06: Verify every level change emits `platform.audit.*` audit event — produces: PyTest audit test — depends-on: [TASK-02]
- TASK-07: Verify Headlamp shows effective level + write path — produces: Playwright UI test — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A1 — LiteLLM (rolling-restart toggle exemplar). A6 — components carrying the toggle. A13 — trace/metric sinks granularity feeds.
### 3.2 Downstream pieces blocked on this
- B6 (shared reload impl), A18 (audit endpoint toggle + level-change audit), A14 (HolmesGPT caller), B9 (Test CLI flag), B13 (kopf operator), A22 (Headlamp `LogLevel` surface).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; ADR 0034 audit adapter (records changes); ADR 0015 tracing backends.

## 4. Parallelizable Subtasks
TASK-02 and TASK-03 run concurrently (independent component paths). After TASK-02: TASK-04, TASK-05, TASK-06, TASK-07 fan out. TASK-01 is independent.

## 5. Test Strategy
- AC-01 → review-gate check (TASK-01, conftest over component manifests).
- AC-02 → PyTest/Chainsaw in-process re-read (TASK-02).
- AC-03, AC-04 → Chainsaw/integration on LiteLLM Deployment + single-replica rejection (TASK-03).
- AC-05 → PyTest tracing distinguishes span-creation from sampling (TASK-04).
- AC-06, AC-07 → integration test on dual-caller + policy gate (TASK-05).
- AC-08 → PyTest audit-event presence (TASK-06).
- AC-09 → Playwright Headlamp effective-level + write path (TASK-07); fake A22 plugin fixture until A22 lands.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0035-log-trace-toggle` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 S · TASK-05 S · TASK-06 S · TASK-07 S. Rollup M. Critical path: TASK-03 (rolling-restart guardrails) is the heaviest single item.

## 8. Rollback / Reversibility
Reverting the review gate re-permits components with no toggle pattern, removing the diagnostic lever
HolmesGPT and the Test CLI depend on. Reverting the LiteLLM guardrails re-permits single-replica
LiteLLM and risks dropped streaming responses on verbosity changes. No data migration — artifacts are
the review gate, the shared reload impl, the Deployment guardrails, and tests.

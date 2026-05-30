# PLAN E1 — Operator training track [PROPOSED]

> spec: SPEC-E1 · kind: COMPONENT · tier: T2
> wave: consumer · estimate: M
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy

Build the operator track as eleven content modules mapped one-to-one to the §12.1 topic list, authored after the platform and its docs exist (consumer tier, §14.5). For each module, first fix the learning outcomes (binary operator competencies), then author the lab (an isolated Approved namespace that reuses the running platform per §12), then the exercises with automated checks, then the knowledge check, then wire the doc links into Workstream C. Reuse existing machinery — the `agent-platform` CLI (B9) and three-layer runners (B14) for checks, tenant-onboarding (A21) for lab namespaces — rather than building new tooling. Sequence the install/configure module (item 3) as the spine because it is the largest and depends on the most components landing first.

## 2. Ordered Task List

- TASK-01: Define the track skeleton and per-module learning outcomes for all eleven §12.1 topics — produces: module outline + outcomes matrix — depends-on: []
- TASK-02: Define the standard lab pattern (isolated namespace, starter state, check harness, knowledge-check format) reusing A21 onboarding + B9/B14 — produces: lab template — depends-on: [TASK-01]
- TASK-03: Author modules 1–2 (platform overview; cluster baseline review) with labs/checks — produces: modules 1–2 — depends-on: [TASK-02]
- TASK-04: Author module 3 (install/configure) as one sub-module per Workstream A component group — produces: module 3 + sub-modules — depends-on: [TASK-02]
- TASK-05: Author module 4 (configuration patterns — SSO, secrets, capabilities, policy bundles) — produces: module 4 — depends-on: [TASK-02]
- TASK-06: Author module 5 (monitoring and alerting) against D1 dashboards + alert rules — produces: module 5 — depends-on: [TASK-02]
- TASK-07: Author module 6 (failure modes and runbooks) inducing failures against C6/F6 runbooks — produces: module 6 — depends-on: [TASK-02]
- TASK-08: Author modules 7–8 (backup/restore drills; upgrade procedures) on dedicated lab substrate — produces: modules 7–8 — depends-on: [TASK-02]
- TASK-09: Author modules 9–10 (security operations; capacity planning) — produces: modules 9–10 — depends-on: [TASK-02]
- TASK-10: Author module 11 (working with HolmesGPT for incident response) with seeded-incident lab — produces: module 11 — depends-on: [TASK-02]
- TASK-11: Wire doc links (C2/C3/C4/C5/C6) into every module and verify no verbatim restatement — produces: linked modules — depends-on: [TASK-03..TASK-10]
- TASK-12: Build the three-layer check/lab-validity tests and a coverage check that the union of modules covers §12.1 — produces: test suite + coverage report — depends-on: [TASK-11]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed

None hard at the spec level (CSV lists no upstream). For real-build, the labs require the running platform; practically the modules cannot be validated until the components they exercise have shipped: A1/A5/A6/A7 (core runtime for most labs), A9/A22 (Headlamp UI exercises), A14 (HolmesGPT, module 11), A18 (audit, monitoring/failure modules), A21 (tenant onboarding for lab namespaces), D1 (operator dashboards, module 5), C2–C6 (docs linked by all modules).

### 3.2 Downstream pieces blocked on this — ids

None — E1 is terminal (no downstream in CSV).

### 3.3 Continuous (non-blocking) inputs

- B14 `agent-platform` test framework — exercise/knowledge checks ride the three-layer runners.
- B9 `agent-platform` CLI — lab driving and check invocation.
- B22 threat model — informs the security-operations module (item 9).
- Workstream C docs — continuously linked; revalidated on doc releases (SPEC §7 versioning).

## 4. Parallelizable Subtasks

After TASK-02 (lab template), the eleven module-authoring tasks are an independent fan-out group: TASK-03, TASK-04, TASK-05, TASK-06, TASK-07, TASK-08, TASK-09, TASK-10 can run concurrently (one author per module). They converge at TASK-11 (doc-link wiring) and TASK-12 (tests/coverage). TASK-04 (install/configure) is the longest pole because its sub-modules track the most components.

## 5. Test Strategy

- Chainsaw: assert lab namespace provisioning and starter-state CRDs (Approved namespace + seeded `Agent`/`Sandbox`), and assert exercise-completion cluster state (e.g. component installed, restore completed). Maps AC-E1-04, AC-E1-05, AC-E1-06, AC-E1-11.
- Playwright: drive Headlamp / Grafana (D1) UI exercises and knowledge-check UI. Maps AC-E1-05 (monitoring), AC-E1-07.
- PyTest: check-logic unit tests, the §12.1 coverage check (eleven modules, union covers all topics), outcome-traceability check, and doc-link presence/no-verbatim check. Maps AC-E1-01, AC-E1-02, AC-E1-03, AC-E1-08, AC-E1-09.
- HolmesGPT seeded-incident lab validated by a Chainsaw+PyTest combo asserting the expected diagnosis path. Maps AC-E1-10.
- Fixtures/fakes: for modules whose upstream component has not yet landed, stub the lab namespace with a recorded starter state so module authoring proceeds before the real component is available.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (real-build after A/B land)

### 6.2 PR — `piece/E1-operator-training-track` → base `wave/consumer`; carries spec-E1 + plan-E1

### 6.3 Merge order — independent of E2 and the other consumer-tier siblings; the consumer wave rolls up to main after Workstreams A/B/C provide the exercised platform + docs.

## 7. Effort Estimate

- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 L (longest pole — per-component-group sub-modules) · TASK-05 S · TASK-06 M · TASK-07 M · TASK-08 M · TASK-09 M · TASK-10 M · TASK-11 S · TASK-12 M.
- Rollup: M (matches CSV estimate). Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-11 → TASK-12.

## 8. Rollback / Reversibility

Fully reversible: E1 is content + lab definitions with no deployed platform surface. Reverting the PR removes the modules and lab templates; nothing downstream breaks (no downstream pieces). Provisioned lab namespaces are torn down by the standard onboarding/teardown path. The only loss on revert is the curriculum itself — no platform component depends on E1.

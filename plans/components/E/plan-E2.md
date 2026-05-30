# PLAN E2 — Developer training track [PROPOSED]

> spec: SPEC-E2 · kind: COMPONENT · tier: T2
> wave: consumer · estimate: M
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy

Build the developer track as thirteen content modules mapped one-to-one to the §12.2 topic list, authored after the platform, its SDKs, and its docs exist (consumer tier, §14.5). For each module, first fix the learning outcomes (binary developer competencies), then author the lab (an isolated Approved namespace that reuses the running platform per §12) and the working starter project, then the exercises with automated checks, then the knowledge check, then wire the doc/how-to links into Workstream C and the B6/B7 how-tos. Reuse existing machinery — the Platform SDK (B6) and agent SDK (B7) for starter projects, the `agent-platform` CLI (B9) and three-layer runners (B14) for checks, tenant-onboarding (A21) for lab namespaces — rather than building new tooling. Sequence the two "first agent" modules (items 2–3) early as the spine because every later module builds on a working agent, and treat the tools/MCP/A2A and evaluation modules as the heaviest poles because they touch the most components.

## 2. Ordered Task List

- TASK-01: Define the track skeleton and per-module learning outcomes for all thirteen §12.2 topics — produces: module outline + outcomes matrix — depends-on: []
- TASK-02: Define the standard lab + starter-project pattern (isolated namespace, scaffolded starter project, check harness, knowledge-check format) reusing A21 onboarding + B6/B7 SDKs + B9/B14 — produces: lab + starter-project template — depends-on: [TASK-01]
- TASK-03: Author module 1 (platform overview — what an agent is in our model) — produces: module 1 — depends-on: [TASK-02]
- TASK-04: Author modules 2–3 (first agent declarative; first agent in code with the SDK) with runnable starter projects — produces: modules 2–3 + starter projects — depends-on: [TASK-02]
- TASK-05: Author module 4 (skills — write, test, version) — produces: module 4 — depends-on: [TASK-04]
- TASK-06: Author module 5 (tools, MCP, A2A — incl. synthetic MCP from OpenAPI) — produces: module 5 — depends-on: [TASK-04]
- TASK-07: Author module 6 (memory, RAG, and the knowledge base) — produces: module 6 — depends-on: [TASK-04]
- TASK-08: Author module 7 (triggers — cron, event, long-running, interactive) — produces: module 7 — depends-on: [TASK-04]
- TASK-09: Author module 8 (evaluation and testing — Langfuse datasets, promptfoo, three-layer tests) — produces: module 8 — depends-on: [TASK-04]
- TASK-10: Author modules 9–10 (A/B testing and rollout; debugging with traces) against D2 dashboards + Langfuse/Tempo — produces: modules 9–10 — depends-on: [TASK-04]
- TASK-11: Author module 11 (cost optimization) against D2 cost-per-success — produces: module 11 — depends-on: [TASK-04]
- TASK-12: Author modules 12–13 (multi-agent patterns — A2A, Teams; capability sets — layered profiles) — produces: modules 12–13 — depends-on: [TASK-04]
- TASK-13: Wire doc/how-to links (C2/C3/C4/C5 + B6/B7 how-tos) into every module and verify no verbatim restatement — produces: linked modules — depends-on: [TASK-03..TASK-12]
- TASK-14: Build the three-layer check/lab-validity/starter-project tests and a coverage check that the union of modules covers §12.2 — produces: test suite + coverage report — depends-on: [TASK-13]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed

None hard at the spec level (CSV lists no upstream). For real-build, the labs require the running platform and SDKs; practically the modules cannot be validated until the components they exercise have shipped: A1/A5/A6/A7 (core runtime for every lab), A2/A13 (Langfuse/Tempo traces, debugging module), A8 (LibreChat, interactive triggers), A10/A11 + Knowledge Base (memory/RAG module), A12 (OpenAPI→MCP, tools module), B6/B7 (SDK starter projects, items 2–3 and onward), B9/B14 (CLI + test framework, evaluation module and all checks), D2 (developer dashboards, A/B + debugging + cost modules), C2–C5 (docs linked by all modules).

### 3.2 Downstream pieces blocked on this — ids

None — E2 is terminal (no downstream in CSV).

### 3.3 Continuous (non-blocking) inputs

- B6/B7 SDKs — starter projects scaffolded with and revalidated against the Platform + agent SDKs on each release.
- B14 `agent-platform` test framework — exercise/knowledge checks ride the three-layer runners; the evaluation module also teaches them.
- B9 `agent-platform` CLI — lab driving, starter-project scaffolding, and check invocation.
- B22 threat model — informs safe-by-default starter projects and the capability-set module.
- Workstream C docs — continuously linked; revalidated on doc releases (SPEC §7 versioning).

## 4. Parallelizable Subtasks

After TASK-04 (the two "first agent" modules establish the working-agent baseline), the remaining module-authoring tasks are an independent fan-out group: TASK-05, TASK-06, TASK-07, TASK-08, TASK-09, TASK-10, TASK-11, TASK-12 can run concurrently (one author per module). TASK-03 (platform overview) can also run in parallel right after TASK-02. They converge at TASK-13 (doc-link wiring) and TASK-14 (tests/coverage). TASK-06 (tools/MCP/A2A) and TASK-09 (evaluation) are the longest poles because they touch the most components.

## 5. Test Strategy

- Chainsaw: assert lab namespace provisioning and starter-project CRD state (Approved namespace + seeded `Agent`/`Sandbox`/`CapabilitySet`), and assert exercise-completion cluster state (e.g. `Agent` runs, `SyntheticMCPServer` created, `CapabilitySet` resolved). Maps AC-E2-04, AC-E2-05, AC-E2-06, AC-E2-10, AC-E2-12.
- Playwright: drive LibreChat interactive-trigger exercises and D2 dashboard / knowledge-check UI. Maps AC-E2-05 (interactive), AC-E2-07.
- PyTest: SDK starter-project unit tests, check-logic unit tests, the §12.2 coverage check (thirteen modules, union covers all topics), outcome-traceability check, and doc-link presence/no-verbatim check. Maps AC-E2-01, AC-E2-02, AC-E2-03, AC-E2-08, AC-E2-09.
- Evaluation module validated by a combined run: a Langfuse-dataset eval, a promptfoo eval, and a three-layer `agent-platform test` invocation, each check-gated. Maps AC-E2-11.
- Fixtures/fakes: for modules whose upstream component or SDK has not yet landed, stub the lab namespace and starter project with a recorded baseline so module authoring proceeds before the real component is available.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (real-build after A/B/C land)

### 6.2 PR — `piece/E2-developer-training-track` → base `wave/consumer`; carries spec-E2 + plan-E2

### 6.3 Merge order — independent of E1 and the other consumer-tier siblings; the consumer wave rolls up to main after Workstreams A/B/C provide the exercised platform + SDKs + docs.

## 7. Effort Estimate

- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 M (spine — two starter projects) · TASK-05 S · TASK-06 L (longest pole — synthetic MCP + A2A) · TASK-07 M · TASK-08 M · TASK-09 L (longest pole — Langfuse datasets + promptfoo + three layers) · TASK-10 M · TASK-11 S · TASK-12 M · TASK-13 S · TASK-14 M.
- Rollup: M (matches CSV estimate). Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-06/TASK-09 → TASK-13 → TASK-14.

## 8. Rollback / Reversibility

Fully reversible: E2 is content + lab definitions + starter projects with no deployed platform surface. Reverting the PR removes the modules, lab templates, and starter projects; nothing downstream breaks (no downstream pieces). Provisioned lab namespaces are torn down by the standard onboarding/teardown path. The only loss on revert is the curriculum itself — no platform component depends on E2.

# PLAN F6 — Production runbook compilation

> spec: SPEC-F6 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes C6 + F1/F2/F4/F5; runs last in v1.0) · estimate: S
> upstream-pieces: [C6, F1, F2, F4, F5, C1] · downstream-pieces: []

## 1. Implementation Strategy

F6 is the closing documentation gate: compile C6's cross-cutting runbooks plus the F1/F2/F4/F5 runbook outputs into one cross-referenced set, verify every per-component (Workstream A 10.7) runbook was exercised at least once, fill cross-cutting incident gaps, publish to the C1 portal, and index into the Knowledge Base for HolmesGPT. No build — compilation, verification matrix, and gap-flagging. It necessarily runs after the other F-pieces (their runbooks are inputs). Critical path: inventory all runbooks → build exercise-verification matrix → author cross-cutting incident runbooks → publish + index → link-check/sign-off.

## 2. Ordered Task List

- **TASK-01:** Runbook inventory — enumerate C6 cross-cutting runbooks + every Workstream A per-component (10.7) runbook + F1/F2/F4/F5 outputs — produces: runbook inventory — depends-on: [].
- **TASK-02:** Exercise-verification matrix — map each runbook → exercise evidence (drill / 3-layer test / real exercise) → owning component; grade evidence strength — produces: verification matrix — depends-on: [TASK-01].
- **TASK-03:** Gap flagging — any unexercised (or weak-evidence) runbook flagged to its owning component — produces: runbook gap list — depends-on: [TASK-02].
- **TASK-04:** Compile + cross-reference — unify into one indexed set; reconcile terminology to Canon; link each runbook to ADR(s) + component spec(s) — produces: compiled runbook set — depends-on: [TASK-01].
- **TASK-05:** Cross-cutting incident runbooks — author the spanning-incident set (chokepoint failure per future §1, broker backlog, correlated gateway/audit/OPA failure, DR invocation) — produces: incident runbooks — depends-on: [TASK-04].
- **TASK-06:** Publish to portal — into Material for MkDocs (C1, ADR 0008) with search/versioning; record content versions reflecting as-shipped v1.0 behavior (ADR 0035 staged-restart) — produces: published runbook set — depends-on: [TASK-04, TASK-05].
- **TASK-07:** Index into Knowledge Base — via C8 into `platform-knowledge-base`; verify HolmesGPT/operator retrieval of a known runbook fact — produces: indexed + retrievable runbooks — depends-on: [TASK-06].
- **TASK-08:** Tests + how-tos — link-check/doc-lint on the compiled set; how-tos (navigate the set; exercise + sign off a runbook); meta-runbook (add/exercise a runbook) — depends-on: [TASK-06, TASK-07].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **C6** — cross-cutting/integrated runbooks (the primary compilation input).
- **F1/F2/F4/F5** — their runbook outputs (retention, DR/restore, security drills, capacity) are folded in; F6 runs after them.
- **C1** — Material for MkDocs portal (publish target, ADR 0008).
- **(Workstream A per-component runbooks)** — the 10.7 deliverables whose exercise F6 verifies.
- **C8** — Knowledge Base indexing pipeline (retrievability for HolmesGPT).

### 3.2 Downstream blocked on this
- None (terminal piece). Operators + HolmesGPT consume the output at runtime.

### 3.3 Continuous (non-blocking) inputs
- **A14/HolmesGPT** (ADR 0012) as runbook consumer. **B14** 3-layer test evidence (exercise proof). **A23/Kargo** promotion/rollback runbook content.

## 4. Parallelizable Subtasks
- TASK-02 (verification matrix) and TASK-04 (compile/cross-reference) can proceed concurrently once the inventory (TASK-01) exists.
- TASK-05 (incident runbooks) parallels the matrix work after the compile baseline.
- TASK-08 (tests/how-tos) parallels after publish/index.

## 5. Test Strategy
- **PyTest (logic):** AC-F6-02/03 (verification matrix completeness + gap-owner assignment), AC-F6-07 (link/Canon/ADR cross-reference check), AC-F6-06 (Knowledge Base retrieval of a known fact).
- **Chainsaw (operator/CRD):** N/A — F6 is documentation; runbook *exercise* evidence is sourced from the owning components' own 3-layer tests, not authored here. (A doc-lint job may run in CI.)
- **Playwright (UI/e2e):** AC-F6-05 (compiled set published + searchable in the MkDocs portal).
- **Fixtures/fakes:** the verification matrix consumes other pieces' exercise evidence — where an upstream runbook/test is missing/late (R2), the matrix records the hole explicitly rather than blocking; weak (test-only) evidence is graded and flagged (R1/OQ1).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (C6, C1, C8 merged to main; F1/F2/F4/F5 merged within the F wave).
### 6.2 PR — `piece/F6-production-runbook-compilation` → base `wave/F`; carries SPEC-F6 + PLAN-F6.
### 6.3 Merge order — **last within the F wave** (depends on F1/F2/F4/F5 runbook outputs); F wave rolls up to main as the production-readiness close-out.

## 7. Effort Estimate
- Inventory + matrix + gap-flag (TASK-01..03): S.
- Compile + incident runbooks (TASK-04,05): S.
- Publish + index + tests/how-tos (TASK-06..08): S.
- **Rollup: S** (matches CSV). **Critical path:** TASK-01 → 02 → 05 → 06 → 07.

## 8. Rollback / Reversibility
Back out by un-publishing the compiled set from the portal and de-indexing it from the Knowledge Base — the per-component and C6 runbooks remain individually available (F6 only compiled/cross-referenced them). Non-destructive. **No downstream break** on revert (F6 is terminal); reverting removes the unified index + verification matrix, re-opening the runbook-readiness gate. Any gaps F6 surfaced remain valid findings for the owning components regardless of F6's own state.

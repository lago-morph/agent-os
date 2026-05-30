# PLAN B9 — agent-platform CLI

> spec: SPEC-B9 · kind: COMPONENT · tier: T1
> wave: W2 · estimate: L
> upstream-pieces: [A1, A2, A5] · downstream-pieces: [B21, B14, B15]

## 1. Implementation Strategy

Build the `agent-platform` CLI as a custom Python container image exposing the nine §8 subcommands as the platform's versioned CI contract (ADR 0030/0010). Implement the orchestration shell first (subcommand surface, config, identity, audit-on-run), then layer in `test` (drives Chainsaw/Playwright/PyTest via the B14 manifest), `eval`/`redteam` (Promptfoo as evaluation, not a fourth layer), `validate`/`scan` (pre-merge static checks), and `promote` (Kargo + OPA gates). Wire `--debug`/`--trace` to the ADR 0035 dynamic toggle with exhaustive restoration (signal handlers + atexit), and make OpenSearch/OTel soft dependencies. Resolve the B9-vs-B14 "who owns the CLI" boundary (OQ-B9-1) with the B14 author before deep implementation: B9 = shell + surface + orchestration; B14 = runner internals + manifest schema + stress probes + result emission.

## 2. Ordered Task List

- **TASK-01:** CLI shell — subcommand registration, config, Keycloak identity resolution, container image — produces: CLI skeleton + image — depends-on: [].
- **TASK-02:** Audit-on-run emission via A18 adapter — produces: run audit — depends-on: [TASK-01].
- **TASK-03:** `--debug`/`--trace` → ADR 0035 toggle with scoped + exhaustive restore (signals + atexit) — produces: toggle wiring — depends-on: [TASK-01].
- **TASK-04:** `test` orchestration over Chainsaw/Playwright/PyTest + aggregation (consumes B14 manifest) — produces: test subcommand — depends-on: [TASK-01, TASK-03].
- **TASK-05:** Soft-dep handling + result-flow split (unit→CI; non-unit→OpenSearch/OTel) — produces: result router — depends-on: [TASK-04].
- **TASK-06:** `eval`/`redteam` (Promptfoo + Langfuse datasets) — produces: eval subcommands — depends-on: [TASK-01].
- **TASK-07:** `validate`/`scan` pre-merge checks (kubeval/conftest/render/dry-run, Trivy/Grype) — produces: static-check subcommands — depends-on: [TASK-01].
- **TASK-08:** `package`/`deploy-preview` — produces: packaging subcommands — depends-on: [TASK-01].
- **TASK-09:** `promote` → Kargo + OPA gate (dry-run aware) — produces: promote subcommand — depends-on: [TASK-01, TASK-07].
- **TASK-10:** `update-base` container-maintenance flow (rebuild/scan/regression-eval/PR) — produces: update-base subcommand — depends-on: [TASK-06, TASK-07].
- **TASK-11:** Env/secret schema + CI endpoint list + semantic-version surface — produces: contract docs — depends-on: [TASK-01..10].
- **TASK-12:** 3-layer tests, dashboard XR (with D3), runbook, CLI how-tos — produces: cross-cutting deliverables — depends-on: [TASK-11].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- A1 (LiteLLM — eval/test gateway calls), A2 (Langfuse — eval datasets), A5 (ARK — validate/package/deploy on Agent CRDs). A18 (audit adapter) for run audit; A7/OPA for `promote` gates; A23 (Kargo) for `promote` runtime (mockable until landed).

### 3.2 Downstream blocked on this
- B14 (test framework drives runners through the CLI), B15 (CI reference pipeline calls the CLI), B21 (dev-environment inner loop).

### 3.3 Continuous (non-blocking) inputs
- B14 manifest schema (co-designed; soft circular — resolve via OQ-B9-1 boundary), B22 threat model, D3 dashboards.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-06, TASK-07, TASK-08 run concurrently (independent subcommands).
- TASK-02 and TASK-03 parallel (audit vs toggle).
- TASK-04→05 sequential; TASK-09 after TASK-07; TASK-10 after TASK-06/07.
- Docs/dashboard/tests (TASK-11, 12) parallel once subcommands stabilize.

## 5. Test Strategy
- **PyTest:** AC-B9-01, -02, -04, -05, -06, -07, -10, -11, -12 (subcommand surface, toggle-restore with kill injection, soft-dep, result-split, scan-block logic).
- **Playwright:** AC-B9-02, -13 (CI-invocation flows, `update-base` PR flow on a fixture repo).
- **Chainsaw:** AC-B9-03, -05, -08, -09 (`test` against cluster, toggle scoping, run audit events, `promote` OPA gate incl. dry-run).
- **Fakes:** mock B14 runners + manifest until B14 lands; fake Kargo + OPA gate (canned allow/deny + `simulated:true`); fake Langfuse/OpenSearch/OTel for soft-dep tests; fixture GitHub repo for `update-base`.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains A1/A2/A5 + W1 specs).
### 6.2 PR — `piece/B9-agent-platform-cli` → base `wave/2`; carries spec-B9 + plan-B9.
### 6.3 Merge order — independent of W2 siblings; coordinate the B9/B14 boundary PR-side before B14 (W3) bases on it. wave/2 rolls up to main.

## 7. Effort Estimate
- TASK-01 M, 02 S, 03 M, 04 L, 05 M, 06 M, 07 M, 08 M, 09 M, 10 M, 11 S, 12 M. Rollup **L**.
- Critical path: TASK-01 → 04 → 05 → 12 (the test-orchestration spine), with TASK-03 toggle and TASK-09 promote on parallel near-critical legs.

## 8. Rollback / Reversibility
The CLI is a versioned container image; rollback = pin the prior image tag. No in-cluster state owned (no reconcile loop). Reverting a subcommand breaks downstream callers (B14 runner invocation, B15 pipeline, B21 inner loop) that pinned the new surface — mitigated by semantic versioning + ≥1-release deprecation window (REQ-B9-11). A bad `--trace` restore is self-healing via the next clean run's restore; the dynamic-toggle service can also be reset out-of-band.

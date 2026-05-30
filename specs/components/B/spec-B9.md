# SPEC B9 — agent-platform CLI

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A1, A2, A5] · downstream: [B21, B14, B15] · adrs: [0011, 0010, 0030, 0034, 0035, 0040] · views: [13, 8, 6.5]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

`agent-platform` is the platform CLI used by developers, operators, and CI (§13, §8). It is the **single integration surface** for CI/CD: v1.0 supports GitHub Actions only, via "option 2" (unified CLI tasks called from native pipelines) — the CLI is the contract; pipelines are syntax around it (§8). The same command runs identically from a developer laptop, from CI, and on schedule (§13, ADR 0011).

The CLI exposes the subcommands enumerated in §8 — `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base` — and orchestrates the three test layers (Chainsaw / Playwright / PyTest) per ADR 0011, reading a manifest of "what runs where," invoking the right runners, and aggregating results. It carries the `--debug` / `--trace` toggle that activates the dynamic log-level / trace-granularity toggle for the duration of a run and restores levels on exit (ADR 0035, ADR 0011). The CLI's **subcommand surface is its versioned API** (semantic versioning, ADR 0030; interface-contract §3.3).

## 2. Scope

### 2.1 In scope
- The `agent-platform` CLI binary, shipped **as a container image** for CI (§8) and runnable on a laptop.
- Subcommands (§8): `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base`.
- `agent-platform test`: reads the test manifest, invokes Chainsaw / Playwright / PyTest runners, aggregates results (§13, ADR 0011). **Promptfoo runs alongside as evaluation, not a fourth layer** (ADR 0011) — surfaced via `eval`/`redteam`.
- `--debug` / `--trace` flags: activate the ADR 0035 toggle for the run's duration, scoped to the components the run exercises, with **exhaustive restoration** on normal exit, Ctrl-C/SIGTERM, and crash (signal handlers + atexit) (§13).
- Soft-dependency handling: OpenSearch (advisory result index) and the OTel collector are soft — **warn, continue, exit non-zero only on actual test failure** (§13).
- Result/diagnostic flow split: unit tests → CI output only; non-unit → OpenSearch advisory index + OTel metrics/logs (ADR 0011).
- Audit emission of every CLI run (start/finish/pass-fail/per-layer) via the platform audit adapter (ADR 0034).
- `promote` invokes Kargo promotion (the CLI's promotion gates honor OPA dry-run, §6.6) consistent with ADR 0040; pre-merge static checks (`validate`/`scan`) complement Kargo runtime gates (§8).
- The schema for required env vars / secrets and the list of CI-reachable endpoints (§8).

### 2.2 Out of scope (and where it lives instead)
- The **test framework runners' implementation** and the test-manifest schema definition + stress-probe harness + result emission wiring — Component **B14** (ADR 0011 — "B14 owns the CLI, the stress-probe harness, OpenSearch/OTel result emission…"). **Reconciliation note:** ADR 0011 attributes CLI *ownership* to B14, while piece-index.csv lists B9 as "agent-platform CLI" with B14 downstream of B9 and B14 titled "agent-platform test framework." This spec scopes **B9 as the CLI shell + subcommand surface + orchestration contract**, and **B14 as the test-framework internals** (runner integration, manifest schema, stress probes, result emission) that the CLI drives. The `agent-platform test` subcommand is B9's surface; its runner internals are B14's. Flagged as OQ-B9-1.
- The **CI/CD reference GitHub Actions pipeline** — Component **B15** (calls the CLI; §8). B9 ships the CLI the pipeline invokes, not the pipeline.
- The **dynamic toggle service** (`LogLevel` reconcile / staged restart) — ADR 0035 owners per component; B9 only *drives* it via `--debug`/`--trace`.
- **Kargo** install + Stages + Warehouse — Component A23 (ADR 0040); `promote` calls Kargo, does not install it.
- The **audit adapter library / endpoint** — Component A18 (ADR 0034); the CLI links it.
- **Test framework dashboards** — Component D3 (ADR 0011).
- The **development environment for agents** — Component B21 (downstream consumer of the CLI).

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM)** — `eval`/`test` exercise gateway calls; CI must reach LiteLLM.
- **A2 (Langfuse)** — `eval` reads/writes Langfuse datasets; CI reaches Langfuse (§8 endpoint list).
- **A5 (ARK)** — `validate`/`test` operate on `Agent` CRDs and AgentRuns; `package`/`deploy-preview` target ARK-reconciled resources.

**Downstream consumers:**
- **B14** (test framework) — drives its runners through the CLI's `test` surface (per piece-index `B9 → B14`).
- **B15** (CI/CD reference pipeline) — native GitHub Actions calling the CLI (`B9 → B15`).
- **B21** (dev environment for agents) — uses the CLI for the inner-loop (`B9 → B21`).

**ADRs honored:**
- **ADR 0011** — three-layer testing with CLI orchestration; `--debug`/`--trace` toggle; soft deps; result-flow split; audit on every run.
- **ADR 0010** — GitHub Actions only; CLI is the integration surface so other CI is a thin shim later.
- **ADR 0030** — CLI subcommand surface is the semantically-versioned API.
- **ADR 0034** — every run emits audit via the adapter.
- **ADR 0035** — `--debug`/`--trace` map to the dynamic toggle, scoped + exhaustively restored.
- **ADR 0040** — `promote` composes with Kargo; pre-merge CLI checks complement Kargo runtime gates.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — B9 owns no CRD. It **operates on** `Agent`/`AgentRun` (A5), drives the `LogLevel` toggle (ADR 0035) via `--debug`/`--trace` rather than authoring `LogLevel` CRs directly, and triggers Kargo promotion (which may create an `Approval` CRD per ADR 0040). No `HealthCheck`/`ScheduledTest` CRDs (deferred, ADR 0011).

### 4.2 APIs / SDK surfaces
- **Exposes** the CLI subcommand surface as the versioned API (ADR 0030; interface-contract §3.3): `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base` — these nine names are Canon (§8) and reproduced verbatim. Per-subcommand flags beyond `--debug`/`--trace` are `[PROPOSED — not in source]`.
- **Consumes:** Langfuse API (datasets/eval), LiteLLM API, Kubernetes/ARK API, ArgoCD (dry-run), OPA bundle store, Kargo (promote), container registry, OpenSearch (advisory index), OTel collector, the audit adapter library — per the §8 CI endpoint list.
- A schema for required env vars / secrets (§8). `[PROPOSED — not in source]` the concrete variable names.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Audit of runs → `platform.audit.*` via adapter (ADR 0034). Eval/red-team results → `platform.evaluation.*` (the namespace covers "Evaluation run started/completed, A/B test results, red-team findings"). `[PROPOSED — not in source]` concrete event-type names (B12 registry). Promotion actions audited under `platform.audit.*`; policy dry-runs under `platform.policy.*` (§6.6).

### 4.4 Data schemas / connection-secret contracts
- Test manifest schema — defined by B14 (ADR 0011 design-time deliverable, backlog §4); B9 consumes it. Non-unit results stream to the OpenSearch advisory index (shape owned by B14/D3). No connection-secret XRD owned. Secrets via CI's encrypted store / env-var schema (§8).

## 5. OSS-vs-Custom Decision

**Build-new.** No OSS CLI provides the platform's unified `validate/eval/redteam/package/test/deploy-preview/promote/scan/update-base` surface across Chainsaw/Playwright/PyTest + Promptfoo + Kargo + audit. The CLI is custom Python, packaged as a container image, wrapping OSS runners (Chainsaw, Playwright, PyTest, Promptfoo, Trivy/Grype for `scan`) and platform APIs. Rationale (§8, ADR 0011/0010): the CLI is the CI contract so GitHub-Actions-only stays mechanical and adding Jenkins/GitLab later is a thin shim. No fork.

## 6. Functional Requirements

- **REQ-B9-01:** The CLI MUST expose the nine subcommands `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base` (§8).
- **REQ-B9-02:** The CLI MUST ship as a container image runnable identically in GitHub Actions CI and on a developer laptop (§8, ADR 0011).
- **REQ-B9-03:** `agent-platform test` MUST read the test manifest, invoke the applicable Chainsaw / Playwright / PyTest runners, and aggregate results (§13, ADR 0011).
- **REQ-B9-04:** Promptfoo MUST run as evaluation (via `eval`/`redteam`), NOT as a fourth structural test layer (ADR 0011).
- **REQ-B9-05:** `--debug` / `--trace` MUST activate the ADR 0035 dynamic toggle for the run's duration only, scoped to the components the run exercises, and MUST restore prior levels on normal exit, on Ctrl-C/SIGTERM, and on crash paths (signal handlers + atexit) (§13).
- **REQ-B9-06:** OpenSearch and the OTel collector MUST be soft dependencies: the CLI warns, continues, and exits non-zero only on actual test failure when either is unreachable (§13).
- **REQ-B9-07:** Unit-test results MUST stay CI-output only; non-unit results MUST stream to the OpenSearch advisory index and publish metrics/logs via OTel (ADR 0011).
- **REQ-B9-08:** Every CLI run MUST emit audit events (start, finish, pass/fail, per-layer) via the platform audit adapter (ADR 0034).
- **REQ-B9-09:** `promote` MUST invoke Kargo promotion and honor OPA promotion-gate decisions (incl. dry-run `simulated: true`) consistent with ADR 0040 / §6.6.
- **REQ-B9-10:** `validate` and `scan` MUST run the pre-merge static checks (kubeval, conftest, helm template / kustomize build render, ArgoCD dry-run; Trivy/Grype + dependency/CRD/CloudEvent schema scans) that complement Kargo runtime gates (§8).
- **REQ-B9-11:** The CLI's subcommand surface MUST be semantically versioned as the platform's CLI API (ADR 0030); deprecated subcommands/flags reachable ≥1 release after replacement.
- **REQ-B9-12:** The CLI MUST publish a schema of required env vars / secrets and the list of network endpoints CI runners must reach (§8).
- **REQ-B9-13:** `update-base` MUST support the container-maintenance flow (rebuild base, scan, regression eval, open PR to bump Agent CRD image refs) per §8.

## 7. Non-Functional Requirements

- **Security:** the CLI is a privileged CI surface (registry push, merge-trigger) — no long-lived static creds; OIDC federation to AWS where possible, scoped tokens, SHA-pinned actions, signed artifacts (§8 security-first design). `scan` gates on critical CVEs.
- **Multi-tenancy (§6.9):** CLI actions resolve to the operator's Keycloak identity / `platform_namespaces`; `deploy-preview`/`promote`/`test` operate only within the caller's authorized namespaces; OPA gates promotion.
- **Observability (§6.5):** non-unit results to OpenSearch advisory index + OTel; `trace_id` correlation (ADR 0015); test-framework-health dashboard fed (§11.1, owned by D3).
- **Scale:** stateless per invocation; result streaming decoupled from execution; laptop runs degrade gracefully without cluster (§13).
- **Versioning (ADR 0030):** semantic versioning; compatibility with gateway/ARK/Langfuse versions documented.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests — N/A — the CLI is a container image + CI artifact, not an in-cluster Deployment (no reconcile loop, ADR 0011). Image build manifests applicable instead.
- Per-product docs (10.5) — **applicable** (CLI reference, env/secret schema, CI endpoint list).
- Runbook (10.7) — **applicable** (CI failure modes, toggle-not-restored recovery, soft-dep degradation).
- Alerts — **applicable** (CI run failure rate, promote-gate denials, scan-block-on-CVE) — mostly via CI/D3.
- Grafana dashboard (Crossplane XR) — **applicable** (test-framework health, §11.1) — owned jointly with D3; delivered as `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — N/A — the CLI is not a CRD admin surface.
- OPA/Rego integration — **applicable** (`promote` and `validate`/`scan` honor OPA gates incl. dry-run; conftest against the Rego library).
- Audit emission (ADR 0034) — **applicable** (every run; core deliverable).
- Knative trigger flow — **applicable** (eval/red-team findings under `platform.evaluation.*`; scheduled CI runs are the v1.0 substitute for periodic in-cluster scheduling, ADR 0011). `[PROPOSED — not in source]` concrete event names.
- HolmesGPT toolset — **applicable** (CLI-invokable diagnostics, e.g., re-run with `--trace`). `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (PyTest for CLI/subcommand/toggle-restore logic; Playwright for CI-invocation flows; Chainsaw for `deploy-preview`/`promote` against cluster). The CLI also *orchestrates* the three layers for other components.
- Tutorials & how-tos — **applicable** (reference GitHub Actions how-to is B15; CLI usage how-tos here).

## 9. Acceptance Criteria

- **AC-B9-01** (REQ-B9-01): `agent-platform --help` lists exactly the nine subcommands; each is invocable. (PyTest)
- **AC-B9-02** (REQ-B9-02): The same image yields identical subcommand behavior in a GitHub Actions job and on a laptop. (PyTest/CI)
- **AC-B9-03** (REQ-B9-03): `agent-platform test` reads a manifest and invokes the declared Chainsaw/Playwright/PyTest runners, returning an aggregated result. (PyTest/Chainsaw)
- **AC-B9-04** (REQ-B9-04): Promptfoo runs under `eval`/`redteam` and does not appear as a structural test layer in `test` aggregation. (PyTest)
- **AC-B9-05** (REQ-B9-05): A `--trace` run raises verbosity only for exercised components and restores prior levels on clean exit, on SIGTERM, and on a forced crash. (Chainsaw/PyTest with kill injection)
- **AC-B9-06** (REQ-B9-06): With OpenSearch and the OTel collector unreachable, a passing run exits 0 with warnings; a failing run exits non-zero. (PyTest)
- **AC-B9-07** (REQ-B9-07): Unit results appear only in CI output; non-unit results appear in the OpenSearch advisory index and OTel. (PyTest/Chainsaw)
- **AC-B9-08** (REQ-B9-08): A run emits start/finish/pass-fail/per-layer audit events via the adapter. (Chainsaw/PyTest)
- **AC-B9-09** (REQ-B9-09): `promote` is denied when OPA denies the promotion action and proceeds when allowed; dry-run returns `simulated: true` without promoting. (Chainsaw)
- **AC-B9-10** (REQ-B9-10): `validate`/`scan` run kubeval/conftest/render/dry-run + Trivy/Grype and block on a critical CVE / schema error. (PyTest/CI)
- **AC-B9-11** (REQ-B9-11): A deprecated subcommand/flag remains usable with a deprecation warning for the documented window. (PyTest)
- **AC-B9-12** (REQ-B9-12): The published env/secret schema and CI endpoint list match what the subcommands actually require. (PyTest)
- **AC-B9-13** (REQ-B9-13): `update-base` rebuilds, scans, runs regression eval, and opens a PR bumping Agent CRD image refs. (CI/Playwright on a fixture repo)

## 10. Risks & Open Questions

- **OQ-B9-1** (high): B9-vs-B14 ownership boundary of "the CLI." ADR 0011 says B14 owns the CLI; piece-index.csv names B9 the CLI and B14 the test framework with B9→B14. Reconciliation (this spec): B9 = CLI shell + subcommand API + orchestration contract; B14 = runner internals + manifest schema + stress probes + result emission. Confirm with the B14 author before implementation. Blast radius: high (duplicated/missing ownership).
- **OQ-B9-2** (med): Per-subcommand flag grammar, env/secret variable names, and the test-manifest schema are design-time (backlog §4). `[PROPOSED — not in source]`.
- **R-B9-1** (med): `--trace` restoration must survive crashes; a missed restore leaves the cluster verbose. Mitigated by signal handlers + atexit + AC-B9-05; blast radius scoped to exercised components.
- **R-B9-2** (med): The CLI is a privileged surface (registry push, merge trigger). Mitigated by §8 security-first design (scoped OIDC tokens, signed artifacts); blast radius high if compromised.
- **OQ-B9-3** (low): Whether `eval` red-team uses Langfuse datasets exclusively or also local fixtures on laptop runs. Reconciliation: soft-dep degradation (REQ-B9-06) applies. `[PROPOSED — not in source]`.

## 11. References

- architecture-overview.md §8 CI/CD integration requirements — subcommand list, option-2 CLI-as-contract, container-maintenance, pre-merge checks complement Kargo, security-first design (~1364–1388).
- architecture-overview.md §13 Testing framework — three layers, CLI coordination, audit emission, soft deps, `--debug`/`--trace` exhaustive restoration, reporting split (~1613–1639).
- architecture-overview.md §11.1 test-framework-health dashboard (~1559).
- ADR 0011 (three-layer testing + CLI orchestration; CLI/B14 ownership note in Consequences), ADR 0010 (GitHub Actions only), ADR 0030 (CLI surface is the versioned API), ADR 0034 (audit on runs), ADR 0035 (dynamic toggle), ADR 0040 (Kargo `promote`).
- Interface-contract §3.3 (`agent-platform` CLI semantic versioning; HTTP `/v1/...`), §2 (`platform.evaluation.*`, `platform.audit.*`, `platform.policy.*`), §5 (audit adapter).
- Related pieces: A1, A2, A5 (upstream); B14, B15, B21 (downstream); A23 (Kargo), D3 (dashboards), A18 (audit).

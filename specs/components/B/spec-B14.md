# SPEC B14 — `agent-platform` test framework

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [B9] · downstream: [B15] · adrs: [0011, 0009, 0015, 0034, 0035, 0030, 0031, 0010]
> · views: [6.5]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B14 is the **`agent-platform` test framework** — the test-orchestration surface every other
component plugs its tests into. Per ADR 0011 the platform standardizes on **three test layers**
— **Chainsaw** (declarative Kubernetes-level e2e), **Playwright** (UI + HTTP API flows), and
**PyTest** (Python unit / callback / kopf-operator tests) — coordinated by the **`agent-platform
test` CLI**, which reads a manifest declaring "what runs where," invokes the right runners, and
aggregates results. The same command runs identically from a developer laptop, from CI, and on
schedule (§13). Because of that, B14 is a **continuous, non-blocking input to every A and B
component** (§14.7: "B14 (test framework) → A, → B" is a fan-out edge, not a wave-depth-setting
dependency).

B14 owns the orchestration CLI surface (a subcommand area of the larger `agent-platform` CLI,
B9), the **stress-probe harness** (low-volume concurrent invocation on top of Chainsaw/Playwright
watching existing observability for saturation), **test-result metrics emission** (non-unit
results stream to OpenSearch advisory index; metrics+logs publish via OTel), **audit emission**
for test runs (ADR 0034), the **`--debug`/`--trace` toggle** wiring (ADR 0035), and **dashboard
provisioning** (the metrics/index views that D3 builds dashboards over).

The problem it solves: without one orchestration contract, each component would invent its own
runner glue, result sink, and CI shim — defeating the "one CLI from laptop, CI, and schedule"
rationale that also makes the GitHub-Actions-only CI choice (ADR 0010) mechanical.

## 2. Scope

### 2.1 In scope
- The **`agent-platform test`** orchestration: manifest-driven invocation of Chainsaw / Playwright
  / PyTest runners and **result aggregation** into one pass/fail.
- The **stress-probe harness**: low-volume N-parallel agent-invocation runs on top of Chainsaw or
  Playwright, watching existing observability for saturation (no locust/k6).
- **Result + metrics emission**: non-unit results → OpenSearch advisory index; metrics + logs →
  OTel collector (Tempo/Mimir/Loki path); **unit results stay CI-output-only** (ADR 0011).
- **Audit emission** of every test run (start/finish/pass-fail/per-layer) via the audit adapter/
  endpoint (ADR 0034).
- **Soft-dependency degradation**: OpenSearch + OTel unreachable ⇒ warn, continue, exit non-zero
  **only on real test failure** (so a laptop run without cluster access still produces local output).
- **`--debug` / `--trace`** flags: activate the dynamic log-level/trace-granularity toggle (ADR 0035)
  for the run, scoped to the components the run touches, with **exhaustive restoration** on normal
  exit, Ctrl-C/SIGTERM, and crash (signal handlers + atexit).
- **Dashboard provisioning**: the OpenSearch result views + metric series that D3 builds on.
- The **test manifest schema** ("what runs where") — a design-time deliverable B14 must specify.

### 2.2 Out of scope (and where it lives instead)
- **The rest of the `agent-platform` CLI** (`validate`, `eval`, `redteam`, `package`, `deploy-preview`,
  `promote`, `scan`, `update-base`) and the CLI's packaging/versioning shell → **B9**. B14 owns the
  `test` subcommand area + the stress/metrics/toggle machinery behind it.
- **The CI/CD reference pipeline** that *calls* the CLI → **B15** (GitHub Actions, ADR 0010).
- **The actual per-component tests** → each A/B component's own deliverable list (§13, §14.7).
- **The test framework dashboards** (Grafana panels over B14's metrics/index) → **D3** (ADR 0011
  consequence: D3 owns the dashboards built on B14's metrics/OpenSearch views).
- **The dynamic-toggle service** itself (`LogLevel` reconcile, rolling-restart machinery) → ADR 0035
  owners / per-component; B14 only *drives* it for a run.
- **The audit adapter/endpoint** → A18 / ADR 0034; B14 *links* the adapter.
- **promptfoo evaluation** — promptfoo is an evaluation tool, **not a fourth test layer** (ADR 0011);
  it runs alongside in CI (B15) but is not part of B14's Chainsaw/Playwright/PyTest contract.
- **Declarative test CRDs** / periodic in-cluster scheduling (`HealthCheck`/`ScheduledTest`) →
  **deferred to future-enhancements** (ADR 0011); v1.0 is on-demand + CI-triggered only.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **B9 (`agent-platform` CLI)** — B14's `test` orchestration is a subcommand of the B9 CLI image;
  B14 binds to B9's CLI framework, packaging, and semantic-versioned subcommand surface (ADR 0030).

**Downstream consumers:**
- **B15** — the GitHub Actions reference pipeline calls `agent-platform test` (and the stress-probe
  harness) as CI steps.
- **D3** — builds the test framework dashboards over B14's OpenSearch result index + OTel metrics.
- **Every A and B component** — contributes Chainsaw/Playwright/PyTest tests that B14 orchestrates
  (continuous fan-out edge, §14.7; non-blocking).

**Soft dependencies (ADR 0011):** OpenSearch (advisory result index) and the OTel collector
(metrics/logs) — B14 degrades gracefully when either is unreachable.

**ADRs honored:**
- **ADR 0011** — three layers + CLI orchestration; promptfoo is not a fourth layer; result-flow
  split (unit→CI-only, non-unit→OpenSearch+OTel); stress probes reuse functional runners; no
  Allure/ReportPortal; no test CRDs in v1.0.
- **ADR 0009** — OpenSearch advisory index role (rebuildable; never authoritative).
- **ADR 0015** — OTel/Tempo+Mimir+Loki path; trace correlation by `trace_id`.
- **ADR 0034** — test runs emit audit through the same adapter/endpoint.
- **ADR 0035** — `--debug`/`--trace` drive the dynamic toggle, scoped + exhaustively restored.
- **ADR 0010** — the CLI is the CI contract; B14 makes adding another CI mechanical.
- **ADR 0030 / 0031** — CLI subcommand surface is versioned; any emitted events use the closed taxonomy.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
**N/A — B14 ships no CRD.** ADR 0011 is explicit: "**No test CRDs** unless we hit a need
declarative test execution can't meet," and the `HealthCheck`/`ScheduledTest` XRDs are
**deferred to future-enhancements**. B14 may *provision* a `GrafanaDashboard` XR for its result
views (see §8), but that XRD is owned by B4. The **`LogLevel` CR** (ADR 0035) is consumed/driven,
not defined here (owned per-component / ADR 0035).

### 4.2 APIs / SDK surfaces
- **`agent-platform test`** CLI surface — the versioned API (ADR 0030: "CLI subcommand surface is
  the versioned API"). Source names the **flags `--debug` / `--trace`** (ADR 0035) and the manifest
  ("reads a manifest declaring what runs where"). Concrete additional flags/subcommand names beyond
  `test`, `--debug`, `--trace` are `[PROPOSED — not in source]`.
- **Test manifest schema** ("what runs where") — a **design-time deliverable** ADR 0011 requires to
  be specified *before* B14 implementation. Its concrete field set is `[PROPOSED — not in source]`
  (held in backlog §4); B14 owns specifying it.
- **Stress-probe harness** — invoked via the same CLI; parameterized by concurrency N. Exact flag
  names `[PROPOSED — not in source]`.
- B14 consumes the **runner CLIs** (Chainsaw, Playwright, PyTest) — vendor surfaces, not Canon.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emits** under `platform.audit.*` — every test run (start/finish/pass-fail/per-layer results) is
  first-class auditable activity (§13, ADR 0034). Per-event-type schemas live in B12.
- **Emits** under `platform.evaluation.*` — `[PROPOSED — not in source]` for evaluation/red-team
  result events if surfaced as CloudEvents (interface-contract §2 maps "evaluation run started/
  completed, A/B test results, red-team findings" here); source states results **stream to
  OpenSearch** and **publish via OTel**, which is the primary path — CloudEvent emission of results
  is not source-stated, hence flagged.
- **Consumes**: none required. B14 reads runner output, not a CloudEvent subscription.
- Non-unit result streaming to OpenSearch + OTel metrics/logs is **not** itself a CloudEvent path —
  it is the OpenSearch advisory index + OTel collector path (ADR 0009 / 0015).

### 4.4 Data schemas / connection-secret contracts
- **OpenSearch advisory result index** — schema of the per-run result document is `[PROPOSED — not
  in source]`; it must be **rebuildable / advisory** per the OpenSearch invariant (ADR 0009) and is
  never a system of record.
- **OTel metrics/logs** — published along the standard collector path (ADR 0015); metric names
  `[PROPOSED — not in source]`.
- **Connection handling** — B14 reaches OpenSearch/OTel as endpoints; both are **soft** (ADR 0011).
  No dedicated connection-secret of its own beyond what the CLI environment provides. The §8 CI
  env-var/secret + network-endpoint schema (Langfuse, Git, registry, ArgoCD, OPA bundle store,
  OpenSearch) is shared with B9/B15.

## 5. OSS-vs-Custom Decision
**Custom orchestration over OSS runners.** Upstream OSS runners: **Chainsaw**, **Playwright**,
**PyTest** (no versions pinned in source — `[PROPOSED — not in source]` to pin in the CLI image).
Approach: **build-new** orchestration + **wrap** the three runners behind one manifest-driven CLI;
the stress-probe harness is built on top of Chainsaw/Playwright (no new load tool — no locust/k6,
ADR 0011). Result sink reuses platform OSS (OpenSearch, OTel collector) rather than a test-only
stack. Rationale (ADR 0011): one CLI across laptop/CI/schedule keeps the GitHub-Actions-only choice
mechanical and avoids a parallel toolchain; Allure/ReportPortal are deferred until felt pain.

## 6. Functional Requirements
- **REQ-B14-01:** B14 MUST orchestrate **Chainsaw, Playwright, and PyTest** via the `agent-platform
  test` CLI, reading a manifest declaring "what runs where" and aggregating results (ADR 0011).
- **REQ-B14-02:** The CLI MUST run **identically from a developer laptop, from CI, and on schedule**
  (same command, same manifest) (§13).
- **REQ-B14-03:** B14 MUST provide a **stress-probe harness** running N parallel agent invocations
  on Chainsaw or Playwright, detecting saturation via **existing observability** (no locust/k6).
- **REQ-B14-04:** **Non-unit** results MUST stream to the **OpenSearch advisory index** and metrics+
  logs MUST publish via **OTel**; **unit** results MUST stay **CI-output-only** (ADR 0011).
- **REQ-B14-05:** Every test run MUST emit **audit events** (start/finish/pass-fail/per-layer) via
  the audit adapter/endpoint (ADR 0034).
- **REQ-B14-06:** When OpenSearch or OTel is unreachable, the CLI MUST **warn, continue, and exit
  non-zero only on an actual test failure** (soft dependencies, ADR 0011).
- **REQ-B14-07:** `--debug` / `--trace` MUST activate the dynamic log/trace toggle (ADR 0035) **for
  the run only**, scoped to the components the run exercises (not the whole platform).
- **REQ-B14-08:** Toggle restoration MUST be **exhaustive** — prior levels restored on normal exit,
  on Ctrl-C / SIGTERM, and on crash (signal handlers + atexit) (ADR 0035).
- **REQ-B14-09:** B14 MUST **specify the test manifest schema** as a design-time deliverable before
  implementation (ADR 0011 / backlog §4).
- **REQ-B14-10:** B14 MUST **provision the result views / metric series** that D3 dashboards build
  on (OpenSearch index + OTel metrics).
- **REQ-B14-11:** The `agent-platform test` subcommand surface MUST be **semantically versioned**;
  the subcommand surface is the versioned API (ADR 0030).
- **REQ-B14-12:** promptfoo MUST be runnable **alongside** the three layers in CI but MUST NOT be
  part of the Chainsaw/Playwright/PyTest structural contract (ADR 0011).
- **REQ-B14-13:** A laptop run **without cluster access** MUST still produce local pass/fail output
  (remote indexing degrades, run still works) (ADR 0011).

## 7. Non-Functional Requirements
- **Security:** the CLI runs with the invoking identity's scope; audit attributes "who ran what test
  against which environment" (ADR 0034). In CI the pipeline (B15) supplies scoped tokens.
- **Multi-tenancy (§6.9):** the `--debug`/`--trace` blast radius is **scoped to the exercised
  components**, not platform-wide, so a debug run in one tenant's namespace does not raise verbosity
  globally (ADR 0035).
- **Observability (§6.5):** shares the production OTel + OpenSearch pipelines rather than a parallel
  test stack; result trend dashboards over the OpenSearch index (D3).
- **Scale:** stress probes target **gross race conditions / resource exhaustion at modest
  concurrency**, explicitly **not** performance benchmarking (ADR 0011).
- **Reliability of restoration:** the toggle must never leave the cluster verbose after an aborted
  run — exhaustive restore is a correctness property, not best-effort (ADR 0035).
- **Versioning (ADR 0030):** the manifest schema + subcommand surface are versioned; B14 ships the
  pinned runner versions in the CLI image.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A (CLI image, not a workload)** — B14 ships as a container image / CLI, not a
  deployed service; no Helm chart of its own. (It *provisions* dashboards as XRs — see below.)
- Per-product docs (10.5) — **applicable** (manifest schema reference, how to add a component's tests,
  stress-probe + `--trace` usage).
- Runbook (10.7) — **applicable** (degraded-backend behavior, stuck-toggle recovery, flaky-test triage).
- Alerts — **applicable** (result-stream failure, toggle-not-restored). `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** — B14 **provisions** the result views/metrics;
  **D3 owns the dashboards** built on them (`GrafanaDashboard` XR, ADR 0021).
- Headlamp plugin — **N/A** — no admin UI; results surface in Grafana/OpenSearch + CI output.
- OPA/Rego integration — **N/A** — orchestration emits no OPA decisions of its own.
- Audit emission (ADR 0034) — **applicable** (every run audited).
- Knative trigger flow — **N/A in v1.0** — periodic in-cluster scheduling (`HealthCheck`/
  `ScheduledTest`) is deferred; v1.0 scheduling is CI-cron invoking the CLI.
- HolmesGPT toolset — **N/A** — no diagnostic toolset contributed.
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable (self-test)** — B14 tests *itself*:
  PyTest for aggregation/soft-degradation/restore logic; Chainsaw/Playwright smoke that orchestration
  invokes each runner and emits audit/results.
- Tutorials & how-tos — **applicable** ("write your component's three-layer tests", "run a stress
  probe", "debug a flaky test with `--trace`").

## 9. Acceptance Criteria
- **AC-B14-01:** Given a manifest, `agent-platform test` invokes Chainsaw, Playwright, and PyTest and
  emits one aggregated pass/fail. (→ REQ-B14-01)
- **AC-B14-02:** The identical command + manifest runs on a laptop, in CI, and from a scheduled job.
  (→ REQ-B14-02)
- **AC-B14-03:** A stress probe runs N parallel invocations and surfaces a saturation signal from
  existing observability (no new load tool present). (→ REQ-B14-03)
- **AC-B14-04:** A non-unit run writes to the OpenSearch index and publishes OTel metrics; a unit run
  writes neither (CI output only). (→ REQ-B14-04)
- **AC-B14-05:** A run emits start/finish/pass-fail/per-layer audit events at the audit endpoint.
  (→ REQ-B14-05)
- **AC-B14-06:** With OpenSearch + OTel made unreachable, a passing run exits 0 with warnings; a
  failing run still exits non-zero. (→ REQ-B14-06, REQ-B14-13)
- **AC-B14-07:** `--trace` raises verbosity only on the exercised components (verified: an untouched
  component's level is unchanged) for the run's duration. (→ REQ-B14-07)
- **AC-B14-08:** Prior log/trace levels are restored after normal exit, after SIGTERM, and after a
  simulated crash. (→ REQ-B14-08)
- **AC-B14-09:** The test manifest schema is published and validated before implementation tasks
  begin. (→ REQ-B14-09)
- **AC-B14-10:** The OpenSearch result index + OTel metric series exist and a D3 dashboard can be
  built against them. (→ REQ-B14-10)
- **AC-B14-11:** A breaking change to the `test` subcommand surface bumps the major version per
  ADR 0030. (→ REQ-B14-11)
- **AC-B14-12:** promptfoo runs alongside the three layers in a CI invocation without being counted
  as a fourth layer in the aggregated layer report. (→ REQ-B14-12)

## 10. Risks & Open Questions
- **R1 (high):** The **test manifest schema is `[PROPOSED — not in source]`** and ADR 0011 requires
  it specified *before* implementation. If it lands late, every component's test contribution stalls.
  Mitigation: REQ-B14-09 makes it the first task. Blast radius: high (fan-out to all A/B).
- **R2 (med):** **Exhaustive toggle restoration** across crash paths is genuinely hard (signal
  handlers + atexit can both be bypassed by SIGKILL). Mitigation: a TTL/`expiresAt` on the `LogLevel`
  CR (ADR 0035 field) as a backstop so an un-restored toggle self-expires. `[PROPOSED — not in source]`
  that B14 sets `expiresAt`.
- **R3 (med):** **Toggle scoping hints** — ADR 0011 says the CLI "passes scoping hints through to the
  dynamic toggle service"; the hint shape is `[PROPOSED — not in source]` and must align with ADR 0035.
- **R4 (low/med):** **OpenSearch result-document schema + OTel metric names** are `[PROPOSED — not in
  source]`; D3 dashboards bind to them, so they must be frozen with D3 in mind.
- **R5 (low):** Boundary with **B9** — which flags belong to the shared CLI shell vs. the `test`
  subcommand. Mitigation: B9 owns shell/packaging/versioning; B14 owns `test` + stress/metrics/toggle.
- **OQ1:** Are non-unit **results also emitted as `platform.evaluation.*` CloudEvents**, or only
  streamed to OpenSearch + OTel? Source states the latter; CloudEvent emission flagged `[PROPOSED]`.
- **OQ2:** Pinned versions of Chainsaw/Playwright/PyTest in the CLI image — `[PROPOSED — not in
  source]`; must be pinned and bumped via the container-maintenance discipline (§8).

## 11. References
- architecture-overview.md §13 (line 1613; three layers, CLI orchestration, audit emission, soft
  deps, `--debug`/`--trace`, reporting to OpenSearch+OTel, stress probes, deferred test CRDs),
  §6.5 (line 370; OTel/Tempo/Mimir/Loki + audit pipeline + dynamic toggle), §8 (line 1364; CLI as
  CI contract, env-var/secret + network-endpoint schema, scan steps), §14.7 (B14 fan-out edge).
- ADR 0011 (three-layer testing + CLI orchestration; promptfoo not a layer; result split; no
  Allure/ReportPortal; no test CRDs; B14 ownership; manifest is a design-time deliverable).
- ADR 0009 (OpenSearch advisory index), ADR 0015 (OTel/Tempo+Mimir+Loki), ADR 0034 (audit),
  ADR 0035 (dynamic toggle), ADR 0010 (CLI is CI contract), ADR 0030/0031 (versioning/taxonomy).
- interface-contract §2 (taxonomy), §3.3 (`agent-platform` CLI versioned subcommand surface),
  §1.5 (`LogLevel`). Glossary: `agent-platform`, `LogLevel`.
- Related pieces: B9 (CLI), B15 (CI pipeline), D3 (dashboards), B12 (event schemas), A18 (audit).

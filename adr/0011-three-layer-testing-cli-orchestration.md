# ADR 0011: Three-layer testing with CLI orchestration

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs automated tests in place from day one across very different surfaces: Kubernetes CRDs and operators, UI and HTTP API flows, and Python libraries (SDKs, callbacks, kopf operators). Tests for each component are part of that component's deliverable list, not an afterthought, and must run identically from a developer laptop, from CI, and on schedule. We also need a way to catch gross race conditions and resource exhaustion at modest concurrency without standing up a full performance lab, and we need test results to feed both PR feedback and longer-term trend dashboards.

## Decision

We adopt three test layers — **Chainsaw** for declarative end-to-end Kubernetes-level tests, **Playwright** for UI and HTTP API flows, and **PyTest** for Python unit and operator tests — with each component contributing tests at whichever layers apply. The **`agent-platform test` CLI** orchestrates all three: it reads a manifest declaring "what runs where," invokes the right runners, and aggregates results.

Result and diagnostic flow is split by layer:

- **Unit tests (PyTest unit scope)** stay CI-output only — PR comments and commit statuses, no streaming to platform observability. The cost of wiring every unit run into OpenSearch and OTel exceeds the diagnostic value at that granularity.
- **All non-unit tests** (Chainsaw, Playwright, PyTest integration/operator) stream per-run test results to **OpenSearch** as an advisory index (per the OpenSearch role in ADR 0009) and publish run metrics + logs via **OpenTelemetry** along the Tempo/Mimir/Loki path (per ADR 0015). This decouples test execution from result analysis and keeps test diagnostics consistent with the rest of the platform's observability model — the same dashboards, the same query surfaces, the same retention story.
- **Test runs themselves emit audit events** through the same audit adapter as the rest of the platform (per ADR 0034), so "who ran what test against which environment" is queryable alongside other platform actions.

The CLI gains a **debug/tracing toggle** (`--debug` / `--trace`) that activates the dynamic log-level and trace-granularity toggle from ADR 0035 for the duration of a single run and disables it on exit. This is the supported path for diagnosing a flaky test: a developer raises verbosity for one run, gets the extra signal in OpenSearch and Tempo, and the cluster returns to baseline observability load afterwards rather than carrying elevated tracing indefinitely.

Stress probes are implemented as low-volume concurrent invocation harnesses on top of Chainsaw or Playwright, watching existing observability for saturation rather than benchmarking.

## Alternatives considered

- **Declarative test CRDs** — Considered as the execution model in place of a CLI. Rejected for v1.0: the CLI keeps execution uniform across laptop, CI, and schedule with no in-cluster control loop to operate, and CRDs would couple test execution to a live cluster. Specifically, CRD-shaped scheduled-test execution (`HealthCheck` / `ScheduledTest` XRDs for periodic platform-health checks) is **deferred to future-enhancements** and is not part of v1.0; CLI orchestration remains the only mechanism. Add CRDs only if declarative test execution becomes a felt need that the CLI cannot meet (backlog 2.14).
- **Custom load testing harness** — Considered for stress and load coverage in place of concurrent Chainsaw / Playwright invocation. Rejected: the goal at v1.0 is finding gross race conditions at low concurrency, not benchmarking, and reusing the same runners that already drive functional tests avoids a parallel toolchain (backlog 2.13).
- **Allure / ReportPortal test reporting from day one** — Considered to provide cross-run trend analysis and per-run inspection beyond what CI output offers. Rejected initially: CI output plus Grafana dashboards built from the test result metrics covers MVP, and adding either tool earns its keep only when trend analysis or defect grouping becomes felt pain (backlog 2.12).

## Consequences

- One CLI runs the same test invocations from laptop, CI, and schedule, which keeps the GitHub-Actions-only CI choice mechanical: adding Jenkins or GitLab CI later is a thin shim that calls the same CLI (see ADR 0010).
- Workstream B14 owns the CLI, the stress-probe harness, OpenSearch/OTel result emission, audit emission, the `--debug` / `--trace` toggle wiring, and dashboard provisioning; Workstream D3 owns the test framework dashboards built on those metrics and the OpenSearch result views.
- The test manifest schema for `agent-platform test` is a design-time deliverable held in backlog section 4 and must be specified before B14 implementation begins.
- Revisit triggers are explicit: adopt Allure or ReportPortal when CI output plus Grafana stops covering trend analysis or per-run inspection (backlog 3.4); adopt test CRDs when the CLI-driven model fights us on scheduled execution or complex composition (backlog 3.7); build a custom load testing harness when a real production race condition slips past the stress probes (backlog 3.8).
- Each component's deliverable list explicitly enumerates Chainsaw, Playwright, and PyTest where each applies, so test ownership is distributed rather than centralized in a QA component.
- No locust or k6 in v1.0; stress probes deliberately reuse the functional runners and rely on existing observability signals for saturation detection.
- Non-unit test result streaming makes OpenSearch and the OTel pipeline soft dependencies of `agent-platform test`: the CLI must degrade gracefully (warn, continue, exit non-zero only on actual test failure) when those backends are unreachable, so a laptop run without cluster access still works.
- The `--debug` / `--trace` toggle is scoped to a single run; the CLI is responsible for restoring prior log/trace levels on exit, including on Ctrl-C and crash paths, so an aborted debug run does not leave the cluster verbose. The toggle's blast radius is the components exercised by the run, not the whole platform — the CLI passes scoping hints through to the dynamic toggle service from ADR 0035.
- Periodic platform-health checks (the use case `HealthCheck` / `ScheduledTest` XRDs would address) run via scheduled CI jobs invoking the same CLI in v1.0; if that approach falls short, CRD-shaped scheduled execution is the named future-enhancement to revisit (backlog 2.14, 3.7).

## References

- architecture-overview.md § 13
- architecture-backlog.md § 2.12, 2.13, 2.14, 3.4, 3.7, 3.8, 4
- ADR 0009 (OpenSearch advisory index role)
- ADR 0010 (GitHub Actions only)
- ADR 0015 (Tempo + Langfuse / OTel Tempo+Mimir+Loki path)
- ADR 0034 (audit pipeline — test runs emit audit through the same adapter)
- ADR 0035 (dynamic log-level and trace-granularity toggle)

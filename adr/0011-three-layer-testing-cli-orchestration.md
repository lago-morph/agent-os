# ADR 0011: Three-layer testing with CLI orchestration

## Status

Accepted

## Context

The platform needs automated testing that covers Kubernetes-level behavior (CRDs, operators, event flows), end-to-end UI and HTTP API flows, and unit-level Python code (SDK, callbacks, kopf operators). It also needs a way to invoke these consistently from a developer laptop, from CI, and on a schedule, without coupling the test harness to a single CI vendor or to a Kubernetes-resident control loop.

Constraints and inputs:

- CI is GitHub Actions only for v1.0 (ADR 0010); the CI job must call the same harness a developer runs locally.
- Tests are owned by each component; they ship alongside the code, not as a separate workstream.
- Stress behavior matters at low concurrency — gross race conditions and resource exhaustion — but is not a benchmarking exercise.
- Trend analysis and per-run inspection should ride existing observability (Mimir, Grafana) before any dedicated test reporting tool is introduced.

## Decision

Adopt three layers of automated tests, coordinated by an `agent-platform test` CLI:

1. **Chainsaw** — declarative end-to-end tests at the Kubernetes level (apply CRDs, assert pods, CloudEvents, Langfuse traces).
2. **Playwright** — UI flows and HTTP API flows (LibreChat journeys, LiteLLM POSTs with expected OPA decisions and audit events).
3. **PyTest** — where it is the natural fit (Python SDK unit tests, custom callback tests, kopf operator unit tests).

Coordination is via the **agent-platform test CLI**. The CLI reads a manifest declaring "what runs where," invokes the appropriate runners, and aggregates results. The same command runs from a developer laptop, from a GitHub Actions job (per ADR 0010), and on schedule.

Reporting:

- Test result metrics emit to Mimir; **Grafana dashboards from test metrics** show pass/fail trends, flake detection, and per-layer health.
- Per-run detail goes to CI output (PR comments, commit statuses).
- No Allure or ReportPortal initially.
- **promptfoo** runs adversarial evaluation and red-teaming as a separate concern (consumed in CI), not as a fourth test layer; eval suites live with the agent code they evaluate.

Stress probes are implemented as Chainsaw or Playwright cases running N concurrent agent invocations and watching existing observability for saturation. No locust/k6, no custom load harness.

Test CRDs are explicitly rejected for v1.0; the CLI is the orchestration surface.

## Consequences

- One CLI invocation reproduces a CI failure on a laptop; CI vendor is replaceable because the CLI carries the orchestration logic.
- Each component's deliverable list explicitly includes Chainsaw + Playwright + PyTest where applicable, keeping tests close to the code under test.
- Trend dashboards reuse the platform's own observability stack; no new tool to operate.
- Without test CRDs, scheduled or composed test runs lean on GitHub Actions schedules and CLI manifests; complex orchestration (fan-out, dependencies between suites) is harder to express declaratively.
- Stress probes catch gross races and saturation but will not produce throughput or latency benchmarks; performance regressions visible only at higher load will go undetected until a real harness is added.
- promptfoo results live alongside test results but are interpreted separately (eval signal, not pass/fail correctness).

## Alternatives considered

- **Allure or ReportPortal for test reporting** (backlog §2.12). Rejected for MVP — CI output plus Grafana dashboards from test metrics covers the felt need. Trigger to revisit (§3.4): cross-run trend analysis or per-run inspection beyond CI/Grafana.
- **Custom load testing harness** (backlog §2.13). Rejected — Chainsaw / Playwright concurrent invocation finds gross race conditions at low concurrency, which is the actual goal. Trigger to revisit (§3.8): a real production race condition that should have been caught by stress probes.
- **Declarative test CRDs** (backlog §2.14). Rejected — CLI orchestration is simpler, runs identically in three contexts (laptop, CI, scheduled), and avoids building yet another controller. Trigger to revisit (§3.7): scheduled execution or complex composition where the CLI-driven model fights us.

## Related

- Architecture overview §13 (Testing framework), §11 (Grafana dashboards), §14.2 component B14 (test framework).
- Architecture backlog §2.12, §2.13, §2.14, §3.4, §3.7, §3.8.
- ADR 0010 — CI is GitHub Actions only and calls the agent-platform test CLI.

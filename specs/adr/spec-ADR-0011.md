# SPEC ADR-0011 — Three-layer testing with CLI orchestration [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [B14] · downstream: [B9, B15, D3, A11, A13, A18] · adrs: [0011] · views: [6.5]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0011 is a settled decision: the platform tests across three layers — **Chainsaw** (declarative Kubernetes-level e2e), **Playwright** (UI + HTTP API flows), and **PyTest** (Python unit + operator) — and the **`agent-platform` test CLI** orchestrates all three from laptop, CI, and schedule identically. This SPEC states what honoring that decision obliges of the CLI, of each component's deliverable list, and of the observability/audit backends, plus acceptance criteria proving the decision is honored. It does not re-argue the choice of runners or the rejection of test CRDs / custom load harnesses.

The problem the decision solves: tests must exist from day one across heterogeneous surfaces with one uniform invocation path, result/diagnostic flow split by layer, graceful degradation when observability backends are unreachable, and a single-run debug toggle that always restores baseline verbosity.

## 2. Scope

### 2.1 In scope
- Obligations the three-layer + CLI-orchestration decision imposes on the `agent-platform` test CLI (component B14 owns implementation).
- The cross-component obligation that every component enumerates Chainsaw/Playwright/PyTest tests where each applies.
- Result-and-diagnostic routing rules: unit = CI-output-only; non-unit = OpenSearch advisory index + OTel along Tempo/Mimir/Loki; all runs emit audit.
- The `--debug` / `--trace` single-run toggle obligations (scoping, restore-on-exit/crash).
- Stress-probe obligation (low-volume concurrent invocation on Chainsaw/Playwright, no separate load tool).

### 2.2 Out of scope (and where it lives instead)
- CLI implementation, stress-probe harness build, dashboard provisioning — component **B14** SPEC.
- Test-framework dashboards / OpenSearch result views — component **D3** SPEC.
- The test manifest schema for `agent-platform test` — design-time deliverable (architecture-backlog §4); not fixed here.
- Dynamic toggle mechanism itself — ADR 0035 / its enforcing component.
- OpenSearch advisory-index role — ADR 0009; audit pipeline — ADR 0034; trace correlation — ADR 0015.

## 3. Context & Dependencies

Upstream consumed: **B14** (`agent-platform` test framework) implements the obligations; **B9** (`agent-platform` CLI) hosts the `test` subcommand surface.
Downstream consumers: **D3** (dashboards over test metrics + OpenSearch result views), **B15** (CI/CD reference pipeline invokes the CLI), **A11** OpenSearch (advisory result index), **A13** Tempo+Mimir (OTel run metrics/logs), **A18** audit endpoint/adapter (test-run audit emission).

ADR decisions honored:
- **ADR 0011** (this) — three layers + CLI orchestration; split result flow; single-run toggle.
- **ADR 0009** — OpenSearch is an advisory index, never system of record, for non-unit results.
- **ADR 0010** — GitHub-Actions-only CI calls the same CLI (no CI-specific runner logic).
- **ADR 0015** — non-unit runs publish via OTel along the Tempo/Mimir/Loki path; emitted traces share `trace_id` correlation.
- **ADR 0034** — test runs emit audit through the standard audit adapter to Postgres + S3 (Postgres-only on kind).
- **ADR 0035** — `--debug`/`--trace` activates the dynamic toggle for one run, scoped to exercised components, restored on exit.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — ADR 0011 explicitly **defers** test CRDs (`HealthCheck` / `ScheduledTest`) to future-enhancements; v1.0 has no test-execution CRD. The `LogLevel` CRD (ADR 0035) is the toggle surface the CLI drives but is owned by ADR 0035.

### 4.2 APIs / SDK surfaces
- `agent-platform test` subcommand of the `agent-platform` CLI (owner B9/B14); subcommand surface is the versioned API per ADR 0030. Method signatures beyond the subcommand surface: not specified in source — `[PROPOSED — not in source]` if detailed.
- `--debug` / `--trace` flags scoped to a single run.

### 4.3 CloudEvents emitted / consumed
- Test-run audit flows under `platform.audit.*` (via the audit adapter, §5 of interface-contract). No new top-level namespace is introduced (closed set per ADR 0031).
- `[PROPOSED — not in source]` any per-event-type test result CloudEvent name — per-event-type names are deferred to B12's registry.

### 4.4 Data schemas / connection-secret contracts
- Non-unit per-run results → OpenSearch advisory index (derived, rebuildable per ADR 0014). Index schema is a B14/D3 design-time deliverable; `[PROPOSED — not in source]`.
- Test-run audit records ride the audit adapter's `audit_events` Postgres table (ADR 0034). No new connection secret introduced.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: ADR names upstream runners **Chainsaw**, **Playwright**, **PyTest** configured/wrapped by the custom **B14** framework; no fork. Rejected alternatives — test CRDs, custom load harness, Allure/ReportPortal — are not built in v1.0.)

## 6. Functional Requirements
- REQ-ADR-0011-01: The platform MUST provide exactly three test layers — Chainsaw, Playwright, PyTest — and no fourth runner in v1.0; no locust/k6/custom load tool.
- REQ-ADR-0011-02: A single CLI (`agent-platform test`) MUST orchestrate all three layers from a manifest, invoking the correct runner per declared entry, identically on laptop, CI, and schedule.
- REQ-ADR-0011-03: Every component's deliverable list MUST enumerate Chainsaw/Playwright/PyTest tests at whichever layers apply (distributed ownership; no central QA component).
- REQ-ADR-0011-04: PyTest **unit-scope** runs MUST stay CI-output-only (PR comments + commit statuses) and MUST NOT stream to OpenSearch or OTel.
- REQ-ADR-0011-05: All **non-unit** runs (Chainsaw, Playwright, PyTest integration/operator) MUST stream per-run results to OpenSearch as an advisory index AND publish run metrics+logs via OTel along the Tempo/Mimir/Loki path.
- REQ-ADR-0011-06: All test runs MUST emit audit events through the standard audit adapter (ADR 0034), capturing who ran what test against which environment.
- REQ-ADR-0011-07: The CLI MUST degrade gracefully — warn, continue, and exit non-zero ONLY on actual test failure — when OpenSearch or the OTel pipeline is unreachable (a laptop run without cluster access still works).
- REQ-ADR-0011-08: `--debug`/`--trace` MUST activate the ADR 0035 dynamic toggle for the duration of one run, scoped to the components the run exercises (not platform-wide), and MUST restore prior log/trace levels on exit, Ctrl-C, and crash.
- REQ-ADR-0011-09: Stress probes MUST be implemented as low-volume concurrent invocations of Chainsaw/Playwright watching existing observability for saturation, not as a separate benchmarking toolchain.

## 7. Non-Functional Requirements
- Observability (§6.5): non-unit test diagnostics MUST use the same dashboards/query surfaces/retention as the rest of the platform (no parallel pipeline).
- Security/multi-tenancy: test-run audit MUST identify actor + target environment; runs against tenant namespaces respect the same RBAC/OPA bounds (ADR 0018) as any actor.
- Versioning (ADR 0030): the `agent-platform` CLI subcommand surface is semantically versioned; manifest schema, once specified, versions with it.
- Reversibility: adopting Jenkins/GitLab later MUST remain a thin shim over the same CLI (ADR 0010).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The standard §14.1 set applies to the enforcing components (B14, D3), not to this decision record.

## 9. Acceptance Criteria
- AC-ADR-0011-01: Honored when the CLI runs all three layers from one manifest and an identical invocation succeeds on laptop, CI, and a scheduled job. (REQ-01/02)
- AC-ADR-0011-02: Honored when a sampled set of component specs each list their applicable Chainsaw/Playwright/PyTest tests. (REQ-03)
- AC-ADR-0011-03: Honored when a unit run produces CI output only and emits zero OpenSearch/OTel records. (REQ-04)
- AC-ADR-0011-04: Honored when a non-unit run produces one OpenSearch advisory result document AND OTel run metrics/logs on the Tempo/Mimir/Loki path. (REQ-05)
- AC-ADR-0011-05: Honored when every test run (unit and non-unit) produces an audit record naming actor + environment. (REQ-06)
- AC-ADR-0011-06: Honored when, with OpenSearch and OTel unreachable, a passing run exits 0 with warnings and a failing run exits non-zero. (REQ-07)
- AC-ADR-0011-07: Honored when a `--debug` run raises verbosity only on exercised components and a forced Ctrl-C/crash leaves baseline log/trace levels restored. (REQ-08)
- AC-ADR-0011-08: Honored when a stress probe runs as concurrent Chainsaw/Playwright invocations with no separate load tool present. (REQ-09)

## 10. Risks & Open Questions
- OQ-1 (med): Test manifest schema is a deferred design-time deliverable (backlog §4); ACs that depend on manifest shape are partial until it lands. `[PROPOSED]`
- R-1 (low): Toggle restore on crash relies on ADR 0035 scoping hints; if the toggle service is down, the CLI cannot guarantee restore — blast radius is exercised components only.
- R-2 (low): OpenSearch/OTel as soft dependencies risk silent loss of diagnostics; mitigated by mandatory warn + non-zero only on real failure.

## 11. References
- ADR 0011 (`adr/0011-three-layer-testing-cli-orchestration.md`) — the decision enforced here.
- architecture-overview.md §13 (testing framework), §6.5 (observability).
- Enforcing/related components: B14 (test framework), B9 (CLI), D3 (dashboards), A11 (OpenSearch), A13 (Tempo+Mimir), A18 (audit).
- ADR 0009, 0010, 0014, 0015, 0034, 0035 (cited constraints).

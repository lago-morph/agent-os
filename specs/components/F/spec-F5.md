# SPEC F5 — Scale evaluation

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [A1, A6, A4, A18, B14] · downstream: [] · adrs: [0035, 0004, 0003, 0034, 0011, 0033] · views: [6.1, 6.2, 6.7, 6.5]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

F5 is the end-of-v1.0 **scale evaluation**: measure the *actual* throughput and latency of the platform's load-bearing paths under realistic load and identify any gaps versus expected v1.0 scale (§14.6). The architecture deploys components in single-instance defaults with one carve-out — ADR 0035 raised LiteLLM to `replicas >= 2` for SSE-safe restarts (future-enhancements.md §1) — and v1.0 does **not** commit SLOs or error budgets (future-enhancements.md §3). F5 therefore produces *measured baselines*, not SLO conformance: gateway throughput, sandbox cold-start time, and broker backlog under load.

F5 builds no new infrastructure and defines no SLO. Per future-enhancements.md §14 (backlog 3.8) and ADR 0011, v1.0 stress probing uses Chainsaw/Playwright concurrent invocation via the B14 test framework rather than a custom load-testing harness. F5's deliverable is a measurement report plus a gap list (where measured scale falls short of expectation) routed to owning components and F6.

## 2. Scope

### 2.1 In scope
- **Gateway throughput**: measured request/response throughput and latency through LiteLLM (A1) at the ADR-0035 two-replica baseline, including the OPA-callback and audit-emission overhead on the hot path.
- **Sandbox cold-start**: measured cold-start time for agent-sandbox (A6) pods (gVisor/Kata), and the effect of `SandboxTemplate` `warmPoolSize` / `hibernationEnabled` on it.
- **Broker backlog under load**: measured NATS JetStream (A4) backlog/lag behavior under sustained CloudEvent load, including consumer keep-up and recovery after a burst.
- **Audit-path scale**: in-flight `audit_events` growth and the ~5-min batch CronJob keep-up under load (ADR 0034), feeding F1's ISM/retention sizing.
- **Gap analysis**: a report comparing measured numbers to expected v1.0 scale, with a prioritized gap list (no SLO commitment).
- **Reusable probes**: the stress probes as repeatable B14 test cases (Chainsaw/Playwright concurrent invocation).

### 2.2 Out of scope (and where it lives instead)
- **SLO/SLI definitions, error budgets, burn-rate alerting, reliability dashboards** → future-enhancements.md §3 (explicitly deferred).
- **HA topologies / clustering beyond the ADR-0035 two-replica baseline** → future-enhancements.md §1.
- **A custom load-testing harness** → future-enhancements.md §14 (backlog 3.8) — v1.0 uses Chainsaw/Playwright concurrency (ADR 0011).
- **The components under test themselves** → A1 (gateway), A6 (sandbox), A4 (broker), A18 (audit) — F5 measures, does not build.
- **The test framework** → **B14**; F5 authors test cases on it.
- **Test-framework dashboards** → **D3**; F5 may emit metrics D3 visualizes.
- **Cross-region / multi-cluster scale** → future-enhancements.md §10, §14; ADR 0026 single-cluster.

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM gateway)** — the throughput path under test, including its OPA callbacks (B2) and audit emission on the hot path; measured at `replicas >= 2` (ADR 0035).
- **A6 (agent-sandbox + Envoy)** — the cold-start path; `SandboxTemplate` `warmPoolSize`/`hibernationEnabled` are the knobs F5 measures.
- **A4 (Knative Eventing + NATS JetStream)** — the broker whose backlog/lag F5 measures under load.
- **A18 (audit endpoint + adapter)** — the audit path whose batch keep-up F5 measures (feeds F1).
- **B14 (agent-platform test framework)** — the stress-probe harness (Chainsaw/Playwright concurrent invocation, ADR 0011) F5 authors against.

**Downstream consumers:** none directly; F1 (retention sizing), F2 (representative reindex volume), F4 (DoS-class baseline), and F6 (runbook capacity notes) consume F5's numbers.

**ADRs honored:**
- **ADR 0035** — LiteLLM is at `replicas >= 2` with SSE-safe rolling-restart tuning; F5 measures at that baseline, not single-instance, and notes restart-drain behavior under load.
- **ADR 0004** — NATS JetStream is the broker backend; backlog/lag semantics are JetStream's; F5 measures stream/consumer behavior (replication factor is single in v1.0 per future §1).
- **ADR 0003** — Envoy egress on the hot path; F5 includes egress-proxy overhead in agent request latency where applicable.
- **ADR 0034** — audit emission is on the hot path but durable+async; F5 measures its overhead and batch keep-up.
- **ADR 0011** — three-layer testing, CLI-orchestrated; stress probing via Chainsaw/Playwright concurrency (no custom harness).
- **ADR 0033** — AWS (EKS) is the measurement substrate for production-representative numbers (kind numbers are not production-representative).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F5 introduces **no new CRD/XRD**. It varies `SandboxTemplate` fields (`warmPoolSize`, `hibernationEnabled`, `resourceLimits` — Canon §1.3) to measure cold-start, and reads `BudgetPolicy` limits when measuring DoS-class behavior (coordinated with F4). A declarative `ScheduledTest`/`HealthCheck` XRD for *recurring* capacity baselining is **out of scope** (future-enhancements.md §5).

### 4.2 APIs / SDK surfaces
N/A — F5 introduces no API/SDK surface. Probes drive the existing gateway HTTP surface, the `AgentRun`/sandbox lifecycle, and the broker, orchestrated through B9/B14. Measured metrics are emitted via the existing OTel/Mimir path (Canon §3.1 OTel emission). `[PROPOSED — not in source]` — the specific stress-probe subcommand is design-time.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
F5 generates load on the `platform.lifecycle.*` (AgentRun/Sandbox) and `platform.gateway.*` paths and measures `platform.observability.*` threshold-crossing behavior under load. It mints no new event types; whether a "scale baseline exceeded" signal should be emitted is `[PROPOSED — not in source]` (SLO-style alerting is deferred, future §3).

### 4.4 Data schemas / connection-secret contracts
N/A — F5 introduces no data schema or connection-secret change. It reads existing telemetry (Mimir metrics, Tempo traces, Langfuse) to compute throughput/latency/backlog; it observes the `audit_events` table growth without altering its schema.

## 5. OSS-vs-Custom Decision
**Measurement using the existing test framework + observability stack — no new build.** Stress probes are **config/wrap** of B14 (Chainsaw/Playwright concurrent invocation, ADR 0011); measurement reads Mimir/Tempo/Langfuse. No custom load-testing harness (future §14 / backlog 3.8), no fork, no new service. Rationale: future-enhancements.md §14 explicitly states v1.0 stress probes via Chainsaw/Playwright concurrency are sufficient until proven otherwise; F5 produces the numbers that would *prove* otherwise, without pre-building the deferred harness.

## 6. Functional Requirements
- **REQ-F5-01:** F5 MUST measure LiteLLM gateway throughput and latency (incl. OPA-callback + audit-emission overhead) at the ADR-0035 `replicas >= 2` baseline under sustained concurrent load.
- **REQ-F5-02:** F5 MUST measure agent-sandbox cold-start time and quantify the effect of `warmPoolSize`/`hibernationEnabled` on it.
- **REQ-F5-03:** F5 MUST measure NATS JetStream backlog/consumer-lag under sustained CloudEvent load and the recovery profile after a burst.
- **REQ-F5-04:** F5 MUST measure audit in-flight `audit_events` growth and ~5-min batch keep-up under load, and feed the numbers to F1 (ISM/retention sizing) and F2 (reindex volume).
- **REQ-F5-05:** F5 MUST produce a report comparing each measured number to expected v1.0 scale and a prioritized gap list, explicitly **without** asserting any SLO/error budget.
- **REQ-F5-06:** Stress probes MUST be repeatable B14 test cases (Chainsaw/Playwright concurrent invocation, ADR 0011) — no custom load harness.
- **REQ-F5-07:** Measurements MUST be taken on the AWS (EKS) substrate for production-representative numbers; any kind-substrate numbers MUST be labeled non-representative (ADR 0033).
- **REQ-F5-08:** F5 MUST verify behavior under a LiteLLM rolling restart while under load (ADR 0035 SSE-safe drain) — measure error/latency impact, not just steady state.

## 7. Non-Functional Requirements
- **Security:** load probes MUST run with scoped credentials in a controlled cluster and MUST NOT bypass OPA/egress controls (the controls' overhead is part of what is measured, not removed).
- **Multi-tenancy (§6.9):** F5 SHOULD measure under multi-tenant load (multiple namespaces) so per-tenant contention/quota behavior is represented, not just single-tenant throughput.
- **Observability (§6.5):** F5 relies on Tempo/Mimir/Langfuse correlation by `trace_id` (ADR 0015) to attribute latency to components; F5 verifies these signals are sufficient to localize a scale gap.
- **Scale:** the load profile MUST be "realistic" per §14.6 (not synthetic-maximal); F5 records the chosen profile so numbers are interpretable and the gap list is meaningful.
- **Versioning (ADR 0030):** the report MUST record the component versions measured; numbers are valid against those versions only.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A — F5 ships no deployable; it loads existing-deployed components.**
- Per-product docs (10.5) — **applicable** (scale-evaluation report; measured baselines; load profile; gap list).
- Runbook (10.7) — **applicable** (capacity/scale runbook: known limits, backlog-recovery, cold-start tuning). Feeds **F6**.
- Alerts — **N/A — SLO-style threshold alerting is deferred (future §3).** `[PROPOSED]` capacity-baseline alert.
- Grafana dashboard (Crossplane XR) — **N/A — reliability/SLO dashboards deferred (future §3); test-framework dashboards are D3.** F5 may emit metrics D3 renders.
- Headlamp plugin — **N/A — measurement activity, no interactive CRD surface.**
- OPA/Rego integration — **N/A — F5 measures OPA-callback overhead; it adds no policy.**
- Audit emission (ADR 0034) — **applicable** — F5 measures the audit hot-path overhead and batch keep-up; emits no new audit type.
- Knative trigger flow — **applicable (load)** — F5 loads the broker/trigger path to measure backlog; introduces no new flow.
- HolmesGPT toolset — **N/A — F5 produces a report, not an operational toolset.** `[PROPOSED]` capacity-posture read tool.
- 3-layer tests — **applicable** (Chainsaw concurrent AgentRun/Sandbox probes; Playwright concurrent gateway/UI invocation; PyTest for metric extraction/aggregation). This *is* the F5 probe set (ADR 0011).
- Tutorials & how-tos — **applicable** (how-to: run a scale probe; reference: interpreting the baseline report).

## 9. Acceptance Criteria
- **AC-F5-01:** A gateway throughput + latency number (with OPA/audit overhead) is recorded at `replicas >= 2` under sustained load. (→ REQ-F5-01)
- **AC-F5-02:** Cold-start times are recorded for warm-pool-on vs warm-pool-off `SandboxTemplate` configs. (→ REQ-F5-02)
- **AC-F5-03:** JetStream backlog/lag under load and post-burst recovery time are recorded. (→ REQ-F5-03)
- **AC-F5-04:** Audit table growth + batch keep-up under load are recorded and handed to F1/F2. (→ REQ-F5-04)
- **AC-F5-05:** A report compares measured to expected scale and lists prioritized gaps, asserting no SLO. (→ REQ-F5-05)
- **AC-F5-06:** Every probe re-runs from the B14 harness and reproduces comparable numbers. (→ REQ-F5-06, REQ-F5-07 labeled by substrate)
- **AC-F5-07:** Gateway error/latency impact during an in-load rolling restart is recorded and within the ADR-0035 SSE-safe expectation. (→ REQ-F5-08)

## 10. Risks & Open Questions
- **R1 (high):** Measured scale may fall materially short of expectation (e.g., OPA-callback overhead dominates gateway latency), forcing a v1.0 scope or HA decision. That is F5's purpose; late discovery is high blast radius. Mitigation: run F5 on AWS early; route gaps to owning components + future §1/§3. Blast radius: high.
- **R2 (med):** "Expected v1.0 scale" is not numerically pinned in source — the gap analysis baseline is itself `[PROPOSED]`. Mitigation: F5 records the assumed expectation explicitly and gets it confirmed before declaring a gap. Blast radius: med.
- **R3 (med):** Chainsaw/Playwright concurrency may not generate enough load to find the real ceiling (the deferred custom harness exists for a reason). Mitigation: note achievable load ceiling; if the probe saturates before the component does, flag the harness limitation (future §14). Blast radius: med.
- **R4 (low):** kind numbers could be mistaken for production numbers. Mitigation: label every number by substrate (REQ-F5-07). Blast radius: low.
- **OQ1:** What is the authoritative "expected v1.0 scale" to compare against? `[PROPOSED]` derive from §14.6 intent + component owners; confirm before gap sign-off.
- **OQ2:** Should F5 feed measured numbers forward as the *seed* for future-§3 SLOs? `[PROPOSED]` yes — record as candidate SLIs, but commit no SLO in v1.0.

## 11. References
- architecture-overview.md §14.6 line 1761 (F5 scope: gateway throughput, sandbox cold-start, broker backlog; identify gaps vs expected scale), §6.5 (observability), §6.7 (eventing/broker), §6.1/§6.2 (gateway/agent-runtime), §11 (alert rules).
- future-enhancements.md §1 (single-instance default + ADR-0035 two-replica carve-out; HA deferred), §3 (SLO/error-budget/reliability framework deferred), §14 (custom load-testing harness deferred — backlog 3.8; Chainsaw/Playwright concurrency sufficient for v1.0).
- Canon interface-contract §1.3 (`SandboxTemplate` `warmPoolSize`/`hibernationEnabled`), §1.4 (`BudgetPolicy`), §2 (`platform.lifecycle.*`/`platform.gateway.*`/`platform.observability.*`), §3.1 (OTel emission).
- ADR 0035 (LiteLLM `replicas>=2`, SSE-safe rolling restart), ADR 0004 (NATS JetStream broker), ADR 0003 (Envoy egress), ADR 0034 (audit hot-path/batch), ADR 0011 (three-layer testing, CLI-orchestrated), ADR 0015 (Tempo/Langfuse trace_id correlation), ADR 0033 (AWS/EKS target).
- Related pieces: A1, A6, A4, A18, B2, B14, D3, F1, F2, F4, F6.

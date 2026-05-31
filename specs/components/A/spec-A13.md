# SPEC A13 — Tempo + Mimir

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [] · adrs: [0015, 0035, 0011, 0030, 0034, 0031] · views: [6.5]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A13 installs and operates **Tempo** (distributed-tracing backend) and **Mimir** (long-term metrics
store) — the platform's general-purpose observability backends (architecture-overview.md §6.5;
ADR 0015). Per ADR 0015 the platform runs **Tempo for general OTel traces** and Langfuse (A2) for
LLM-grade traces, correlated by a shared `trace_id`; a developer in Grafana/Tempo can deep-link into
Langfuse and back. **Mimir holds long-term metrics; Prometheus stays the scrape layer** (§6.5).

A13 lands early (W0 foundation; §14.7 "in early so other components can … emit traces from the
start") so every other component can send OTel spans and publish metrics from day one. Tempo + Mimir
sit downstream of the OTel collector and Prometheus respectively, feeding Grafana. They are two of
the four backends HolmesGPT's observability toolset consumes (Tempo + Mimir + Loki + Langfuse,
ADR 0015), and they carry non-unit test run metrics/logs/traces via OTel per ADR 0011.

## 2. Scope

### 2.1 In scope
- Tempo Helm install + values (general OTel trace storage, retention, query).
- Mimir Helm install + values (long-term metrics store fed from Prometheus).
- Wiring Tempo behind the OTel collector and Mimir behind Prometheus (collector + Prometheus are
  k8-platform baseline; A13 provides the backend targets).
- Object/block storage backing for Tempo and Mimir via the `ObjectStore` substrate XRD (B4) where
  required `[PROPOSED — not in source: exact storage backing per backend is design-time]`.
- The **Grafana→Langfuse deep-link** correlation contract on the Tempo side (ADR 0015): Tempo spans
  carry the `trace_id` that joins to Langfuse; A13 supplies the link config (the dashboards
  themselves are Workstream D / §11).
- `LogLevel`-gated trace granularity in Tempo (ADR 0035) — span ingestion conditional on live
  config; off-mode has zero cost.
- Carrying non-unit test run metrics/logs/traces via the same OTel/Tempo/Mimir path (ADR 0011).
- Standard §14.1 Workstream-A deliverable set (see §8).

### 2.2 Out of scope (and where it lives instead)
- LLM-grade traces — **Langfuse (A2)**; non-LLM spans never enter Langfuse (ADR 0015).
- Logs (Loki) and baseline Prometheus/Grafana — **k8-platform baseline** (§6.5 marks Loki,
  Prometheus, Grafana "baseline"). A13 is Tempo + Mimir only.
- The OTel collector and Prometheus scrape layer — **baseline**; A13 consumes them as inputs.
- Emitting `trace_id` into spans — **agent SDK (B7) and LiteLLM (A1)** (load-bearing per ADR 0015);
  A13 only stores/correlates.
- Audit records — **audit endpoint + adapter (A18)**; Tempo/Mimir are not audit sinks.
- Object-store provisioning Composition — **Crossplane (B4)** (`ObjectStore`).
- SLI/SLO definitions, error budgets, alert→remediation loops — **deferred to future enhancements**
  (§6.5 scope note); A13 provides the collection/storage path, not the policy on top.
- Per-component dashboards' content — owned by each component / Workstream D; A13 ships its own.

## 3. Context & Dependencies

**Upstream consumed:** none (W0 foundation). Binds to the baseline OTel collector, Prometheus, and
Grafana, and to B4's `ObjectStore` for backing storage (foundation band).

**Downstream consumers:** none declared in piece-index. Functionally every component that emits OTel
traces/metrics and HolmesGPT (A14, dotted toolset edge `A13 -.-> A14`) consume A13's backends — a
continuous runtime relationship, not a build edge.

**ADR decisions honored:**
- **ADR 0015** — Tempo for general OTel traces, correlated with Langfuse by `trace_id`; Grafana
  deep-links from Tempo into Langfuse; HolmesGPT consumes both; trace granularity gated by ADR 0035;
  v1.0 does **not** run always-on sample-and-drop.
- **ADR 0035** — Tempo trace granularity is `LogLevel`-gated; span creation conditional on live
  config so off-mode has no perf cost and Tempo cost scales with verbosity, not traffic.
- **ADR 0011** — non-unit test runs publish metrics/logs via OTel along the Tempo/Mimir/Loki path;
  test diagnostics share the platform's observability surfaces.
- **ADR 0034** — A13 is NOT an audit sink.
- **ADR 0031** — A13 events fall under `platform.observability.*`; per-event names deferred to B12.
- **ADR 0030** — A13 ships no platform CRD; versioning applies to any glue HTTP surface.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- A13 defines **no platform CRD**. It **consumes**:
  - `ObjectStore` (XRD, B4) — `bucketName`, `lifecycle`, `connectionSecretRef`, `substrateClass`
    — for Tempo/Mimir block/object storage (capability-parity caveat: kind path may lack archive
    lifecycle).
  - `LogLevel` (ADR 0035) — `componentSelector`, `level`, `traceGranularity`, `scope`, `expiresAt`
    — to gate Tempo trace granularity.
  - `GrafanaDashboard` (XR, B4) — `dashboardJson`, `folder`, `visibility` — A13's own dashboards.

### 4.2 APIs / SDK surfaces
- **Tempo ingest/query API** and **Mimir remote-write/query API** (upstream product APIs) — Tempo
  receives spans from the OTel collector; Mimir receives metrics from Prometheus remote-write and
  serves queries to Grafana. No new platform HTTP API beyond the products' own.
- **trace_id correlation contract:** Tempo stores the `trace_id` emitted by SDK/gateway unchanged so
  Grafana can deep-link to the matching Langfuse trace (ADR 0015).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.observability.*` for threshold crossings / backend health
  `[PROPOSED — not in source: specific event names deferred to B12]`.
- **Consumed:** none required for core function.

### 4.4 Data schemas / connection-secret contracts
- Tempo/Mimir storage consumes the **`ObjectStore` uniform connection secret**
  (host/port/user/password/dbname or per-primitive equivalent; ADR 0044 canonical shape).
- Trace/metric schemas are the **upstream product schemas**; the only platform-owned invariant is
  the `trace_id` match with Langfuse (ADR 0015) and OTel semantic-convention compliance for spans.

## 5. OSS-vs-Custom Decision
- **Named projects:** Tempo (glossary: "Distributed tracing backend (A13; ADR 0015)") and Mimir
  (glossary: "Metrics backend (A13)").
- **Mode:** install + configure + operate (Helm); unmodified upstream, no fork.
- **Version:** pin tested Tempo and Mimir versions `[PROPOSED — not in source]`.
- **ADR linkage:** ADR 0015 (Tempo chosen as the general OTel backend alongside Langfuse; Mimir as
  long-term metrics).
- **Rationale:** a generic OTel/metrics backend is the right home for non-LLM spans and long-term
  metrics (ADR 0015 rejects Langfuse-only); Mimir provides metrics LTS while Prometheus stays the
  scrape layer (§6.5).

## 6. Functional Requirements
- **REQ-A13-01:** A13 SHALL install Tempo and Mimir via Helm with platform values as ArgoCD-synced
  releases.
- **REQ-A13-02:** A13 SHALL configure Tempo to receive general OTel spans from the baseline OTel
  collector and Mimir to receive metrics via Prometheus remote-write.
- **REQ-A13-03:** A13 SHALL back Tempo and Mimir storage with the `ObjectStore` XRD where required,
  consuming the uniform connection secret.
- **REQ-A13-04:** A13 SHALL store the emitter-supplied `trace_id` in Tempo unchanged, preserving
  correlation with Langfuse (ADR 0015) — A13 SHALL NOT rewrite `trace_id`.
- **REQ-A13-05:** A13 SHALL configure Grafana→Langfuse deep-link targets from Tempo spans so a
  developer can open the correlated Langfuse trace without manual ID copy-paste.
- **REQ-A13-06:** A13 SHALL honor a `LogLevel` CR gating Tempo trace granularity: at baseline no
  always-on sample-and-drop runs and off-mode has zero performance cost (ADR 0035).
- **REQ-A13-07:** A13 SHALL accept non-unit test run metrics/logs/traces over the same OTel/Tempo/
  Mimir path (ADR 0011) without a parallel test-only pipeline.
- **REQ-A13-08:** A13 SHALL deliver per-component Grafana dashboards (`GrafanaDashboard` XR) for
  Tempo and Mimir and alert rules for backend failure conditions (ingest down, store unreachable,
  query failures).
- **REQ-A13-09:** A13 SHALL contribute HolmesGPT toolset entries exposing Tempo trace queries and
  Mimir metric queries (dotted toolset edge `A13 -.-> A14`).
- **REQ-A13-10:** A13 SHALL emit `platform.observability.*` CloudEvents on backend health/threshold
  conditions and audit operator actions via the adapter (ADR 0034) — A13 is not an audit *sink*.

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** trace/metric query access SSO'd and claim-scoped via Grafana;
  per-tenant trace/metric partitioning `[PROPOSED — not in source: design-time]`; RBAC floor + OPA
  restriction (ADR 0018).
- **Observability (§6.5):** Tempo/Mimir self-emit metrics/traces through the standard path; they are
  themselves observable (meta-observability).
- **Scale:** Mimir provides long-term metrics retention; Tempo cost scales with verbosity not
  traffic (ADR 0035). Concrete retention/throughput targets deferred to F1/F5.
- **Versioning (ADR 0030):** pin tested upstream versions; URL-path versioning on any glue surface.

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable** (Tempo + Mimir).
- Per-product docs (10.5) — **applicable**.
- Runbook (10.7) + backup/restore — **applicable** (object-store-backed; backup/restore covers the
  `ObjectStore` data; kind path may lack archive lifecycle per the parity caveat).
- Alerts — **applicable** (ingest/store/query failures for both backends).
- Grafana dashboard (Crossplane XR) — **applicable** (Tempo + Mimir health/usage dashboards).
- Headlamp plugin — **N/A as A13-owned editor** — observability is consumed via Grafana; no A13 row
  in ADR 0039's editor set. Read-only inspection via the generic framework.
- OPA/Rego integration — **applicable** (admission on A13 workloads; query-access gating via claims).
- Audit emission (ADR 0034) — **applicable** (operator actions; A13 is not an audit *sink*).
- Knative trigger flow — **applicable** (emits `platform.observability.*` on backend health).
- HolmesGPT toolset — **applicable** (Tempo trace-query + Mimir metric-query tools).
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable**.
- Tutorials & how-tos — **applicable** (observability how-to: query a trace in Tempo, a metric in
  Mimir/Grafana, deep-link to Langfuse).

## 9. Acceptance Criteria
- **AC-A13-01 (REQ-A13-01):** ArgoCD sync brings Tempo and Mimir to Ready. *(Chainsaw)*
- **AC-A13-02 (REQ-A13-02):** A span sent to the OTel collector lands in Tempo; a metric scraped by
  Prometheus is queryable in Mimir. *(PyTest)*
- **AC-A13-03 (REQ-A13-03):** Tempo/Mimir connect using only the `ObjectStore` connection-secret
  fields; data survives a pod restart. *(PyTest)*
- **AC-A13-04 (REQ-A13-04):** A span emitted with `trace_id=X` is stored in Tempo under `trace_id=X`
  and is joinable to a Langfuse trace with the same `trace_id`. *(PyTest)*
- **AC-A13-05 (REQ-A13-05):** From a Tempo span in Grafana, the deep link opens the matching
  Langfuse trace. *(Playwright)*
- **AC-A13-06 (REQ-A13-06):** With `LogLevel` at baseline no spans are created for a target
  component (zero cost); raising granularity produces spans; reverting stops them. *(Chainsaw +
  PyTest)*
- **AC-A13-07 (REQ-A13-07):** A non-unit test run publishes metrics/traces that appear in Mimir/Tempo
  via the standard path (no separate pipeline). *(PyTest)*
- **AC-A13-08 (REQ-A13-08):** The Tempo and Mimir `GrafanaDashboard` XRs render; a simulated
  ingest-down fires the alert. *(Chainsaw + PyTest)*
- **AC-A13-09 (REQ-A13-09):** HolmesGPT can run a Tempo trace query and a Mimir metric query via the
  toolset. *(PyTest)*
- **AC-A13-10 (REQ-A13-10):** A backend-health threshold crossing emits a `platform.observability.*`
  event; an operator action emits an audit record via the adapter. *(PyTest)*

## 10. Risks & Open Questions
- **R1 (med):** `trace_id` correlation depends on emitters (A1/B7) — an emitter that omits `trace_id`
  breaks the Tempo↔Langfuse join (ADR 0015 consequence); out of A13's control, covered by SDK tests.
- **R2 (low):** Per-tenant partitioning of traces/metrics is `[PROPOSED — not in source]`; blast
  radius limited to query scoping (floor is RBAC/OPA).
- **R3 (low):** Storage backing per backend (`ObjectStore` vs local) is `[PROPOSED]`; kind parity
  caveat (no archive lifecycle) may reduce retention on kind by design (ADR 0044).
- **R4 (low):** Retention durations deferred to F1; A13 ships the path, not the policy. No blast
  radius for v1.0 install.

## 11. References
- architecture-overview.md §6.5 (observability architecture, lines ~370–434 — Tempo/Mimir/Prometheus
  roles, deep links, ADR 0035 toggle, ADR 0011 test publication, scope note); §14.7 (foundation
  notes — A13 in early); §14.1 (A13 row, line ~1679).
- ADR 0015 (Tempo + Langfuse correlated by `trace_id`); ADR 0035 (log/trace toggle); ADR 0011
  (three-layer testing — OTel publication); ADR 0034 (audit — A13 not a sink); ADR 0031
  (CloudEvents); ADR 0030 (versioning); ADR 0044 (`ObjectStore` connection-secret shape).
- View V6-05 (Observability architecture).
- Related pieces: A2 (Langfuse correlation), A1/B7 (`trace_id` emitters), A14 (HolmesGPT toolset
  consumer), B4 (`ObjectStore`/`GrafanaDashboard`), A18 (audit — distinct path).

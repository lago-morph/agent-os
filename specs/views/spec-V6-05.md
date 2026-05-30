# SPEC V6-05 — Observability architecture `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0011, 0015, 0034, 0035] · views: [6.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the observability slice: the slice in which every platform component emits OTel telemetry, LLM-grade traces, and audit through one shared set of pipelines. **Tempo** and **Langfuse** correlate by `trace_id`; **Mimir** holds long-term metrics; audit flows through the **audit adapter** library to a single **audit endpoint** with Postgres + S3 as system of record and OpenSearch as advisory fanout. It is not a buildable component; it is realized by Langfuse (A2), Tempo + Mimir (A13), and the operator/developer dashboards (D1, D2).

The problem it solves: an agent platform must be observable across two correlated planes — conventional telemetry (traces/metrics/logs) and LLM-grade observability (prompts, costs, evals) — plus a durable audit plane that never silently drops records. By fixing the `trace_id` correlation invariant, the adapter/endpoint audit invariant, and the dynamic log-level toggle contract, this view ensures a developer can pivot Grafana ↔ Langfuse on one id, audit ingestion always succeeds even when OpenSearch is down, and verbosity can be raised per-component/tenant/event-class without a parallel always-on tracing pipeline.

## 2. Scope
### 2.1 In scope
- The `trace_id` correlation invariant: every span from the agent SDK and gateway carries the same `trace_id`, enabling Grafana ↔ Langfuse deep-links (ADR 0015).
- The OTel collection path: agent pods, LiteLLM, Envoy, ARK, UIs → OTel collector → Tempo (traces) / Loki (logs) / Mimir (metrics, via Prometheus scrape).
- The LLM-grade plane: Langfuse receives prompts, costs, evals, and red-team (promptfoo) results.
- The audit-emission invariant (ADR 0034): no component writes audit directly; all link the audit adapter and post to the audit endpoint; Postgres + S3 system of record; OpenSearch advisory fanout.
- The dynamic log-level / trace-granularity toggle contract (ADR 0035): in-process toggle (default) vs rolling-restart toggle (non-reloadable services); zero off-mode cost.
- Test-result + OTel publication (ADR 0011): non-unit test results stream to OpenSearch advisory; metrics/logs publish via OTel through the shared collector.
- Per-component dashboards and failure-condition alert rules (the committed scope).

### 2.2 Out of scope (and where it lives instead)
- Audit pipeline store topology in depth (XRD, batch CronJob) — view V6-03 / component A18 / ADR 0034.
- The audit adapter library + audit endpoint implementation — component A18 (consumed, not defined here).
- Dashboards as namespaced `GrafanaDashboard` XRs implementation — ADR 0021 / component B4 (consumed here).
- SLI/SLO definitions, error budgets, automated alert→remediation loops — deferred to future-enhancements (explicit scope note).
- HolmesGPT self-management / AlertManager→HolmesGPT trigger — views V6-07 / V6-10.
- CloudEvent schema registry — component B12.

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A2 Langfuse** — the LLM-grade observability plane (prompts, costs, evals); correlated to Tempo by `trace_id`.
- **A13 Tempo + Mimir** — distributed tracing backend (Tempo) and long-term metrics store (Mimir); Prometheus stays as the scrape layer.
- **D1 Operator integrated dashboards** — operator-facing Grafana dashboards over the shared pipelines.
- **D2 Developer dashboards** — developer-facing Grafana dashboards over the shared pipelines.

ADR decisions honored:
- **ADR 0011** — three-layer testing with CLI orchestration; non-unit test results stream to OpenSearch advisory; metrics/logs via OTel.
- **ADR 0015** — Tempo + Langfuse correlated by `trace_id`.
- **ADR 0034** — audit pipeline: durable adapter, Postgres + S3 system of record, OpenSearch advisory only.
- **ADR 0035** — dynamic log-level / trace-granularity toggle with staged restart for non-reloadable services.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Consumed:
- `LogLevel` (owner: per-component) — `componentSelector`, `level`, `traceGranularity`, `scope` (component/tenant/eventClass), `expiresAt` (ADR 0035).
- `GrafanaDashboard` (XR, owner B4) — `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled) (ADR 0021); dashboards are namespaced XRs.
- `AuditLog` (XRD, owner B4) — `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec`, `endpointReplicas` (ADR 0034); provisions the audit pipeline this view's audit plane runs on.

All namespaced; versioning per ADR 0030; `LogLevel` versioning per-component, XRD versioning owned by B4.

### 4.2 APIs / SDK surfaces
- **Platform SDK (B6)** OTel emission — agent-side telemetry surface carrying `trace_id`.
- **Audit adapter library (A18)** — the single Python library every component links to emit audit; posts to the audit endpoint; no direct store writes.
- OTel collector ingest — the shared collection path for agent pods, LiteLLM, Envoy, ARK, UIs.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the observability slice (per-event-type names deferred to B12 registry):
- `platform.audit.*` — all audit emission, into the Postgres + S3 system of record with OpenSearch advisory fanout (ADR 0034).
- `platform.observability.*` — threshold crossings and alert routing (e.g. budget-exceeded notifications, SLA-style alerts).
Consumed:
- `platform.observability.*` — by alert-routing trigger flows (e.g. budget-exceeded → email user, defined in V6-07).

### 4.4 Data schemas / connection-secret contracts
- Audit system of record = Postgres `audit_events` table; on AWS a ~5-min batch CronJob writes verified immutable S3 objects then deletes Postgres rows; on kind Postgres alone is authoritative (ADR 0034).
- OpenSearch audit + test-result indexes are advisory and rebuildable from primaries; if OpenSearch is down, ingestion still succeeds.
- Telemetry stores (Tempo, Mimir, Loki) hold derived/observability data, not systems of record for platform state.

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom for Langfuse, Tempo/Mimir, and the dashboards lives in component SPECs A2, A13, D1, D2; the audit adapter/endpoint lives in A18).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-05-01:** Every span emitted by the agent SDK and the gateway MUST carry the same `trace_id` (LLM-flavored or not) so Grafana (Tempo) and Langfuse correlate and deep-link bidirectionally (ADR 0015). (§6.5, lines ~406–421)
- **REQ-V6-05-02:** All components MUST emit telemetry through the shared OTel collector path (Tempo/Loki/Mimir) and MUST NOT run a parallel observability stack; Mimir holds long-term metrics, Prometheus stays the scrape layer. (§6.5, lines ~393–421)
- **REQ-V6-05-03:** No component may write audit records directly to OpenSearch, Postgres, or S3; every component MUST link the audit adapter library and post to the single audit endpoint (ADR 0034). (§6.5, lines ~409–423)
- **REQ-V6-05-04:** The audit system of record MUST be Postgres + S3 (S3 AWS-only); OpenSearch MUST be advisory fanout only, rebuildable from primaries, and audit ingestion MUST still succeed if OpenSearch is unavailable (ADR 0034). (§6.5, line ~423)
- **REQ-V6-05-05:** Verbosity (log-level / trace-granularity) MUST be dynamically togglable per component/tenant/event-class via a `LogLevel` mechanism; in-house components use an in-process toggle, non-reloadable services (e.g. LiteLLM) use a rolling-restart toggle, and the platform MUST record which pattern applies (ADR 0035). (§6.5, lines ~425–428)
- **REQ-V6-05-06:** A non-reloadable rolling-restart toggle MUST be safe by construction: `replicas >= 2`, readiness probe gating Service traffic, `preStop: sleep 10-15s`, `terminationGracePeriodSeconds` 60–300s so streaming completes (ADR 0035). (§6.5, line ~428)
- **REQ-V6-05-07:** Off-mode (baseline verbosity) MUST have zero performance cost — no always-on trace-then-sample-and-drop pipeline; cost is paid only when verbosity is raised. (§6.5, line ~430)
- **REQ-V6-05-08:** Non-unit test results (integration, e2e, evaluation, red-team) MUST stream to OpenSearch as an advisory index and metrics/logs MUST publish via OTel through the shared collector path, not a test-only stack (ADR 0011). (§6.5, line ~432)
- **REQ-V6-05-09:** The architecture MUST commit collection, storage, correlation, per-component dashboards, and failure-condition alert rules; SLI/SLO/error-budget/auto-remediation work is explicitly deferred to future-enhancements. (§6.5, line ~434)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** `GrafanaDashboard` visibility is RBAC + OPA-controlled; dashboards are namespaced XRs; audit captures cross-component activity for tenant-scoped review.
- **Observability (self):** the slice is the observability plane; durability invariant (REQ-04) guarantees audit survives OpenSearch outage.
- **Scale:** Mimir for long-term metrics; off-mode zero-cost telemetry (REQ-07) bounds steady-state overhead.
- **Versioning (ADR 0030):** `LogLevel` per-component; `GrafanaDashboard`/`AuditLog` XRD versioning owned by B4.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A2, A13, D1, D2, and the audit adapter/endpoint A18).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-05-01:** A single request's `trace_id` resolves both a Tempo trace and a Langfuse trace, and a Grafana panel deep-links to the matching Langfuse view and back. (→ REQ-01)
- **AC-V6-05-02:** All emitting components are confirmed to route through one OTel collector; no parallel collector/stack exists. (→ REQ-02)
- **AC-V6-05-03:** A code/audit scan shows no direct writes to OpenSearch/Postgres/S3 for audit; every audit path goes through the adapter to the endpoint. (→ REQ-03)
- **AC-V6-05-04:** With OpenSearch down, audit ingestion still succeeds and the advisory index rebuilds from S3 (AWS) / Postgres (kind) once OpenSearch returns. (→ REQ-04)
- **AC-V6-05-05:** A `LogLevel` change raises verbosity for a chosen component/tenant/event-class and lowers it back; the platform reports whether the component used an in-process or rolling-restart toggle. (→ REQ-05)
- **AC-V6-05-06:** A rolling-restart verbosity toggle on LiteLLM completes with no dropped in-flight streaming response (deployment shape per REQ-06). (→ REQ-06)
- **AC-V6-05-07:** At baseline verbosity, no tracing overhead is measurable beyond baseline; cost appears only when verbosity is raised. (→ REQ-07)
- **AC-V6-05-08:** Integration/e2e/eval/red-team results appear in the OpenSearch advisory index and their metrics/logs in the shared OTel path. (→ REQ-08)
- **AC-V6-05-09:** Per-component dashboards and failure-condition alerts exist; no SLI/SLO/error-budget artifacts are required for v1.0 (deferred). (→ REQ-09)

## 10. Risks & Open Questions
- **OQ-1 (low):** SLI/SLO/error-budget scope is deferred; this view commits only collection paths and per-component dashboards/alerts.
- **OQ-2 (med):** `trace_id` propagation depends on every harness emitting OTel correctly; unsupported third-party harnesses (V6-02) may emit non-conformant spans — correlation degrades but enforcement does not.
- **R-1 (med):** A component that bypasses the audit adapter would create a silent audit gap; mitigated by REQ-03 scan/conformance and the adapter being the only linkable path.
- **R-2 (low):** Frequent rolling-restart verbosity toggles on LiteLLM churn the gateway; mitigated by the ADR 0035 deployment shape and by recording the toggle cost to operators (REQ-05/06).

## 11. References
- architecture-overview.md §6.5 Observability architecture (lines ~370–434); §6.3 (audit pipeline stores); §6.1 (rolling-restart deployment shape, line ~214).
- ADRs: 0011 (three-layer testing / OTel publication), 0015 (Tempo + Langfuse trace_id), 0034 (audit pipeline), 0035 (log-level toggle).
- Realizing components: A2 (Langfuse), A13 (Tempo + Mimir), D1 (operator dashboards), D2 (developer dashboards); audit adapter/endpoint A18; dashboards-as-XR via B4.
- Related views: V6-01, V6-02, V6-03, V6-06, V6-07, V6-10.

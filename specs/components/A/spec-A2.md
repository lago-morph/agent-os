# SPEC A2 — Langfuse

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [B1, B6, B9, B10] · adrs: [0015, 0035, 0030, 0034, 0031] · views: [6.5]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A2 installs and operates **Langfuse**, the platform's LLM-grade observability and tracing
backend (architecture-overview.md §6.5). The platform produces two flavors of trace data:
general OpenTelemetry spans (handled by Tempo, component A13) and LLM-grade traces — prompts,
completions, costs, evaluator scores, dataset/red-team runs — that need first-class tooling.
Per ADR 0015 these two backends are run side-by-side and **correlated by a shared `trace_id`**.

Langfuse is the destination for LLM-flavored spans emitted by the agent SDK, the LiteLLM gateway
(A1), and promptfoo red-team results from CI. It gives developers prompt-level detail reachable
by deep-link from Grafana/Tempo and back, and is one of the four backends HolmesGPT's
observability toolset consumes (Tempo + Mimir + Loki + Langfuse). A2 is a W0 foundation
component so other components can emit LLM traces from day one.

## 2. Scope

### 2.1 In scope
- Langfuse Helm install + values, branding, and operation as a platform service.
- Backing-store wiring: Langfuse-resident state in the shared Postgres deployment (provisioned
  via the `XPostgres` substrate XRD, owned by B4); object storage for large payloads via the
  `XObjectStore` substrate XRD where Langfuse requires it `[PROPOSED — not in source]`.
- SSO via Keycloak / oauth2-proxy handoff (config owned by B1; A2 exposes the surface).
- The ingestion endpoint configuration that the agent SDK and LiteLLM callbacks target.
- `trace_id` correlation contract honored on the receiving side (ADR 0015).
- Grafana/Tempo → Langfuse deep-link wiring contributions (link target config; the Grafana
  dashboards themselves are A13/Workstream D).
- `LogLevel`-driven trace-granularity gating on the Langfuse-ingest path (ADR 0035).
- Standard §14.1 Workstream-A deliverable set (see §8).

### 2.2 Out of scope (and where it lives instead)
- General OTel trace storage — **Tempo (A13)**.
- Metrics LTS — **Mimir (A13)**; baseline Prometheus/Loki/Grafana — k8-platform baseline.
- Emitting the `trace_id` into spans — **agent SDK (B7) and LiteLLM gateway (A1)** are the
  load-bearing emitters per ADR 0015; A2 only receives and correlates.
- Audit records — **audit endpoint + adapter (A18)**; Langfuse is not an audit sink.
- Postgres / object-store provisioning Compositions — **Crossplane Compositions (B4)**.
- SSO proxy config — **B1**; A2 consumes it.
- Platform SDK OTel-emission surface — **B6**.
- Coach/HolmesGPT consumption logic — **B10 / A14**; A2 only exposes the backend.

## 3. Context & Dependencies

**Upstream consumed:** none (W0 foundation). A2 consumes only the assumed k8-platform baseline
(Grafana/Prometheus/Loki) and the shared Postgres substrate (via B4's `XPostgres`, available in
the foundation band).

**Downstream consumers:**
- **B1** — SSO/auth proxy layer puts oauth2-proxy in front of the Langfuse UI.
- **B6** — Platform SDK's OTel emission surface targets Langfuse for LLM-grade spans.
- **B9** — `agent-platform` CLI surfaces Langfuse links / trace lookups.
- **B10** — Coach Component observes traces in Langfuse to propose prompt/skill changes.

**ADR decisions honored:**
- **ADR 0015** — Tempo + Langfuse correlated by `trace_id`. A2 must accept and key on the same
  `trace_id` the SDK/gateway emit; must support Grafana→Langfuse deep links; LLM-grade data only
  (non-LLM spans never enter Langfuse).
- **ADR 0035** — trace granularity in Langfuse is gated by the dynamic log/trace toggle
  (`LogLevel`); v1.0 does not run always-on sample-and-drop. Langfuse is a non-in-house service,
  so the toggle on its own deployment follows the rolling-restart pattern where its config is not
  hot-reloadable `[PROPOSED — not in source]` (ADR 0035 only names LiteLLM explicitly).
- **ADR 0034** — A2 is NOT an audit sink; audit flows through the adapter/endpoint only.
- **ADR 0031** — any CloudEvents A2 emits fall under `platform.observability.*` (threshold
  crossings) and/or `platform.lifecycle.*`; per-event names are deferred to B12.
- **ADR 0030** — A2 ships no platform CRD; versioning policy applies to any HTTP surface it
  exposes (URL-path versioning for custom glue, if any).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
A2 defines **no platform CRD**. It consumes:
- `XPostgres` (XRD, owner B4) — `version`, `size`, `storage`, `connectionSecretRef`,
  `substrateClass` — for Langfuse-resident state.
- `LogLevel` (per-component, ADR 0035) — `componentSelector`, `level`, `traceGranularity`,
  `scope` (component/tenant/eventClass), `expiresAt` — A2 honors this to gate ingest verbosity.
- `GrafanaDashboard` (XR, owner B4) — `dashboardJson`, `folder`, `visibility` — A2's
  per-component dashboard is delivered as a `GrafanaDashboard` XR (§8).

### 4.2 APIs / SDK surfaces
- **Langfuse ingestion API** (upstream Langfuse product API) — the endpoint the agent SDK (B6)
  and LiteLLM callbacks (A1/B2) post LLM-grade spans to. A2 configures and exposes it; the exact
  emission method signatures live in **B6** (Platform SDK §3.1: "OTel emission") and are not
  specified in source.
- **Langfuse web UI** — fronted by oauth2-proxy (B1), scoped from platform JWT claims.
- No new platform HTTP API is introduced by A2 beyond Langfuse's own product surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.observability.*` for threshold/health crossings on the Langfuse
  service `[PROPOSED — not in source: specific event names deferred to B12]`.
- **Consumed:** none required for core function. `[PROPOSED — not in source]`
- Per-event-type names/schemas are owned by **B12's registry** (deferred per interface-contract).

### 4.4 Data schemas / connection-secret contracts
- Langfuse Postgres connection consumes the **uniform connection secret** from `XPostgres`:
  `host`, `port`, `user`, `password`, `dbname` (ADR 0041 canonical shape).
- Langfuse internal trace schema is the **upstream product schema** — not a platform-owned
  contract; the only platform-owned invariant is that `trace_id` matches the value Tempo holds
  (ADR 0015).

## 5. OSS-vs-Custom Decision
- **Named project:** Langfuse (glossary: "LLM observability / tracing (A2)").
- **Mode:** install + configure + operate (Helm). Unmodified upstream; no fork.
- **Version:** pin a tested Langfuse chart/app version `[PROPOSED — not in source: exact version
  is design-time]`.
- **ADR linkage:** ADR 0015 (chosen as the LLM-grade backend alongside Tempo).
- **Rationale:** purpose-built for LLM workflows (prompts/costs/evals); a generic OTel backend
  would under-serve these surfaces (ADR 0015 "Alternatives considered").

## 6. Functional Requirements
- **REQ-A2-01:** A2 SHALL install Langfuse via Helm with platform values (branding, replicas,
  resource sizing via Kustomize overlay) as a single ArgoCD-synced release.
- **REQ-A2-02:** A2 SHALL persist Langfuse-resident state in the shared Postgres provisioned via
  the `XPostgres` XRD, consuming the uniform connection secret (host/port/user/password/dbname).
- **REQ-A2-03:** A2 SHALL expose the Langfuse ingestion endpoint such that the agent SDK (B6) and
  LiteLLM callbacks (A1) can post LLM-grade spans to it.
- **REQ-A2-04:** A2 SHALL store and key LLM-grade traces by the `trace_id` supplied by the
  emitter, preserving correlation with Tempo (ADR 0015) — A2 SHALL NOT rewrite or regenerate
  `trace_id` on ingest.
- **REQ-A2-05:** A2 SHALL accept promptfoo red-team results and LiteLLM prompt/cost/eval data
  into Langfuse (ADR 0015), and SHALL NOT accept non-LLM spans.
- **REQ-A2-06:** A2 SHALL configure Grafana/Tempo→Langfuse deep-link targets so a Tempo span can
  open the correlated Langfuse trace without manual ID copy-paste.
- **REQ-A2-07:** A2 SHALL honor a `LogLevel` CR targeting the Langfuse-ingest path: at baseline
  verbosity no trace-storage cost is incurred beyond what emitters send; raised verbosity is
  applied per ADR 0035 (rolling-restart pattern for the non-reloadable Langfuse service).
- **REQ-A2-08:** A2 SHALL front the Langfuse UI with Keycloak SSO (via B1's oauth2-proxy),
  scoping visibility from platform JWT claims.
- **REQ-A2-09:** A2 SHALL deliver a per-component Grafana dashboard as a `GrafanaDashboard` XR and
  alert rules for Langfuse-service failure conditions (ingest down, store unreachable, backlog).
- **REQ-A2-10:** A2 SHALL contribute a HolmesGPT toolset entry exposing Langfuse trace lookups
  (dotted toolset-contribution edge `A2 -.-> A14`).

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** UI access SSO'd and JWT-claim-scoped; Langfuse project /
  trace visibility partitioned per tenant `[PROPOSED — not in source: exact partitioning is
  design-time]`. RBAC-as-floor / OPA-as-restrictor (ADR 0018) governs admin operations.
- **Observability (§6.5):** A2 self-emits OTel spans/metrics through the standard collector path;
  it is itself observable in Tempo/Mimir.
- **Scale:** sized to absorb the platform's LLM-trace volume at raised verbosity without dropping;
  off-mode has zero added cost (ADR 0035). Concrete throughput target deferred to F5.
- **Versioning (ADR 0030):** pin tested upstream version; any platform glue HTTP surface uses
  URL-path versioning (`/v1/...`).

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable**.
- Per-product docs (10.5) — **applicable**.
- Runbook (10.7) + backup/restore — **applicable** (Langfuse state is in shared Postgres; inherits
  its backup/PITR posture).
- Alerts — **applicable** (ingest/store/backlog failure conditions).
- Grafana dashboard (Crossplane `GrafanaDashboard` XR) — **applicable**.
- Headlamp plugin — **N/A for v1.0** — no A2-specific editor; Langfuse UI is the native surface
  (no A2 row in ADR 0039's editor set). Read-only inspection via the generic Headlamp framework.
- OPA/Rego integration — **applicable** (admission policy on A2 workloads; UI action gating via
  JWT claims).
- Audit emission (ADR 0034) — **applicable** (A2 admin actions emit via the adapter; A2 is not an
  audit *sink*).
- Knative trigger flow — **applicable** (emits `platform.observability.*` on service health).
- HolmesGPT toolset — **applicable** (Langfuse trace-query tool).
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable**.
- Tutorials & how-tos — **applicable** (observability how-to: find an agent's LLM trace).

## 9. Acceptance Criteria
- **AC-A2-01 (REQ-A2-01):** `helm`/ArgoCD sync brings Langfuse to Ready; pods pass readiness;
  one ArgoCD application owns the release. *(Chainsaw/PyTest)*
- **AC-A2-02 (REQ-A2-02):** Langfuse connects using only the five connection-secret fields from
  `XPostgres`; killing/recreating the pod retains traces (state is in Postgres). *(PyTest)*
- **AC-A2-03 (REQ-A2-03):** A span POSTed to the configured ingest endpoint with a valid key is
  retrievable in Langfuse. *(PyTest)*
- **AC-A2-04 (REQ-A2-04):** A span emitted with `trace_id=X` is stored under `trace_id=X` and is
  joinable to a Tempo span carrying the same `trace_id`. *(PyTest)*
- **AC-A2-05 (REQ-A2-05):** A promptfoo red-team payload lands in Langfuse; a synthetic non-LLM
  OTel span is rejected/absent. *(PyTest)*
- **AC-A2-06 (REQ-A2-06):** From a Tempo span in Grafana, the deep link opens the matching
  Langfuse trace. *(Playwright)*
- **AC-A2-07 (REQ-A2-07):** Applying a `LogLevel` CR raising trace granularity triggers the
  documented rolling-restart and increases captured detail; reverting restores baseline with no
  residual cost. *(Chainsaw + PyTest)*
- **AC-A2-08 (REQ-A2-08):** Unauthenticated UI access is redirected to Keycloak; a user sees only
  JWT-claim-permitted projects. *(Playwright)*
- **AC-A2-09 (REQ-A2-09):** The `GrafanaDashboard` XR renders the Langfuse dashboard; a simulated
  ingest-down condition fires the alert rule. *(Chainsaw + PyTest)*
- **AC-A2-10 (REQ-A2-10):** HolmesGPT can invoke the Langfuse-lookup tool and return a trace by
  `trace_id`. *(PyTest)*

## 10. Risks & Open Questions
- **R1 (med):** ADR 0035 names only LiteLLM as rolling-restart; whether Langfuse ingest is
  hot-reloadable for `LogLevel` is unstated → flagged `[PROPOSED — not in source]`. *Open:
  confirm Langfuse reload behavior during implementation.*
- **R2 (low):** Per-tenant trace partitioning model in Langfuse is design-time
  `[PROPOSED — not in source]`. Blast radius limited to UI scoping; enforcement floor is RBAC/OPA.
- **R3 (low):** Object-store need for large payloads (`XObjectStore`) is `[PROPOSED]`; may be
  unnecessary if Postgres suffices.
- **R4 (med):** `trace_id` correlation is only as good as emitters (A1/B6); an emitter that omits
  `trace_id` breaks correlation (ADR 0015 consequence) — out of A2's control; covered by SDK tests.

## 11. References
- architecture-overview.md §6.5 (observability architecture, lines ~370–434); §6.13 (versioning);
  §14.1 (Workstream A deliverables, A2 row, line ~1668).
- ADR 0015 (Tempo + Langfuse correlated by `trace_id`).
- ADR 0035 (dynamic log/trace toggle); ADR 0034 (audit pipeline — A2 not a sink);
  ADR 0031 (CloudEvent taxonomy); ADR 0030 (versioning); ADR 0041 (connection-secret shape).
- View V6-05 (Observability architecture).
- Related pieces: A13 (Tempo + Mimir), A1 (LiteLLM emitter), B6 (SDK OTel emission), B10 (Coach),
  A14 (HolmesGPT toolset consumer), B1 (SSO), B4 (XPostgres/GrafanaDashboard).

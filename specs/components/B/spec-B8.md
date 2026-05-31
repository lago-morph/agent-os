# SPEC B8 — Knative event adapter services

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A4] · downstream: [B12] · adrs: [0031, 0023, 0030, 0034] · views: [6.7, 6.2]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B8 is purely a **consumer** of event namespaces: it takes explicit consumer dependencies on the owning components for each namespace it emits to or subscribes from — B13 (`platform.capability`), B19 (`platform.approval`), B14 (`platform.evaluation`), A5 (`platform.lifecycle`), A1 (`platform.gateway`), A7 (`platform.policy`/`platform.security`), A18 (`platform.audit`), A21 (`platform.tenant`), A13 (`platform.observability`). Schemas are authored and registered (in B12) by those owners; B8 never co-owns a namespace. Any security-relevant event the SDK detects MUST also be emitted under `platform.security` (schema owned by A7), in addition to local handling.

Knative Eventing has no native sink for `AgentRun` CRs or for Argo Workflows (§9). B8 supplies the two **thin Python adapter services** that bridge CloudEvents into platform resources: **CloudEvent → AgentRun** (creates an `AgentRun` CR for ARK) and **CloudEvent → Workflow** (submits an Argo Workflow). These are the two initial trigger-flow sinks in the eventing architecture (§6.7 diagram: `ARC[CloudEvent → AgentRun CR]`, `WFC[CloudEvent → Argo Workflow]`).

The adapters are **pure field-mapping services with no decision logic** — all filtering happens upstream at the Knative Trigger (declarative, GitOps-able, audited and policy-checked through Knative), so every event flows through Knative's audit and policy layer rather than through adapter-internal branching (§6.7). Sources differ per environment (AwsSqsSource on AWS, webhook receivers on kind) and are documented exceptions to substrate abstraction (ADR 0023); B8's adapters sit downstream of the Trigger and are source-agnostic.

## 2. Scope

### 2.1 In scope
- Two adapter Deployments: **CloudEvent → AgentRun** and **CloudEvent → Workflow** (§6.7).
- Field-mapping from the inbound CloudEvent envelope/data to the target resource spec: `AgentRun` (`agentRef`, `inputs`, `traceId`, `triggeredBy`, `state`) and an Argo Workflow submission.
- HTTP receiver conforming to the CloudEvents binding so a Knative Trigger can sink to the adapter.
- Idempotent resource creation keyed on a stable CloudEvent identity (so redelivery does not double-create).
- URL-path-versioned (`/v1/...`) adapter HTTP surface (interface-contract §3.3).
- Audit emission of each adapter action via the platform audit adapter (ADR 0034).
- Trace-context propagation: carry `traceId` from the CloudEvent into the created `AgentRun` for correlation (ADR 0015).

### 2.2 Out of scope (and where it lives instead)
- **Knative Eventing + NATS JetStream broker** install — Component A4.
- **Triggers / filters** themselves — declarative Knative Trigger resources authored by the emitting/consuming components (§6.7 — "filtering happens at the Knative Trigger"); B8 ships reference Triggers for its two flows but filtering logic is not adapter code.
- **Event sources** (AwsSqsSource, webhook receivers, PingSource, ApiServerSource) — environment-specific, ADR 0023; not B8.
- **CloudEvent schemas / schema registry** — Component **B12** (downstream of B8). B8 maps fields per whatever schema B12 publishes; it does not author schemas.
- **ARK operator** (reconciles the created `AgentRun`) — Component A5. **Argo Workflows** engine — Component A3. B8 only creates/submits; it does not reconcile or run.
- The **AlertManager → HolmesGPT** and **budget-exceeded → email** specific flows — those are wired by A14/B2 + Triggers; B8 provides the AgentRun/Workflow sinks they may target, not the flow definitions.

## 3. Context & Dependencies

**Upstream consumed:**
- **A4 (Knative Eventing + NATS JetStream broker)** — the broker, Trigger machinery, and CloudEvents delivery contract the adapters receive from.

**Downstream consumers:**
- **B12 (CloudEvent schema registry)** — depends on B8 (piece-index.csv `B8 → B12`); B8's adapters are the first concrete consumers exercising the schemas B12 registers; the AgentRun/Workflow mapping informs the registry's `platform.lifecycle.*` shapes.

**ADRs honored:**
- **ADR 0031** — every CloudEvent the adapters emit (e.g., lifecycle of the created AgentRun) falls under exactly one committed namespace; adapters carry `specversion` + per-event `schemaVersion`.
- **ADR 0023** — sources are environment-specific by design; adapters are downstream of the Trigger and source-agnostic, so they work identically across AWS/kind.
- **ADR 0030** — adapter HTTP API uses URL-path versioning (`/v1/...`); deprecated versions reachable ≥1 platform release after replacement.
- **ADR 0034** — adapter actions emit audit via the platform adapter, never direct writes.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — B8 owns no CRD. It **creates** `AgentRun` (interface-contract §1.2: `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state`; "Created by Knative event adapters or workflow steps") and **submits** Argo Workflows (no CRD owned by B8). It does not reconcile either — ARK (A5) and Argo (A3) own reconciliation.

### 4.2 APIs / SDK surfaces
- **Exposes** two CloudEvents HTTP receivers (one per adapter), versioned `/v1/...` (interface-contract §3.3). The receiver conforms to the CloudEvents HTTP binding (structured/binary). `[PROPOSED — not in source]` exact route paths/payload field-mapping table (design-time; informed by B12).
- **Consumes** the Kubernetes API to create `AgentRun` and submit Workflows.
- **Consumes** the platform audit adapter library (A18) for emission.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed:** CloudEvents delivered by a Knative Trigger; the trigger flows that target the AgentRun sink are typically `platform.lifecycle.*` or source-injected events. B8 does not filter — it maps.
- **Emitted:** lifecycle events for the resource it creates (e.g., AgentRun created) → `platform.lifecycle.*`; audit of the adapter action → `platform.audit.*`. `[PROPOSED — not in source]` concrete event-type names (B12 registry).

### 4.4 Data schemas / connection-secret contracts
- Inbound CloudEvent envelope per CloudEvents spec; data schema per B12 registry (not owned here). No connection-secret XRD owned. Trace correlation via `traceId` (ADR 0015 `trace_id`).

## 5. OSS-vs-Custom Decision

**Build-new (thin glue).** Upstream: **Knative Eventing** (A4) provides delivery + Triggers; **NATS JetStream** is the broker. B8 builds two small Python services because Knative has no native sink for `AgentRun` or Argo Workflows (§9 — "two thin Python adapter services (CloudEvent → resource); filtering happens at Knative Trigger"). Deliberately minimal — no decision logic, to keep all filtering at the Trigger layer where it is audited and policy-checked. No fork. ADR linkage: ADR 0023 (source-agnostic, downstream of Trigger), ADR 0031 (taxonomy).

## 6. Functional Requirements

- **REQ-B8-01:** B8 MUST provide a **CloudEvent → AgentRun** adapter that, on receiving a sinked CloudEvent, creates an `AgentRun` CR with `agentRef`, `inputs`, `traceId`, `triggeredBy`, and initial `state` mapped from the event (§6.7; interface-contract §1.2).
- **REQ-B8-02:** B8 MUST provide a **CloudEvent → Workflow** adapter that, on receiving a sinked CloudEvent, submits an Argo Workflow mapped from the event (§6.7).
- **REQ-B8-03:** The adapters MUST contain **no filtering or decision logic** — all event selection happens at the Knative Trigger (§6.7).
- **REQ-B8-04:** Adapter resource creation MUST be **idempotent** on redelivery keyed on a stable CloudEvent identity, so at-least-once delivery does not double-create the resource.
- **REQ-B8-05:** The adapters MUST be **source-agnostic** — identical behavior whether the upstream source is AwsSqsSource (AWS) or a webhook receiver (kind) (ADR 0023).
- **REQ-B8-06:** Each adapter MUST emit an audit event for its action via the platform audit adapter (ADR 0034); no direct writes.
- **REQ-B8-07:** The adapters MUST propagate the CloudEvent `traceId` into the created `AgentRun` for trace correlation (ADR 0015).
- **REQ-B8-08:** The adapter HTTP receivers MUST be URL-path-versioned (`/v1/...`) per ADR 0030 / interface-contract §3.3.
- **REQ-B8-09:** Lifecycle events the adapter emits for the created resource MUST fall under `platform.lifecycle.*` carrying `specversion` + `schemaVersion` (ADR 0031/0030).
- **REQ-B8-10:** The adapters MUST surface a clear failure (and not silently drop) when target-resource creation fails, so Knative's redelivery/dead-letter machinery can act.
- **REQ-B8-11:** B8 MUST ship reference Knative Trigger manifests for its two flows as examples, without embedding their filter logic in adapter code.

## 7. Non-Functional Requirements

- **Security:** adapters run with least-privilege ServiceAccounts (create `AgentRun` / submit Workflow only) in the namespaces they serve; cross-namespace creation governed by RBAC + OPA (§6.9).
- **Multi-tenancy (§6.9):** the created `AgentRun` lands in the tenant namespace implied by the event/Trigger routing; adapters must not cross tenant boundaries beyond their RBAC grant.
- **Observability (§6.5):** adapters emit metrics (events in, resources created, failures) on the OTel path; `traceId` propagation enables end-to-end traces (ADR 0015).
- **Scale:** stateless, horizontally scalable; idempotency keying enables safe replay; backlog absorbed by the broker (§11 capacity dashboard).
- **Versioning (ADR 0030):** `/v1/...` HTTP versioning; deprecated routes reachable ≥1 release after replacement.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests — **applicable** (two Deployments + reference Triggers, single manifest set; substrate-agnostic).
- Per-product docs (10.5) — **applicable** (adapter mapping reference, "how to add a trigger flow").
- Runbook (10.7) — **applicable** (adapter-down / redelivery-storm / resource-create-failure).
- Alerts — **applicable** (adapter error rate, create-failure, broker backlog at the adapter).
- Grafana dashboard (Crossplane XR) — **applicable** (events in/out, create success rate) — `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — N/A — B8 owns no CRD/admin surface; AgentRun is visible via ARK/Headlamp native views.
- OPA/Rego integration — **applicable** (admission/RBAC on adapter resource-creation; minimal). `[PROPOSED — not in source]` no named adapter policy in Canon.
- Audit emission (ADR 0034) — **applicable** (each adapter action).
- Knative trigger flow — **applicable** (this component *is* the sink half of the eventing fabric; ships reference Triggers for both flows).
- HolmesGPT toolset — **applicable** (query adapter health / failed creations). `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (Chainsaw: "emit CloudEvent on broker → expect AgentRun CR / Workflow"; PyTest: field-mapping + idempotency logic; Playwright: HTTP receiver flow).
- Tutorials & how-tos — **applicable** ("trigger an agent from an event" tutorial, §10.1).

## 9. Acceptance Criteria

- **AC-B8-01** (REQ-B8-01): A CloudEvent sinked to the AgentRun adapter produces an `AgentRun` with correctly mapped `agentRef`/`inputs`/`traceId`/`triggeredBy`/`state`. (Chainsaw)
- **AC-B8-02** (REQ-B8-02): A CloudEvent sinked to the Workflow adapter submits the expected Argo Workflow. (Chainsaw)
- **AC-B8-03** (REQ-B8-03): The adapter creates/submits for any event it receives and performs no content-based filtering. (PyTest)
- **AC-B8-04** (REQ-B8-04): Redelivering the same CloudEvent does not create a second `AgentRun`/Workflow. (Chainsaw/PyTest)
- **AC-B8-05** (REQ-B8-05): The same event payload produces the same result whether arriving via an AWS-style or webhook source. (PyTest)
- **AC-B8-06** (REQ-B8-06): Each adapter action emits an audit event via the adapter; no direct store write. (Chainsaw/PyTest)
- **AC-B8-07** (REQ-B8-07): The created `AgentRun.traceId` equals the inbound CloudEvent's trace id. (Chainsaw)
- **AC-B8-08** (REQ-B8-08): Adapter receivers respond under `/v1/...` and reject unknown versions appropriately. (Playwright)
- **AC-B8-09** (REQ-B8-09): Emitted lifecycle events carry `platform.lifecycle.*` type + `specversion` + `schemaVersion`. (Chainsaw)
- **AC-B8-10** (REQ-B8-10): A forced create-failure surfaces an error to Knative (non-2xx / dead-letter), not a silent drop. (PyTest/Chainsaw)
- **AC-B8-11** (REQ-B8-11): Reference Trigger manifests ship and route to the adapters without adapter-side filtering. (Chainsaw)

## 10. Risks & Open Questions

- **OQ-B8-1** (med): Exact CloudEvent-data → resource-spec field mapping depends on B12's schemas, not yet authored. Reconciliation: B8 ships against a documented provisional mapping and re-binds when B12 registers the `platform.lifecycle.*` types (B8 is upstream of B12, so B8 informs the shapes). `[PROPOSED — not in source]`.
- **R-B8-1** (med): At-least-once delivery means idempotency is load-bearing; a weak idempotency key causes duplicate agent runs. Mitigated by AC-B8-04; blast radius: spurious workloads/cost.
- **OQ-B8-2** (low): How the adapter chooses the target namespace for the created resource (from event attribute vs. Trigger config) — design-time. `[PROPOSED — not in source]`.
- **R-B8-2** (low): Workflow submission contract is Argo-version-specific (A3); pinned version bounds it.
- **OQ-B8-3** (low): Dead-letter / retry policy ownership (Trigger config vs adapter response codes) — sits at the Knative layer; B8 only signals failure.

## 11. References

- architecture-overview.md §6.7 Eventing — adapter diagram (~578–600), "filtering happens at the Knative Trigger / adapters are pure field-mapping" (~602), namespaces table (~608–620), initial trigger flows (~623–626).
- architecture-overview.md §9 Knative fill-in rows ("two thin Python adapter services", ~1413–1414).
- architecture-overview.md §6.2 / §7.2 AgentRun creation by event adapters.
- ADR 0023 (environment-specific sources), ADR 0031 (CloudEvent taxonomy), ADR 0030 (`/v1/...` versioning), ADR 0034 (audit via adapter), ADR 0015 (trace_id correlation).
- Interface-contract §1.2 (`AgentRun` fields; "Created by Knative event adapters"), §2 (`platform.lifecycle.*`, `platform.audit.*`), §3.3 (HTTP URL-path versioning).
- Related pieces: A4 (broker), A5 (ARK), A3 (Argo), B12 (schema registry, downstream).

# SPEC A4 — Knative Eventing + NATS JetStream broker

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A19, B8, B12] · adrs: [0004, 0031, 0023, 0034, 0030, 0002] · views: [6.7]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05

## 1. Purpose & Problem Statement

A4 is the platform's **event mesh**: Knative Eventing for declarative event routing and **NATS JetStream** as the broker backend (ADR 0004). The same broker runs in dev (kind) and prod (managed Kubernetes), so trigger flows, ordering semantics, and runbooks are identical end to end. Event *sources* differ per environment (AwsSqsSource on AWS, webhook receivers in kind — ADR 0023), but everything downstream of the source — broker, Triggers, adapters — is uniform.

The problem A4 solves is **a single declarative, audited, policy-checked transport for every platform CloudEvent**. Filtering happens at the Knative **Trigger** (GitOps-able, audited, OPA-filterable); adapters are pure field-mapping services with no decision logic. This keeps every event flowing through one observable, governed layer and lets the eventing fabric grow organically: each component designs its own trigger flows as it lands, without big-bang design.

A4 is **T0, contract-owning** — it owns the broker contract that the CloudEvent taxonomy (ADR 0031), the event adapters (B8), the schema registry (B12), the audit pipeline fanout (ADR 0034), and the Mattermost bridge (A19) all build on.

## 2. Scope

### 2.1 In scope

- Knative Eventing install (controller + the Broker abstraction).
- **NATS JetStream** install via Helm, same in dev and prod, with stream configuration managed declaratively (stream sizing, retention).
- The **Broker** (NATS JetStream-backed) onto which all `platform.*` CloudEvents flow.
- The **Trigger** machinery (declarative type-prefix filtering; GitOps-able; audited; **OPA-driven Trigger filtering** support per §6.7).
- Broker arrival / dispatch observability (metrics, audit events for arrival + dispatch outcome).
- Idempotency contract for consumers (at-least-once delivery from JetStream).
- The §14.1 standard deliverable set (Helm, docs, runbook, alerts, dashboard XR, Headlamp plugin integration, OPA integration, audit emission, Knative trigger-flow design, HolmesGPT toolset, 3-layer tests, tutorials).

### 2.2 Out of scope (and where it lives instead)

- **Event sources** (AwsSqsSource, Azure source, webhook receivers, PingSource, ApiServerSource) — environment-specific, **ADR 0023**; these are documented exceptions to the substrate-abstraction pattern. Owned by the components that need them (e.g. A19 Mattermost bridge, A14/HolmesGPT alert flow). A4 provides the broker they feed.
- **Adapter services** (CloudEvent → `AgentRun` CR; CloudEvent → Argo Workflow; the two initial trigger flows) — **B8** (Knative event adapter services).
- **CloudEvent per-event-type schemas + registry** — **B12** (schema registry).
- **CloudEvent top-level taxonomy** itself — fixed by **ADR 0031** (A4 enforces routing on it but does not define it).
- **Audit endpoint / adapter library** — **A18**; the `platform.audit.*` Trigger sinks into the audit pipeline, but A4 does not own the pipeline.
- **Mattermost bidirectional bridge + channel-routing OPA filter library** — **A19**.
- **OPA engine + Rego** — **A7** / **B16**.
- **Substrate XRD wrapping of the broker** — explicitly **not** wrapped; event sources are documented exceptions to ADR 0044, and NATS itself is the same install both substrates.

## 3. Context & Dependencies

**Upstream consumed:** None (W0 foundation). Installs onto the k8-platform baseline.

**Downstream consumers (what they consume):**
- **B8** — consumes the Broker + Trigger API to ship the two adapter services and the two initial trigger flows.
- **B12** — the schema registry registers event types that flow on this broker; A4 enforces only namespace-prefix routing, not schemas.
- **A19** — the Mattermost bridge is the first non-trivial cross-broker consumer (ADR 0004 consequence); uses Triggers + OPA-driven filtering.
- Every component that emits/consumes CloudEvents (cross-cutting, non-blocking): A1, A7, A18, ARK (A5), etc.

**ADR decisions honored:**
- **ADR 0004** — NATS JetStream is the broker backend; same install dev + prod; at-least-once delivery → consumers must be idempotent; broker selection is encapsulated below the Trigger layer (swappable without rewriting triggers/adapters).
- **ADR 0031** — the ten top-level CloudEvent namespaces are an architectural invariant; Triggers filter on namespace prefix; introducing a new top-level namespace is a breaking change requiring a new ADR. Every event carries `specversion` + `schemaVersion`.
- **ADR 0023** — environment-specific Knative sources are documented exceptions to ADR 0044; A4 does not unify source kinds behind a Composition.
- **ADR 0034** — `platform.audit.*` is the namespace the audit adapter consumes into Postgres + S3; OpenSearch indexing is advisory fanout. A4 routes; it does not store.
- **ADR 0002** — OPA-driven Trigger filtering (e.g. Mattermost channel routing) evaluates against OPA; admission of Knative/Trigger CRs goes through Gatekeeper.
- **ADR 0030** — CloudEvent schema versioning policy (`specversion` + `schemaVersion`); CRD versioning for any A4-owned config CRDs.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A4 installs Knative's own CRDs (`Broker`, `Trigger`, source kinds) — these are **upstream Knative CRDs**, not platform CRDs, and are not in the platform CRD inventory (§6.12). No platform-owned CRD is introduced by A4.

| Resource | Owner | Notes |
|---|---|---|
| `Broker` (Knative) | Knative / A4 install | NATS JetStream-backed; the single platform broker |
| `Trigger` (Knative) | Knative / declared per consuming component | filters by CloudEvent `type` prefix; OPA-filterable |
| Source kinds (`PingSource`, `ApiServerSource`, `AwsSqsSource`, webhook receivers) | per ADR 0023, owned by consuming components | **out of scope for A4** |

Platform CRDs owned here: **N/A — A4 introduces no platform CRD; it installs upstream Knative + NATS.** `[PROPOSED — not in source]` whether stream/retention config is expressed as a CRD or Helm values — source says "stream configuration managed declaratively" without specifying the mechanism; treat as Helm values unless Canon says otherwise.

### 4.2 APIs / SDK surfaces

- **Knative Broker / Trigger API** — the declarative surface consumers (B8, A19) use to route events. A4 provides the Broker instance + the filtering contract.
- **NATS JetStream admin** — stream configuration (sizing, retention) managed declaratively via Helm; operational surface only, not a platform API.
- No bespoke HTTP API. Adapters (B8) expose their own `/v1/...` HTTP surfaces, not A4.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

A4 is the **transport**, not a primary emitter of domain events. It emits operational/audit events about its own behavior:
- `platform.audit.*` — **event arrival and dispatch outcome** at the broker (the audit hook point listed for "Knative broker" in §6.6), via the audit adapter.
- `platform.observability.*` — broker-health threshold crossings (e.g. dispatch backlog). `[PROPOSED — not in source]` exact event-type names → B12.

Consumed: A4 transports **all** namespaces (`platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, `platform.security.*`) but interprets only the `type` prefix for routing. Every event carries `specversion` + `schemaVersion` (ADR 0031/0030); A4 does not validate schemas (that is B12's registry concern).

### 4.4 Data schemas / connection-secret contracts

- JetStream is durable transport, **not** a system of record; events in flight only. The reproducibility invariant (ADR 0014) means anything durable must live in a primary store (Postgres/S3/object) elsewhere.
- No connection-secret XRD (event sources are documented exceptions to ADR 0044). NATS auth/credentials are install-config. `[PROPOSED — not in source]` NATS client-credential secret shape — not specified.

## 5. OSS-vs-Custom Decision

- **Upstream projects:** Knative Eventing + NATS JetStream. **Decision: config** — install both via Helm and configure the Broker + streams declaratively. No fork, no wrap-service.
- **ADR 0004 linkage:** NATS JetStream chosen over Kafka / RabbitMQ / SNS+SQS for same-broker-dev-and-prod, light footprint, and Knative channel maturity. Selection is encapsulated below the Trigger layer, so a future swap rewrites neither Triggers nor adapters.
- Custom code lives in **B8** (adapters) and **A19** (Mattermost bridge), not in A4. A4's only "custom" is declarative stream/retention config + the standard deliverable set.

## 6. Functional Requirements

- **REQ-A4-01:** A4 SHALL install Knative Eventing and provision a single platform **Broker** backed by **NATS JetStream**.
- **REQ-A4-02:** The same NATS JetStream install SHALL run in dev (kind) and prod (managed Kubernetes) with identical downstream (Broker/Trigger/adapter) behavior.
- **REQ-A4-03:** NATS JetStream stream configuration (sizing, retention) SHALL be managed declaratively (GitOps-reconciled), not by imperative admin commands.
- **REQ-A4-04:** Triggers SHALL filter events declaratively by CloudEvent `type` namespace prefix, and SHALL support **OPA-driven filtering** for sink routing (e.g. Mattermost channel routing).
- **REQ-A4-05:** The broker SHALL provide at-least-once delivery; the consumer contract SHALL require idempotent handling (ADR 0004).
- **REQ-A4-06:** A4 SHALL reject/route only on the ten committed top-level namespaces (ADR 0031); it SHALL NOT introduce a new top-level namespace.
- **REQ-A4-07:** Broker event **arrival** and **dispatch outcome** SHALL emit audit events via the platform audit adapter (ADR 0034); A4 SHALL NOT write audit directly to any store.
- **REQ-A4-08:** A4 SHALL NOT own event sources; environment-specific sources (ADR 0023) are provided by consuming components and feed the single broker.
- **REQ-A4-09:** A4 SHALL NOT act as a system of record; events are in-flight transport only (ADR 0014).
- **REQ-A4-10:** Admission of Knative/Trigger/source CRs SHALL pass through Gatekeeper (ADR 0002).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** Trigger filtering can be OPA-gated per tenant/channel (RBAC-as-floor / OPA-restrictor, ADR 0018). Cross-tenant publish events route under `platform.tenant.*` and are policy-checked at the Trigger.
- **Availability / durability:** JetStream provides durable in-flight transport with at-least-once delivery; consumers idempotent. Broker outage must not silently drop events (durability), but A4 is not the system of record.
- **Observability (§6.5):** broker arrival + dispatch metrics to Mimir; OTel traces; per-component Grafana dashboard via `GrafanaDashboard` XR.
- **Portability (ADR 0004/0026):** runs on kind, EKS, Azure equivalents with small footprint; independent-cluster model.
- **Versioning (ADR 0030/0031):** CloudEvent `specversion` + `schemaVersion`; backward-compatible additions bump minor, breaking changes mint a new event type.

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | applicable — Knative + NATS JetStream charts |
| Per-product docs (10.5) | applicable |
| Operator runbook (10.7) | applicable — stream sizing, retention, backlog triage, broker recovery |
| Backup / restore | applicable-with-caveat — broker is in-flight transport, not SoR; "restore" = redeploy + replay from primaries; no broker backup loop |
| Alert rules | applicable — broker down, dispatch backlog, JetStream storage pressure, undeliverable events |
| Grafana dashboard (`GrafanaDashboard` XR) | applicable |
| Headlamp plugin | applicable — Trigger/Broker visibility (A4 ensures integration in our env) |
| OPA/Rego integration | applicable — Gatekeeper admission of Knative CRs; OPA-driven Trigger filtering helpers (Rego in B16) |
| Audit emission (ADR 0034) | applicable — arrival + dispatch outcome via adapter |
| Knative trigger flow | applicable — A4 owns the broker/Trigger machinery the two initial flows (B8) ride on |
| HolmesGPT toolset | applicable — broker-health, backlog, dispatch-failure query tools |
| 3-layer tests | applicable — Chainsaw (Broker/Trigger), PyTest (filtering/idempotency), Playwright (Headlamp view) |
| Tutorials & how-tos | applicable — "author a Trigger flow", "subscribe an external sink" |

## 9. Acceptance Criteria

- **AC-A4-01** (REQ-A4-01): A platform `Broker` exists, is Ready, and reports NATS JetStream as its backing. *(Chainsaw)*
- **AC-A4-02** (REQ-A4-02): The same NATS JetStream Helm release deploys on kind and on a managed-K8s target; a Trigger flow behaves identically on both. *(Chainsaw, two substrates)*
- **AC-A4-03** (REQ-A4-03): Stream sizing/retention are set from Git config; an out-of-band imperative change is reconciled back. *(Chainsaw)*
- **AC-A4-04** (REQ-A4-04): A Trigger filtering `platform.audit.*` receives audit events and not `platform.gateway.*` events; an OPA-gated Trigger routes only policy-allowed events to its sink. *(PyTest)*
- **AC-A4-05** (REQ-A4-05): A redelivered (duplicate) event is handled idempotently by a reference consumer; at-least-once is demonstrated under a forced retry. *(PyTest)*
- **AC-A4-06** (REQ-A4-06): An event with a `type` outside the ten namespaces is detectable as non-conformant (no Trigger routes it); no new top-level namespace is created. *(PyTest)*
- **AC-A4-07** (REQ-A4-07): Event arrival and dispatch-outcome produce audit records at the audit endpoint via the adapter, with no direct store write from A4. *(PyTest)*
- **AC-A4-08** (REQ-A4-08): A4 ships no source kind; a webhook/PingSource provided by a consuming component feeds the broker successfully. *(Chainsaw)*
- **AC-A4-09** (REQ-A4-09): Broker restart loses only in-flight-undelivered events per JetStream durability config; no durable platform state is sourced from the broker. *(Chainsaw)*
- **AC-A4-10** (REQ-A4-10): Creating a `Trigger` CR passes through Gatekeeper admission; a policy-violating Trigger is denied. *(Chainsaw)*

## 10. Risks & Open Questions

- **R1 (med):** Stream-config mechanism (CRD vs Helm values) is `[PROPOSED — not in source]`; pick one and document — affects GitOps reconcile shape.
- **R2 (med):** At-least-once delivery pushes idempotency onto every consumer; if a downstream consumer is non-idempotent (e.g. an adapter creating `AgentRun`), duplicate side effects occur. *Reconciliation:* B8 adapters must dedupe; A4 documents the contract. Blast radius med.
- **R3 (low):** Source/auth credential secret shape not specified → `[PROPOSED — not in source]`.
- **R4 (low):** A future broker swap is claimed cheap (encapsulated below Triggers), but JetStream-specific ordering/retention semantics could leak into consumer assumptions. Document the abstraction boundary.
- **OQ1:** Does the platform want broker-level dead-letter handling for undeliverable events? Not specified in source → `[PROPOSED — not in source]`.

## 11. References

- architecture-overview.md §6.7 Eventing architecture (line ~553), §6.6 (~436, audit/OPA hook points — Knative broker), §6.5 (~370, observability), §6.13 (~981, versioning).
- ADR 0004 (NATS JetStream broker), ADR 0031 (CloudEvent taxonomy), ADR 0023 (environment-specific Knative sources), ADR 0034 (audit pipeline — `platform.audit.*` consumer), ADR 0002 (OPA/Gatekeeper), ADR 0030 (versioning), ADR 0014 (Postgres primary / OpenSearch retrieval — reproducibility invariant), ADR 0026 (independent-cluster install).
- Related pieces: B8 (event adapters + initial flows), B12 (schema registry), A19 (Mattermost bridge), A18 (audit pipeline), A7 (OPA).

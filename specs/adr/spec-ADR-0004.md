# SPEC ADR-0004 — NATS JetStream as the Knative broker backend [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [A19;B8;B12] · adrs: [0004] · views: [6.7]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0004 fixes NATS JetStream as the broker backend behind Knative Eventing. This SPEC states what honoring that decision requires: the **same** NATS JetStream install runs in dev (kind) and prod (managed Kubernetes), all `platform.*` CloudEvents flow through this single broker, stream configuration is declarative, and broker selection stays encapsulated below the Knative Trigger layer so trigger filtering and the CloudEvent taxonomy are backend-independent. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- NATS JetStream as component A4's Knative broker backend in both kind and managed Kubernetes.
- The constraint that all `platform.*` CloudEvents transit this single broker, dispatched by Knative Triggers.
- Declarative stream configuration; at-least-once delivery + durability semantics; consumer idempotency requirement.
- Encapsulation: Trigger filtering and CloudEvent taxonomy (ADR 0031) are unaffected by the backend choice.

### 2.2 Out of scope (and where it lives instead)
- Knative Eventing + NATS install mechanics + deliverables — component A4 SPEC.
- Event-source selection (per-environment) — ADR 0023 (AwsSqsSource on AWS, webhook receivers on kind).
- CloudEvent taxonomy + per-event schemas — ADR 0031 + B12 registry.
- Knative event adapters — B8; Mattermost cross-broker consumer — ADR 0036.

## 3. Context & Dependencies
Upstream consumed: none (A4 is foundation). Downstream consumers: A19 (Mattermost adapter), B8 (event adapters), B12 (schema registry) build on the broker + Trigger layer.
ADR decisions honored: **0004** — NATS JetStream is the single dev+prod broker, declaratively configured; **0023** — environment-specific sources feed this single broker; **0031** — taxonomy is backend-independent; **0026** — small footprint suits independent-cluster install; **0030** — event versioning via `specversion` + `schemaVersion`.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — this ADR introduces no platform CRD/XRD. It fixes the broker backend below Knative Eventing's own resources.

### 4.2 APIs / SDK surfaces
No new SDK surface. The Knative Trigger filter API is the consumer contract; broker selection is below it.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
All ten top-level namespaces transit this broker: `platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, `platform.security.*`. Every event carries `specversion` + `schemaVersion`. Per-event-type names live in B12.

### 4.4 Data schemas / connection-secret contracts
N/A — NATS JetStream stream sizing/retention is declarative Knative/Helm config, not a substrate XRD connection-secret primitive (broker is a documented non-wrapped exception in spirit; not a substrate-asymmetric primitive).

## 5. OSS-vs-Custom Decision
Upstream project: **NATS JetStream**, provisioned by Helm with declarative stream config (A4). Mode: install + config + pin. Rationale per ADR 0004: same broker in dev (kind) and prod removes "works in dev, breaks in prod" eventing bugs; small footprint suits kind and the independent-cluster model. Rejected: Kafka (heaviest footprint, awkward in kind), RabbitMQ (heavier, less-maintained Knative channel), SNS+SQS (no native Knative channel, AWS-only, splits runbooks).

## 6. Functional Requirements
- REQ-ADR-0004-01: NATS JetStream MUST be the Knative broker backend in both kind and managed Kubernetes (same install topology).
- REQ-ADR-0004-02: All `platform.*` CloudEvents MUST transit this single broker and be dispatched by Knative Triggers.
- REQ-ADR-0004-03: Stream configuration MUST be declarative (Helm/GitOps), not hand-applied.
- REQ-ADR-0004-04: The broker MUST provide at-least-once delivery + durability; consumers MUST be idempotent.
- REQ-ADR-0004-05: Knative Trigger filtering and the CloudEvent taxonomy MUST be unaffected by the backend choice (encapsulated below the Trigger layer).
- REQ-ADR-0004-06: Broker arrival and dispatch metrics MUST be observed via the standard observability stack.

## 7. Non-Functional Requirements
- Resilience: at-least-once + durability for in-flight CloudEvents; idempotent consumers (a pre-existing CloudEvent requirement).
- Portability: identical broker, ordering semantics, and runbooks across kind/EKS/Azure equivalents (ADR 0026).
- Observability: broker arrival/dispatch metrics in the standard stack (§6.5).
- Versioning: events carry `specversion` + `schemaVersion`; breaking change mints a new event type (ADR 0030/0031).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1 deliverable set is owned by component A4; conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0004-01: The kind and managed-Kubernetes installs use the same NATS JetStream broker topology (manifest diff is environment-config only). (REQ-01)
- AC-ADR-0004-02: An event published under each `platform.*` namespace is dispatched to a subscribing Trigger sink through the broker. (REQ-02)
- AC-ADR-0004-03: Stream config exists only as declarative manifests; a hand-applied stream change is reverted by GitOps. (REQ-03)
- AC-ADR-0004-04: A redelivered event (duplicate) is handled idempotently by a reference consumer; no duplicate side effect. (REQ-04)
- AC-ADR-0004-05: Swapping the backend in a test harness leaves Trigger filters and taxonomy unchanged. (REQ-05)
- AC-ADR-0004-06: Broker arrival and dispatch metrics appear in the observability stack. (REQ-06)

## 10. Risks & Open Questions
- Event volumes outgrowing NATS JetStream (blast radius: low) — Knative abstraction permits a backend swap without rewriting triggers/adapters; captured as future evolution.
- Stream sizing/retention defaults (low) — Workstream A responsibility; settle in A4 plan.
- Cross-broker consumer validation (low) — Mattermost (ADR 0036) is the first non-trivial validator.

## 11. References
- ADR 0004 (`adr/0004-nats-jetstream-broker.md`) — the decision.
- Enforcing components: A4 (Knative Eventing + NATS JetStream broker, owner), B8 (event adapters), B12 (schema registry), A19 (Mattermost cross-broker consumer).
- architecture-overview.md §5, §6.5, §6.7, §14.1; architecture-backlog.md §2.7, §2.8.
- Related: ADR 0023, 0031, 0036, 0026, 0030.

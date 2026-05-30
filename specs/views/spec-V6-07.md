# SPEC V6-07 — Eventing architecture (Knative + NATS JetStream) `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0004, 0023, 0030, 0031, 0036] · views: [6.7]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the eventing slice: the slice in which **Knative Eventing** is the event mesh and **NATS JetStream** is the broker backend (same broker in dev and prod). Every platform event is a CloudEvent; filtering happens declaratively at the **Knative Trigger**; adapters are pure field-mapping services with no decision logic. It is not a buildable component; it is realized by the Knative Eventing + NATS JetStream install (A4), the Knative event adapter services (B8), and the CloudEvent schema registry (B12).

The problem it solves: an agent platform must route events between many producers and consumers without each component re-inventing transport, filtering, or auditing. By fixing the closed ten-namespace CloudEvent taxonomy, the Trigger-does-the-filtering invariant, the adapters-have-no-logic invariant, and the same-broker-everywhere invariant — while allowing **environment-specific event sources** as documented exceptions to substrate abstraction — this view ensures the event fabric grows organically (each component designs its own flows) while every event stays governed and observable.

## 2. Scope
### 2.1 In scope
- The mesh invariant: Knative Eventing for the event mesh, NATS JetStream as the broker backend, same broker in dev and prod (ADR 0004).
- The source invariant: event sources are environment-specific by design (PingSource, ApiServerSource, AwsSqsSource on AWS, webhook receivers in kind), a documented exception to substrate abstraction (ADR 0023).
- The filtering invariant: filtering happens at the Knative Trigger (declarative, GitOps-able, audited, policy-checked).
- The adapter invariant: adapters are pure field-mapping services with no decision logic (CloudEvent → `AgentRun` CR; CloudEvent → Argo Workflow).
- The CloudEvent taxonomy invariant: the closed set of ten top-level namespaces; every emitted event falls under exactly one; schemas live in B12's registry (ADR 0031).
- Event versioning: CloudEvents `specversion` + per-event-type `schemaVersion`; backward-compatible additions bump minor; breaking changes mint a new event type (ADR 0030/0031).
- The two initial v1.0 trigger flows: AlertManager → HolmesGPT; budget-exceeded → email user.
- Mattermost bidirectional integration via Knative Eventing with OPA-driven Trigger filtering, generic chat-platform driver pattern (ADR 0036).

### 2.2 Out of scope (and where it lives instead)
- CloudEvent per-event-type names and concrete schemas — component B12 registry (deferred per ADR 0031).
- The CloudEvent schema-registry implementation — component B12 (consumed, not defined here).
- HolmesGPT self-management behavior on receipt of AlertManager events — view V6-10.
- Budget enforcement / `BudgetPolicy` semantics — view V6-06 (the budget-exceeded CloudEvent originates from a LiteLLM callback).
- ARK `AgentRun` reconciliation — view V6-02 (the adapter only creates the CR).
- Argo Workflows engine — component A3 (the adapter only submits the workflow).
- Mattermost Keycloak GitLab-OAuth-spoof auth detail — view V6-11 / §9 (referenced here).

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A4 Knative Eventing + NATS JetStream broker** — the event mesh and broker backend; Triggers + filters; the same broker in dev and prod (ADR 0004).
- **B8 Knative event adapter services** — the thin Python field-mapping services (CloudEvent → `AgentRun` CR; CloudEvent → Argo Workflow) with no decision logic.
- **B12 CloudEvent schema registry** — the Git JSON-schema registry for event types under the ten namespaces; external subscribers subscribe declaratively.

ADR decisions honored:
- **ADR 0004** — NATS JetStream as the Knative broker backend.
- **ADR 0023** — Knative broker is in the architecture; sources are environment-specific by design (documented exception to substrate abstraction).
- **ADR 0030** — versioning policy: event versioning carries `schemaVersion`; breaking changes mint new event types.
- **ADR 0031** — CloudEvent top-level type taxonomy: closed set of ten namespaces; introducing a new namespace is a breaking change requiring a new ADR.
- **ADR 0036** — two-way Mattermost integration via Knative Eventing as the v1.0 chat-platform pattern.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Produced by an adapter (owned by ARK, A5):
- `AgentRun` — `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state` (created by Knative event adapters or workflow steps).

The eventing slice owns no CRDs of its own; Knative `Broker`, `Trigger`, and source kinds are Knative-native resources (not platform CRDs). Environment-specific sources (AwsSqsSource, webhook receivers) are NOT wrapped behind a Crossplane Composition (ADR 0023).

### 4.2 APIs / SDK surfaces
- CloudEvents over the Knative broker (NATS JetStream backend) — the transport every producer/consumer uses.
- Knative Trigger filters — declarative, GitOps-able filtering surface (the only place filtering happens).
- Adapter HTTP services — URL-path versioned (`/v1/...`) per interface-contract §3.3; pure field-mapping, no decision logic.
- B12 schema registry — Git JSON schemas per event type; external systems subscribe declaratively.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
The closed set of ten top-level namespaces — every platform event falls under exactly one:
`platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, `platform.security.*`.
- Per-event-type names within each namespace are design-time per component and deferred to B12's registry.
- `platform.capability.changed` is the specific capability-change event (ADR 0013).
- Initial v1.0 flows: AlertManager alerts route in as CloudEvents → Trigger → HolmesGPT; a LiteLLM callback emits a budget CloudEvent (`platform.observability.*`) → Trigger → notification adapter emails the user.

### 4.4 Data schemas / connection-secret contracts
- Every CloudEvent carries CloudEvents-native `specversion` plus a per-event-type `schemaVersion`; backward-compatible additions bump minor, breaking changes mint a new event type (ADR 0030/0031).
- Concrete per-event JSON schemas live in B12's Git registry — `[PROPOSED — not in source]` for any specific event-type field set (deferred to B12).
- Mattermost integration uses the same broker + Trigger machinery; OPA-driven Trigger filtering decides which events route to which channel.

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom for Knative/NATS, the adapter services, and the schema registry lives in component SPECs A4, B8, B12).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-07-01:** The platform MUST use Knative Eventing as the event mesh with NATS JetStream as the broker backend, and the same broker MUST be used in dev and prod (ADR 0004). (§6.7, lines ~553, ~569)
- **REQ-V6-07-02:** Event sources MUST be environment-specific by design (PingSource, ApiServerSource, AwsSqsSource on AWS, webhook receivers in kind) and are a documented exception to the substrate-abstraction pattern — source kinds MUST NOT be unified behind a Crossplane Composition (ADR 0023). (§6.7, line ~555)
- **REQ-V6-07-03:** All event filtering MUST happen at the Knative Trigger (declarative, GitOps-able, audited and policy-checked); no filtering logic may live in adapters. (§6.7, line ~602)
- **REQ-V6-07-04:** Adapters MUST be pure field-mapping services with no decision logic — one creates an `AgentRun` CR, the other submits an Argo Workflow — so every event flows through Knative's audit and policy layer. (§6.7, line ~602)
- **REQ-V6-07-05:** Every CloudEvent a platform component emits MUST fall under exactly one of the closed set of ten top-level namespaces; introducing a new top-level namespace is a breaking change requiring a new ADR (ADR 0031). (§6.7, lines ~606–620)
- **REQ-V6-07-06:** Event schemas MUST live in the B12 Git registry; every event MUST carry CloudEvents `specversion` plus a per-event-type `schemaVersion`; backward-compatible additions bump minor and breaking changes mint a new event type (ADR 0030/0031). (§6.7, line ~621)
- **REQ-V6-07-07:** The two initial v1.0 trigger flows (AlertManager → HolmesGPT; budget-exceeded → email user) MUST ship to exercise the path end-to-end; each subsequent component MUST design its own trigger flows as a standard deliverable. (§6.7, lines ~623–628)
- **REQ-V6-07-08:** Mattermost MUST integrate as a bidirectional surface via Knative Eventing using the same broker + Trigger machinery, with OPA-driven Trigger filtering and a generic chat-platform driver pattern; only Mattermost Team Edition ships in v1.0 (ADR 0036). (§6.7, line ~630)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** Triggers are policy-checked; OPA-driven Trigger filtering controls which events reach which sink (including Mattermost channels); cross-tenant publish events ride `platform.tenant.*`.
- **Observability (§6.5):** broker event arrival and dispatch outcome are an enumerated audit point (§6.6); events flow through Knative's audit layer.
- **Scale / portability:** same broker (NATS JetStream) in dev and prod removes environment skew; environment-specific sources isolate substrate differences at the edge.
- **Versioning (ADR 0030/0031):** event versioning via `schemaVersion`; new event types instead of breaking subscribers; new namespaces require a new ADR.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A4, B8, B12).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-07-01:** The broker is Knative-on-NATS-JetStream and is the same in kind and AWS. (→ REQ-01)
- **AC-V6-07-02:** On AWS an AwsSqsSource feeds the broker and on kind a webhook receiver does; no Crossplane Composition wraps the source kinds. (→ REQ-02)
- **AC-V6-07-03:** Changing a Trigger filter (GitOps) reroutes events without touching adapter code; the filter change is audited. (→ REQ-03)
- **AC-V6-07-04:** The CloudEvent→`AgentRun` adapter and the CloudEvent→Argo Workflow adapter contain no branching/decision logic (field-mapping only). (→ REQ-04)
- **AC-V6-07-05:** A conformance check confirms every emitted event type maps to exactly one of the ten namespaces; an attempt to emit outside the set fails the check. (→ REQ-05)
- **AC-V6-07-06:** Each event type has a schema in B12's registry and carries `specversion` + `schemaVersion`; a backward-compatible change bumps minor and a breaking change appears as a new event type. (→ REQ-06)
- **AC-V6-07-07:** An AlertManager alert reaches HolmesGPT via a Trigger, and a budget-exceeded LiteLLM callback CloudEvent reaches the email-notification adapter via a Trigger. (→ REQ-07)
- **AC-V6-07-08:** A Mattermost message flows in and a platform event flows out through the same broker/Trigger machinery, with OPA deciding channel routing; Team Edition is the deployed tier. (→ REQ-08)

## 10. Risks & Open Questions
- **OQ-1 (med):** Per-event-type names and schemas are deferred to B12; this view fixes only the namespace taxonomy and versioning rules (`[PROPOSED — not in source]` for concrete schemas).
- **OQ-2 (low):** Mattermost Team Edition lacks native OIDC/SAML and is authenticated via Keycloak GitLab-OAuth-spoof protocol mappers; the auth detail is owned by the identity view (V6-11 / §9).
- **R-1 (med):** Putting decision logic into an adapter would bypass Knative's audit/policy layer; mitigated by REQ-04 "no logic in adapters" conformance.
- **R-2 (low):** Adding an eleventh top-level namespace to fit a new event would be a silent breaking change; mitigated by REQ-05 (new namespace requires a new ADR).

## 11. References
- architecture-overview.md §6.7 Eventing architecture (lines ~553–630); §6.6 (broker as an enumerated audit point, line ~520); §6.5 (observability alert routing); interface-contract.md §2 (CloudEvent taxonomy).
- ADRs: 0004 (NATS JetStream broker), 0023 (broker in architecture / environment-specific sources), 0030 (versioning), 0031 (CloudEvent taxonomy), 0036 (Mattermost via Knative).
- Realizing components: A4 (Knative + NATS JetStream), B8 (adapter services), B12 (schema registry); flows touch A14 (HolmesGPT), A3 (Argo Workflows), A5 (`AgentRun`), A19 (Mattermost adapter).
- Related views: V6-02, V6-05, V6-06, V6-10, V6-11.

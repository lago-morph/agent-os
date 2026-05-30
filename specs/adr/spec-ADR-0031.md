# SPEC ADR-0031 — CloudEvent top-level type taxonomy `[PROPOSED]`

> kind: ADR · workstream: — · tier: T0
> upstream: [A4] · downstream: [B12; B8; B19; A18; A19; A20; A23] · adrs: [0031] · views: [6.7]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0031 is a settled decision: the platform commits to a **closed set of ten top-level
CloudEvent type namespaces** as an architectural invariant. Every CloudEvent any platform
component emits MUST fall under exactly one of them. This SPEC states what honoring that
decision requires of producing and consuming components, the schema-registry, and the
Knative Trigger layer — it does not re-argue the taxonomy.

The problem the decision solves: without a fixed namespace set, each component would coin
ad-hoc top-level types, Knative Triggers would have to match arbitrary patterns, and
audit-category alignment with the durable audit pipeline (ADR 0034) would break. The
invariant makes declarative routing, fan-out, and external subscription tractable.

## 2. Scope

### 2.1 In scope
- The closed set of ten top-level namespaces as a conformance contract on every emitter.
- The `specversion` + per-event-type `schemaVersion` carriage requirement (ADR 0030).
- Trigger-filtering contract: subscribers match on namespace prefix.
- The amendment rule: a new top-level namespace is a breaking change requiring a new ADR.

### 2.2 Out of scope (and where it lives instead)
- Per-event-type **names and schemas** under each namespace — deferred to component B12
  (CloudEvent schema registry) per architecture-backlog §4.
- Broker / Trigger mechanics — owned by A4 (Knative Eventing + NATS JetStream) and B8
  (Knative event adapter services).
- Concrete trigger flow designs (e.g. AlertManager → HolmesGPT) — owned by the shipping
  components (A14, observability flows).

## 3. Context & Dependencies

Upstream consumed: A4 (Knative Eventing + NATS JetStream broker) supplies the transport on
which CloudEvent `type` is the routing key. Downstream consumers: B12 registers concrete
per-event-type schemas under these namespaces; B8 normalizes inbound payloads into them;
B19, A18, A19, A20, A23 emit/consume events bound to specific namespaces.

ADR decisions honored:
- **ADR 0031** (this): exactly-one-of-ten namespace membership; new namespace ⇒ new ADR.
- **ADR 0030**: every event carries `specversion` + `schemaVersion`; breaking change mints a
  new event type rather than breaking subscribers.
- **ADR 0004**: NATS JetStream is the broker backend; routing is by CloudEvent `type`.
- **ADR 0034**: `platform.audit.*` is the namespace the audit adapter consumes.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — this ADR imposes no CRD/XRD. (Schema registry artifacts are B12-owned, not CRDs here.)

### 4.2 APIs / SDK surfaces
The B12 schema registry is the single source of truth for concrete schemas. OTel emission
surface on the Platform SDK (B6) tags events with a `type` under one of the ten namespaces.
Specific method signatures are **not specified in source**.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
The closed set of ten top-level namespaces (verbatim Canon):

- `platform.lifecycle.*` — Agent, AgentRun, Sandbox, Workflow, MemoryStore lifecycle.
- `platform.audit.*` — audit-emission events; consumed by the audit adapter (ADR 0034).
- `platform.gateway.*` — LiteLLM non-audit events (routing, failover, MCP health, A2A handoffs).
- `platform.policy.*` — OPA decisions, policy violations, dynamic registration accept/deny.
- `platform.capability.*` — capability-registry changes (`platform.capability.changed`, ADR 0013).
- `platform.evaluation.*` — evaluation runs, A/B results, red-team findings.
- `platform.approval.*` — approval requested / OPA-elevated / decided / timed out.
- `platform.observability.*` — threshold crossings, alert routing (e.g. budget-exceeded).
- `platform.tenant.*` — tenant onboarding, namespace association changes, cross-tenant publish.
- `platform.security.*` — security events distinct from audit (sandbox-escape, authn failures).

Carriage: CloudEvents-native `specversion` + per-event-type `schemaVersion`. Backward-compatible
additions bump minor; breaking changes mint a new event type.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface. Per-event-type data schemas are deferred to B12.

## 5. OSS-vs-Custom Decision
N/A — ADR (no component build decision; taxonomy is an invariant). Realizing components make
their own OSS-vs-custom calls in their own SPECs.

## 6. Functional Requirements

- REQ-ADR-0031-01: Every CloudEvent emitted by any platform component MUST carry a `type`
  whose top-level namespace is exactly one of the ten Canon namespaces in §4.3.
- REQ-ADR-0031-02: No platform component MUST emit a CloudEvent under a top-level namespace
  not in §4.3; introducing one MUST be gated behind a new ADR + architecture amendment.
- REQ-ADR-0031-03: Every CloudEvent MUST carry CloudEvents-native `specversion` and a
  per-event-type `schemaVersion` (ADR 0030).
- REQ-ADR-0031-04: A backward-compatible schema change MUST bump the `schemaVersion` minor; a
  breaking change MUST mint a new event type rather than mutate an existing one.
- REQ-ADR-0031-05: Concrete per-event-type schemas MUST be registered in B12's schema registry;
  emitters MUST NOT publish an event type absent from the registry.
- REQ-ADR-0031-06: Knative Triggers MUST be able to select a consumption set by top-level
  namespace prefix alone, without coupling to component-internal event names.
- REQ-ADR-0031-07: `platform.audit.*` MUST be the sole namespace consumed by the audit adapter
  path (ADR 0034); `platform.security.*` MUST remain distinct from `platform.audit.*`.

## 7. Non-Functional Requirements
- Security/multi-tenancy: namespace membership is a reviewable contract enabling per-category
  audit alignment; cross-tenant publish events fall under `platform.tenant.*`.
- Observability (§6.5): namespace prefix is the stable subscription key for sinks/dashboards.
- Versioning (ADR 0030): `specversion` + `schemaVersion` on every event; new-type-on-breaking.
- Scale: namespace-prefix matching keeps Trigger filters O(prefix) regardless of event-type count.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. (The realizing components — B12, B8, emitters — carry the §14.1 deliverables in
their own SPECs; conformance to this taxonomy is verified per §9.)

## 9. Acceptance Criteria

- AC-ADR-0031-01: Honored when a registry/CI conformance check rejects any registered event
  type whose top-level namespace is not one of the ten. (→ REQ-01, REQ-02, REQ-05)
- AC-ADR-0031-02: Honored when every event sampled from the broker carries both `specversion`
  and `schemaVersion`. (→ REQ-03)
- AC-ADR-0031-03: Honored when a breaking schema change is shown to produce a new event type,
  and a backward-compatible change is shown to bump only the minor `schemaVersion`. (→ REQ-04)
- AC-ADR-0031-04: Honored when a Trigger filtering on `platform.audit.*` receives all audit
  events and zero `platform.security.*` events in a mixed-emission test. (→ REQ-06, REQ-07)
- AC-ADR-0031-05: Honored when an attempt to register/emit a new top-level namespace without a
  corresponding ADR fails the architecture-conformance gate. (→ REQ-02)

## 10. Risks & Open Questions
- (low) Emitters could attach a correct namespace prefix but a malformed sub-name; mitigated by
  REQ-05 registry gating. Sub-name grammar is B12's concern.
- (med) `[PROPOSED]` — the exact CI/registry mechanism that enforces the closed set (conftest
  rule vs. registry admission) is not specified in source; flagged for B12 design.
- Open: which namespace a chatops invocation lands under — ADR 0036 says "likely
  `platform.lifecycle.*`"; final binding is design-time per the emitting component.

## 11. References
- ADR 0031 (this decision). Enforcing/realizing components: B12 (schema registry), B8 (event
  adapters), A4 (broker), A18 (`platform.audit.*` consumer), A19/A20/A23/B19 (emitters).
- architecture-overview.md §6.7 (eventing). architecture-backlog.md §4 (deferred schemas), §6
  (invariant). ADR 0004, ADR 0030, ADR 0034.

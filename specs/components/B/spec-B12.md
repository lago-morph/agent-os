# SPEC B12 — CloudEvent schema registry

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [A4, B8] · downstream: [B19] · adrs: [0031, 0030, 0004, 0013, 0034, 0023] · views: [6.7, 6.13]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B12 is the **CloudEvent schema registry**: the Git-versioned set of **JSON schemas** for the
platform's CloudEvent types plus the tooling that validates, versions, and publishes them. Every
platform event flows through Knative Eventing as a CloudEvent on the NATS JetStream broker (A4,
ADR 0004) under the **closed set of ten top-level `platform.*` namespaces** fixed by ADR 0031 / §6.7.
Per-event-type names and schemas *within* each namespace are design-time per component and were
**deferred** at architecture level — B12 is the single source of truth where those concrete schemas
land as components author them (§6.7: "We maintain a JSON schema registry in Git for our event
types (component B12)").

Without a registry, each emitting component (LiteLLM callbacks, the kopf operator, ARK adapters, the
approval system, etc.) would invent ad-hoc payload shapes, breaking declarative Knative Trigger
filtering, external subscription, and the audit-pipeline alignment (ADR 0034). B12 provides the
namespace-conformance guardrail (every event under exactly one Canon namespace), the dual versioning
contract (`specversion` + per-event-type `schemaVersion`), and the validation/publishing tooling. It
is T0 because the approval system (B19) and every event-emitting component bind to its schemas and
conformance rules.

## 2. Scope

### 2.1 In scope
- The **Git-versioned JSON schema registry** holding per-event-type schemas under the ten Canon
  `platform.*` namespaces (§6.7, ADR 0031).
- **Namespace-conformance tooling**: validate that every registered event type falls under **exactly
  one** Canon top-level namespace; reject ad-hoc or new top-level namespaces (those require an ADR).
- **Event-versioning enforcement** (ADR 0030 / 0031): every schema carries CloudEvents-native
  `specversion` plus a per-event-type **`schemaVersion`**; backward-compatible additions bump minor,
  breaking changes **mint a new event type** rather than break subscribers — enforced by tooling.
- **Schema validation / contribution tooling**: a component drops its event schema in the registry;
  CI validates structure, namespace membership, versioning rules, and (where applicable) compatibility
  with the prior version.
- **Publishing / consumption surface**: schemas published so emitters validate before publish and
  external subscribers can subscribe declaratively against the namespace contract.
- **The `platform.capability.changed` schema** (Canon event name, ADR 0013) as a registry entry the
  platform SDK (B6) and B7 deserialize.
- Schemas for the **two initial v1.0 trigger flows** as exemplars (AlertManager → HolmesGPT;
  budget-exceeded → email user), under `platform.security.*`/`platform.observability.*` as applicable.

### 2.2 Out of scope (and where it lives instead)
- **The top-level namespace set itself** — fixed by **ADR 0031**; B12 enforces, does not define;
  adding a namespace needs a new ADR.
- **Knative Eventing + NATS JetStream broker install** → **A4**; **Knative event adapter services**
  (CloudEvent → AgentRun / → Workflow) → **B8** (consume schemas, map fields).
- **Environment-specific Knative sources** (AwsSqsSource, webhook receivers) → ADR 0023 / per-env;
  B12 schematizes the CloudEvents, not the source kinds.
- **Authoring each component's event *content*** — each component owns its event types; B12 owns the
  registry, conformance, and versioning rules they land under.
- **Audit pipeline storage/retention** → A18 / ADR 0034 / F1; B12 only schematizes `platform.audit.*`.
- **CRD/XRD and SDK versioning** → ADR 0030 owners (B13, A5, A6, B4, B6); B12 owns only event-schema
  versioning.

## 3. Context & Dependencies

**Upstream consumed:**
- **A4 (Knative Eventing + NATS JetStream broker)** — the transport carrying the CloudEvents B12
  schematizes; B12's schemas describe payloads flowing on this broker.
- **B8 (Knative event adapter services)** — the adapters that consume CloudEvents and map fields;
  B12's schemas are the contract the adapters validate against.

**Downstream consumers:**
- **B19** (generalized approval system) — `platform.approval.*` schemas (requested / OPA-elevated /
  decided / timed out) live here.
- Every event-emitting component (B2 LiteLLM callbacks → `platform.gateway.*`/`platform.observability.*`;
  B13 kopf → `platform.capability.*`; A18 → `platform.audit.*`; ARK → `platform.lifecycle.*`; A20/B3
  decision points → `platform.policy.*`; B6/B7 consume `platform.capability.changed`).

**ADRs honored:**
- **ADR 0031** — closed set of ten top-level namespaces; every event under exactly one; per-event-type
  schemas live in B12; a new top-level namespace is a breaking change requiring a new ADR.
- **ADR 0030** — `specversion` + `schemaVersion`; additive=minor, breaking=new event type.
- **ADR 0004** — events ride the NATS JetStream broker behind Knative.
- **ADR 0013** — `platform.capability.changed` is the specific capability-change event (registry entry).
- **ADR 0034** — `platform.audit.*` schemas align with the durable audit pipeline (Postgres + S3
  system of record; OpenSearch advisory).
- **ADR 0023** — environment-specific sources are exceptions; B12 schematizes events, not source kinds.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B12 defines no CRD. Its artifacts are **JSON schemas in Git**, not Kubernetes resources. It
references CRD-derived event payloads (e.g. `platform.capability.changed` carrying affected
Agent/CapabilitySet identifiers per ADR 0013) but owns no CRD.

### 4.2 APIs / SDK surfaces
- **Registry layout / schema-file convention** (the primary interface). The ten Canon namespaces
  (`platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`,
  `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`,
  `platform.tenant.*`, `platform.security.*`) are Canon; the **directory/file layout, schema-id
  convention, and registry index format are `[PROPOSED — not in source]`.**
- **Schema envelope fields**: CloudEvents-native **`specversion`** and per-event-type **`schemaVersion`**
  are Canon (§6.13, ADR 0030/0031). Any additional envelope fields are `[PROPOSED — not in source]`.
- **Validation/contribution tooling CLI/CI gate**: validates namespace membership, versioning rules,
  and backward-compatibility. Tooling command names are `[PROPOSED — not in source]`.
- **Per-event-type names**: explicitly **deferred to component design** (§6.7, ADR 0031). B12 does NOT
  invent event-type names; it hosts those the owning components author. The only Canon-named event
  types are **`platform.capability.changed`** (ADR 0013) and the two trigger-flow exemplars described
  by behavior (budget-exceeded notification; AlertManager → HolmesGPT) — their concrete event-type
  *names* beyond `platform.capability.changed` are `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- B12 itself is a **registry**, not a runtime emitter: it emits **no** CloudEvents of its own.
- It is the **schema authority** for all ten Canon namespaces. It **enforces** that every contributed
  event type falls under exactly one Canon namespace and carries `specversion` + `schemaVersion`.
- **`platform.capability.changed`** (Canon) is a hosted schema consumed by B6/B7 (ADR 0013).
- `[PROPOSED — not in source]` whether B12 publishes a `platform.*` event on registry change (e.g. a
  "schema-registered" event) — defaulting to **no self-emission** to avoid coining an event type.

### 4.4 Data schemas / connection-secret contracts
N/A — no datastore, no connection secret. B12's "schemas" are JSON-schema files in Git; the registry
is reconciled like other GitOps artifacts. It owns no substrate primitive (ADR 0041).

## 5. OSS-vs-Custom Decision
**Build-new tooling over an OSS schema standard.** Upstream: **CloudEvents** (the event format,
inherent via Knative/A4) and **JSON Schema** (the schema language). B12 **builds-new** the registry
layout, the namespace-conformance + versioning validators, and the CI/publish tooling; it **wraps**
JSON-Schema validation rather than forking it. No suitable OSS registry enforces the platform's
closed-namespace + `schemaVersion` mint-on-break invariants out of the box. Python is the default
implementation language (ADR 0006 rationale). Pinned JSON-Schema/CloudEvents tooling versions
`[PROPOSED — not in source]`. ADR linkage: 0031 (namespaces), 0030 (versioning), 0004 (broker).

## 6. Functional Requirements
- **REQ-B12-01:** B12 MUST host a **Git-versioned JSON schema registry** of per-event-type schemas
  organized under the ten Canon `platform.*` top-level namespaces (§6.7, ADR 0031).
- **REQ-B12-02:** Tooling MUST validate that **every registered event type falls under exactly one**
  Canon top-level namespace and MUST reject any event type outside the closed set (new namespace ⇒
  requires an ADR, not a registry change).
- **REQ-B12-03:** Every schema MUST carry CloudEvents-native **`specversion`** and a per-event-type
  **`schemaVersion`** (ADR 0030 / 0031).
- **REQ-B12-04:** Versioning tooling MUST enforce that **backward-compatible additions bump the minor**
  `schemaVersion` and **breaking changes mint a new event type** rather than mutating a published one.
- **REQ-B12-05:** B12 MUST provide **contribution tooling / a CI gate** so a component can drop its
  event schema and have namespace membership, versioning rules, and backward-compatibility validated
  automatically.
- **REQ-B12-06:** B12 MUST **publish** schemas so emitters can validate before publish and external
  subscribers can subscribe declaratively against the namespace contract.
- **REQ-B12-07:** B12 MUST host the **`platform.capability.changed`** schema (ADR 0013) consumable by
  the platform SDK (B6) and the agent harness (B7).
- **REQ-B12-08:** B12 MUST host schemas for the **two initial v1.0 trigger-flow** events
  (budget-exceeded notification under `platform.observability.*`; AlertManager → HolmesGPT routing
  under `platform.security.*` or `platform.observability.*` as applicable) as exemplars.
- **REQ-B12-09:** B12 MUST NOT invent **per-event-type names** for other components; it hosts what
  owning components author and only enforces the conformance + versioning contract.

## 7. Non-Functional Requirements
- **Security:** the registry is GitOps-reconciled and review-gated; schema changes that would break
  subscribers are blocked by the mint-on-break rule (REQ-B12-04); `platform.security.*` schemas are
  first-class (distinct from audit, ADR 0031).
- **Multi-tenancy (§6.9):** schemas carry tenant/namespace identifiers where the event is tenant-scoped
  (e.g. `platform.tenant.*`); the registry itself is platform-global, not per-tenant.
- **Observability (§6.5):** the registry is the contract enabling Trigger filtering by namespace and
  audit-category alignment; schema-validation failures are visible in CI.
- **Scale:** schema validation is build-time/publish-time, off the event hot path; emitters validate
  against cached published schemas, not the registry per event.
- **Versioning (ADR 0030):** the registry's own change process follows additive-minor / new-type-on-break;
  introducing a top-level namespace is a breaking platform change requiring a new ADR.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable (minimal)** — GitOps reconciliation of the published registry; no
  long-running workload of its own. `[PROPOSED — not in source]` for a publish/serving manifest.
- Per-product docs (10.5) — **applicable** (registry layout, how-to-add-an-event-schema, versioning
  rules, namespace catalog).
- Runbook (10.7) — **applicable** (schema-validation-failure / breaking-change remediation).
- Alerts — **N/A** — no runtime service emitting alerts; CI surfaces validation failures.
- Grafana dashboard (Crossplane XR) — **N/A** — no runtime metrics owned here.
- Headlamp plugin — **N/A** — no operator UI surface (a schema browser, if any, is not committed).
- OPA/Rego integration — **N/A** — the registry is schema tooling, not a decision point. `[PROPOSED]`
  optional admission/CI policy that emitters reference only registered types.
- Audit emission (ADR 0034) — **N/A (defines schemas only)** — B12 schematizes `platform.audit.*`;
  emission is by the audit adapter (A18).
- Knative trigger flow — **applicable (defines contracts)** — B12 schematizes the events Triggers
  filter on, including the two v1.0 exemplar flows; B12 emits none itself.
- HolmesGPT toolset — **N/A** — registry, not a self-management surface.
- 3-layer tests — **applicable** (PyTest for validators/versioning tooling; Chainsaw N/A — no CRD;
  Playwright N/A — no UI).
- Tutorials & how-tos — **applicable** (how-to-add-an-event-schema).

## 9. Acceptance Criteria
- **AC-B12-01:** The registry contains JSON schemas organized under the ten Canon namespaces; a sample
  schema loads and validates. (→ REQ-B12-01)
- **AC-B12-02:** A schema declaring a non-Canon top-level namespace is rejected by tooling/CI. (→ REQ-B12-02)
- **AC-B12-03:** A schema missing `specversion` or `schemaVersion` fails validation. (→ REQ-B12-03)
- **AC-B12-04:** A backward-compatible change that bumps minor passes; a breaking change to a published
  type fails unless a new event type is minted. (→ REQ-B12-04)
- **AC-B12-05:** A contributor drops a new event schema and CI validates namespace, versioning, and
  compatibility without registry-framework edits. (→ REQ-B12-05)
- **AC-B12-06:** A published schema is retrievable by an emitter for pre-publish validation and by an
  external subscriber. (→ REQ-B12-06)
- **AC-B12-07:** The `platform.capability.changed` schema exists and validates a sample event carrying
  affected Agent/CapabilitySet identifiers. (→ REQ-B12-07)
- **AC-B12-08:** Schemas for the budget-exceeded and AlertManager→HolmesGPT exemplar flow events exist
  under Canon namespaces. (→ REQ-B12-08)
- **AC-B12-09:** A review confirms B12 hosts no invented per-event-type names beyond Canon/exemplars;
  contributed types are owned by their components. (→ REQ-B12-09)

## 10. Risks & Open Questions
- **R1 (high):** Registry layout, schema-id convention, and tooling command names are `[PROPOSED — not
  in source]`. B19 and every emitter bind to them; late change is a cross-component break. Mitigation:
  B12 freezes layout + conformance contract first; downstreams bind. Blast radius: high.
- **R2 (med):** Per-event-type names are deferred (ADR 0031); risk that B12 over-reaches and coins names
  owned by other components (REQ-B12-09). Mitigation: B12 hosts + enforces only; names land with their
  owning component's spec.
- **R3 (med):** The `platform.capability.changed` payload field set is `[PROPOSED]` beyond "affected
  Agent / CapabilitySet identifiers" (ADR 0013). B6/B7 deserialize it; misalignment breaks the
  notification client. Mitigation: B12 freezes this schema early as the B6/B7 contract.
- **R4 (med):** Exemplar trigger-flow event-type *names* are `[PROPOSED]`; must not preempt the owning
  components (LiteLLM callbacks / HolmesGPT flow). Mitigation: flag as exemplars, coordinate names with
  B2/A14.
- **R5 (low):** Backward-compat detection for JSON Schema is non-trivial; mitigated by a conservative
  validator that errs toward requiring a new event type on ambiguity (ADR 0030 intent).
- **OQ1:** Is the registry served at runtime (a schema endpoint) or consumed purely from Git at build
  time? §6.7 says "in Git"; runtime serving is unstated. `[PROPOSED]` Git-first, optional published
  artifact for external subscribers.
- **OQ2:** Does registry change emit a `platform.*` event (e.g. schema-registered)? `[PROPOSED]` no
  self-emission to avoid coining a type; revisit if a consumer needs it.

## 11. References
- architecture-overview.md §6.7 (lines 604–621; CloudEvent standardization, B12 registry in Git, the
  ten top-level namespaces, `schemaVersion` per §6.13; lines 623–628 two initial trigger flows),
  §6.13 (line 988; CloudEvent schema versioning, `specversion` + `schemaVersion`, mint-on-break).
- interface-contract.md §2 (CloudEvent taxonomy; per-event-type names deferred, schemas in B12;
  `platform.capability.changed`; event versioning), §6 (gaps: per-event-type names/schemas → B12).
- ADR 0031 (top-level taxonomy), ADR 0030 (versioning), ADR 0004 (NATS JetStream broker), ADR 0013
  (`platform.capability.changed`), ADR 0034 (audit pipeline alignment), ADR 0023 (env-specific sources).
- Related pieces: A4 (Knative+NATS), B8 (event adapters), B19 (approval events), B2/B13/A18/A20/ARK
  (emitters), B6/B7 (capability-change consumers).

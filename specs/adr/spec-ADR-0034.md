# SPEC ADR-0034 — Audit pipeline — durable adapter, Postgres + S3 system of record, OpenSearch advisory only `[PROPOSED]`

> kind: ADR · workstream: — · tier: T0
> upstream: [B4; A11; A7] · downstream: [A18; B4; A23; B12] · adrs: [0034] · views: [6.5]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0034 is a settled decision: audit emission is mediated by a single platform **audit adapter**
(a Python library) that every component links against; **no component writes audit records
directly** to Postgres, S3, or OpenSearch. The adapter posts to a single deployable **audit
endpoint**; the **system of record is Postgres + S3**; **OpenSearch is advisory fanout only**. The
whole pipeline is provisioned by the `AuditLog` XRD. This SPEC states the interface contract that
honoring the decision imposes on every emitting component and on the pipeline — it does not
re-argue the topology.

The problem the decision solves: §6.5 commits to broad audit emission from every component, and the
reproducibility invariant requires that anything in OpenSearch be rebuildable from a primary source.
A single mediated emission path removes storage-specific code from every component, makes Postgres+S3
the authoritative store, and keeps OpenSearch a rebuildable index that can fail without losing audit.

## 2. Scope

### 2.1 In scope
- The single-adapter emission contract: every component emits audit **only** through the adapter library to the audit endpoint.
- The audit-adapter interface and `audit_events` schema are frozen (D-05); a freeze-gate MUST pass before any component emits audit events.
- The system-of-record topology: in-flight rows in Postgres `audit_events`; AWS batch CronJob → immutable verified S3 object → only-then Postgres delete; kind = Postgres-only, batch disabled.
- OpenSearch as advisory async fanout: rebuildable, never authoritative, non-blocking to ingestion.
- The `AuditLog` XRD as the provisioner of the whole pipeline (endpoint Deployment, Postgres, S3+lifecycle (AWS), indexer, batch CronJob (AWS)).
- Audit events carried under the `platform.audit.*` CloudEvent namespace (ADR 0031).
- The audit endpoint owning its own `LogLevel` toggle (ADR 0035).

### 2.2 Out of scope (and where it lives instead)
- Audit **retention durations, redaction rules, lifecycle specifics** — deferred to Workstream F (component F1) per architecture-backlog §1.13.
- Per-event-type audit **schemas** under `platform.audit.*` — owned by component B12 (schema registry).
- The substrate XRDs (`Postgres`, `ObjectStore`) the `AuditLog` XRD composes — owned by ADR 0044 / B4.
- The dynamic-toggle mechanism itself — owned by ADR 0035.
- The audit-endpoint component build and adapter-library build — owned by component A18.

## 3. Context & Dependencies

Upstream consumed: B4 (Crossplane Compositions) supplies `Postgres` + `ObjectStore` that the
`AuditLog` XRD composes; A11 (OpenSearch) is the advisory fanout target; A7 (OPA/Gatekeeper) admits
the XR only when a Composition matches the cluster's substrate label (ADR 0044). Downstream
consumers: A18 (builds the adapter library + audit endpoint), B4 (owns the `AuditLog` Composition),
A23 (Kargo promotes the `AuditLog` XR uniformly across substrates), B12 (registers `platform.audit.*`
event schemas). Every platform component is an emitter that MUST link the adapter.

ADR decisions honored:
- **ADR 0034** (this): single adapter; no direct writes; Postgres+S3 SoR; OpenSearch advisory; `AuditLog` XRD provisions the pipeline.
- **ADR 0031**: audit events carry a `type` under `platform.audit.*`.
- **ADR 0035**: the audit endpoint owns its own dynamic `LogLevel` toggle.
- **ADR 0044**: `AuditLog` is a canonical substrate-abstraction instance composing `Postgres` + `ObjectStore`; kind may produce "no archive" (graceful degradation).
- **ADR 0040**: Kargo promotes the `AuditLog` XR uniformly across substrates.
- **ADR 0014 / ADR 0033**: Postgres dual-mode (RDS on AWS, in-cluster on kind); kind functionally complete without cloud services.
- **ADR 0003**: audit-endpoint egress to S3 and managed OpenSearch goes through the Envoy egress proxy.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- **`AuditLog` (XRD)**, namespaced. Source-stated fields: `postgresRef`, `s3BucketRef`, `indexerRef`,
  `batchScheduleSpec`, `endpointReplicas`. (XRD kind: `AuditLog`, created as a namespaced XR directly in the tenant namespace per Crossplane v2.) One XR provisions: Postgres
  backing store, S3 bucket + lifecycle (AWS only), OpenSearch indexer config, the ~5-minute batch
  CronJob (AWS only), and the audit endpoint Deployment. Composes `Postgres` (system of record on
  kind and AWS) and `ObjectStore` (S3 archive on AWS; MinIO or no-op on kind). XRD versioning
  follows ADR 0030 (conversion webhooks + deprecation windows on both Compositions, per ADR 0044).

### 4.2 APIs / SDK surfaces
- **Audit adapter — a Python library** every component links against (architecture-backlog §6: all
  custom code is Python). It is the **sole** path to audit emission; it posts to the single audit
  endpoint. No component talks to Postgres / S3 / OpenSearch for audit directly.
- **Audit endpoint** — a single deployable component (a Deployment). It writes received events into
  the Postgres `audit_events` table and owns its own `LogLevel` toggle (ADR 0035) so verbosity rises
  per-tenant / per-event-class without redeploying callers (callers only know the endpoint).
- HTTP API of the audit endpoint follows URL-path versioning (`/v1/...`) per ADR 0030 §3.3; concrete
  method signatures and the adapter's per-method surface are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Audit events flow under **`platform.audit.*`** — the sole namespace the audit adapter path consumes.
  `platform.security.*` is deliberately distinct (security signals are not audit emission). Per-event-type
  names/schemas under `platform.audit.*` are deferred to B12. Every event carries `specversion` +
  `schemaVersion` (ADR 0030).

### 4.4 Data schemas / connection-secret contracts
- **Postgres `audit_events` table** holds in-flight rows (the authoritative store on kind; the
  in-flight buffer on AWS). Exact column set is **not specified in source** `[PROPOSED — not in source]`.
- **AWS durability cycle**: batch CronJob (~5 min) aggregates pending rows → writes an immutable S3
  object → verifies the write (object exists, byte count, checksum) → **only then** deletes the
  corresponding Postgres rows. On failure, rows stay in Postgres and the batch retries (no data loss).
- **kind**: S3 not provisioned; Postgres alone is the system of record; the batch step is disabled.
- **OpenSearch** receives events via an async indexer; the audit index is rebuildable from S3 (AWS)
  or Postgres (kind); OpenSearch is never authoritative; if down, audit ingestion still succeeds.
- The `AuditLog` XRD's composed Postgres / S3 backends present the ADR 0044 uniform connection-secret
  shape (`host`, `port`, `user`, `password`, `dbname` or equivalent) to the endpoint.

## 5. OSS-vs-Custom Decision
N/A — ADR (topology decision). The realizing component A18 builds custom Python (adapter library +
audit endpoint); the pipeline is provisioned by the `AuditLog` Composition (B4) over Postgres
(CloudNativePG on kind / RDS on AWS), S3 (or MinIO/no-op on kind), and OpenSearch (A11). Those
OSS-vs-custom calls live in A18 / B4 SPECs.

## 6. Functional Requirements

- REQ-ADR-0034-01: Every platform component that emits audit MUST do so **only** through the audit adapter library; no component MUST write audit records directly to Postgres, S3, or OpenSearch.
- REQ-ADR-0034-02: The audit adapter MUST post to a single deployable audit endpoint; callers MUST NOT know about the backing stores.
- REQ-ADR-0034-03: The audit endpoint MUST write received events into the Postgres `audit_events` table; in-flight rows live in Postgres.
- REQ-ADR-0034-04: On AWS, a ~5-minute batch CronJob MUST aggregate pending rows, write an immutable S3 object, verify it (exists + byte count + checksum), and ONLY THEN delete the corresponding Postgres rows.
- REQ-ADR-0034-05: If the S3 write or verification fails, the batch MUST leave the rows in Postgres and retry — no audit data is lost.
- REQ-ADR-0034-06: On kind, S3 MUST NOT be required: Postgres alone is the system of record and the batch step MUST be disabled (the pipeline degrades gracefully — "no archive").
- REQ-ADR-0034-07: OpenSearch MUST be advisory fanout only: an async indexer; the audit index MUST be rebuildable from S3 (AWS) or Postgres (kind); OpenSearch MUST never be the system of record; if OpenSearch is unavailable, audit ingestion MUST still succeed.
- REQ-ADR-0034-08: Audit events MUST carry a CloudEvent `type` under `platform.audit.*`, distinct from `platform.security.*` (ADR 0031).
- REQ-ADR-0034-09: The `AuditLog` XRD MUST provision the whole pipeline from one XR: Postgres store, S3 bucket + lifecycle (AWS only), OpenSearch indexer, batch CronJob (AWS only), and the audit endpoint Deployment.
- REQ-ADR-0034-10: The audit endpoint MUST own its own `LogLevel` toggle (ADR 0035) so verbosity rises per-tenant / per-event-class without redeploying callers.
- REQ-ADR-0034-11: Audit-endpoint egress to S3 and AWS-managed OpenSearch MUST go through the Envoy egress proxy (ADR 0003); the endpoint is not a special case.
- REQ-ADR-0034-12: The audit-adapter interface and the `audit_events` schema MUST be frozen (D-05); a freeze-gate MUST pass before any component emits audit events, so no emitter ships against an unfrozen contract.

## 7. Non-Functional Requirements
- Reproducibility invariant: OpenSearch contents MUST always be rebuildable from S3 (AWS) or Postgres (kind).
- Failure bounding: OpenSearch down ⇒ ingestion still succeeds; S3 write/verify fails ⇒ rows retained + retried; Postgres down ⇒ endpoint fails closed for that caller (same surface as any Postgres-backed control-plane op).
- Multi-tenancy (§6.9): verbosity is raiseable per-tenant / per-event-class via the endpoint's `LogLevel` toggle without caller redeploy.
- Observability (§6.5): the §6.5 diagram is logical — components emit via adapter → endpoint → Postgres (authoritative) / S3 (archive, AWS) / OpenSearch (advisory).
- Versioning (ADR 0030): `AuditLog` XRD schema changes are versioning events on both Compositions; the endpoint HTTP API uses URL-path versioning; events carry `schemaVersion`.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing component A18 carries the §14.1 deliverables (adapter library, endpoint
Deployment manifests, runbook, audit emission, HolmesGPT toolset, 3-layer tests); B4 carries the
`AuditLog` Composition; conformance to this interface is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0034-01: Honored when a code/CI conformance check shows no platform component imports a Postgres/S3/OpenSearch client for audit — emission flows only through the adapter to the endpoint. (→ REQ-01, REQ-02)
- AC-ADR-0034-02: Honored when an event posted to the endpoint appears as a row in Postgres `audit_events`. (→ REQ-03)
- AC-ADR-0034-03: Honored when, on AWS, the batch produces an immutable S3 object whose existence + byte count + checksum verify, and the corresponding Postgres rows are deleted only after that verification. (→ REQ-04)
- AC-ADR-0034-04: Honored when an injected S3 write/verify failure leaves the rows in Postgres and the next batch retries them with zero loss. (→ REQ-05)
- AC-ADR-0034-05: Honored when, on kind, the pipeline ingests and serves audit with no S3 provisioned and the batch CronJob absent/disabled. (→ REQ-06)
- AC-ADR-0034-06: Honored when OpenSearch is taken down mid-ingest, ingestion still succeeds, and the audit index is shown to rebuild from S3 (AWS) / Postgres (kind). (→ REQ-07)
- AC-ADR-0034-07: Honored when every sampled audit event carries a `type` under `platform.audit.*` and no audit event is emitted under `platform.security.*`. (→ REQ-08)
- AC-ADR-0034-08: Honored when one `AuditLog` claim is shown to provision the endpoint Deployment, Postgres store, and (on AWS) S3 bucket + lifecycle + indexer + batch CronJob. (→ REQ-09)
- AC-ADR-0034-09: Honored when a `LogLevel` change on the audit endpoint raises verbosity per-tenant / per-event-class with no caller redeploy. (→ REQ-10)
- AC-ADR-0034-10: Honored when endpoint egress to S3 / managed OpenSearch is observed traversing the Envoy egress proxy. (→ REQ-11)
- AC-ADR-0034-11: Honored when the audit-adapter interface and `audit_events` schema are marked frozen (D-05) and a CI freeze-gate fails any pipeline in which a component emits audit events before that gate passes. (→ REQ-12)

## 10. Risks & Open Questions
- (high) On AWS the in-flight Postgres `audit_events` table grows unbounded while S3 is unrecoverable; REQ-05 retains data but operators must alert on table growth — alerting threshold is an A18 design detail, not specified here.
- (med) `[PROPOSED]` — the `audit_events` column set and the adapter/endpoint method signatures are not specified in source; flagged for A18 / B12 design.
- (med) A regulatory regime requiring synchronous durable archive (S3 write before ack) reopens the ~5-minute batch cadence; explicitly out of scope (ADR 0034 revisit trigger).
- Open: retention durations / redaction rules — deferred to Workstream F (F1); the S3 lifecycle on the `AuditLog` XRD is the surface they attach to.

## 11. References
- ADR 0034 (this decision). Enforcing/realizing components: A18 (adapter library + audit endpoint), B4 (`AuditLog` Composition), A11 (OpenSearch advisory), A7 (admission), A23 (Kargo claim promotion), B12 (`platform.audit.*` schemas).
- architecture-overview.md §6.5 (observability/audit), §6.3 (data architecture). architecture-backlog.md §1.13 (retention deferred to F), §6 (reproducibility invariant; all custom code Python). ADR 0003, 0009, 0011, 0014, 0021, 0031, 0033, 0035, 0040, 0044.

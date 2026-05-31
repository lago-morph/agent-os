# SPEC A18 — Audit endpoint + audit adapter library

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [A11, A7] · downstream: [] · adrs: [0003, 0009, 0011, 0014, 0021, 0030, 0031, 0033, 0034, 0035, 0040, 0044] · views: [6.5, 6.6, 6.7]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

A18 is the platform's **single audit emission path**. Per ADR 0034, audit emission is mediated by one **Python audit adapter library** that every component links against, posting to one deployable **audit endpoint** Deployment. **No component writes audit records directly** to Postgres, S3, or OpenSearch. This is the load-bearing invariant of the whole audit subsystem: it lets every other component import a library instead of carrying storage-specific code, and it gives the platform exactly one place to enforce the system-of-record contract, the dual-mode (kind/AWS) hosting rule, and the reproducibility invariant (anything in OpenSearch must be rebuildable from a primary source).

The system of record is **Postgres + S3**. In-flight rows live in the Postgres `audit_events` table written by the endpoint. On AWS a ~5-minute batch **CronJob** aggregates pending rows into an immutable S3 object, verifies it (exists + byte count + checksum), and only then deletes the Postgres rows; S3 is the durable archive. On kind, S3 is not provisioned — Postgres alone is the system of record and the batch step is disabled. **OpenSearch is advisory fanout only**: an async indexer ships events for query/dashboards, is rebuildable from S3 (AWS) or Postgres (kind), and is never authoritative — if it is down, audit ingestion still succeeds. The whole pipeline is provisioned by the **`AuditLog`** XRD (Postgres store, S3 bucket + lifecycle on AWS, OpenSearch indexer, batch CronJob on AWS, endpoint Deployment). The endpoint owns its own `LogLevel` toggle (ADR 0035) so verbosity rises per-tenant / per-event-class without redeploying callers. Because **A18 is T0 and every component depends on the adapter interface**, the adapter library's surface (interface-contract §5) is specified here exhaustively.

## 2. Scope

### 2.1 In scope
- **Python audit adapter library** — the single import every component uses to emit audit; posts to the audit endpoint; the only component that may know storage details (and even it only knows the endpoint URL, not the stores).
- **Audit endpoint Deployment** — receives adapter posts, writes the Postgres `audit_events` table, owns its own `LogLevel` toggle (ADR 0035), fans out to the async OpenSearch indexer.
- **Postgres `audit_events` schema + migrations** — the in-flight system-of-record table.
- **OpenSearch indexer service** — async, advisory; rebuildable; non-blocking on audit ingestion.
- **S3 batch CronJob** (AWS only) — ~5-min aggregate → immutable S3 object → verify (exists + byte count + checksum) → delete Postgres rows.
- **`AuditLog` Crossplane composition** — provisions the whole pipeline (Postgres via `Postgres`, S3 via `ObjectStore` on AWS, OpenSearch indexer, batch CronJob on AWS, endpoint Deployment); composes substrate primitives per ADR 0044.
- Standard §14.1 deliverables: Helm/manifests, per-product docs, runbook, alerts, Grafana dashboard (XR), Headlamp plugin (audit inspection), OPA integration (admission of `AuditLog` XR + endpoint access), audit (self-emission), Knative trigger-flow design (consumes `platform.audit.*`), HolmesGPT toolset (audit queries), 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)
- **Audit retention durations, redaction rules, S3 lifecycle specifics, OpenSearch index rollover** → **Workstream F (F1)** / backlog §1.13. A18 ships the *topology and attachment surface*, not the policy.
- **The substrate XRDs themselves** (`Postgres`, `ObjectStore`, `SearchIndex`) and the Crossplane Composition machinery → **B4**. A18 authors the `AuditLog` XRD/composition that *consumes* them (build-order dependency on B4; `[PROPOSED — not in source]` since CSV lists only A11/A7).
- **Per-event-type CloudEvent names + JSON schemas** under `platform.audit.*` → **B12** schema registry. A18 commits to the namespace and the adapter envelope, not per-type schemas.
- **OpenSearch install** → **A11**; **Postgres-on-kind / object store** substrate installs → B4/substrate primitives.
- **The `LogLevel` reconciler mechanism** (declarative toggle, staged restart) → ADR 0035 / its owning component; A18 *consumes* `LogLevel` for the endpoint.
- **OPA engine + framework + content** → A7 / B3 / B16; A18 contributes only its own Rego.
- **Knative broker + Triggers + adapters** → A4 / B8; A18 designs which `platform.audit.*` events it consumes but does not own the broker.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A11 (OpenSearch)** — the search/vector store the advisory indexer targets (in-cluster on kind, AWS-managed on AWS).
- **A7 (OPA / Gatekeeper)** — admission of the `AuditLog` XR and any access policy on the endpoint.

**Effective build-order dependencies (named here, not in CSV):**
- **B4 (Crossplane Compositions)** — owns `Postgres` / `ObjectStore` / `SearchIndex` the `AuditLog` XRD composes. `[PROPOSED — not in source]` hard dependency (waves.md notes B4 owns the XRDs A18 consumes; CSV upstream cell omits it).
- **A4 / B8 (Knative + adapters)** — if `platform.audit.*` events arrive via the broker; A18 consumes them.
- **ADR 0035 `LogLevel`** owner — the toggle the endpoint honors.

**Downstream consumers:** none in CSV, but **every audit-emitting component links the adapter** — per §6.6 the emission points are LiteLLM callbacks, Gatekeeper admission, Envoy egress, Knative broker, ARK, agent-sandbox, ESO, ArgoCD, LibreChat, Headlamp plugin actions, approval workflows, Coach, HolmesGPT, and the B13 kopf operator. A18's adapter is the universal dependency for all of them.

**ADRs honored:**
- **ADR 0034** — durable adapter; Postgres+S3 system of record; OpenSearch advisory; `AuditLog` XRD; endpoint Deployment; failure modes (Postgres down → fail closed; S3 verify fail → retain + retry; OpenSearch down → ingest still succeeds).
- **ADR 0035** — endpoint owns its own dynamic `LogLevel` / trace-granularity toggle.
- **ADR 0014** — Postgres as primary store; same control-plane failure surface.
- **ADR 0009 / 0033** — OpenSearch dual-hosting; kind functionally complete without cloud services.
- **ADR 0021 / 0044** — `AuditLog` is a canonical substrate-abstraction instance composing `Postgres` + `ObjectStore`; explicit kind "no archive" capability-parity caveat.
- **ADR 0003** — endpoint egress to S3 + managed OpenSearch rides the Envoy egress proxy; not a special case.
- **ADR 0011** — test results stream via the same OpenSearch + OTel path through the adapter, not direct.
- **ADR 0031** — `platform.audit.*` namespace. **ADR 0030** — versioning (adapter is a versioned SDK surface; URL-path versioning for the endpoint HTTP API).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
**`AuditLog`** (namespace-scoped XR, Crossplane v2; owner B4-composed, defined/consumed here per ADR 0034). Source-stated fields (interface-contract §1.6): `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec`, `endpointReplicas`. One AuditLog XR provisions: Postgres store (via `Postgres`), S3 bucket + lifecycle (via `ObjectStore`, AWS only), OpenSearch indexer config, 5-min batch CronJob (AWS only), audit endpoint Deployment.
- Composes `Postgres` (system of record, both substrates) and `ObjectStore` (S3 on AWS; MinIO/no-op on kind) per ADR 0034/0044.
- **Capability-parity caveat:** on kind the archive path is **no-op** (no S3, batch disabled) — `ready`/`endpoint`/`version` status remain substrate-agnostic; the absence of an archive is by design.
- `LogLevel` (namespaced; consumed): the endpoint honors a `LogLevel` whose `componentSelector` targets the audit endpoint, with `scope` of component/tenant/eventClass (ADR 0035).
- Versioning per ADR 0030/0044: XR schema changes go through conversion webhooks + ≥1-release deprecation; ownership of the `AuditLog` versioning lifecycle sits with A18 (the component owning the pipeline).

### 4.2 APIs / SDK surfaces — **THE AUDIT ADAPTER INTERFACE (interface-contract §5)**
This is the T0 contract every component binds to. Source (ADR 0034 / interface-contract §5) fixes the **behavioral contract**; concrete method/field names not in source are tagged `[PROPOSED — not in source]`.

**Source-fixed behavioral contract (NOT proposed — these are binding):**
1. The adapter is a **Python library**; every component **links against it**; no component writes Postgres/S3/OpenSearch directly.
2. The adapter **posts to a single audit endpoint** (a Deployment); callers know only the endpoint, not the stores.
3. Events flow under the **`platform.audit.*`** CloudEvent namespace (carrying `specversion` + per-event-type `schemaVersion`, ADR 0030/0031).
4. **Postgres is authoritative in-flight; S3 archive on AWS; OpenSearch advisory.** The adapter/endpoint MUST make audit ingestion succeed even when OpenSearch is down.
5. If **Postgres is down**, the endpoint **fails closed** for the emitting component (same surface as any Postgres-backed control-plane op).
6. The endpoint owns a **`LogLevel` toggle** so verbosity rises per-tenant/per-event-class **without redeploying callers**.

**`[PROPOSED — not in source]` concrete surface (to be frozen by A18, since downstreams bind to it):**
- **Adapter entrypoint** — a single `emit(event)` call (Python; TS not required — adapter is Python per ADR 0034 / backlog §6 "all custom code is Python") that takes a `platform.audit.*` CloudEvent envelope and posts to the endpoint. `[PROPOSED]`.
- **CloudEvent envelope fields the adapter populates:** CloudEvents-native `id`, `source`, `type` (a `platform.audit.*` value from B12), `time`, `specversion`, plus `schemaVersion` and a `data` payload. `time`/`specversion`/`schemaVersion` are source-named; the rest of the field list is `[PROPOSED]`.
- **Delivery semantics:** synchronous POST to the endpoint with bounded retry; fail-closed surfaced to the caller when the endpoint/Postgres is unavailable. Buffering/at-least-once vs at-most-once is **`[PROPOSED]`** (source states only "fails closed" on Postgres-down and "ingestion still succeeds" on OpenSearch-down).
- **Endpoint HTTP API:** URL-path-versioned `/v1/...` ingest route (interface-contract §3.3). Route name `[PROPOSED]`.
- **Versioning:** the adapter is a versioned SDK surface (semantic versioning, ADR 0030); a compatibility matrix against the endpoint version `[PROPOSED]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumes / ingests `platform.audit.*`** — audit-emission events from any component (gateway request/response, admission decisions, egress connections, secret access, simulator dry-runs routed here under policy). The endpoint is the consumer of record.
- **Namespace ownership (QN-03):** A18 (audit endpoint) is the **single owner of the `platform.audit` namespace** — A18 authors and registers its schema in B12 (the event catalogue). Every audit-emitting component takes an explicit dependency on A18 as the owner; A18 is the sink and namespace owner, while per-event-type names under it are registered in B12.
- **Freeze-gate (D-05):** the audit-adapter interface and the `audit_events` schema are **frozen**. A18 publishes the frozen interface/schema and the **freeze-gate that MUST pass before any component emits audit events** — no component (A1 gateway, A5 ARK, A6 sandbox, A7 OPA, etc.) may emit audit until the audit-adapter freeze-gate (D-05) passes.
- **Cross-cutting security emission (QN-03):** for any security-relevant event A18 detects (e.g. a forbidden/malformed audit write, a privileged admin action on the audit store), A18 MUST perform its existing local handling AND additionally emit the event to the bus under `platform.security` (schema owned by A7).
- **Emits `platform.observability.*`** for its own threshold crossings (e.g. in-flight table growth, S3-verify failures) — `[PROPOSED — not in source]` (no specific A18 event named; namespace is the commitment).
- **Self-audit:** A18's own privileged actions (endpoint config changes, batch deletions) emit `platform.audit.*` through the same adapter (dogfooding).
- Per-event-type schemas → **B12**. A18 commits only to namespaces + the envelope.

### 4.4 Data schemas / connection-secret contracts
- **Postgres `audit_events` table** — the in-flight system-of-record table written by the endpoint. Source names the **table name** (`audit_events`) only; **column schema is `[PROPOSED — not in source]`** and owned by A18's migrations. Proposed minimum columns: `id`, `event_time`, `event_type` (`platform.audit.*`), `tenant`, `source_component`, `schema_version`, `payload` (JSON), `archived_at` (nullable, set by batch). All `[PROPOSED]` except the table name and the `platform.audit.*` typing.
- **Connection secret** (ADR 0044) from `Postgres`/`ObjectStore`: `host`, `port`, `user`, `password`, `dbname` (+ object-store equivalent: bucket name, endpoint). The endpoint and CronJob consume these; A18 does not redefine the shape.
- **S3 object** — immutable aggregate of pending rows; verified by **existence + byte count + checksum** before Postgres deletion (source-fixed). Object key/layout `[PROPOSED]`.
- **XR status:** substrate-agnostic `ready`, `endpoint`, `version` only.

## 5. OSS-vs-Custom Decision
- **Audit adapter library** — **build-new** (custom Python). No OSS equivalent provides the platform's single-endpoint, fail-closed, `platform.audit.*` contract. Rationale: the whole point (ADR 0034) is one platform-owned emission path.
- **Audit endpoint** — **build-new** thin Python Deployment (write Postgres, fan to indexer, honor `LogLevel`).
- **OpenSearch indexer** — **build-new / wrap** thin async shipper to OpenSearch (A11). No fork of OpenSearch.
- **Postgres / S3 / OpenSearch backing stores** — **config + wrap** via substrate primitives (CloudNativePG/RDS, MinIO/S3, in-cluster/AWS OpenSearch) through the `AuditLog` composition; no fork.
- **Batch CronJob** — **build-new** Python job (aggregate → S3 → verify → delete).
- No pinned upstream versions stated in source → `[PROPOSED — not in source]` to pin at implementation. Custom code is Python per backlog §6.

## 6. Functional Requirements
- **REQ-A18-01:** A18 MUST ship a **Python audit adapter library** that every component links against; the library MUST be the only way components emit audit, and MUST NOT expose Postgres/S3/OpenSearch endpoints to callers.
- **REQ-A18-02:** The adapter MUST post audit events to a **single audit endpoint Deployment**; callers MUST know only the endpoint, not the stores.
- **REQ-A18-03:** The audit endpoint MUST write received events into the Postgres **`audit_events`** table as the authoritative in-flight system of record.
- **REQ-A18-04:** On **AWS**, a batch CronJob MUST run ~every 5 minutes: aggregate pending rows → write an **immutable S3 object** → **verify (object exists + byte count + checksum)** → **only then delete** the corresponding Postgres rows.
- **REQ-A18-05:** On **kind**, S3 MUST NOT be provisioned, the batch step MUST be disabled, and Postgres alone MUST be the system of record (functionally complete without cloud services).
- **REQ-A18-06:** The OpenSearch indexer MUST be **asynchronous and advisory only**; audit ingestion MUST succeed even when OpenSearch is unavailable; the index MUST be **rebuildable** from S3 (AWS) or Postgres (kind).
- **REQ-A18-07:** If **Postgres is unavailable**, the endpoint MUST **fail closed** for the emitting component (no silent drop).
- **REQ-A18-08:** If the **S3 write or verification fails**, the batch MUST leave the rows in Postgres and **retry** — no data loss, only in-flight table growth until recovery.
- **REQ-A18-09:** The audit endpoint MUST honor its own **`LogLevel`** toggle (ADR 0035) to raise verbosity per-tenant / per-event-class **without redeploying callers**.
- **REQ-A18-10:** The whole pipeline MUST be provisioned by a single **`AuditLog`** XR composing `Postgres` + `ObjectStore` (and indexer/CronJob/endpoint), with the same XR schema on kind and AWS (kind producing no archive by design).
- **REQ-A18-11:** All audit events MUST carry the **`platform.audit.*`** type plus `specversion` + `schemaVersion`; per-event-type schemas resolve from **B12**.
- **REQ-A18-12:** The adapter and endpoint HTTP API MUST be **versioned** (semantic versioning for the library; URL-path `/v1/...` for the endpoint) per ADR 0030, with a deprecation window before removing a version.
- **REQ-A18-13:** Endpoint egress to S3 and AWS-managed OpenSearch MUST route through the **Envoy egress proxy** (ADR 0003); the endpoint is not a special case.
- **REQ-A18-14:** A18 MUST contribute Rego for admission of the `AuditLog` XR and for any access policy on the audit endpoint, and MUST self-emit audit for its own privileged actions.
- **REQ-A18-15:** The `audit_events` schema MUST be delivered as versioned **migrations** owned by A18.

## 7. Non-Functional Requirements
- **Security:** Audit integrity is a named asset (§6.6 threat model). The endpoint is the only writer of `audit_events`; S3 objects are immutable; OpenSearch (rebuildable) is never authoritative. Adapter prevents components from forging direct writes. Endpoint access is OPA-gated; egress via Envoy. Redaction is **deferred to F1** — A18 leaves the attachment surface.
- **Multi-tenancy (§6.9):** `audit_events` MUST carry tenant attribution so audit data respects namespace/tenant boundaries; `LogLevel` `scope` supports per-tenant verbosity. `AuditLog` XR is namespaced.
- **Observability (§6.5):** Grafana dashboard (XR) for ingest rate, in-flight table size, batch lag, S3-verify failures, OpenSearch indexer lag. Alerts on Postgres-down (fail-closed), in-flight growth, S3-verify failure, indexer lag.
- **Scale / availability:** `endpointReplicas` (XRD field) scales the endpoint; OpenSearch-down and S3-down are **bounded, non-data-loss** failure modes (REQ-A18-06/-08). Postgres-down is fail-closed by design.
- **Versioning (ADR 0030):** adapter is the most-depended-on SDK surface — breaking changes require coordinated bumps + compatibility matrix; `AuditLog` XR schema changes use conversion webhooks.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (`AuditLog` XR/composition, endpoint Deployment, indexer, CronJob, migrations Job).
- Per-product docs (10.5) — **applicable** (adapter usage guide — the cross-component contract; pipeline topology).
- Runbook (10.7) — **applicable** (Postgres-down recovery, S3-verify-failure backlog drain, OpenSearch rebuild from S3/Postgres, `LogLevel` raise for incident).
- Backup/restore — **applicable** (system of record = Postgres+S3; rebuild procedure documented).
- Alerts — **applicable** (in-flight growth, batch lag, S3-verify fail, indexer lag, fail-closed). `[PROPOSED]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **applicable** (audit inspection / search view over OpenSearch advisory index).
- OPA/Rego integration — **applicable** (`AuditLog` admission, endpoint access).
- Audit emission (ADR 0034) — **applicable (A18 IS the audit subsystem; self-emits via its own adapter)**.
- Knative trigger flow — **applicable** (consumes `platform.audit.*`; designs the ingestion flow). Any new emitted trigger `[PROPOSED]`.
- HolmesGPT toolset — **applicable** (audit-query tool over the advisory index).
- 3-layer tests — **applicable** (Chainsaw: `AuditLog` reconcile + admission; PyTest: adapter fail-closed / OpenSearch-down / S3-verify-then-delete logic; Playwright: Headlamp audit view).
- Tutorials & how-tos — **applicable** ("emit audit from your component using the adapter").

## 9. Acceptance Criteria
- **AC-A18-01:** A component importing the adapter emits an audit event with no Postgres/S3/OpenSearch endpoint in its config; a code scan finds no direct store writes outside A18. (→ REQ-A18-01, -02)
- **AC-A18-02:** An emitted event lands as a row in the Postgres `audit_events` table. (→ REQ-A18-03)
- **AC-A18-03:** On AWS, after a batch run, rows are written to an immutable S3 object, verified (exists + byte count + checksum), and only then deleted from Postgres; a forced verify-failure leaves rows intact and retries. (→ REQ-A18-04, -08)
- **AC-A18-04:** On kind, no S3 is provisioned, the CronJob is absent/disabled, and Postgres alone holds the records; the install is functionally complete. (→ REQ-A18-05)
- **AC-A18-05:** With OpenSearch stopped, an audit emit still succeeds (returns OK, row in Postgres); restarting OpenSearch and rebuilding repopulates the index from Postgres/S3. (→ REQ-A18-06)
- **AC-A18-06:** With Postgres stopped, an emit causes the endpoint to fail closed and the caller observes the failure (no silent success). (→ REQ-A18-07)
- **AC-A18-07:** Setting a `LogLevel` targeting the endpoint with a per-tenant/per-event-class scope raises verbosity without redeploying any caller. (→ REQ-A18-09)
- **AC-A18-08:** A single `AuditLog` XR provisions Postgres, OpenSearch indexer, endpoint (and on AWS S3 + CronJob); the identical XR applies on kind producing no archive. (→ REQ-A18-10)
- **AC-A18-09:** Every emitted event carries a `platform.audit.*` type, `specversion`, and `schemaVersion`. (→ REQ-A18-11)
- **AC-A18-10:** The adapter library and endpoint advertise versions; a deprecated `/v1` route stays reachable ≥1 platform release after a successor lands. (→ REQ-A18-12)
- **AC-A18-11:** Endpoint→S3 and endpoint→managed-OpenSearch traffic is observed transiting the Envoy egress proxy. (→ REQ-A18-13)
- **AC-A18-12:** The `AuditLog` admission Rego rejects a malformed XR; A18's own privileged action emits a `platform.audit.*` event. (→ REQ-A18-14)
- **AC-A18-13:** Running the migrations creates the `audit_events` table at a known schema version; a re-run is idempotent. (→ REQ-A18-15)

## 10. Risks & Open Questions
- **R1 (high):** The concrete **adapter method/envelope field names** are `[PROPOSED — not in source]`. Every component binds to this surface; divergence is catastrophic. Mitigation: A18 freezes the adapter contract **first** (T0, W1) and publishes it before downstreams emit. Blast radius: platform-wide.
- **R2 (high):** **`audit_events` column schema** is `[PROPOSED]` (only the table name is source-stated). Downstream queries (Headlamp view, HolmesGPT tool, F1 retention) bind to it. Mitigation: A18 owns the migrations and versions the schema.
- **R3 (med):** **Delivery semantics** (at-least-once vs at-most-once, buffering on transient endpoint failure) are `[PROPOSED]`; source fixes only fail-closed-on-Postgres-down and ingest-survives-OpenSearch-down. Affects audit completeness guarantees the threat model relies on.
- **R4 (med):** In-flight table **unbounded growth** if S3/verify fails for long (REQ-A18-08) — bounded but operationally risky. Mitigation: alert + runbook.
- **R5 (med):** B4 build-order dependency is `[PROPOSED]` (CSV omits it); if B4's `ObjectStore`/`Postgres` lag, A18 cannot compose `AuditLog`. Mitigation: fakes for not-yet-landed XRDs in tests.
- **R6 (low):** kind "no archive" capability-parity caveat means kind cannot exercise S3-verify-then-delete failure modes (mirrors ADR 0023's cost note) — those surface only in AWS integration tests.
- **OQ1:** Does the adapter buffer/queue on transient endpoint unavailability, or fail closed immediately? Deferred — `[PROPOSED]` fail-closed-on-Postgres, bounded-retry-on-endpoint.
- **OQ2:** Is the indexer fed from the endpoint directly or off the `platform.audit.*` broker stream? `[PROPOSED]` endpoint-driven async fanout (ADR 0034 reads as endpoint→stores).
- **OQ3:** Who owns `LogLevel` reconciliation for the endpoint (in-process vs rolling-restart, ADR 0035)? A18 consumes it; `[PROPOSED]` in-process toggle since the endpoint is reloadable.

## 11. References
- architecture-overview.md §6.5 (observability + audit; logical fanout diagram), §6.6 (line 511; audit emission points — the full caller list that links the adapter; line 544 simulator runs audited under `platform.policy.*`), §6.7 (line 611; `platform.audit.*` consumed by the audit adapter), §14.1 (line 1684; A18 deliverable).
- ADR 0034 (durable adapter; Postgres+S3 system of record; OpenSearch advisory; `AuditLog` XRD; failure modes). interface-contract §5 (audit adapter interface — the binding behavioral contract).
- ADR 0035 (endpoint `LogLevel` toggle). ADR 0014 (Postgres primary). ADR 0009/0033 (OpenSearch dual-hosting). ADR 0021/0044 (`AuditLog` substrate-abstraction instance; connection secret). ADR 0003 (Envoy egress). ADR 0011 (test results via same path). ADR 0030/0031 (versioning / event taxonomy). ADR 0040 (Kargo promotes `AuditLog` XR).
- architecture-backlog.md §1.13 (retention deferred to F1), §6 (reproducibility invariant; all custom code is Python).
- Related pieces: A11 (OpenSearch), A7/B3/B16 (OPA), B4 (substrate XRDs), A4/B8 (Knative), B12 (event schemas), F1 (retention).

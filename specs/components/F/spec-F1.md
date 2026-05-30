# SPEC F1 — Audit retention policy

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [A18, B4] · downstream: [] · adrs: [0034, 0041, 0035, 0009, 0014, 0030] · views: [6.5, 6.3]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

F1 settles the audit **retention policy** that ADR 0034 and the interface contract (§5) explicitly
deferred to Workstream F: concrete retention durations for the Postgres `audit_events` in-flight
table, the S3 archive (AWS only), and the OpenSearch advisory index; lifecycle/rollover rules;
the redaction strategy for sensitive fields; and a compliance review. ADR 0034 fixes the audit
*topology* (durable adapter → audit endpoint → Postgres system of record, S3 archive on AWS,
OpenSearch advisory fanout) but states "this ADR fixes the topology, not the policy." F1 supplies
the policy and attaches it to the surfaces ADR 0034 already exposed: the S3 lifecycle policy on
the `AuditLog` XRD and the OpenSearch index rollover.

F1 runs at the **end** of v1.0 (production-readiness): it hardens and verifies an already-built
pipeline rather than building new infrastructure. Its deliverable is a policy specification plus
the concrete `AuditLog` XRD field values (lifecycle, batch schedule) and an OpenSearch index
state management (ISM) configuration that realize it, verified against the dual-mode (kind/AWS)
substrate behavior ADR 0041 guarantees.

## 2. Scope

### 2.1 In scope
- **Retention schedules** for each tier: Postgres `audit_events` (in-flight, system of record on
  kind), S3 archive (durable archive, AWS only), OpenSearch advisory index.
- **Lifecycle rules**: S3 object lifecycle (transition to cheaper storage classes, expiry) wired
  through the `AuditLog` XRD `s3BucketRef` lifecycle; OpenSearch index rollover/ISM.
- **Redaction strategy**: which audit fields are redacted/tokenized, at which stage (emission
  vs. archive vs. index), honoring that the adapter (A18) is the single write path.
- **Compliance review**: a documented mapping of retention durations to the platform's stated
  compliance posture, including the legal-hold gap (ADR 0034 revisit trigger) called out as `[PROPOSED]`.
- **Dual-mode parity statement**: kind has no S3 archive (Postgres alone is system of record;
  batch disabled) per ADR 0034 — retention semantics differ by substrate, documented explicitly.

### 2.2 Out of scope (and where it lives instead)
- **The audit pipeline / adapter / endpoint itself** → **A18** (audit endpoint + adapter library).
- **The `AuditLog` XRD and its Compositions** → **B4** (Crossplane Compositions); F1 sets field
  *values*, not the XRD schema.
- **CloudEvent per-event-type schemas under `platform.audit.*`** → **B12** (schema registry).
- **DR restore of the audit stores** → **F2** (DR testing).
- **Ongoing backup/DR cadence with measured RTO/RPO** → future-enhancements.md §1.
- **Dynamic verbosity (log-level toggle) on the audit endpoint** → **A18 / ADR 0035**; F1 does
  not change verbosity, only retention.

## 3. Context & Dependencies

**Upstream consumed:**
- **A18 (audit endpoint + adapter)** — the system of record this policy governs; the `audit_events`
  Postgres table, the 5-minute batch CronJob, the OpenSearch indexer. F1 consumes the existing
  schema and write path; it does not alter them.
- **B4 (Crossplane Compositions)** — owns the `AuditLog` XRD that composes `XPostgres` and
  `XObjectStore`. F1 supplies `batchScheduleSpec` and S3 lifecycle values into the claim.

**Downstream consumers:** none (terminal production-readiness piece).

**ADRs honored:**
- **ADR 0034** — Postgres + S3 are the system of record; OpenSearch is advisory and rebuildable;
  retention/redaction/lifecycle are exactly F1's deferred surface (S3 lifecycle on the XRD +
  OpenSearch rollover). Legal-hold / synchronous-archive are revisit triggers, out of scope.
- **ADR 0041** — substrate abstraction: kind `XObjectStore` may be a no-op archive; retention
  policy MUST tolerate the no-archive kind path (capability-parity not promised).
- **ADR 0009 / 0014** — OpenSearch advisory-only; Postgres primary. Retention on OpenSearch never
  makes it authoritative.
- **ADR 0035** — the endpoint's `LogLevel` toggle affects verbosity, not retention; F1 keeps them
  orthogonal.
- **ADR 0030** — any new field convention F1 needs is versioned per-component.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F1 sets **values** on the existing `AuditLog` (XRD) (Canon §1.6): `batchScheduleSpec` (the ~5-min
batch cadence, AWS only), `s3BucketRef` → S3 lifecycle, `indexerRef` → OpenSearch ISM. F1 defines
**no new XRD**. Concrete retention **durations** are not stated in source — all numeric values
below are `[PROPOSED — not in source]`:
- Postgres `audit_events`: in-flight only on AWS (rows deleted after verified S3 write); on kind it
  is the system of record and retained per a `[PROPOSED]` Postgres retention window.
- S3 archive (AWS): `[PROPOSED]` hot→infrequent-access transition, then long-term retention, then
  expiry — concrete durations set during compliance review.
- OpenSearch advisory index: `[PROPOSED]` rollover by age/size + ISM delete; rebuildable from S3/Postgres.

### 4.2 APIs / SDK surfaces
N/A — F1 introduces no API/SDK surface. It consumes the A18 adapter contract (Canon §5) and the
`AuditLog` claim. The redaction strategy is realized in the adapter/endpoint config (A18), not a
new surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Consumes the existing `platform.audit.*` namespace (Canon §2) as the data F1 governs. F1 emits no
new event types; per-event redaction applies to fields carried under `platform.audit.*`. A lifecycle
or retention-violation alert, if emitted, would fall under `platform.observability.*` —
`[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts
Governs the Postgres `audit_events` table (A18-owned) and the S3 archive objects (immutable,
checksum-verified per ADR 0034). Uses the standard `XPostgres` / `XObjectStore` connection-secret
shape (`host`, `port`, `user`, `password`, `dbname`; Canon §4) — F1 adds no fields. Redaction
operates on record *contents*, not the connection contract.

## 5. OSS-vs-Custom Decision
**Config/policy over existing OSS + custom infra** — no new build. S3 lifecycle is native object-store
lifecycle config (AWS S3; MinIO no-op on kind); OpenSearch retention is native **ISM** (Index State
Management); Postgres retention is a `[PROPOSED]` CronJob/`DELETE` window. Approach: **config** the
`AuditLog` XRD + OpenSearch ISM + (proposed) Postgres prune job; redaction is **config** of the A18
adapter. No fork, no new service. Rationale: ADR 0034 deliberately left a "concrete surface to
attach to" (S3 lifecycle + OpenSearch rollover); F1 attaches policy, it does not build a pipeline.

## 6. Functional Requirements
- **REQ-F1-01:** F1 MUST define a concrete retention duration for each tier — Postgres
  `audit_events`, S3 archive (AWS), OpenSearch advisory index — with substrate-specific values.
- **REQ-F1-02:** S3 archive lifecycle (transitions + expiry) MUST be expressed via the `AuditLog`
  XRD `s3BucketRef` lifecycle and applied on AWS only (no-op/absent on kind per ADR 0041).
- **REQ-F1-03:** OpenSearch advisory-index retention MUST be expressed as ISM rollover + delete,
  and MUST NOT make OpenSearch authoritative — the index stays rebuildable from S3/Postgres (ADR 0009/0034).
- **REQ-F1-04:** On kind, where no S3 archive exists, the policy MUST keep Postgres as the system
  of record and define its standalone retention window without the batch-delete path (ADR 0034).
- **REQ-F1-05:** F1 MUST define a **redaction strategy** naming the sensitive field classes, the
  redaction stage (emission/archive/index), and the irreversibility guarantee for archived records.
- **REQ-F1-06:** A **compliance review** MUST map each retention duration to the platform compliance
  posture and explicitly record the legal-hold gap as out of scope (ADR 0034 revisit trigger).
- **REQ-F1-07:** Retention and redaction config MUST be Git-reconciled (ArgoCD) and versioned per
  ADR 0030; changing a duration is a reviewed, audited change.
- **REQ-F1-08:** The policy MUST NOT delete Postgres rows on AWS before the ADR 0034 verified-S3-write
  invariant holds (object exists + byte count + checksum) — retention never overrides durability.

## 7. Non-Functional Requirements
- **Security:** redaction MUST prevent sensitive data persisting beyond its retention class; archived
  S3 objects remain immutable (write-once) so redaction is decided at emission/archive, not by mutation.
- **Multi-tenancy (§6.9):** retention/redaction MUST be expressible per-tenant or per-event-class
  (the endpoint already scopes by tenant/event-class for verbosity, ADR 0035) — `[PROPOSED]` to reuse
  that scoping for retention overrides.
- **Observability (§6.5):** in-flight `audit_events` table growth MUST be alertable (batch-stall
  detection) so a stuck S3 write doesn't silently grow Postgres (ADR 0034 failure mode).
- **Scale:** OpenSearch ISM rollover MUST bound index size at v1.0 audit volume (validated against F5
  scale numbers where available).
- **Versioning (ADR 0030):** retention-duration changes are versioned config; a shortening of
  retention is treated as a breaking compliance change requiring review.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (ISM policy, S3 lifecycle values, proposed Postgres prune CronJob; via `AuditLog` claim).
- Per-product docs (10.5) — **applicable** (retention policy reference; redaction matrix).
- Runbook (10.7) — **applicable** (retention change procedure; batch-stall / table-growth response). Feeds **F6**.
- Alerts — **applicable** (`audit_events` growth / batch stall; ISM failure). `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** (retention/lifecycle health: archive lag, index size). `[PROPOSED]`.
- Headlamp plugin — **N/A** — retention is Git-reconciled config, not an interactive CRD editor surface.
- OPA/Rego integration — **applicable** — `[PROPOSED]` admission guard rejecting a retention shorter than the compliance floor.
- Audit emission (ADR 0034) — **N/A (governs audit data; does not itself emit)** — F1 is policy over the audit pipeline.
- Knative trigger flow — **N/A** — no new event flow; retention is scheduled/lifecycle, not event-driven.
- HolmesGPT toolset — **applicable** — `[PROPOSED]` read tool to report retention posture / archive lag during triage.
- 3-layer tests — **applicable** (Chainsaw for `AuditLog` claim lifecycle values; PyTest for redaction + Postgres prune logic).
- Tutorials & how-tos — **applicable** (how-to: change a retention duration; how-to: add a redaction rule).

## 9. Acceptance Criteria
- **AC-F1-01:** Each tier has a documented, concrete retention duration with kind/AWS variants. (→ REQ-F1-01)
- **AC-F1-02:** On AWS, an S3 archive object transitions/expires per the configured lifecycle; on kind the lifecycle is absent/no-op. (→ REQ-F1-02)
- **AC-F1-03:** OpenSearch ISM rolls over and deletes old indices; a deleted index is rebuilt from S3/Postgres in a test. (→ REQ-F1-03)
- **AC-F1-04:** On kind, audit rows persist in Postgres per the standalone window with no batch-delete attempted. (→ REQ-F1-04)
- **AC-F1-05:** A sensitive field is redacted at the defined stage and is absent from the archived S3 object. (→ REQ-F1-05)
- **AC-F1-06:** The compliance review document maps every duration to a posture statement and records the legal-hold gap. (→ REQ-F1-06)
- **AC-F1-07:** A retention-duration change merges via Git/ArgoCD and is itself audited. (→ REQ-F1-07)
- **AC-F1-08:** A simulated S3 verify-failure leaves Postgres rows intact (no deletion) until verification succeeds. (→ REQ-F1-08)

## 10. Risks & Open Questions
- **R1 (high):** All concrete durations are `[PROPOSED — not in source]`; compliance review must
  set them. If set wrong, either cost (too long) or compliance breach (too short). Mitigation:
  compliance sign-off gates REQ-F1-01. Blast radius: high (legal/cost).
- **R2 (med):** Redaction at emission is irreversible for archived S3 objects (immutable); a wrong
  redaction rule loses data permanently. Mitigation: redaction-rule review + test on kind first.
- **R3 (med):** kind/AWS retention divergence (no S3 on kind) risks tests passing on kind but
  archive behavior never exercised. Mitigation: AWS-substrate Chainsaw run required (F2 overlaps).
- **R4 (low):** OpenSearch ISM misconfig could delete an index faster than rebuild capacity.
  Mitigation: rebuild drill (shared with F2 reindex).
- **OQ1:** Is there a regulatory legal-hold requirement that would reopen ADR 0034's advisory-only
  stance? `[PROPOSED]` assume no for v1.0; flag for the compliance reviewer.
- **OQ2:** Should retention be overridable per-tenant (reusing ADR 0035 scoping) or platform-global
  only? `[PROPOSED]` global default + per-event-class override.

## 11. References
- architecture-overview.md §6.5 (observability/audit emission points, line 511), §6.3 (audit
  retention paragraph, line 335 dual-mode Postgres).
- Canon interface-contract §5 (audit adapter interface; "retention deferred to F1"), §1.6 (`AuditLog` XRD), §2 (`platform.audit.*`), §4 (connection secret).
- ADR 0034 (audit pipeline — retention/redaction/lifecycle deferred to F; S3 lifecycle + OpenSearch rollover surface; verified-write invariant; revisit triggers).
- ADR 0041 (substrate abstraction; kind no-archive parity caveat). ADR 0009/0014 (OpenSearch advisory; Postgres primary). ADR 0035 (verbosity toggle, kept orthogonal). ADR 0030 (versioning).
- Related pieces: A18 (audit endpoint/adapter), B4 (`AuditLog` XRD), B12 (event schemas), F2 (DR), F6 (runbook compilation).

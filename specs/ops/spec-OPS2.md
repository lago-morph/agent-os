# SPEC OPS2 — Disaster Recovery Architecture

> kind: COMPONENT (cross-cutting NFR) · workstream: — (operational/NFR layer) · tier: T1
> upstream: [B4, A18, A10, B11, A11, A21, B19, A7, A4, A23] · downstream: [F2, F6] · adrs: [0034, 0041, 0014, 0009, 0033, 0028, 0037, 0040, 0030, 0025, 0020, 0021] · views: [6.3, 6.5, 6.6]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

OPS2 is the platform's **disaster-recovery (DR) architecture**: the cross-cutting design that
defines what platform and tenant state must survive a destructive event, how fast it must come
back (RTO) and how much of it may be lost (RPO), and in what order the control plane and data
plane are recovered. The source architecture specifies the *functional* shape of every
state-bearing piece (B4 substrate XRDs, A18 audit Postgres+S3 system of record, A10/B11 memory,
B19 approval state, A21 tenant onboarding) but treats backup/restore and failover thinly — most
of it is deferred into Workstream F (F2 "DR testing") and `future-enhancements.md` §1. F2 owns
the *testing* of DR; OPS2 owns the *DR design* that F2 tests against.

The problem OPS2 solves: without a single DR architecture, each state-bearing component would
make its own (or no) backup decision, RPO/RTO would be undefined and therefore unverifiable, and
a region or cluster loss would have an unbounded, undocumented recovery procedure. OPS2 is a
constraint-and-realization layer: it classifies platform/tenant state into **data classes**,
assigns each class an RPO/RTO target, fixes the **restore-ordering** (control plane before data
plane, system-of-record before advisory rebuild), and states the v1.0 **cross-AZ/region posture**
— without re-deciding the genuinely open questions, which are surfaced as proposed ADRs in §10.

## 2. Scope

### 2.1 In scope
- A **data-classification taxonomy** for all platform and tenant state, each class mapped to an
  RPO and RTO target (§4.4, §6).
- **Backup topology** per data class: what is backed up, by what mechanism, to where, and how the
  kind/AWS substrate split (ADR 0041) changes the available durable target.
- **Control-plane recovery**: Kubernetes/etcd + CRD/XRD object restore — the GitOps-reconciled
  declarative state (ArgoCD/Crossplane) vs the non-reconcilable runtime state.
- **Data-plane recovery**: Postgres (`XPostgres` — CloudNativePG/RDS), object store
  (`XObjectStore` — MinIO/S3), search/index (`XSearchIndex` — OpenSearch), Mongo
  (`XMongoDocStore`), and the audit system of record (A18 Postgres+S3), memory backend (A10
  Letta / B11), per-agent/tenant/user databases (`XAgentDatabase`).
- **Restore ordering**: the dependency-respecting sequence that brings the platform back without
  violating the system-of-record / advisory-rebuild contract (e.g. OpenSearch rebuilt from S3 or
  Postgres, never restored as authoritative).
- **Region / AZ failover posture** for v1.0 (a *stated posture*, not necessarily an active-active
  build — the actual target is a proposed ADR in §10).
- The **NFR obligations OPS2 pushes back into component specs** (which existing spec must absorb a
  backup/restore obligation — §3, §7).

### 2.2 Out of scope (and where it lives instead — name the owning piece)
- **DR testing / drills execution** → **F2 (DR testing)**. OPS2 defines the targets and the
  restore procedure; F2 builds and runs the drills that prove them.
- **Production runbook compilation** → **F6**. OPS2 specifies the restore-ordering and per-class
  procedures; F6 compiles them into the operator runbook set.
- **Audit retention / lifecycle / redaction policy** → **F1**. Retention durations affect how far
  back a restore can go but the policy itself is F1's (interface-contract §5).
- **The substrate XRDs + Compositions + connection-secret contract** → **B4**. OPS2 consumes the
  XRDs and the uniform connection secret; it does not redefine them.
- **The audit adapter + endpoint + system-of-record mechanics** (Postgres+S3, batch CronJob,
  OpenSearch rebuild) → **A18**. OPS2 sets the audit-class RPO/RTO and restore ordering; A18 owns
  the pipeline.
- **Secret / key backup & rotation, KMS/envelope, ESO topology** → **OPS3** (key & secret
  management). OPS2 references secret-material recoverability as a hard dependency but does not own
  rotation/KMS design.
- **Scale/capacity/cost model** → **OPS1**. Restore *throughput* sizing references OPS1's capacity
  model but OPS2 does not own it.
- **Crossplane provider-stack selection / cluster bootstrap / resource sizing / region selection**
  → install-time config, intentionally NOT wrapped (ADR 0041 documented exceptions); OPS2 states
  the recovery posture, not the bootstrap automation.
- **Promotion fabric (Kargo)** → **A23**. OPS2 uses GitOps/Kargo state as a recovery *source of
  truth* for declarative config; it does not own promotion.

## 3. Context & Dependencies

**Upstream pieces consumed (and exactly what is consumed):**
- **B4 (Crossplane v2 Compositions)** — the `XPostgres`, `XSearchIndex`, `XObjectStore`,
  `XMongoDocStore` substrate XRDs; the higher-level `AuditLog`, `XAgentDatabase`,
  `TenantOnboarding`, `GrafanaDashboard` XRDs; the **uniform connection-secret shape** and the
  substrate-agnostic XR status. OPS2's backup/restore is expressed against these claim shapes so
  it is substrate-portable.
- **A18 (audit endpoint + adapter)** — the **Postgres + S3 system of record**, the immutable
  verified S3 archive, and the **OpenSearch-rebuildable** advisory invariant. The audit data class
  inherits A18's failure-mode contract.
- **A10 (Letta) / B11 (memory backend adapter)** — agent memory state and the `MemoryStore`
  access modes (ADR 0025); a memory data class with its own RPO/RTO.
- **A11 (OpenSearch)** — the advisory search/vector store that is **rebuilt, not restored as
  authoritative** (ADR 0009/0033).
- **A21 (tenant onboarding) / `TenantOnboarding` XRD** — tenant provisioning state
  (`tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`) that must be
  reconstructable for tenant recovery.
- **B19 (approval system)** — in-flight `Approval` CRD state + any backing store; an open-approval
  data class whose loss has governance impact.
- **A7 (OPA / Gatekeeper)** — admission must continue to enforce substrate-match (ADR 0002) during
  a restore on a target cluster so a restored claim resolves to a valid Composition.
- **A4 (Knative + NATS JetStream)** — broker/stream state; in-flight event backlog is a recovery
  consideration (broker durability vs replay).
- **A23 (Kargo) / ArgoCD / Crossplane** — GitOps-reconciled declarative config is the **recovery
  source of truth** for control-plane object state.

**Downstream consumers:**
- **F2 (DR testing)** — builds drills that assert OPS2's RPO/RTO and restore ordering.
- **F6 (production runbook compilation)** — compiles OPS2's restore procedures into runbooks.

**ADR decisions honored (cite + constraint):**
- **ADR 0034** — Postgres + S3 are the audit system of record; OpenSearch is advisory and
  **rebuildable**. *Constraint:* OPS2 MUST treat OpenSearch as rebuilt-not-restored and order
  restore so the system-of-record returns first.
- **ADR 0041** — one XRD + one Composition per substrate; uniform connection secret;
  capability-parity not promised; documented non-wrapped exceptions. *Constraint:* OPS2 backup/
  restore is expressed against claim shapes; kind may lack a durable archive target (no S3) so the
  kind RPO/RTO posture differs by design.
- **ADR 0014** — Postgres as primary control-plane-adjacent store. *Constraint:* Postgres recovery
  is the critical path of the data-plane RTO.
- **ADR 0009 / 0033** — OpenSearch dual-hosting; kind functionally complete without cloud services.
  *Constraint:* search-index recovery is a rebuild from a primary source, with a longer RTO than
  the primary it is rebuilt from.
- **ADR 0028 / 0037** — `TenantOnboarding` provisions tenants; CapabilitySets intentionally not
  coupled. *Constraint:* tenant recovery reconstructs onboarding state + namespaces + OIDC mapping;
  CapabilitySets recover via GitOps, not via onboarding.
- **ADR 0040** — Kargo promotion fabric; declarative config flows through Git. *Constraint:* OPS2
  treats Git + Kargo Warehouse/Stage state as the authoritative recovery source for declarative
  config.
- **ADR 0030** — versioning. *Constraint:* a restore onto a newer-version cluster MUST be able to
  run conversion webhooks; OPS2 must not assume identical CRD/XRD versions across backup and
  restore points.
- **ADR 0025 / 0020 / 0021** — memory access modes, per-scope databases, dashboards. *Constraint:*
  these define additional data classes OPS2 must classify.

## 4. Interfaces & Contracts

Use ONLY names in `_meta/interface-contract.md` / `_meta/glossary.md`. Anything not in Canon is
tagged `[PROPOSED — not in source]`. OPS2 is a cross-cutting NFR piece: it mints **no new CRD or
event type** of its own; it expresses obligations against existing Canon surfaces.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
OPS2 introduces **no new CRD/XRD**. It consumes the following existing Canon XRDs as the units of
backup/restore (all namespaced; fields per interface-contract §1.6):

| XRD / store | Source-stated fields used by DR | Backup unit | Restore class |
|---|---|---|---|
| `XPostgres` ↔ `Postgres` | `version`, `storage`, `connectionSecretRef`, `substrateClass` | DB snapshot / PITR | data-plane primary |
| `XObjectStore` ↔ `ObjectStore` | `bucketName`, `lifecycle`, `connectionSecretRef` | bucket + versioning | durable archive (AWS) |
| `XSearchIndex` ↔ `SearchIndex` | `nodeCount`, `storage`, `connectionSecretRef` | **rebuild source ref** | advisory (rebuilt) |
| `XMongoDocStore` ↔ `MongoDocStore` | `version`, `storage`, `connectionSecretRef` | DB snapshot | data-plane primary |
| `AuditLog` (XR `XAuditLog`) | `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec` | composes the above | audit class |
| `XAgentDatabase` ↔ `AgentDatabase` | `engine`, `scope`, `ownerRef`, `credentialsSecretRef` | per-scope snapshot | tenant data class |
| `MemoryStore` (XR) | `accessMode`, `backendType` | backend snapshot | memory class |
| `TenantOnboarding` | `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` | GitOps object | control-plane (declarative) |

> `[PROPOSED — not in source]`: a backup-policy attachment surface. The source does not provide a
> field on any XRD to declare backup schedule/retention/target for DR. OPS2 proposes that DR
> backup policy attach **out-of-band** (operator-owned, e.g. CloudNativePG/RDS backup config and
> S3 versioning/replication) rather than as a new XRD field, to avoid coining a CRD shape outside
> Canon. Whether DR policy should instead become a first-class XRD field is **OPS2-ADR-A** (§10).

### 4.2 APIs / SDK surfaces
N/A — OPS2 exposes no SDK. Its "interface" is the **restore procedure** (an ordered runbook,
verified by F2) and the **RPO/RTO targets** (testable, §6/§9). It consumes the existing
connection-secret shape (§4.4) and the substrate-agnostic XR status (`ready`, `endpoint`,
`version`) as the signals a restore is complete.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
OPS2 mints **no new CloudEvent type and no new top-level namespace** (a new namespace would be a
breaking change requiring its own ADR — interface-contract §2). DR-relevant signals map onto
existing namespaces:
- **`platform.observability.*`** — backup-job failure, backup-staleness (RPO-at-risk) threshold
  crossings, restore-drill SLA alerts. `[PROPOSED — not in source]` (no specific event named;
  namespace is the commitment; per-event-type schemas → B12).
- **`platform.lifecycle.*`** — store/XR lifecycle during restore (created/started/failed).
- **`platform.security.*`** — a destructive-event / failover signal, if the failover trigger is
  modelled as a security event. `[PROPOSED]` — whether DR triggers an event at all is a B12/B8
  design point.
- **`platform.audit.*`** — restore actions are privileged and self-audited via the A18 adapter
  (dogfooding), like any other privileged platform action.

### 4.4 Data schemas / connection-secret contracts
- **No new connection-secret shape.** Restore reuses the uniform shape (`host`, `port`, `user`,
  `password`, `dbname` or documented per-primitive equivalent, ADR 0041) written by B4
  Compositions; a restored store re-emits the same secret via `connectionSecretRef`, so consumers
  reconnect without branching on whether the store is fresh or restored.
- **Data-class table (the load-bearing DR schema)** — each platform/tenant state class with its
  store, system-of-record status, RPO, and RTO. The class set is Canon-derived; the **numeric
  RPO/RTO values are `[PROPOSED — not in source]`** (the source states no targets — `future-
  enhancements.md` §1 defers them) and are proposed here as testable defaults for v1.0:

  | Class | Backing store(s) | System of record? | RPO (proposed) | RTO (proposed) |
  |---|---|---|---|---|
  | Declarative config (CRDs/XRDs, CapabilitySets, policy bundles) | Git + ArgoCD/Kargo + etcd | Git (authoritative) | **0 (Git is source of truth)** | **≤ 1 h** (reconcile from Git) |
  | Audit (in-flight) | Postgres `audit_events` | Postgres+S3 | **≤ 5 min** (matches A18 batch cadence) | **≤ 2 h** |
  | Audit (archive) | S3 immutable objects (AWS) | S3 | **≤ 5 min** (post-batch durable) | **≤ 4 h** (rebuild index after) |
  | Tenant/agent/user data | `XAgentDatabase` (Postgres/Mongo) | the DB | **≤ 15 min** (PITR) | **≤ 4 h** |
  | Agent memory | A10 Letta / `MemoryStore` | the backend | **≤ 1 h** | **≤ 4 h** |
  | Search/vector index | `XSearchIndex` (OpenSearch) | **none — rebuilt** | **N/A (rebuilt)** | **≤ 8 h** (rebuild) |
  | Approval (in-flight) | `Approval` CRD + backing store | etcd / DB | **≤ 15 min** | **≤ 2 h** |
  | Object store (general) | `XObjectStore` | the bucket | **≤ 15 min** (versioning) | **≤ 4 h** |

  > All numeric RPO/RTO values above are `[PROPOSED — not in source]`. They are stated as concrete,
  > testable defaults so F2 has something to verify and so reviewers can challenge specific numbers
  > rather than an absent target. The **authoritative per-class numbers are a proposed ADR**
  > (**OPS2-ADR-B**, §10). On **kind**, archive-class RPO/RTO are **N/A by design** (no S3; ADR
  > 0041 capability-parity caveat) — kind's DR posture is "Git-reconcilable config + single-store
  > data, no cross-region durability."

## 5. OSS-vs-Custom Decision
- **Postgres backup/PITR** — **config + wrap** the substrate-native mechanism: CloudNativePG
  backups on kind, RDS automated backups / snapshots / PITR on AWS (both reached via `XPostgres`).
  No fork; no custom backup engine. Version pins `[PROPOSED — not in source]`.
- **Object-store durability** — **config** S3 versioning + (proposed) cross-region replication on
  AWS; MinIO on kind has no cross-region story by design (ADR 0041 "no archive" parity caveat).
- **Search/index** — **no backup engine**: rebuilt from the system of record (S3/Postgres) per ADR
  0034/0009 — reuses A18's rebuild path; OPS2 adds the rebuild-ordering, not a new tool.
- **Control-plane object state** — **config**: GitOps (ArgoCD) + Kargo (A23) are the recovery
  source of truth; etcd backup is the substrate-native mechanism (managed control plane on AWS;
  declared cluster-bootstrap concern on kind). OPS2 adds **no custom etcd backup tool** for v1.0;
  whether to add one is **OPS2-ADR-C** (§10).
- **Restore orchestration** — **build-new (thin)**: a documented, F2-tested restore runbook +
  optional ordering automation; not a heavyweight DR product in v1.0. A managed DR/orchestration
  product is a `future-enhancements.md` §1 item, surfaced as **OPS2-ADR-D** (§10).

## 6. Functional Requirements
- **REQ-OPS2-01:** OPS2 MUST define a **data-classification taxonomy** covering all platform and
  tenant state (the §4.4 class table at minimum), and every class MUST be assigned a backup
  mechanism, an RPO, and an RTO.
- **REQ-OPS2-02:** **Declarative config** (CRDs/XRDs, CapabilitySets, OPA/Rego bundles, Kargo/
  ArgoCD config) MUST be recoverable **from Git** as the authoritative source (RPO 0); a control-
  plane restore MUST reconcile cluster object state from Git/Kargo, not from an etcd backup, as the
  primary path.
- **REQ-OPS2-03:** The **audit system of record** (A18 Postgres + S3) MUST be backed up such that
  the in-flight audit RPO is **≤ 5 minutes** (matching the A18 batch cadence) and the durable S3
  archive is treated as the long-term record; restore MUST NOT treat OpenSearch as authoritative.
- **REQ-OPS2-04:** **OpenSearch / `XSearchIndex`** MUST be **rebuilt from a primary source**
  (S3 on AWS, Postgres on kind), never restored as a system of record (ADR 0034/0009/0033); its
  recovery is gated on the primary source being restored first.
- **REQ-OPS2-05:** **Tenant/agent/user databases** (`XAgentDatabase` → `XPostgres`/`XMongoDocStore`)
  MUST support point-in-time recovery (PITR) on AWS and substrate-native backup on kind, meeting
  the tenant-data RPO/RTO in §4.4.
- **REQ-OPS2-06:** **Agent memory** (A10 Letta / `MemoryStore`) MUST have a defined backup
  mechanism and RPO/RTO; memory recovery MUST honor the `MemoryStore.accessMode` boundary
  (ADR 0025) so a restore does not widen access.
- **REQ-OPS2-07:** OPS2 MUST define a **restore-ordering** that brings the control plane back
  before data-plane dependents, restores each **system of record before** any advisory store that
  is rebuilt from it, and restores backing stores before the components that consume their
  connection secrets.
- **REQ-OPS2-08:** **Tenant recovery** MUST reconstruct `TenantOnboarding` state (namespaces, SAs,
  OIDC claim mapping) and the tenant's databases without coupling CapabilitySets into onboarding
  (ADR 0028/0037); CapabilitySets recover via GitOps.
- **REQ-OPS2-09:** A restore MUST re-emit the **uniform connection secret** (ADR 0041) for each
  restored store so consumers reconnect without code change, and MUST pass through **Gatekeeper
  substrate-match admission** (ADR 0002/0041) on the target cluster.
- **REQ-OPS2-10:** OPS2 MUST state the **v1.0 region/AZ failover posture** explicitly, including
  whether v1.0 targets active-active, active-passive, or backup-restore-only across regions, and
  MUST mark any cross-region capability that depends on substrate (e.g. S3 replication) as
  AWS-only with kind degraded by design (ADR 0041).
- **REQ-OPS2-11:** A restore MUST tolerate a **version skew** between the backup point and the
  target cluster by running CRD/XRD conversion webhooks (ADR 0030); OPS2 MUST NOT assume identical
  versions across backup and restore.
- **REQ-OPS2-12:** All restore actions MUST be **self-audited** via the A18 audit adapter, and
  backup-failure / RPO-at-risk conditions MUST surface as `platform.observability.*` threshold
  crossings (per-event-type schema → B12).
- **REQ-OPS2-13:** OPS2 MUST enumerate the **NFR obligations each component spec must absorb**
  (the backup/restore responsibility that lands on A18, B4, A10/B11, A21, B19) so the obligation is
  traceable, and MUST NOT silently re-own those mechanisms.
- **REQ-OPS2-14:** Every RPO/RTO target MUST be **objectively measurable** by an F2 drill: a
  destructive-event scenario, a measured restore, and a pass/fail against the target.

## 7. Non-Functional Requirements
- **Security:** A restore re-creates credentialed connection secrets; restored secret material MUST
  remain under the same RBAC-as-floor / OPA-as-restrictor controls (ADR 0018) and the same access
  modes (`MemoryStore.accessMode`, `GrafanaDashboard.visibility`). **Secret/key recoverability is a
  hard dependency on OPS3** — if KMS/envelope keys are unrecoverable, encrypted backups are
  unrecoverable; OPS2 names this dependency and defers the key-recovery design to OPS3. Backups
  themselves are sensitive assets (audit integrity, tenant data) and inherit the audit-integrity
  threat-model treatment (§6.6 / B22).
- **Multi-tenancy (§6.9):** restore MUST preserve tenant isolation — a tenant's data restores into
  its own namespaces; cross-tenant data MUST NOT leak via a shared restore; per-tenant restore
  (recovering one tenant without others) is a `[PROPOSED]` capability gated on per-tenant backup
  granularity (**OPS2-ADR-E**, §10).
- **Observability (§6.5):** backup freshness, backup-job success, restore-drill duration, and
  RPO/RTO headroom MUST be dashboarded (a `GrafanaDashboard` XR) and alerted
  (`platform.observability.*`). The signal that a class is **out of RPO** MUST be a first-class
  alert, not a derived one.
- **Scale:** restore *throughput* (DB size / s, index rebuild rate) sets the achievable RTO and
  MUST be sized against OPS1's capacity model; OPS2 references OPS1, does not own sizing.
- **Versioning (ADR 0030):** the DR procedure is itself a versioned operational artifact; a change
  to restore ordering or RPO/RTO targets is a documented, reviewed change (the targets' authority
  is **OPS2-ADR-B**).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable (config only)** — backup policy config on `XPostgres`/`XObjectStore`
  (operator-native), a restore-ordering manifest/job set. No new CRD.
- Per-product docs (10.5) — **applicable** (per-class backup/restore procedure; kind-vs-AWS posture).
- Runbook (10.7) — **applicable** — the restore runbook is the primary OPS2 artifact; **compiled by
  F6**, authored here.
- Backup/restore — **applicable (this IS the piece)** — per-class backup mechanism + restore order.
- Alerts — **applicable** (backup-job failure, backup staleness / RPO-at-risk, restore-drill SLA).
  `[PROPOSED]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable** — DR posture dashboard authored as a
  `GrafanaDashboard` claim (backup freshness, RPO/RTO headroom, last successful drill).
- Headlamp plugin — **N/A** — no dedicated DR UI in v1.0; restore is operator-driven.
- OPA/Rego integration — **applicable (contract)** — restored claims MUST pass Gatekeeper
  substrate-match admission (ADR 0002); OPS2 states the requirement, B16 owns the Rego.
- Audit emission (ADR 0034) — **applicable** — restore actions self-audit via the A18 adapter.
- Knative trigger flow — **N/A** — OPS2 mints no event types; backup-failure events ride existing
  namespaces (schemas → B12).
- HolmesGPT toolset — **applicable (proposed)** — a read-only "DR posture / backup freshness" query
  toolset for the self-management agent. `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** — Chainsaw for claim restore → connection-secret re-emit →
  admission; PyTest for RPO/RTO measurement harness; **chaos/DR drill (F2)** for end-to-end.
  Playwright **N/A** — no UI.
- Tutorials & how-tos — **applicable** ("restore a tenant", "run a DR drill", "rebuild the search
  index").

## 9. Acceptance Criteria
- **AC-OPS2-01:** The data-classification table exists and **every** class has a named backup
  mechanism, an RPO, and an RTO; no platform/tenant state class is unclassified. (→ REQ-OPS2-01)
- **AC-OPS2-02:** A drill that destroys cluster object state and reconciles from Git/Kargo restores
  all declarative config (CRDs/XRDs/CapabilitySets/policy bundles) with **RPO = 0** and within the
  config-class RTO (proposed **≤ 1 h**), without resorting to an etcd backup. (→ REQ-OPS2-02)
- **AC-OPS2-03:** A drill measures audit in-flight data loss after a Postgres-loss event at **≤ 5
  minutes** (matching A18 batch cadence) and shows OpenSearch was **not** used as the recovery
  source of record. (→ REQ-OPS2-03)
- **AC-OPS2-04:** After a primary-source restore, the OpenSearch / `XSearchIndex` index is
  **rebuilt** (not restored from an OpenSearch backup) from S3 (AWS) or Postgres (kind) and a test
  asserts the rebuild source, completing within the search-class RTO (proposed **≤ 8 h**).
  (→ REQ-OPS2-04)
- **AC-OPS2-05:** An `XAgentDatabase`-backed tenant DB is recovered via PITR (AWS) / substrate
  backup (kind) to a target point with measured RPO **≤ 15 min** and RTO **≤ 4 h** (proposed).
  (→ REQ-OPS2-05)
- **AC-OPS2-06:** Agent memory (`MemoryStore`) is restored within the memory-class RPO/RTO and a
  test asserts the restored store's `accessMode` is unchanged from backup (no access widening).
  (→ REQ-OPS2-06)
- **AC-OPS2-07:** A full-platform restore drill follows the documented ordering and **fails** the
  test if any advisory store is brought up before its system of record, or any consumer before its
  backing store's connection secret. (→ REQ-OPS2-07)
- **AC-OPS2-08:** A tenant-recovery drill reconstructs `TenantOnboarding` (namespaces, SAs, OIDC
  mapping) and the tenant DB, and a test asserts **no CapabilitySet** was created by onboarding
  (they reconcile from Git). (→ REQ-OPS2-08)
- **AC-OPS2-09:** Each restored store re-emits the uniform connection secret (`host`/`port`/`user`/
  `password`/`dbname` or documented per-primitive map) and a restored claim is **admitted** by
  Gatekeeper substrate-match on the target cluster. (→ REQ-OPS2-09)
- **AC-OPS2-10:** The spec states a single, unambiguous v1.0 region/AZ failover posture, and any
  cross-region capability is marked AWS-only with the kind degradation documented. (→ REQ-OPS2-10)
- **AC-OPS2-11:** A restore from a backup taken at version `vN-1` onto a `vN` cluster succeeds via
  conversion webhooks; a test asserts the converted objects reconcile. (→ REQ-OPS2-11)
- **AC-OPS2-12:** Restore actions appear in `audit_events` via the A18 adapter, and an induced
  backup failure raises a `platform.observability.*` RPO-at-risk alert. (→ REQ-OPS2-12)
- **AC-OPS2-13:** The spec enumerates, per component (A18, B4, A10/B11, A21, B19), the exact
  backup/restore obligation that component must absorb; each is traceable to a REQ. (→ REQ-OPS2-13)
- **AC-OPS2-14:** Every RPO/RTO in the class table has a corresponding F2 drill that produces a
  measured number and a pass/fail verdict against the target. (→ REQ-OPS2-14)

## 10. Risks & Open Questions

Each item carries a blast-radius rating. Genuinely open decisions are surfaced as **proposed ADR
titles** (NOT auto-decided): the user authors them only if asked.

**Proposed ADRs (titles + one-line rationale):**
- **OPS2-ADR-A — "Backup/DR policy attachment: out-of-band operator config vs first-class XRD
  field."** *(blast radius: med)* — Source provides no XRD field to declare backup schedule/
  retention/target; deciding whether DR policy becomes Canon CRD shape or stays operator-native
  changes B4's XRD surface and is a versioning commitment.
- **OPS2-ADR-B — "Authoritative per-data-class RPO/RTO targets for v1.0."** *(blast radius: high)*
  — The source states no RPO/RTO numbers (`future-enhancements.md` §1 defers them); these numbers
  define every F2 drill's pass/fail and bound cost/architecture, so they need a deliberate decision
  rather than inheriting OPS2's proposed defaults.
- **OPS2-ADR-C — "etcd / control-plane backup posture: GitOps-reconcile-only vs etcd snapshot
  backups."** *(blast radius: high)* — OPS2 proposes Git/Kargo as the primary control-plane
  recovery source, but non-reconcilable runtime state (e.g. in-flight `AgentRun`, broker offsets)
  may justify etcd snapshots; this decides whether the platform ships an etcd-backup tool.
- **OPS2-ADR-D — "v1.0 cross-region/AZ failover model: backup-restore-only vs active-passive vs
  active-active."** *(blast radius: high)* — `future-enhancements.md` §1 leaves region failover
  open; the choice drives whether S3 cross-region replication, multi-region RDS, and a second
  cluster are in scope, with large cost/complexity implications and a hard kind/AWS asymmetry.
- **OPS2-ADR-E — "Per-tenant restore granularity: whole-cluster restore vs per-tenant point-in-time
  recovery."** *(blast radius: med)* — Recovering one tenant without disturbing others requires
  per-tenant backup boundaries on `XAgentDatabase`/`MemoryStore`; this is a multi-tenancy isolation
  decision (§6.9) that interacts with shared vs dedicated stores.

**Risks / open questions:**
- **R1 (high):** RPO/RTO numbers are `[PROPOSED]` (OPS2-ADR-B). If components are built assuming
  looser targets, the drill targets become unachievable retroactively. Mitigation: ratify
  OPS2-ADR-B before F2 builds drills.
- **R2 (high):** **Secret/key recoverability is owned by OPS3, not OPS2.** If KMS/envelope keys are
  lost, encrypted backups are unrecoverable regardless of OPS2's mechanics. Mitigation: hard
  cross-reference; OPS3 must guarantee key-material survivability as an OPS2 precondition.
- **R3 (med):** **kind has no durable cross-region archive** (no S3; ADR 0041 parity caveat), so
  kind's DR posture is structurally weaker than AWS. Risk: a developer validates DR on kind and
  assumes AWS parity. Mitigation: explicit kind-vs-AWS posture documentation (REQ-OPS2-10).
- **R4 (med):** **Broker / NATS JetStream in-flight state** (A4) and in-flight `AgentRun` are not
  clearly system-of-record-backed; an event backlog may be lost on cluster loss. Mitigation: scope
  decision under OPS2-ADR-C; default posture is "events replay from source, in-flight runs are
  re-derivable, not restored."
- **R5 (med):** **Version-skew restore** (REQ-OPS2-11) depends on conversion webhooks being present
  and correct at restore time; a restore onto a cluster missing a webhook fails. Mitigation: drill
  the `vN-1 → vN` path explicitly (AC-OPS2-11).
- **R6 (low):** No upstream backup-tool versions are pinned in source (`[PROPOSED]`); pin at
  implementation.
- **OQ1:** Does **B19 approval in-flight state** live only in `Approval` CRDs (etcd-backed, hence
  GitOps/etcd recovery) or in a separate backing store needing its own backup? Source does not say;
  classified provisionally as etcd-backed pending B19 confirmation.
- **OQ2:** Is the **DR failover trigger** modelled as a `platform.security.*` event, or is failover
  purely an out-of-band operator action with no event? `[PROPOSED]` — B12/B8 own any schema.

## 11. References
- `_meta/pending-operational-nfr-layer.md` (OPS2 scope; relates ADR 0034, F2, `future-
  enhancements.md` §1).
- `_meta/interface-contract.md` §1.6 (XRD field tables), §4 (connection-secret contract), §5 (audit
  adapter / system of record), §2 (CloudEvent taxonomy — closed set).
- `_meta/glossary.md` (substrate, connection secret, `MemoryStore`, `TenantOnboarding`, XRDs).
- `_meta/waves.md` (F-pieces are `consumer` wave, real-build after A/B; OPS2 is a post-W4
  cross-cutting layer).
- Related specs: `specs/components/B/spec-B4.md` (substrate XRDs + connection secret),
  `specs/components/A/spec-A18.md` (audit Postgres+S3 system of record, OpenSearch rebuild),
  A10 (Letta) / B11 (memory), A21 (`TenantOnboarding`), B19 (`Approval`), A11 (OpenSearch),
  A4 (Knative/NATS), A23 (Kargo), A7 (OPA/Gatekeeper).
- ADRs: 0034 (audit system of record), 0041 (substrate abstraction / connection secret /
  capability-parity), 0014 (Postgres primary), 0009/0033 (OpenSearch dual-hosting), 0028/0037
  (tenant onboarding), 0040 (Kargo), 0030 (versioning/conversion webhooks), 0025/0020/0021 (memory
  access modes / per-scope DBs / dashboards), 0002 (Gatekeeper substrate-match admission).
- Cross-cutting siblings: OPS1 (scale/cost — restore throughput sizing), OPS3 (key/secret
  recoverability — hard precondition), F1 (audit retention), F2 (DR testing — downstream),
  F6 (runbook compilation — downstream).

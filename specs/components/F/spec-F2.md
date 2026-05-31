# SPEC F2 — DR testing

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [A18, A11, B4, A10, A23] · downstream: [] · adrs: [0034, 0044, 0009, 0014, 0026, 0040] · views: [6.3, 6.5]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

F2 is the one-time **disaster-recovery drill** that future-enhancements.md §1 commits to: exercise, on a real substrate, the restore paths for every stateful platform store before v1.0 is declared production-ready. It proves that the platform's stated durability invariants (ADR 0034's Postgres + S3 system of record; ADR 0009/0014's "OpenSearch is rebuildable retrieval optimization, never authoritative"; the ADR 0044 connection-secret contract) actually recover data, not just that they were configured.

F2 does not build backup/restore *automation* and it does not establish an ongoing DR cadence with measured RTO/RPO — both are explicitly deferred to future-enhancements.md §1. F2 is a verification deliverable: a documented, repeatable drill that runs Postgres restore, OpenSearch reindex from primary sources, cloud secret-store recovery, and a full end-to-end platform restore, capturing observed (not contractual) recovery behavior and feeding any gaps into F6 runbooks.

## 2. Scope

### 2.1 In scope
- **Postgres restore drill**: restore the substrate-managed Postgres (`Postgres` — RDS on AWS, CloudNativePG on kind) from its backup/snapshot, including the `audit_events`-bearing instance and per-agent/tenant databases (`AgentDatabase`), verifying the connection-secret contract still resolves post-restore.
- **OpenSearch reindex drill**: rebuild the OpenSearch (`SearchIndex`) advisory index from its primary source — S3 archive on AWS, Postgres on kind — proving ADR 0034's "rebuildable" claim and ADR 0009/0014's advisory-only stance.
- **Secret recovery drill**: recover secrets from the cloud secret store (AWS Secrets Manager via ESO) and confirm ESO re-injection restores `connection secret` material and MCP-service credentials.
- **Full restore drill**: a from-cold rebuild of a cluster's platform state — GitOps re-sync (ArgoCD), stateful-store restore, secret recovery, capability-registry reconciliation — to a working platform, on the ADR 0026 independent-cluster topology.
- **Gap capture**: documented observed recovery behavior + a defect/gap list routed to F6 (runbooks) and future-enhancements.md §1 (automation/cadence).

### 2.2 Out of scope (and where it lives instead)
- **Backup-restore automation + ongoing DR cadence with RTO/RPO targets** → future-enhancements.md §1 (explicitly deferred).
- **Cross-region / cross-AZ replication** → future-enhancements.md §1, §10; ADR 0026 (single region per environment in v1.0).
- **The stateful stores / XRDs themselves** → **B4** (Crossplane Compositions: `Postgres`, `SearchIndex`, `ObjectStore`, `AgentDatabase`, `AuditLog`).
- **Audit retention/redaction policy that governs what is in the archive** → **F1**.
- **Multi-cluster federation / failover** → future-enhancements.md §14 (backlog 3.13).
- **The runbooks themselves** (final form) → **C6 / F6**; F2 produces the *drill record* that F6 verifies has been exercised.

## 3. Context & Dependencies

**Upstream consumed:**
- **A18 (audit endpoint + adapter)** — the `audit_events` Postgres table and the S3 archive + OpenSearch indexer whose restore/rebuild F2 exercises; F2 consumes the durability invariants, does not change them.
- **A11 (OpenSearch)** — the advisory index F2 rebuilds; F2 uses the documented reindex path, not a new one.
- **A10 (Letta memory backend)** — memory store(s) on Postgres/`MemoryStore` whose restore is part of the full drill.
- **B4 (Crossplane Compositions)** — owns `Postgres`, `SearchIndex`, `ObjectStore`, `AgentDatabase`; F2 drives their restore behavior and verifies the connection-secret contract (Canon §4) survives restore.
- **A23 (Kargo)** — the promotion fabric whose Git-sourced desired state drives the GitOps half of the full restore.

**Downstream consumers:** none directly; F6 consumes the drill record to confirm DR runbooks were exercised.

**ADRs honored:**
- **ADR 0034** — Postgres + S3 are the system of record; OpenSearch is rebuildable advisory fanout. F2 *proves* rebuildability and that audit ingestion does not depend on OpenSearch.
- **ADR 0044** — restore must yield the same substrate-agnostic connection-secret shape (`host`,`port`,`user`,`password`,`dbname`) and status (`ready`,`endpoint`,`version`); kind vs AWS recovery paths differ and are documented per the capability-parity caveat (kind `ObjectStore` may have no archive → reindex source is Postgres).
- **ADR 0009 / 0014** — OpenSearch reindex never makes OpenSearch authoritative; primary source is S3 (AWS) / Postgres (kind).
- **ADR 0026** — independent-cluster topology, no v1.0 federation; restore is per-cluster.
- **ADR 0040** — full restore re-syncs Git desired state through ArgoCD/Kargo.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F2 introduces **no new CRD/XRD**. It exercises restore of existing Canon XRDs: `Postgres`, `SearchIndex`, `ObjectStore`, `AgentDatabase`, `AuditLog` (XRD), `MemoryStore` (XR) (Canon §1.6). It asserts post-restore that each XR re-reports `ready`/`endpoint`/`version` and re-writes its connection secret. A declarative `ScheduledTest`/`HealthCheck` XRD for *recurring* drills is **out of scope** (future-enhancements.md §5).

### 4.2 APIs / SDK surfaces
N/A — F2 introduces no API/SDK surface. The drill is orchestrated through the `agent-platform` CLI / test framework (B9/B14) and substrate-native restore tooling (RDS snapshot restore, CloudNativePG restore, ESO resync, ArgoCD sync). `[PROPOSED — not in source]` — the specific CLI subcommand for a DR drill is design-time.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
F2 emits no new event types. A restore drill, if it emits telemetry, falls under `platform.observability.*` (drill start/result) and reads `platform.audit.*` records as the data being restore-verified — both `[PROPOSED — not in source]`. Per-event schemas remain B12-owned.

### 4.4 Data schemas / connection-secret contracts
F2's central assertion is the **connection-secret contract** (Canon §4): after restore, each substrate primitive re-emits `host`,`port`,`user`,`password`,`dbname` (or per-primitive equivalent) and consumers reconnect without code change. It also asserts ADR 0034's verified-write archive (object exists + byte count + checksum) is intact and usable as a reindex source. F2 adds no fields.

## 5. OSS-vs-Custom Decision
**Verification using substrate-native tooling + the platform test harness — no new build.** Restore uses RDS automated snapshots / point-in-time restore (AWS), CloudNativePG backup restore (kind), ESO resync against AWS Secrets Manager, ArgoCD re-sync, and OpenSearch's documented reindex. Orchestration is **config/wrap** of B9/B14 (drill scripts as test cases). No fork, no new service. Rationale: future-enhancements.md §1 scopes F2 as a *one-time drill*, not automation; building restore machinery would pre-empt the deferred automation work.

## 6. Functional Requirements
- **REQ-F2-01:** F2 MUST execute a Postgres restore of the `audit_events`-bearing instance (and at least one `AgentDatabase`) from substrate backup, and verify row-level data recovery against a pre-drill fixture.
- **REQ-F2-02:** After Postgres restore, the `Postgres`/`AgentDatabase` connection secret MUST resolve to the Canon §4 shape and dependent components MUST reconnect without config change.
- **REQ-F2-03:** F2 MUST rebuild the OpenSearch advisory index from its primary source — S3 on AWS, Postgres on kind — and verify the rebuilt index matches the source record set.
- **REQ-F2-04:** F2 MUST demonstrate that audit ingestion succeeds while OpenSearch is unavailable/rebuilding (ADR 0034 advisory-only invariant).
- **REQ-F2-05:** F2 MUST recover platform secrets from the cloud secret store via ESO and confirm re-injection into the consuming workloads (gateway, agents, MCP services).
- **REQ-F2-06:** F2 MUST perform a full from-cold platform restore on a cluster: ArgoCD/Kargo Git re-sync + stateful-store restore + secret recovery + capability-registry reconciliation, ending in a serviceable platform.
- **REQ-F2-07:** F2 MUST document, per substrate (kind/AWS), the observed recovery path and any divergence driven by the capability-parity caveat (e.g., kind reindex-from-Postgres because no S3 archive).
- **REQ-F2-08:** F2 MUST produce a drill record (steps, observed outcomes, defects/gaps) routed to F6 (runbook verification) and future-enhancements.md §1 (automation/cadence backlog), with no measured RTO/RPO commitment.
- **REQ-F2-09:** The drill MUST be repeatable (scripted via B9/B14) so it can be re-run, even though scheduling it is out of scope.

## 7. Non-Functional Requirements
- **Security:** secret-recovery drill MUST NOT print recovered secret material to logs/drill records; only success/failure of re-injection is recorded. Restored stores MUST re-apply RBAC/OPA gating before being declared serviceable.
- **Multi-tenancy (§6.9):** restore MUST preserve tenant/namespace boundaries; an `AgentDatabase` restore MUST land in the owning tenant's namespace with original `ownerRef`.
- **Observability (§6.5):** drill steps MUST be observable (trace/metric/log) so a failed restore is diagnosable; archive-lag / reindex-progress signals reused from F1 where present.
- **Scale:** reindex drill SHOULD be run against a realistic record count (coordinated with F5 volume numbers) so rebuild time is representative, even though the *time* is recorded informationally only (no RTO target in v1.0).
- **Versioning (ADR 0030):** restore MUST tolerate a stored backup taken at the then-current CRD/XRD version; if a version skew is found, it is logged as a gap (conversion-webhook coverage), not silently reconciled.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A — F2 ships no deployable; it scripts restore of existing-deployed stores.**
- Per-product docs (10.5) — **applicable** (DR drill procedure per store; substrate divergence notes).
- Runbook (10.7) — **applicable** (Postgres restore, OpenSearch reindex, secret recovery, full restore runbooks). Primary feed into **F6**.
- Alerts — **N/A — verification deliverable; restore-failure alerting is part of the deferred automation (future §1).** `[PROPOSED]` to surface drill-result as an alert.
- Grafana dashboard (Crossplane XR) — **N/A — one-time drill; ongoing DR dashboards are deferred (future §1, §3).**
- Headlamp plugin — **N/A — restore is operator-run procedure, not an interactive CRD surface.**
- OPA/Rego integration — **N/A — F2 verifies restored stores re-apply existing OPA gating; it adds no policy.**
- Audit emission (ADR 0034) — **applicable** — F2 *verifies* audit durability/recovery; a drill-execution audit record is `[PROPOSED]`.
- Knative trigger flow — **N/A — drill is operator-initiated, not event-driven.**
- HolmesGPT toolset — **applicable** — `[PROPOSED]` read tool reporting last-drill status / restore health during triage.
- 3-layer tests — **applicable** (Chainsaw for XR restore/ready assertions; PyTest for data-equivalence and connection-secret checks; Playwright N/A).
- Tutorials & how-tos — **applicable** (how-to: run the DR drill; how-to: rebuild the OpenSearch index from primary source).

## 9. Acceptance Criteria
- **AC-F2-01:** A Postgres instance is restored from backup and a known pre-drill row set is present post-restore. (→ REQ-F2-01)
- **AC-F2-02:** Post-restore, a consuming component reconnects using only the standard connection secret, no config edit. (→ REQ-F2-02)
- **AC-F2-03:** The OpenSearch index is deleted and rebuilt from S3 (AWS) / Postgres (kind); rebuilt doc count matches source. (→ REQ-F2-03)
- **AC-F2-04:** With OpenSearch down, a new audit event is still durably written to Postgres (+ S3 on AWS). (→ REQ-F2-04)
- **AC-F2-05:** A deleted/rotated secret is recovered via ESO and the consuming workload returns to ready. (→ REQ-F2-05)
- **AC-F2-06:** A cluster restored from cold (Git re-sync + store restore + secrets) serves an end-to-end agent request. (→ REQ-F2-06)
- **AC-F2-07:** The drill record documents kind and AWS recovery paths and notes the no-S3-archive kind divergence. (→ REQ-F2-07)
- **AC-F2-08:** A gap/defect list exists and is referenced from F6 and future-enhancements.md §1. (→ REQ-F2-08)
- **AC-F2-09:** The full drill can be re-invoked from the B9/B14 harness and reproduces the same pass/fail outcomes. (→ REQ-F2-09)

## 10. Risks & Open Questions
- **R1 (high):** A restore path may simply not work (e.g., reindex assumptions wrong, snapshot policy never enabled). That is the *point* of F2 — failure here is a finding, but late discovery is high blast radius. Mitigation: run F2 against AWS substrate early enough to fix gaps before GA. Blast radius: high (production durability).
- **R2 (med):** kind/AWS divergence means a drill green on kind may not exercise the S3-archive reindex or RDS restore at all. Mitigation: AWS-substrate run is mandatory for AWS-only paths (overlaps F1 R3). Blast radius: med.
- **R3 (med):** Secret-recovery drill could leak secret material into the drill record. Mitigation: NFR forbids printing material; assert on workload-ready, not on value. Blast radius: med (security).
- **R4 (low):** Backup taken at one CRD/XRD version, restored under a newer version → conversion-webhook gap. Mitigation: log as versioning gap (ADR 0030). Blast radius: low.
- **OQ1:** Should the full-restore drill target a throwaway cluster or a staging Stage (ADR 0040)? `[PROPOSED]` throwaway/ephemeral cluster to avoid perturbing a live Stage.
- **OQ2:** Is a measured RTO/RPO even informally recorded in v1.0, or strictly out of scope? `[PROPOSED]` record observed timings informationally only; no target committed (future §1).

## 11. References
- architecture-overview.md §6.3 (data architecture / Postgres-primary, audit dual-mode), §6.5 (observability/audit emission points), §14.6 line 1758 (F2 scope), §6.7 (eventing — OpenSearch-independent audit).
- future-enhancements.md §1 (one-time DR drill is F2; automation + RTO/RPO + cross-region deferred), §5 (recurring tests deferred), §14 (multi-cluster federation deferred).
- Canon interface-contract §1.6 (`Postgres`/`SearchIndex`/`ObjectStore`/`AgentDatabase`/`AuditLog`/`MemoryStore`), §4 (connection-secret contract), §5 (audit adapter durability invariants), §2 (`platform.observability.*`).
- ADR 0034 (system of record + rebuildable OpenSearch + verified-write), ADR 0044 (substrate abstraction + connection secret), ADR 0009/0014 (OpenSearch advisory), ADR 0026 (independent cluster), ADR 0040 (Kargo/GitOps re-sync).
- Related pieces: A18, A11, A10, B4, A23, F1, F6.

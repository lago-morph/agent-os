# PLAN F2 — DR testing

> spec: SPEC-F2 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes A/B; runs at end of v1.0) · estimate: M
> upstream-pieces: [A18, A11, B4, A10, A23] · downstream-pieces: []

## 1. Implementation Strategy

F2 is a one-time, scripted verification drill — no automation, no cadence (both deferred to future §1). The approach: build repeatable drill scripts on the B9/B14 harness, one per store-class (Postgres restore, OpenSearch reindex, secret recovery), then compose them into a full from-cold restore on a throwaway cluster, and capture observed behavior + gaps. Substrate matters: AWS-specific paths (RDS restore, S3-archive reindex) MUST be exercised on AWS; kind exercises the no-S3 divergence (reindex-from-Postgres). Critical path: pre-drill fixtures + backups confirmed → individual restore drills (parallel) → full-restore composition → drill record + gap routing to F6/future-§1.

## 2. Ordered Task List

- **TASK-01:** Pre-drill fixtures — seed known row sets in Postgres (`audit_events` + an `AgentDatabase`), index docs in OpenSearch, plant secrets; confirm backups/snapshots exist — produces: drill fixtures + backup baseline — depends-on: [].
- **TASK-02:** Postgres restore drill — restore `Postgres`/`AgentDatabase` from snapshot; assert row recovery + connection-secret re-resolution + reconnect-without-config — produces: Postgres restore script + record — depends-on: [TASK-01].
- **TASK-03:** OpenSearch reindex drill — delete index, rebuild from S3 (AWS) / Postgres (kind); assert doc-count match; assert audit ingestion succeeds while OpenSearch down — produces: reindex script + record — depends-on: [TASK-01].
- **TASK-04:** Secret recovery drill — rotate/delete a secret, recover via ESO from AWS Secrets Manager, assert workload returns ready; no secret material in record — produces: secret-recovery script + record — depends-on: [TASK-01].
- **TASK-05:** Full from-cold restore — ArgoCD/Kargo Git re-sync + store restore (TASK-02/03) + secret recovery (TASK-04) + capability-registry reconcile → serviceable platform serving an e2e agent request — produces: full-restore script + record — depends-on: [TASK-02, TASK-03, TASK-04].
- **TASK-06:** Substrate-divergence documentation — kind (no S3) vs AWS recovery paths; version-skew (backup vs current CRD/XRD) gaps — produces: substrate/divergence notes — depends-on: [TASK-02, TASK-03].
- **TASK-07:** Drill record + gap register — observed outcomes, defects, informational timings (no RTO/RPO commit); route to F6 + future-§1 — depends-on: [TASK-05, TASK-06].
- **TASK-08:** Runbooks (Postgres restore / OpenSearch reindex / secret recovery / full restore) + how-tos; proposed HolmesGPT last-drill-status read tool — depends-on: [TASK-07].
- **TASK-09:** 3-layer tests (Chainsaw XR restore/ready; PyTest data-equivalence + connection-secret + leak-scan) mapping ACs — depends-on: [TASK-02, TASK-03, TASK-04, TASK-05].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A18** — audit stores (Postgres `audit_events`, S3 archive, OpenSearch indexer) restore/rebuild targets.
- **A11** — OpenSearch reindex path. **A10** — memory store(s) in the full restore.
- **B4** — `Postgres`/`SearchIndex`/`ObjectStore`/`AgentDatabase`/`AuditLog`/`MemoryStore` restore behavior + connection-secret contract.
- **A23** — Kargo/ArgoCD Git desired-state for the GitOps half of full restore.

### 3.2 Downstream blocked on this
- None (terminal). F6 consumes the drill record to confirm DR runbooks exercised.

### 3.3 Continuous (non-blocking) inputs
- **B14** harness (Chainsaw/PyTest). **F5** realistic record counts for reindex sizing. **F1** retention/archive config (what is in the archive). **F4** overlaps on secret-recovery security posture.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-04 are an independent fan-out (Postgres / OpenSearch / secrets).
- TASK-06 (divergence docs) parallels once TASK-02/03 land.
- TASK-08 (runbooks) and TASK-09 (tests) parallelize after the drills.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-F2-01 (restore → rows present), AC-F2-03 (index rebuilt, count match), AC-F2-05 (secret recovered → workload ready), AC-F2-06 (full restore → e2e request) — XR `ready`/`endpoint` assertions.
- **PyTest (logic):** AC-F2-02 (reconnect via connection secret, no config edit), AC-F2-04 (audit durable while OpenSearch down), AC-F2-07 (divergence doc), AC-F2-08 (gap register), secret-leak scan of the drill record.
- **Playwright:** N/A — operator-run procedure, no UI surface (minimal e2e request check folds into Chainsaw AC-F2-06).
- **Fixtures/fakes:** AWS substrate required for RDS-restore + S3-reindex (kind cannot exercise — shared AWS fixture with F1); throwaway/ephemeral cluster for full restore (OQ1); OpenSearch-down injector; CRD-version-skew backup fixture.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (A18, A11, A10, B4, A23 merged to main).
### 6.2 PR — `piece/F2-dr-testing` → base `wave/F`; carries SPEC-F2 + PLAN-F2.
### 6.3 Merge order — independent of F-siblings; F6 references F2's drill record post-merge.

## 7. Effort Estimate
- Fixtures + individual drills (TASK-01..04): M (AWS-substrate setup is the bulk).
- Full restore (TASK-05): M (composition + ephemeral cluster).
- Docs + record + tests (TASK-06..09): S.
- **Rollup: M** (matches CSV). **Critical path:** TASK-01 → 02/03/04 → 05 → 07.

## 8. Rollback / Reversibility
F2 is a drill — it makes no lasting platform change; rollback is tearing down the throwaway cluster and discarding drill fixtures. The drill scripts (B14 test cases) and the drill record persist as artifacts. **No downstream break** on revert (F2 is terminal); reverting only removes the verification evidence, which would re-open the production-readiness gate.

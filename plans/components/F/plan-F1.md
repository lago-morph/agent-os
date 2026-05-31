# PLAN F1 — Audit retention policy

> spec: SPEC-F1 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes A/B; runs at end of v1.0) · estimate: S
> upstream-pieces: [A18, B4] · downstream-pieces: []

## 1. Implementation Strategy

F1 is a policy-attachment piece, not a build: it sets concrete values on surfaces ADR 0034 already exposed (S3 lifecycle on the `AuditLog` XRD, OpenSearch ISM rollover) and config on the A18 adapter (redaction). The approach is compliance-first — derive the retention durations from a compliance review, then express them as Git-reconciled config (XR values, OpenSearch ISM policy, a proposed Postgres prune CronJob), then verify against both substrates (kind = Postgres-only system of record; AWS = full S3 archive path). Critical path: compliance review (sets the `[PROPOSED]` numbers) → encode as config → dual-substrate verification → redaction + durability-invariant tests.

## 2. Ordered Task List

- **TASK-01:** Compliance review — set retention durations per tier (Postgres, S3 archive, OpenSearch), record legal-hold gap as out-of-scope — produces: retention-duration table + compliance mapping — depends-on: [].
- **TASK-02:** Redaction strategy — name sensitive field classes + redaction stage (emission/archive/index) + irreversibility note — produces: redaction matrix — depends-on: [].
- **TASK-03:** Encode S3 lifecycle (transition + expiry) into the `AuditLog` XR `s3BucketRef` lifecycle (AWS only) — produces: AuditLog XR values — depends-on: [TASK-01]. (B4 XRD available.)
- **TASK-04:** Encode OpenSearch ISM rollover + delete policy; assert rebuildable-from-source — produces: ISM policy — depends-on: [TASK-01].
- **TASK-05:** Proposed Postgres standalone retention (kind system-of-record) — prune CronJob / window, gated on no-S3 path — produces: Postgres prune job — depends-on: [TASK-01].
- **TASK-06:** Apply redaction config to the A18 adapter/endpoint (single write path) — produces: adapter redaction config — depends-on: [TASK-02].
- **TASK-07:** Dual-substrate verification — AWS archive transition/expiry + kind Postgres-only no-batch-delete — produces: substrate-parity record — depends-on: [TASK-03, TASK-04, TASK-05].
- **TASK-08:** Durability-invariant test — simulate S3 verify-failure, assert no Postgres deletion (ADR 0034) — produces: invariant test — depends-on: [TASK-03].
- **TASK-09:** Cross-cutting — proposed retention-floor OPA guard, archive-lag/batch-stall alert + dashboard XR, HolmesGPT retention-posture read tool — depends-on: [TASK-07].
- **TASK-10:** 3-layer tests + how-tos (change a retention duration; add a redaction rule); runbook (retention change, batch-stall response) feeding F6 — depends-on: [TASK-06, TASK-07, TASK-08].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A18** — `audit_events` table, 5-min batch CronJob, OpenSearch indexer, single adapter write path. Consumed: the schema + write path F1 governs.
- **B4** — `AuditLog` XRD composing `Postgres`/`ObjectStore`. Consumed: the XR fields (lifecycle, batchScheduleSpec) F1 sets values into.

### 3.2 Downstream blocked on this
- None (terminal). F2 (DR reindex/restore) and F6 (runbook) reference F1 outputs.

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework (Chainsaw/PyTest). **F5** scale numbers (ISM/retention sizing). **B22** threat model (audit-integrity targets). Compliance reviewer sign-off.

## 4. Parallelizable Subtasks
- TASK-01 and TASK-02 are independent (durations vs redaction) and can run concurrently.
- After TASK-01: TASK-03, TASK-04, TASK-05 are an independent fan-out (S3 / OpenSearch / Postgres tiers).
- TASK-06 (redaction config) parallels the tier-encoding tasks.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-F1-02 (`AuditLog` XR lifecycle on AWS / absent on kind), AC-F1-03 (ISM rollover+delete then rebuild), AC-F1-04 (kind Postgres window, no batch-delete).
- **PyTest (logic):** AC-F1-05 (redaction stage + absence from archive), AC-F1-08 (verify-failure leaves Postgres intact), AC-F1-07 (retention change is itself audited), Postgres prune logic.
- **Playwright:** N/A — F1 has no UI surface (Headlamp editor is N/A).
- **Fixtures/fakes:** AWS-substrate fixture for S3 lifecycle (kind cannot exercise it — share with F2); a sensitive-field fixture for redaction; an S3-verify-failure injector.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (production-readiness; A18, B4 already merged to main).
### 6.2 PR — `piece/F1-audit-retention-policy` → base `wave/F`; carries SPEC-F1 + PLAN-F1.
### 6.3 Merge order — independent of F-siblings; F2/F6 reference but do not block on F1 merging first.

## 7. Effort Estimate
- Compliance review + redaction (TASK-01,02): S (analysis-bound, gated on reviewer).
- Config encoding (TASK-03..06): S.
- Verification + cross-cutting + tests (TASK-07..10): S.
- **Rollup: S** (matches CSV). **Critical path:** TASK-01 → 03/04 → 07 → 10.

## 8. Rollback / Reversibility
Back out by reverting the retention/redaction config commit (Git/ArgoCD) — the `AuditLog` XR reverts to its pre-F1 (B4 default) values, ISM policy is removed, redaction config is dropped. Non-destructive for already-archived S3 objects (immutable). **Caution:** redaction is irreversible for objects already written under it; rollback restores future behavior, not redacted historical data. No downstream component breaks (F1 is terminal).

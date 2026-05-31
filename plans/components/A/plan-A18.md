# PLAN A18 — Audit endpoint + audit adapter library

> spec: SPEC-A18 · kind: COMPONENT · tier: T0
> wave: W1 · estimate: L
> upstream-pieces: [A11, A7] · downstream-pieces: []

## 1. Implementation Strategy

A18 is the most-depended-on audit surface, so the **adapter interface is frozen first** (interface-contract §5) and published before any other component emits — every emission point (LiteLLM, Gatekeeper, Envoy, Knative, ARK, agent-sandbox, ESO, ArgoCD, LibreChat, Headlamp, approval, Coach, HolmesGPT, B13) links against it. Build adapter-contract-first → audit endpoint + `audit_events` migrations (the in-flight system of record) → async OpenSearch advisory indexer (must be droppable without breaking ingestion) → the AWS-only S3 batch CronJob (aggregate → verify exists+bytes+checksum → only-then-delete) → the `AuditLog` Crossplane composition that provisions the whole pipeline across substrates (kind produces no archive by design). The fail-closed-on-Postgres / survive-OpenSearch-down / retain-and-retry-on-S3-verify-fail behaviors are the core correctness tests. Mock B4's `Postgres`/`ObjectStore`/`SearchIndex` and the `LogLevel` reconciler until they land.

## 2. Ordered Task List

- **TASK-01:** Freeze + publish the **audit adapter interface** (Python `emit`, CloudEvent envelope, fail-closed semantics, versioning) — produces: adapter contract doc + library skeleton — depends-on: [] (interface-contract §5).
- **TASK-02:** Implement the adapter library (post-to-endpoint, `platform.audit.*` envelope, bounded retry, fail-closed surfacing) — produces: Python adapter package — depends-on: [TASK-01].
- **TASK-03:** `audit_events` schema + versioned migrations — produces: migration set — depends-on: [TASK-01].
- **TASK-04:** Audit endpoint Deployment (ingest route, write Postgres, honor own `LogLevel`) — produces: endpoint service — depends-on: [TASK-02, TASK-03]. (Mock `LogLevel` reconciler.)
- **TASK-05:** Async OpenSearch advisory indexer (non-blocking; rebuildable) — produces: indexer service — depends-on: [TASK-04] (A11 available).
- **TASK-06:** S3 batch CronJob (AWS only): aggregate → write immutable object → verify exists+bytes+checksum → delete rows; retain+retry on failure — produces: batch job — depends-on: [TASK-04]. (Mock `ObjectStore`.)
- **TASK-07:** `AuditLog` Crossplane composition (compose `Postgres` + `ObjectStore`; provision indexer, CronJob (AWS), endpoint; kind no-archive path) — produces: `AuditLog` XRD/composition — depends-on: [TASK-04, TASK-05, TASK-06] (B4 XRDs). 
- **TASK-08:** Envoy egress wiring for endpoint→S3 / endpoint→managed-OpenSearch (ADR 0003) — produces: egress config — depends-on: [TASK-05, TASK-06].
- **TASK-09:** OPA Rego — `AuditLog` admission + endpoint access; A18 self-audit of privileged actions — produces: A18 Rego + self-audit — depends-on: [TASK-04] (B3 contract).
- **TASK-10:** Cross-cutting — `GrafanaDashboard` XR (ingest rate, in-flight size, batch lag, verify failures, indexer lag), alerts, runbook (Postgres-down, S3-verify-backlog, OpenSearch-rebuild, LogLevel-raise), HolmesGPT audit-query tool — depends-on: [TASK-07, TASK-09].
- **TASK-11:** 3-layer tests mapping all ACs — depends-on: [TASK-07, TASK-09].
- **TASK-12:** Tutorials & how-tos ("emit audit from your component using the adapter") — depends-on: [TASK-02, TASK-10].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A11 (OpenSearch)** — advisory indexer target (kind in-cluster / AWS-managed).
- **A7 (OPA/Gatekeeper)** — `AuditLog` admission + endpoint access policy.
- **B4 (Crossplane Compositions)** — `Postgres`/`ObjectStore`/`SearchIndex` the `AuditLog` XRD composes. `[PROPOSED]` hard dep (not in CSV upstream). Mockable until landed.

### 3.2 Downstream blocked on this
- None in CSV, but **every audit-emitting component** (§6.6 list) links the adapter — A18's adapter contract gates all of them. ADR 0035 `LogLevel` consumers.

### 3.3 Continuous (non-blocking) inputs
- **B12** CloudEvent schema registry (`platform.audit.*` per-type schemas). **A4/B8** Knative (if audit events arrive via broker). **F1** retention policy (attaches to S3 lifecycle / index rollover later). **B14** test framework. **B22** threat model (audit integrity is a named asset).

## 4. Parallelizable Subtasks
- After TASK-04: TASK-05 (indexer), TASK-06 (batch), TASK-09 (Rego/self-audit) are independent fan-out.
- TASK-10 (cross-cutting) and TASK-12 (docs) parallelize once the pipeline + adapter exist.
- TASK-02 (adapter) and TASK-03 (migrations) parallelize after TASK-01.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-A18-08 (single `AuditLog` claim provisions pipeline, kind+AWS), -04 (kind no-S3/no-CronJob), -12 (`AuditLog` admission rejects malformed), -13 (migrations idempotent).
- **PyTest (logic):** AC-A18-01 (no direct store writes), -02 (row written), -03 (batch verify-then-delete + verify-fail-retain), -05 (OpenSearch-down ingest succeeds + rebuild), -06 (Postgres-down fail-closed), -07 (LogLevel raise no redeploy), -09 (envelope fields), -10 (version/deprecation).
- **Playwright (UI/e2e):** Headlamp audit-inspection view over the advisory index.
- **Fakes/fixtures:** mock `Postgres`/`ObjectStore`/`SearchIndex` (B4), mock `LogLevel` reconciler, a stoppable OpenSearch + Postgres for failure-mode tests. AWS-only S3-verify-then-delete needs a cloud integration fixture (kind cannot exercise the archive path — ADR 0023/0034 caveat).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A11, A7 specs A18 builds on).
### 6.2 PR — `piece/A18-audit-endpoint-adapter` → base `wave/1`; carries SPEC-A18 + PLAN-A18.
### 6.3 Merge order — independent of W1 siblings; **publish the adapter contract (TASK-01) early** so downstream wave authors bind to it. wave/1 rolls up to main.

## 7. Effort Estimate
- Adapter + endpoint + migrations (TASK-01..04): M (the fail-closed/versioned contract is exacting but small).
- Indexer + batch + composition (TASK-05..08): M (S3 verify-then-delete correctness + dual-substrate composition are the bulk).
- Rego + cross-cutting + tests + docs (TASK-09..12): M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 (freeze contract) → 02 → 04 → 06 (S3 verify-then-delete) → 07 (`AuditLog` composition) → 11.

## 8. Rollback / Reversibility
Back out by deleting the `AuditLog` claim (deprovisions endpoint/indexer/CronJob) and unpinning the adapter library. **Destructive caution:** deleting the Postgres store / S3 bucket loses the system of record — drain/archive first. **Downstream break:** every component linking the adapter loses its audit path; with the endpoint gone, callers **fail closed on Postgres-unavailable** (no silent audit loss, by design) — so a careless rollback halts audited operations platform-wide. Reverting the adapter contract is the highest-blast-radius action: pin a prior adapter version and keep the endpoint backward-compatible across one release (ADR 0030).

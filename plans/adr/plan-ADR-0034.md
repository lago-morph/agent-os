# PLAN ADR-0034 — Audit pipeline — durable adapter, Postgres + S3 system of record, OpenSearch advisory only `[PROPOSED]`

> spec: SPEC-ADR-0034 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B4; A11; A7] · downstream-pieces: [A18; B4; A23; B12]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0034 is enforced primarily at **A18**, which builds the single
audit adapter library and the audit endpoint and is the natural gate for the "no direct writes /
single emission path" invariant; at **B4**, which authors the `AuditLog` Composition realizing the
Postgres+S3 SoR + OpenSearch-advisory topology with kind graceful degradation; and at the
architecture-CI layer, which asserts no component imports a storage client for audit. The durability
cycle (verify-then-delete), the OpenSearch-rebuildable property, and the `platform.audit.*` namespace
boundary are each verified by targeted tests. The endpoint's `LogLevel` toggle (ADR 0035) and Envoy
egress (ADR 0003) are honored by reusing those mechanisms, not re-inventing them.

## 2. Ordered Task List
- TASK-01: Encode the single-emission-path invariant as a code/CI conformance check (no Postgres/S3/OpenSearch client imported for audit) — produces: conformance rule — depends-on: []
- TASK-02: Verify post-to-endpoint → row-in-Postgres `audit_events` — produces: PyTest ingestion test — depends-on: [TASK-01]
- TASK-03: Verify AWS durability cycle: S3 object verified (exists+bytes+checksum) before Postgres delete; failure retains+retries — produces: PyTest/Chainsaw batch test — depends-on: [TASK-02]
- TASK-04: Verify kind degraded mode: Postgres-only SoR, batch disabled, no S3 — produces: Chainsaw kind-mode test — depends-on: [TASK-02]
- TASK-05: Verify OpenSearch advisory: ingest survives OpenSearch outage; index rebuilds from S3/Postgres — produces: PyTest fault-injection test — depends-on: [TASK-02]
- TASK-06: Verify `platform.audit.*` namespace carriage + distinctness from `platform.security.*` — produces: eventing conformance test — depends-on: [TASK-01]
- TASK-07: Verify one `AuditLog` XR provisions endpoint + Postgres + (AWS) S3+lifecycle+indexer+batch — produces: Chainsaw XRD test — depends-on: []
- TASK-08: Verify endpoint `LogLevel` toggle raises verbosity without caller redeploy; egress traverses Envoy — produces: integration test — depends-on: [TASK-02]
- TASK-09: Encode the D-05 freeze-gate: assert the audit-adapter interface + `audit_events` schema are frozen and fail any pipeline emitting audit events before the gate passes — produces: CI freeze-gate check — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B4 — Crossplane Compositions (`XPostgres`, `XObjectStore` that `AuditLog` composes).
- A11 — OpenSearch (advisory fanout target).
- A7 — OPA/Gatekeeper (admits the XR only on matching substrate Composition).
### 3.2 Downstream pieces blocked on this
- A18 (adapter + endpoint build), B4 (`AuditLog` Composition), A23 (Kargo XR promotion), B12 (`platform.audit.*` schemas).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (fault-injection harness); F1 (retention policy — attaches to S3 lifecycle later); ADR 0035 toggle mechanism.

## 4. Parallelizable Subtasks
TASK-07 runs concurrently with TASK-01. After TASK-02: TASK-03, TASK-04, TASK-05, TASK-08 fan out independently. TASK-06 runs after TASK-01.

## 5. Test Strategy
- AC-01 → conformance check (TASK-01, PyTest/conftest over imports).
- AC-02 → PyTest ingestion (TASK-02).
- AC-03, AC-04 → PyTest/Chainsaw batch durability + failure-retry (TASK-03); fake S3 (MinIO) fixture until AWS available.
- AC-05 → Chainsaw kind-mode (TASK-04).
- AC-06 → PyTest fault-injection on OpenSearch (TASK-05).
- AC-07 → eventing conformance (TASK-06).
- AC-08 → Chainsaw `AuditLog` XRD provisioning (TASK-07); fake B4 Compositions until B4 lands.
- AC-09, AC-10 → integration test on endpoint toggle + Envoy egress (TASK-08).
- AC-11 → freeze-gate (D-05): CI check that the audit-adapter interface + `audit_events` schema are marked frozen and that any pipeline emitting audit events before the gate passes fails (TASK-09).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0034-audit-pipeline` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 S · TASK-05 M · TASK-06 S · TASK-07 M · TASK-08 S. Rollup M. Critical path: TASK-01 → TASK-02 → TASK-03.

## 8. Rollback / Reversibility
Reverting the single-emission-path conformance gate re-permits direct-to-store audit writes,
breaking the reproducibility invariant and re-coupling every component to storage. Reverting the
`AuditLog` XRD forces parallel manifest provisioning per substrate. No audit-data migration to back
out — the artifacts are conformance rules, the Composition, and tests. In-flight Postgres rows are
the durability handle during any rollback.

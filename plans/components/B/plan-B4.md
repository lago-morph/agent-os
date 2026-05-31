# PLAN B4 — Crossplane v2 Compositions

> spec: SPEC-B4 · kind: COMPONENT · tier: T0
> wave: W1 · estimate: XL
> upstream-pieces: [] · downstream-pieces: [A18, A21, A23, B19, B11]

## 1. Implementation Strategy
Build the substrate-abstraction layer bottom-up: first the four **substrate XRDs** (`Postgres`,
`SearchIndex`, `ObjectStore`, `MongoDocStore`) each with a kind + AWS Composition writing the
**same connection-secret shape** and substrate-agnostic status (the tested invariant, ADR 0044);
then the **higher-level XRDs** that compose them (`AuditLog`, `AgentDatabase`, `GrafanaDashboard`,
`TenantOnboarding`); then the **agent-facing composed XRs** (`MemoryStore`, `AgentEnvironment`,
`SyntheticMCPServer`). The connection-secret conformance harness and the label-driven
Composition-selection + admission-rejection requirement are first-class deliverables, not
afterthoughts. Backing operators for kind (CloudNativePG / OpenSearch operator / MinIO / Bitnami
MongoDB) are installed as prerequisites; AWS provider-stack selection is install-time config and is
NOT wrapped (ADR 0044 exception). Everything is built kind-first so the whole stack is exercisable
without cloud, then the AWS Composition is added per XRD for parity.
All XRDs are namespace-scoped composite resources (Crossplane v2: single XR layer, no claims, no `X`-prefix). The `AgentEnvironment` Composition emits `ResourceQuota`+`LimitRange` from `quotas`, and `TenantOnboarding` admission requires `cpu`/`memory` quota entries present (concrete or `unlimited: true`) — quota presence is mandatory (#5/#9; coordinate with A21).

## 2. Ordered Task List
- **TASK-01:** Pin Crossplane v2 + author the XRD package scaffold + `platform.io/environment`
  selection convention — produces: Crossplane package skeleton, `substrateClass` mapping — depends-on: []
- **TASK-02:** Freeze the **connection-secret per-primitive mapping table** (host/port/user/password/
  dbname + per-primitive equivalents) and the substrate-agnostic status contract (`ready`/`endpoint`/
  `version` + `status.substrateDetail` "do-not-depend") — produces: secret-shape contract doc —
  depends-on: [TASK-01]
- **TASK-03:** Install kind backing operators (CloudNativePG, OpenSearch operator, MinIO, Bitnami
  MongoDB) as prerequisites — produces: kind prereq manifests — depends-on: [TASK-01]
- **TASK-04:** `Postgres` XRD + kind Composition (CloudNativePG) + AWS Composition (RDSInstance),
  both writing the secret shape — produces: `Postgres` XRD+2 Compositions — depends-on: [TASK-02, TASK-03]
- **TASK-05:** `SearchIndex` XRD + kind (OpenSearch operator) + AWS (managed OpenSearch) Compositions
  — produces: `SearchIndex` XRD+2 Compositions — depends-on: [TASK-02, TASK-03]
- **TASK-06:** `ObjectStore` XRD + kind (MinIO/no-op) + AWS (S3) Compositions; document "no archive"
  on kind — produces: `ObjectStore` XRD+2 Compositions — depends-on: [TASK-02, TASK-03]
- **TASK-07:** `MongoDocStore` XRD + kind (Bitnami MongoDB) + AWS (DocumentDB/self-managed)
  Compositions — produces: `MongoDocStore` XRD+2 Compositions — depends-on: [TASK-02, TASK-03]
- **TASK-08:** **Connection-secret conformance harness** — reads the secret from each substrate XRD's
  kind AND AWS Composition, asserts identical shape (the tested invariant) — produces: conformance
  test suite — depends-on: [TASK-04, TASK-05, TASK-06, TASK-07]
- **TASK-09:** Substrate-mismatch **admission requirement + input shape** handed to A7/B16; verify a
  no-matching-Composition XR is rejected at admission — produces: admission contract + Chainsaw
  test — depends-on: [TASK-04..TASK-07]
- **TASK-10:** `AuditLog` XRD composing `Postgres`+`ObjectStore`+indexer + (AWS) batch
  CronJob + endpoint Deployment; kind degrades Postgres-only — produces: `AuditLog` XRD+Compositions —
  depends-on: [TASK-04, TASK-06]
- **TASK-11:** `AgentDatabase` XRD composing `Postgres`|`MongoDocStore` per
  `engine` — produces: `AgentDatabase` XRD+Compositions — depends-on: [TASK-04, TASK-07]
- **TASK-12:** `GrafanaDashboard` XRD with RBAC+OPA `visibility` — produces:
  `GrafanaDashboard` XRD+Compositions — depends-on: [TASK-01]
- **TASK-13:** `TenantOnboarding` XRD (namespaces/SAs/OIDC mapping; CapabilitySets NOT coupled) —
  produces: `TenantOnboarding` XRD+Compositions — depends-on: [TASK-01]
- **TASK-14:** Agent-facing composed XRs `MemoryStore` (accessMode), `AgentEnvironment`,
  `SyntheticMCPServer` (mcpServerRef back-link) — produces: 3 composed XRs — depends-on: [TASK-04, TASK-05, TASK-06]
- **TASK-15:** XRD **versioning wiring** (ADR 0030): conversion-webhook + deprecation-window pattern on
  both Compositions for a sample XR schema change — produces: versioning pattern + test — depends-on:
  [TASK-04]
- **TASK-16:** Per-XRD **substrate-difference docs**, runbook (stuck-reconcile/drift/secret-mismatch),
  operability `GrafanaDashboard` XR, alerts — produces: docs+runbook+dashboard+alerts — depends-on:
  [TASK-10..TASK-14]
- **TASK-17:** 3-layer tests rollup (Chainsaw XR→Composition→secret; PyTest conformance) + how-tos —
  produces: test suite + tutorials — depends-on: [TASK-08, TASK-09, TASK-15]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None as build predecessors. **Install-time prerequisites** (not pieces): Crossplane v2; kind backing
  operators (CloudNativePG/OpenSearch operator/MinIO/Bitnami MongoDB); AWS provider stack (install-time
  config, NOT wrapped — ADR 0044).
### 3.2 Downstream pieces blocked on this
- A18 (consumes `AuditLog`/`AuditLog`), A21 (`TenantOnboarding`), A23 (promotes XRs), B19
  (datastores via connection-secret contract), B11 (`MemoryStore` + secret).
### 3.3 Continuous (non-blocking) inputs
- B14 (test framework) for the 3-layer harness; B22 (threat model) — sets security standards for the
  connection-secret/admission/visibility surfaces (security review before complete, ADR 0027).

## 4. Parallelizable Subtasks
- After TASK-02+TASK-03, the four substrate XRDs **TASK-04 / TASK-05 / TASK-06 / TASK-07** are an
  independent fan-out group (one engineer per primitive).
- **TASK-12** (`GrafanaDashboard`) and **TASK-13** (`TenantOnboarding`) depend only on TASK-01 and run
  concurrently with the substrate-XRD group.
- TASK-10 / TASK-11 / TASK-14 each fan out once their composed substrate primitives exist.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-B4-01..08 (XR installs/resolution), AC-B4-04 (AuditLog topology),
  AC-B4-10 (admission rejection), AC-B4-11 (conversion webhook), AC-B4-14 (kind prereqs).
- **PyTest (logic):** AC-B4-09 (no substrate leak into status), **AC-B4-12 (connection-secret
  conformance — the invariant)**, AC-B4-13 (substrate-difference docs presence check).
- **Playwright:** N/A — no UI.
- **Fixtures/fakes:** AWS Compositions tested against a Crossplane provider in a mocked/localstack-style
  harness until a real AWS cluster is available; consumers (A18/B11) faked via the connection-secret
  contract so B4 lands before them.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (foundation band; contains B4's siblings).
### 6.2 PR — `piece/B4-crossplane-compositions` → base `wave/1`; carries SPEC-B4 + PLAN-B4 + the XRD/
Composition packages.
### 6.3 Merge order — independent of W1 siblings; substrate-XRD subtasks can be sub-PRs stacked under
the piece branch; the piece rolls up into `wave/1` → main. Must merge before A18/A21/A23/B19/B11 land.

## 7. Effort Estimate
- TASK-01 S, TASK-02 M, TASK-03 M, TASK-04 L, TASK-05 L, TASK-06 M, TASK-07 L, TASK-08 M, TASK-09 S,
  TASK-10 L, TASK-11 M, TASK-12 M, TASK-13 M, TASK-14 L, TASK-15 M, TASK-16 M, TASK-17 L.
- **Rollup: XL** (matches CSV). **Critical path:** TASK-01 → TASK-02 → TASK-04 → TASK-10/TASK-14 →
  TASK-16 → TASK-17 (substrate primitive → composing XRDs → docs/tests).

## 8. Rollback / Reversibility
Back out by reverting the XRD/Composition package; XRs become unresolvable but existing
substrate resources persist (Crossplane orphans, not deletes, on XRD removal — verify deletion
policy). **Downstream breakage if reverted:** A18 audit pipeline, A21 onboarding, A23 promotion,
B11 memory, B19 backing stores all lose their provisioning path. Reversibility is per-XRD — a single
problematic Composition can be reverted without dropping the whole package. Connection-secret-shape
changes are breaking (ADR 0030) and require the conversion-webhook path, not a bare revert.

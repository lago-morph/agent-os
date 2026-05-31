# SPEC B4 — Crossplane v2 Compositions

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [] · downstream: [A18, A21, A23, B19, B11] · adrs: [0044, 0013, 0032, 0034, 0020, 0021, 0025, 0030, 0002] · views: [6.3, 6.6, 6.8, 6.12]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B4 owns the **substrate-abstraction layer** of the platform: the Crossplane v2 XRDs and
Compositions that wrap every substrate-asymmetric primitive (Postgres, search/index, object
store, Mongo-compatible document store) behind a uniform XR schema, plus the higher-level
XRDs (`AuditLog`, `TenantOnboarding`, `AgentDatabase`, `GrafanaDashboard`) and the
agent-facing composed XRs (`MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer`). ADR 0044
commits the platform to "one XRD, one Composition per supported substrate (kind + AWS), all
writing the same connection-secret shape and exposing the same substrate-agnostic status."
B4 is where that commitment becomes concrete artifacts.

Without B4, every consumer (audit pipeline A18, tenant onboarding A21, Kargo promotion A23,
approval system B19, memory adapter B11) would have to branch on substrate, parallel manifest
sets would proliferate, and Kargo cross-substrate promotion (ADR 0040) would silently break.
B4 is foundation (T0, W1): it owns the XRD set that A18/A21/A23/B19/B11 consume, and the
**connection-secret contract is a tested architectural invariant**, not a convention.

## 2. Scope

### 2.1 In scope
- The four **substrate XRDs** committed for v1.0 (§14, §6.3, interface-contract §1.6): `Postgres`,
  `SearchIndex`, `ObjectStore`, `MongoDocStore` — each with **one Composition per substrate**
  (kind + AWS), both writing the **same connection-secret shape** (§4.4) and exposing the same
  substrate-agnostic XR status (`ready`, `endpoint`, `version`).
- The **higher-level XRDs** that compose those substrate primitives: `AuditLog` (XR `AuditLog`,
  ADR 0034), `AgentDatabase` (claim `AgentDatabase`, ADR 0020), `GrafanaDashboard`
  (XR `GrafanaDashboard`, ADR 0021), `TenantOnboarding` (ADR 0028/0037).
- The **agent-facing composed XRs**: `MemoryStore` (ADR 0025 access modes), `AgentEnvironment`,
  `SyntheticMCPServer` (back-link to `MCPServer` produced by A12).
- The **connection-secret contract** as a tested invariant: shape, status-field normalization,
  substrate-specific subfield discipline ("do not depend on").
- The **substrate-selection mechanism**: `platform.io/environment=kind|aws` label-driven
  Composition selection, and the **Gatekeeper admission policy contract** that rejects claims
  with no matching Composition (ADR 0002 / 0044 — B4 owns the *requirement*; the Rego content
  is authored in B16).
- XRD-level **CRD/API versioning** wiring (ADR 0030): conversion webhooks + deprecation windows
  applied identically to XRDs when a XR schema changes.
- The **kind-side backing operators** the kind Compositions depend on being installed
  (CloudNativePG, in-cluster OpenSearch operator, MinIO, Bitnami MongoDB) — declared as
  prerequisites, installed where B4 owns them.

### 2.2 Out of scope (and where it lives instead)
- **OPA/Gatekeeper install + engine** → A7. **The substrate-mismatch admission Rego content** →
  B16 (B4 states the requirement and the input shape; B16 authors the rule).
- **The audit endpoint Deployment + audit adapter library** → A18; B4's `AuditLog` XRD
  *provisions* the pipeline topology (Postgres store, S3 bucket + lifecycle on AWS, OpenSearch
  indexer, batch CronJob on AWS, endpoint Deployment) but does not own the adapter code.
- **Tenant onboarding reconciler / Headlamp flow** → A21 (consumes `TenantOnboarding`).
- **Kargo promotion fabric** → A23 (promotes XRs B4 defines).
- **Memory backend adapter logic** → B11 (consumes `MemoryStore` + connection secret).
- **OpenAPI→MCP conversion** → A12 (produces the `MCPServer` that `SyntheticMCPServer` back-links).
- **`MCPServer`/`MemoryStore`-CRD reconciliation into LiteLLM** → B13 (kopf); B4 composes the XR.
- **Resource sizing / replicas / region / cluster bootstrap / Knative sources** → intentionally
  NOT wrapped (ADR 0044 documented exceptions); Kustomize overlays / install-time config.
- **Audit retention / lifecycle / redaction policy** → Workstream F (F1).

## 3. Context & Dependencies

**Upstream consumed:** None as hard build predecessors (B4 is placed in W1 foundation band per
`_meta/waves.md`; the source lists no wave for B4). B4 depends at *install* time on Crossplane v2
being present and on the backing operators it composes (CloudNativePG, OpenSearch operator, MinIO,
Bitnami MongoDB on kind; the AWS provider stack on AWS — provider stack selection is install-time
config per ADR 0044, NOT wrapped here).

**Downstream consumers (HARD):**
- **A18 (audit endpoint + adapter)** — consumes `AuditLog`/`AuditLog`, which composes `Postgres`
  + `ObjectStore` + an indexer.
- **A21 (tenant onboarding reconciler)** — consumes `TenantOnboarding`.
- **A23 (Kargo)** — promotes B4's XRs with identical shape across substrates.
- **B19 (approval system)** — consumes platform datastores via the connection-secret contract
  (and any XRD-provisioned backing store the `Approval`/Argo path needs).
- **B11 (memory backend adapter)** — consumes `MemoryStore` + its connection secret.

**ADRs honored:**
- **ADR 0044** — one XRD + one Composition per substrate; uniform connection-secret shape;
  substrate-agnostic status; label-driven selection; admission rejects unmatched claims;
  capability-parity not promised; documented non-wrapped exceptions.
- **ADR 0013** — capability CRDs are owned by B13; B4 composes the XRs (`SyntheticMCPServer`
  back-links to `MCPServer`) without redefining capability CRD shapes.
- **ADR 0032** — overlay semantics are a CapabilitySet concern (B13/B17); B4 does not implement
  overlay but `AgentEnvironment.defaultCapabilitySetRef` references a resolved set.
- **ADR 0034** — `AuditLog` XRD provisions Postgres+S3 system of record, OpenSearch advisory
  fanout, batch CronJob (AWS only); kind degrades to Postgres-only.
- **ADR 0020** — `AgentDatabase` provisions per-agent/tenant/user Postgres or MongoDB.
- **ADR 0021** — `GrafanaDashboard` is a namespaced Crossplane-composed XR.
- **ADR 0025** — `MemoryStore.accessMode` lives on the XR (private / namespace-shared / RBAC-OPA).
- **ADR 0030** — XRD schema change go through new `vN` + conversion webhooks + deprecation.
- **ADR 0002** — Gatekeeper admission rejects claims with no matching Composition.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

All XRDs are **namespace-scoped XRs** (Crossplane v2: a single layer — XRs are namespace-scoped
directly, with no claims and no `X`-prefixed cluster-scoped composites; `spec.claimNames` is not used — ADR 0044 / glossary). All v1.0 platform CRDs are namespaced;
there are **no cluster-scoped platform CRDs**. Fields below are **only** those in
interface-contract §1.6. Anything else is tagged `[PROPOSED — not in source]`.

**Substrate XRDs (one Composition per substrate — kind + AWS):**

| XRD (XR ↔ claim) | Key fields (source) | kind Composition | AWS Composition |
|---|---|---|---|
| `Postgres` ↔ `Postgres` | `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass` | CloudNativePG `Cluster` | Crossplane `RDSInstance` |
| `SearchIndex` ↔ `SearchIndex` | `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass` | in-cluster OpenSearch operator | AWS-managed OpenSearch (Crossplane MR) |
| `ObjectStore` ↔ `ObjectStore` | `bucketName`, `lifecycle`, `connectionSecretRef`, `substrateClass` | MinIO (or no-op) | Crossplane S3 bucket |
| `MongoDocStore` ↔ `MongoDocStore` | `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass` | Bitnami MongoDB | DocumentDB or self-managed |

> `[PROPOSED — not in source]`: `substrateClass` is named in interface-contract §1.6 as a field but
> its **value grammar** is not enumerated; B4 proposes it carry the resolved Composition selector
> (`kind`|`aws`) mirrored from the cluster label, validated against `platform.io/environment`.

**Higher-level XRDs (compose substrate primitives):**

| XRD (XR ↔ claim) | Key fields (source) | Composes |
|---|---|---|
| `AuditLog` ↔ `AuditLog` (XR `AuditLog`) | `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec`, `endpointReplicas` | `Postgres` + `ObjectStore` + indexer + (AWS) batch CronJob + endpoint Deployment |
| `AgentDatabase` ↔ `AgentDatabase` | `engine` (postgres/mongodb), `scope` (agent/tenant/user), `ownerRef`, `credentialsSecretRef` | `Postgres` **or** `MongoDocStore` per `engine` |
| `GrafanaDashboard` ↔ `GrafanaDashboard` (XR `GrafanaDashboard`) | `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled) | Grafana folders (kind) / Crossplane-provisioned Grafana (AWS) |
| `TenantOnboarding` ↔ `TenantOnboarding` | `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` | namespaces + SAs + OIDC claim mapping (CapabilitySets intentionally NOT coupled) |

**Agent-facing composed XRs:**

| XR | Key fields (source) |
|---|---|
| `MemoryStore` | `accessMode` (private / namespace-shared / RBAC-OPA), `backendType` (ADR 0025) |
| `AgentEnvironment` | `region`, `quotas`, `defaultCapabilitySetRef` |
| `SyntheticMCPServer` | `openApiSpecRef`, `authConfigRef`, `mcpServerRef` (back-link; produced by A12) |

**Status fields (every XR, substrate-agnostic — ADR 0044):** `ready`, `endpoint`, `version`.
Substrate-specific values (RDS ARN, in-cluster Service path) are **deliberately absent** from
user-visible status; where unavoidable they live in a substrate-specific subfield marked
"implementation detail, do not depend on." `[PROPOSED — not in source]`: the subfield name is not
enumerated in source; B4 proposes `status.substrateDetail` with a do-not-depend annotation.

**Quota schema (`AgentEnvironment.spec.quotas` / `TenantOnboarding.spec.quotas`, #5/#9):** `spec.quotas` is an **extensible typed list**; each entry carries a `quotaType` discriminator plus type-specific fields. The `AgentEnvironment` Composition MUST emit a `ResourceQuota` and a `LimitRange` derived from `quotas`. `TenantOnboarding` admission MUST verify `cpu` and `memory` entries are present (each a concrete limit or `unlimited: true`); quota presence on `TenantOnboarding` is **mandatory**, not optional or deferred (coordinated with A21).

### 4.2 APIs / SDK surfaces
N/A — B4 exposes no programmatic SDK. Its "API" is the **XR API** (the namespace-scoped XRs above);
tenants create XRs directly in their own namespace — there is no separate claim API. Its other surface is
the **connection-secret shape** (§4.4). Consumers interact via Kubernetes XR objects and
the written connection secret, not a B4 library.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
B4 itself is reconciled by Crossplane; the **XR/Sandbox/MemoryStore lifecycle** events flow under
`platform.lifecycle.*` (interface-contract §2 explicitly lists MemoryStore lifecycle there).
Concrete per-event-type names/schemas are **deferred to B12** (interface-contract §2, §6). B4 does
not mint event types. `[PROPOSED — not in source]`: whether Crossplane reconcile of B4 XRs emits a
`platform.lifecycle.*` event directly or via an adapter is a B12/B8 design point, not fixed here.

### 4.4 Data schemas / connection-secret contracts
**The connection-secret contract (architectural invariant, tested — ADR 0044 / interface-contract §4):**
- Each substrate XRD's **both Compositions write the same secret shape**: `host`, `port`, `user`,
  `password`, `dbname` — *"or the equivalent fields per primitive."* The five are the canonical
  shape; the exact per-primitive field list beyond these is **not enumerated in source**
  (`[PROPOSED — not in source]` — e.g. `ObjectStore` likely substitutes `bucket`/`accessKeyId`/
  `secretAccessKey` for `dbname`/`user`/`password`; B4 proposes a documented per-primitive mapping
  table, all five canonical keys present or explicitly mapped).
- **Secret-shape conformance is tested**, not convention (REQ-B4-12 / AC-B4-12).
- `connectionSecretRef` on each XRD names where the secret is written; consumers read it without
  branching on substrate.
- **Capability-parity is not promised**: claim *shape* is uniform; runtime *behaviour* may differ
  (e.g. kind `ObjectStore` may produce "no archive"; kind `AuditLog` is Postgres-only). Each XRD
  documents its substrate-specific behavioural differences.

## 5. OSS-vs-Custom Decision
**Custom Compositions on top of OSS Crossplane v2 (wrap, not fork).** Upstream: Crossplane v2
(no pinned version stated in source — `[PROPOSED — not in source]` to pin in the Composition
package). Backing operators wrapped: CloudNativePG, in-cluster OpenSearch operator, MinIO, Bitnami
MongoDB (kind); AWS provider MRs `RDSInstance`, managed OpenSearch, S3, DocumentDB (AWS). Approach:
**build-new** XRDs + Compositions; **wrap** the backing operators/MRs. Rationale (ADR 0044):
Crossplane Compositions are the canonical pattern for substrate abstraction; alternatives (parallel
manifests, Kustomize-only, Helm conditionals) were rejected in ADR 0044 for drift, substrate-leak,
and broken cross-substrate promotion.

## 6. Functional Requirements
- **REQ-B4-01:** B4 MUST define the four substrate XRDs `Postgres`, `SearchIndex`, `ObjectStore`,
  `MongoDocStore`, each as a **namespace-scoped XR** with the source-stated fields (§4.1).
- **REQ-B4-02:** Each substrate XRD MUST have **exactly one Composition per supported substrate**
  (kind + AWS for v1.0), selected by the `platform.io/environment=kind|aws` cluster label.
- **REQ-B4-03:** The kind Compositions MUST compose CloudNativePG (`Postgres`), in-cluster
  OpenSearch operator (`SearchIndex`), MinIO/no-op (`ObjectStore`), Bitnami MongoDB
  (`MongoDocStore`); the AWS Compositions MUST compose RDS, managed OpenSearch, S3, and
  DocumentDB/self-managed Mongo respectively (ADR 0044).
- **REQ-B4-04:** B4 MUST define the `AuditLog` XRD composing `Postgres` +
  `ObjectStore` + an OpenSearch indexer + (AWS only) a batch CronJob + the audit endpoint
  Deployment; on kind it MUST degrade to Postgres-only with the batch step disabled (ADR 0034).
- **REQ-B4-05:** B4 MUST define `AgentDatabase` composing `Postgres`
  **or** `MongoDocStore` per the declared `engine`, scoped agent/tenant/user (ADR 0020).
- **REQ-B4-06:** B4 MUST define `GrafanaDashboard` as a namespace-scoped XR with
  `dashboardJson`, `folder`, and RBAC+OPA-controlled `visibility` (ADR 0021).
- **REQ-B4-07:** B4 MUST define `TenantOnboarding` provisioning `namespaces[]`,
  `defaultServiceAccounts[]`, and `clusterOIDCClaimMapping` for a `tenantId`; CapabilitySets MUST
  NOT be coupled into it (ADR 0028/0037).
- **REQ-B4-08:** B4 MUST define the composed XRs `MemoryStore` (with `accessMode` ∈ {private,
  namespace-shared, RBAC-OPA}, ADR 0025), `AgentEnvironment` (`region`, `quotas`,
  `defaultCapabilitySetRef`), and `SyntheticMCPServer` (`openApiSpecRef`, `authConfigRef`,
  `mcpServerRef` back-link).
- **REQ-B4-09:** Every XR's user-visible **status MUST be substrate-agnostic** (`ready`, `endpoint`,
  `version`); substrate-specific values MUST NOT appear in user-visible status (ADR 0044).
- **REQ-B4-10:** Substrate selection MUST be **label-driven** on `platform.io/environment`; an XR
  whose target substrate has **no matching Composition** MUST be rejected at admission (B4 supplies
  the admission requirement + input shape to A7/B16; ADR 0002/0044).
- **REQ-B4-11:** XRD schema change MUST follow ADR 0030 (new `vN` API group + conversion
  webhooks + ≥1-release deprecation window), applied to **both** Compositions.
- **REQ-B4-12:** Both Compositions of each substrate XRD MUST write the **same connection-secret
  shape** (`host`, `port`, `user`, `password`, `dbname` or documented per-primitive equivalent),
  referenced by `connectionSecretRef`; this conformance MUST be tested as an invariant (ADR 0044).
- **REQ-B4-13:** Each XRD MUST **document its substrate-specific behavioural differences**
  (capability-parity caveat — e.g. kind `ObjectStore` "no archive", kind `AuditLog` Postgres-only).
- **REQ-B4-14:** B4 MUST declare/install the kind backing operators it composes (CloudNativePG,
  OpenSearch operator, MinIO, Bitnami MongoDB) as prerequisites; provider-stack selection on AWS is
  install-time config and MUST NOT be wrapped (ADR 0044 exception).
- **REQ-B4-15:** The `AgentEnvironment` Composition MUST emit a `ResourceQuota` and a `LimitRange`
  from `spec.quotas`; `TenantOnboarding` admission MUST verify `cpu`/`memory` quota entries are
  present (concrete limit or `unlimited: true`). Quota presence on `TenantOnboarding` is mandatory
  (#5/#9; coordinated with A21).
- **REQ-B4-16:** Emission of audit events through the `AuditLog` pipeline is gated on the
  audit-adapter freeze-gate (D-05); the `audit_events` schema and audit-adapter interface are frozen.
- **REQ-B4-17:** Any security-relevant event B4 detects (e.g. quota-bypass attempts, an XR targeting
  an unauthorized substrate) MUST also be emitted to the event bus under `platform.security` (schema
  owned by A7), in addition to local handling.

## 7. Non-Functional Requirements
- **Security:** connection secrets carry credentials — written to the namespace `connectionSecretRef`
  names; ESO mediates MCP-service secrets (not B4's). `GrafanaDashboard.visibility` and
  `MemoryStore.accessMode` are RBAC+OPA-gated. Gatekeeper admission (REQ-B4-10) prevents silent
  stuck reconciliation. RBAC-as-floor/OPA-as-restrictor (ADR 0018) applies to all XR admission.
- **Multi-tenancy (§6.9):** all XRDs are namespaced; `TenantOnboarding` is the tenant-provisioning
  XRD; cross-tenant sharing is opt-in and not coupled into onboarding.
- **Observability (§6.5):** XR lifecycle flows under `platform.lifecycle.*`; XR status exposes a
  normalized `ready` view; `GrafanaDashboard` XR is itself the dashboard-provisioning mechanism.
- **Scale:** Compositions are declarative; day-2 drift reconciliation is standard Crossplane
  behaviour (substrate-side console edits reconciled back to declared state — ADR 0044).
- **Versioning (ADR 0030):** per-component XRD ownership sits with B4; breaking XR schema change
  are coordinated because A18/A21/A23/B19/B11 bind to the XRs and the secret shape.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (XRD + Composition packages; kind backing-operator installs).
- Per-product docs (10.5) — **applicable** (per-XR reference, substrate-difference docs,
  connection-secret mapping table).
- Runbook (10.7) — **applicable** (stuck-reconcile recovery, drift handling, secret-shape mismatch).
- Alerts — **applicable** (XR not-ready, Composition selection failure, secret-shape conformance
  fail). `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** — B4 *owns* the `GrafanaDashboard` XR primitive;
  a B4 operability dashboard is itself authored as a `GrafanaDashboard` XR.
- Headlamp plugin — **N/A** — XR editing/visibility surfaces are A22 (graphical editors) / A21
  (tenant onboarding flow); B4 ships no plugin.
- OPA/Rego integration — **applicable (contract only)** — B4 states the substrate-mismatch admission
  requirement + input shape; Rego content is B16.
- Audit emission (ADR 0034) — **applicable (provisions pipeline)** — `AuditLog` XRD provisions the
  audit topology; emission code is A18's adapter.
- Knative trigger flow — **N/A** — B4 mints no event types; lifecycle event schemas are B12.
- HolmesGPT toolset — **N/A** — no self-management toolset shipped by B4.
- 3-layer tests — **applicable** (Chainsaw for XR→Composition→secret-shape; PyTest for
  secret-shape conformance harness; Playwright **N/A** — no UI).
- Tutorials & how-tos — **applicable** ("add a substrate XRD", "consume a connection secret").

## 9. Acceptance Criteria
- **AC-B4-01:** Each of `Postgres`/`SearchIndex`/`ObjectStore`/`MongoDocStore` installs as a
  namespace-scoped XR (XRD) with the source-stated fields. (→ REQ-B4-01)
- **AC-B4-02:** For each substrate XRD, exactly one Composition matches `platform.io/environment=kind`
  and one matches `=aws`; an XR resolves to the matching one. (→ REQ-B4-02)
- **AC-B4-03:** On kind, XRs provision CloudNativePG/OpenSearch-operator/MinIO/Bitnami-Mongo; on
  AWS they provision RDS/managed-OpenSearch/S3/DocumentDB. (→ REQ-B4-03)
- **AC-B4-04:** An `AuditLog` claim on AWS provisions Postgres+S3+indexer+batch-CronJob+endpoint; on
  kind it provisions Postgres+indexer+endpoint with the batch step disabled. (→ REQ-B4-04)
- **AC-B4-05:** An `AgentDatabase` XR with `engine=postgres` composes `Postgres`; with
  `engine=mongodb` composes `MongoDocStore`. (→ REQ-B4-05)
- **AC-B4-06:** A `GrafanaDashboard` claim provisions a namespaced dashboard with RBAC+OPA-gated
  `visibility`. (→ REQ-B4-06)
- **AC-B4-07:** A `TenantOnboarding` claim creates the declared namespaces, SAs, and OIDC claim
  mapping and does NOT create any CapabilitySet. (→ REQ-B4-07)
- **AC-B4-08:** `MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer` XRs install with their
  source-stated fields; `MemoryStore.accessMode` accepts only the three modes. (→ REQ-B4-08)
- **AC-B4-09:** Each XR's user-visible status exposes only `ready`/`endpoint`/`version`; a test
  asserts no RDS ARN / Service path leaks into user-visible status. (→ REQ-B4-09)
- **AC-B4-10:** A XR submitted on a cluster whose label has no matching Composition is **rejected
  at admission** (not stuck-reconciling). (→ REQ-B4-10)
- **AC-B4-11:** A XR schema change ships a new `vN` group with a working conversion webhook and the
  prior version remains for ≥1 release on both Compositions. (→ REQ-B4-11)
- **AC-B4-12:** A conformance test reads the connection secret from each substrate XRD's kind **and**
  AWS Composition and asserts identical key shape (canonical or documented per-primitive map).
  (→ REQ-B4-12)
- **AC-B4-13:** Each XRD's docs enumerate its substrate-specific behavioural differences; kind
  `ObjectStore` "no archive" and kind `AuditLog` "Postgres-only" are documented. (→ REQ-B4-13)
- **AC-B4-14:** On a fresh kind cluster, B4's prerequisite installs bring up CloudNativePG/OpenSearch
  operator/MinIO/Bitnami-Mongo before any XR is reconciled. (→ REQ-B4-14)

## 10. Risks & Open Questions
- **R1 (high):** Per-primitive connection-secret field lists beyond the five canonical keys are
  **not in source** (`[PROPOSED]`). If A18/B11 assume different key names for `ObjectStore`, the
  invariant breaks. Mitigation: B4 freezes the per-primitive mapping table first; consumers bind.
- **R2 (med):** `substrateClass` value grammar is `[PROPOSED]`; must not collide with the
  `platform.io/environment` label semantics. Mitigation: mirror the label, admission-validate.
- **R3 (med):** `status.substrateDetail` subfield name is `[PROPOSED]`; risk that a consumer depends
  on it despite the "do not depend" annotation. Mitigation: lint/test that no platform consumer
  reads it.
- **R4 (med):** Capability-parity gaps (kind "no archive", kind audit Postgres-only) can surprise a
  developer who tested on kind then promotes to AWS — mitigated by explicit per-XRD docs (REQ-B4-13)
  but residual.
- **R5 (low):** Crossplane v2 version not pinned in source (`[PROPOSED]`); pin in the package.
- **OQ1:** Does B4 own installing the AWS provider stack, or is it purely install-time config? ADR
  0044 says provider-stack selection is install-time and NOT wrapped → B4 declares it a prerequisite,
  does not own it. Confirm with A-workstream install owner.
- **OQ2:** Is the substrate-mismatch CloudEvent emitted as `platform.security.*` (policy-bypass-ish)
  or only an admission deny with no event? `[PROPOSED]` admission deny + audit under
  `platform.policy.*`; B12 owns the schema.

## 11. References
- architecture-overview.md §6.3 (lines 294–347: data architecture, dual-mode hosting, substrate
  abstraction, audit pipeline), §6.6 (line 534: OPA decision points lists every B4 XR), §6.8
  (capability registries), §6.12 (CRD inventory), §14 (committed substrate XRDs).
- ADR 0044 (substrate abstraction — primary), 0034 (audit pipeline), 0020 (AgentDatabase / initial
  MCP services), 0021 (GrafanaDashboard XRs), 0025 (memory access modes), 0028/0037 (tenant
  onboarding), 0013 (capability CRDs), 0032 (overlay), 0030 (versioning), 0002 (Gatekeeper).
- interface-contract.md §1.6 (XRD field tables), §4 (connection-secret contract), §5 (audit adapter).
- Related pieces: A18 (audit), A21 (tenant onboarding), A23 (Kargo), B19 (approval), B11 (memory),
  A12 (OpenAPI→MCP), B13 (kopf), B16 (admission Rego content), A7 (OPA/Gatekeeper).

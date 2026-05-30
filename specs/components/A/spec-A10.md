# SPEC A10 — Letta memory backend

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A11] · downstream: [B11] · adrs: [0005, 0025] · views: [6.3]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

A10 installs and operates **Letta** as the platform's memory backend (ADR 0005). Letta runs as a memory service behind ARK's `Memory` CRD: Platform Agents reach it only through the Platform SDK's `memory.*` API (B6), never directly (§6.3, ADR 0005). Letta persists state to the platform's managed Postgres system of record and keeps any retrieval-optimized indexes in OpenSearch reproducible from primary sources (§6.3 rule of thumb). It is provisioned as a Crossplane v2 `MemoryStore` XR (B4), and a thin memory backend adapter (B11) wraps Letta-specific calls.

The problem A10 solves is durable, tiered agent memory consumed declaratively. The architecture deliberately keeps storage roles separate — Postgres is the system of record for state and checkpoints; OpenSearch optimizes retrieval; object storage is the immutable archive (§6.3). Letta is the OS-style tiered-memory service that sits over that storage, while access policy is governed by the `MemoryStore` `accessMode` (ADR 0025), not by Letta itself.

A10's scope is the install + configure + operate package (Helm, docs, runbook, dashboards, OPA, audit, tests), plus wiring Letta to Postgres/OpenSearch through the substrate-abstracted primitives and honoring the three memory access modes declared on `MemoryStore`.

## 2. Scope

### 2.1 In scope

- Letta install (Helm values/manifests) at a pinned tested version, namespace-scoped per the multi-tenancy mitigation (ADR 0005 consequence; §9).
- Wiring Letta state persistence to managed Postgres (in-cluster CloudNativePG on kind / RDS on AWS via the `XPostgres` substrate, §6.3, ADR 0014/0033/0041).
- Wiring Letta retrieval indexes to OpenSearch (A11) such that they remain reproducible from Postgres / object storage (OpenSearch never a system of record for memory — §6.3, ADR 0009/0014).
- Honoring the three `MemoryStore` access modes (private / namespace-shared / RBAC-OPA) declared on the XR (ADR 0025) — Letta-backed service must surface the same access semantics regardless of backend.
- Consumption path: Letta is reachable **only** through ARK's `Memory` CRD and the Platform SDK `memory.*` API; A10 exposes no direct agent-facing Letta endpoint.
- Audit emission for memory reads/writes attributed to agent identity + store access mode (ADR 0025 consequence; §6.6).
- Lifecycle CloudEvents under `platform.lifecycle.*` via `MemoryStore` lifecycle (ADR 0005 consequence; §6.7).
- Gatekeeper admission for `Memory` / `MemoryStore` (§6.6; ADR 0025 admission of `accessMode`).
- Backup/restore: inherits Postgres backup/PITR/DR posture (§6.3); A10 documents the memory-restore procedure.
- Standard Workstream A deliverables: per-product docs (10.5), runbook (10.7), alerts, Grafana dashboard (`GrafanaDashboard` XR), Headlamp plugin (where useful), OPA integration, audit, Knative trigger-flow design, HolmesGPT toolset, 3-layer tests, tutorials/how-tos (§14.1).

### 2.2 Out of scope (and where it lives instead)

- The `Memory` CRD reconciler — **A5** (ARK). A10 supplies the backend the `Memory` binding resolves to.
- The `MemoryStore` XR + its Compositions and the `XPostgres`/`XSearchIndex` substrate XRDs — **B4** (Crossplane). A10 consumes the provisioned backends.
- The Platform SDK `memory.*` / `rag.*` API surface — **B6**.
- The Letta-specific call wrapper (memory backend adapter) — **B11** (the named downstream).
- OpenSearch install/operation — **A11** (upstream).
- Detailed memory namespace/sharing semantics (naming, lifecycle, cross-store joins, eviction) — **deferred** per architecture-backlog §4 / ADR 0025 (this component fixes only the three-mode contract behavior at the backend).
- Access-mode policy authoring (Rego) — **B16** content; A10 contributes admission targets.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A11** (OpenSearch) — Letta's retrieval-optimization indexes live in OpenSearch; A10 cannot wire retrieval until A11 exists. (OpenSearch is advisory/retrieval only; primary memory state is Postgres.)

**Upstream consumed (effective, via Canon even though CSV lists only A11):**
- Managed Postgres via B4 `XPostgres` substrate (§6.3) — Letta's system of record. `[PROPOSED — not in source]` the CSV upstream column lists only A11; the Postgres dependency is stated in §6.3/ADR 0005 and is treated as a HARD runtime dependency here, satisfied by B4. Flagged because it is not in the CSV edge set.
- A5 (ARK `Memory` CRD), B6 (SDK), B11 (adapter) — the declarative + call path; consumed as they land.

**Downstream consumers:** **B11** (memory backend adapter wraps Letta calls).

**ADR decisions honored:**
- **ADR 0005** — Letta is THE memory backend; consumed only through `Memory` CRD + `memory.*` API; state in managed Postgres; OpenSearch indexes reproducible; provisioned via `MemoryStore` XR; namespace-scoped instances mitigate Letta multi-tenancy gaps; SDK keeps a pinned compatibility matrix against Letta.
- **ADR 0025** — `MemoryStore.accessMode` ∈ {private, namespace-shared, RBAC/OPA-controlled}; backend choice does not change access semantics; private/namespace-shared enforced structurally; RBAC/OPA mode follows RBAC-as-floor / OPA-as-restrictor (ADR 0018); audit attributes events to agent identity + store mode.
- **ADR 0014 / 0009** — Postgres primary; OpenSearch retrieval-optimization only, reproducible from primary.
- **ADR 0041** — substrate abstraction via Crossplane; uniform connection-secret; substrate-agnostic status.
- **ADR 0030 / 0031 / 0034** — versioning, CloudEvent taxonomy, audit adapter (as for all A components).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A10 **owns no CRD/XRD**; it consumes ones owned elsewhere:

| Resource | Owner | Source-stated fields A10 relies on |
|---|---|---|
| `Memory` | A5 (ARK) | `memoryStoreRef` |
| `MemoryStore` (XR) | B4 (Crossplane) | `accessMode` (private / namespace-shared / RBAC-OPA), `backendType` |
| `XPostgres` (XRD) | B4 | `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass` |
| `XSearchIndex` (XRD) | B4 | `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass` |

A10 configures Letta to consume the connection secrets these XRDs write; it does not extend their schemas. `[PROPOSED — not in source]` `MemoryStore.backendType` value naming Letta is not enumerated in Canon; A10 uses whatever value B4 assigns and does not coin it.

### 4.2 APIs / SDK surfaces

- A10 exposes **no platform agent-facing API of its own**. The only memory surface is the Platform SDK `memory.*` API (B6), which terminates at Letta via the B11 adapter (§6.3; ADR 0005).
- `[PROPOSED — not in source]` Letta's internal service endpoint/port is an upstream implementation detail not specified in Canon; A10 keeps it cluster-internal, reachable only through the SDK/adapter path.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **Emitted:** Letta-backed resource lifecycle (`Memory`, `MemoryStore` created/started/paused/resumed/completed/failed/deleted) under **`platform.lifecycle.*`** (ADR 0005 consequence; §6.7). Memory read/write audit under **`platform.audit.*`** via the audit adapter.
- **Consumed:** none required for v1.0 beyond admission/reconcile triggers. 
- Per-event-type names/schemas deferred to **B12** registry; each carries `specversion` + `schemaVersion`. `[PROPOSED — not in source]` concrete memory event-type names are not in Canon.

### 4.4 Data schemas / connection-secret contracts

- Letta consumes the **uniform connection secret** (`host`, `port`, `user`, `password`, `dbname` or per-primitive equivalent) written by the `XPostgres` and `XSearchIndex` Compositions (interface-contract §4; ADR 0041) — A10 branches on neither substrate.
- Letta state schema in Postgres is an upstream-owned detail; A10 does not redefine it. The architectural invariant A10 must uphold: **everything Letta puts in OpenSearch is reproducible from Postgres or object storage** (§6.3).

## 5. OSS-vs-Custom Decision

- **Upstream project:** Letta, chosen over Mem0 / Zep+Graphiti / Cognee / LangMem (ADR 0005; architecture-backlog §2.9) for fully-OSS posture and OS-style tiered memory.
- **Mode:** **config + wrap**. Install unmodified at a pinned version; wrap behind the `Memory` CRD (A5) and the B11 adapter; run namespace-scoped to mitigate multi-tenancy gaps (ADR 0005/§9). No fork.
- **Rationale:** fully self-hostable, no vendor dependency; the `Memory` CRD + SDK insulate agents so a future backend swap only touches B11 (ADR 0005 consequence).
- `[PROPOSED — not in source]` exact Letta version/chart coordinates not in Canon; pinned at install and recorded in runbook.

## 6. Functional Requirements

- REQ-A10-01: A10 installs Letta via Helm at a single pinned tested version, namespace-scoped, ArgoCD-synced.
- REQ-A10-02: Letta persists its state to managed Postgres consumed via the `XPostgres` connection secret; on kind it uses in-cluster CloudNativePG and on AWS RDS, without A10 branching on substrate.
- REQ-A10-03: Any retrieval index Letta maintains in OpenSearch (via `XSearchIndex`) is reproducible from Postgres/object storage; OpenSearch is never the memory system of record.
- REQ-A10-04: Letta is reachable by agents **only** through the `Memory` CRD binding (A5) and the Platform SDK `memory.*` path (B6/B11); no direct agent-facing Letta endpoint is exposed.
- REQ-A10-05: A `Memory` referencing a `MemoryStore` with `accessMode: private` permits reads only by the writing Platform Agent; `namespace-shared` permits reads by any Platform Agent in the same namespace; `RBAC/OPA-controlled` follows RBAC-as-floor / OPA-as-restrictor.
- REQ-A10-06: Memory read and write actions emit structured audit events attributing the agent identity and the store's declared access mode, through the audit adapter library.
- REQ-A10-07: Letta-backed resource lifecycle emits CloudEvents under `platform.lifecycle.*` only, each carrying `specversion` + `schemaVersion`.
- REQ-A10-08: `Memory` and `MemoryStore` are subject to Gatekeeper admission; a `MemoryStore` with missing or out-of-range `accessMode` is rejected (ADR 0025).
- REQ-A10-09: A10 documents and tests a memory backup/restore procedure inheriting the Postgres backup/PITR/DR posture.
- REQ-A10-10: A10 contributes a HolmesGPT memory toolset (store inspection, memory-state queries) and OPA admission targets to B16.
- REQ-A10-11: A10 delivers per-product docs, runbook, component-failure alerts, a `GrafanaDashboard` XR, and (where useful) a Headlamp plugin, plus tutorials/how-tos.
- REQ-A10-12: A10 owns its install version pin and participates in the SDK's version-pinned compatibility matrix against Letta (ADR 0005; §6.13).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** Letta runs namespace-scoped; tenant isolation is the namespace boundary plus the `accessMode` contract. RBAC/OPA mode never grants beyond the RBAC floor (ADR 0018). Threat focus: unintended access excess across memory stores (ADR 0027 / B22) — audit must make excess detectable.
- **Observability (§6.5):** OTel emission; memory metrics/dashboards; alerts on Letta unavailability, Postgres connection loss, OpenSearch reindex lag.
- **Data durability (§6.3):** Postgres is the system of record; OpenSearch reproducibility is an invariant; object storage holds source documents/archive. Restore must reconstruct OpenSearch indexes from primaries.
- **Scale:** memory service throughput must serve agent `memory.*` traffic via the adapter without becoming a bottleneck; design-time choice of SDK-API vs filesystem-mount access is per-agent (§6.4 analogue).
- **Versioning (ADR 0030):** per-component pin; SDK compatibility matrix.

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm values / manifests in Git | Applicable |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable |
| Backup / restore | Applicable — inherits Postgres PITR/DR; memory-restore procedure documented |
| Alert rules | Applicable — Letta down, Postgres conn loss, OpenSearch reindex lag |
| Grafana dashboard (Crossplane XR) | Applicable |
| Headlamp plugin | Applicable (where useful) — memory-store inspector |
| OPA / Rego integration | Applicable — `Memory`/`MemoryStore` admission + access-mode enforcement targets to B16 |
| Audit emission (ADR 0034) | Applicable — memory read/write attributed to identity + mode |
| Knative trigger flow | Applicable — `platform.lifecycle.*` via `MemoryStore` lifecycle |
| HolmesGPT toolset | Applicable — memory inspection toolset |
| 3-layer tests | Applicable — Chainsaw (admission/lifecycle), Playwright (plugin), PyTest (access-mode + reproducibility logic) |
| Tutorials & how-tos | Applicable |

## 9. Acceptance Criteria

- AC-A10-01 (REQ-A10-01): ArgoCD sync installs Letta at the pinned version, namespace-scoped; pods become ready.
- AC-A10-02 (REQ-A10-02): On both kind and AWS, Letta starts against the `XPostgres` connection secret with no substrate-specific config in Letta values.
- AC-A10-03 (REQ-A10-03): Deleting and rebuilding the OpenSearch memory index reconstructs it from Postgres/object storage with no memory loss.
- AC-A10-04 (REQ-A10-04): No route reaches Letta from an agent pod except through the `memory.*`/adapter path; a direct connection attempt fails.
- AC-A10-05 (REQ-A10-05): For each access mode, the cross-agent and cross-namespace read matrix matches the spec (private denies non-writer; namespace-shared allows same-ns; RBAC/OPA denies beyond floor and allows within).
- AC-A10-06 (REQ-A10-06): Each memory read/write produces one audit event carrying agent identity and access mode.
- AC-A10-07 (REQ-A10-07): Every emitted CloudEvent has a `platform.lifecycle.*` type with non-empty `specversion`/`schemaVersion`.
- AC-A10-08 (REQ-A10-08): A `MemoryStore` with absent or invalid `accessMode` is denied at admission with a policy reason.
- AC-A10-09 (REQ-A10-09): A documented restore procedure recovers memory state from a Postgres backup and rebuilds OpenSearch indexes.
- AC-A10-10 (REQ-A10-10): The HolmesGPT memory toolset answers a store-inspection query; OPA admission targets are present in B16.
- AC-A10-11 (REQ-A10-11): Docs, runbook, alerts, `GrafanaDashboard` XR, and (if shipped) Headlamp plugin exist and the dashboard renders memory metrics.
- AC-A10-12 (REQ-A10-12): The Letta version pin appears in the SDK compatibility matrix.

## 10. Risks & Open Questions

- R-A10-1 (med): Letta multi-tenancy gaps (ADR 0005/§9). Mitigation: namespace-scoped instances + `Memory` CRD wrapping. Flag into B22.
- R-A10-2 (med): the Postgres runtime dependency is not in the CSV upstream edge set but is stated in §6.3/ADR 0005. `[PROPOSED]` treated as HARD via B4 `XPostgres`; confirm with B4 sequencing.
- R-A10-3 (low): `MemoryStore.backendType` value for Letta not in Canon. `[PROPOSED]`; use B4-assigned value.
- R-A10-4 (med): RBAC/OPA access-mode enforcement semantics for memory reads/writes are deferred (ADR 0025 / architecture-backlog §4). Open question: how is per-request OPA invoked on a memory read — at the SDK/adapter, at Letta, or both? Resolve with B11/B16. Blast radius med (affects cross-namespace sharing correctness).
- R-A10-5 (low): OpenSearch reproducibility must be enforced/tested, not assumed; a non-reproducible index would violate the §6.3 invariant.

## 11. References

- architecture-overview.md §6.3 (memory & data, lines 294–347), §6.4 (KB access patterns, 349–368), §6.5 (observability), §6.6 (audit/OPA points), §6.7 (eventing), §6.9 (multi-tenancy), §6.13 (versioning), §9 (OSS limitations), §14.1 (deliverables, A10 row 1676).
- ADR 0005 (Letta backend); ADR 0025 (memory access modes); ADR 0014 (Postgres primary / OpenSearch retrieval); ADR 0009 (OpenSearch); ADR 0041 (substrate abstraction); ADR 0018 (RBAC-floor/OPA-restrictor); ADR 0030/0031/0034.
- interface-contract §1.6 (`MemoryStore`, `XPostgres`, `XSearchIndex`), §4 (connection-secret), §2 (CloudEvents), §6 (deferred gaps).
- glossary (Letta, MemoryStore, OpenSearch, connection secret). Related: A5, A11, B4, B6, B11, B16.

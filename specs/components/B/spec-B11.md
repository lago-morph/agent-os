# SPEC B11 — Memory backend adapter

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A10] · downstream: [] · adrs: [0005, 0025, 0014, 0016, 0018, 0030, 0034, 0041] · views: [6.3, 6.2]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B11 is the **thin memory backend adapter** that wraps Letta-specific calls behind the platform's
stable memory contract (ADR 0005: "a thin memory backend adapter (B11) wraps Letta-specific
calls"). It is the single seam between the Letta memory service (A10) and the rest of the
platform — ARK's `Memory` CRD, the `MemoryStore` XR (B4), and the Platform SDK's `memory.*` API
(B6). The architecture deliberately routes **all** agent memory access through ARK's `Memory`
CRD and the SDK's `memory.*` surface; **agent code never calls Letta directly** (§6.2, §6.3,
ADR 0005). B11 is what makes that indirection real on the backend side.

The problem it solves: Letta is the chosen OSS backend today, but the platform must not be
coupled to Letta's API shape. ADR 0005 states plainly that "a migration off Letta would require
re-implementing the B11 memory backend adapter; the SDK API contract insulates Platform Agents
from that change." B11 therefore localizes every Letta-specific assumption — request/response
mapping, state persistence into Postgres, retrieval-index reproducibility in OpenSearch — and
enforces the per-store **access modes** (ADR 0025) at the backend boundary.

## 2. Scope

### 2.1 In scope
- The **adapter layer** translating the platform's memory operations onto Letta's API, behind
  ARK's `Memory` CRD and the SDK `memory.*` contract (B6).
- **Honoring the `MemoryStore` access mode** (private / namespace-shared / RBAC-OPA) declared on
  the XR (ADR 0025) — enforcing private & namespace-shared structurally, and deferring the
  RBAC/OPA-controlled mode to the RBAC-floor / OPA-restrictor model (ADR 0018).
- **Persistence wiring**: Letta state → Postgres system of record; any Letta retrieval indexes in
  OpenSearch kept **reproducible** from Postgres/object storage (the OpenSearch invariant, §6.3,
  ADR 0014).
- **Namespace-scoped Letta instances** (the §9 multi-tenancy mitigation for Letta).
- **Audit emission** for memory reads/writes attributed to agent identity + store mode (ADR 0025,
  ADR 0034).
- Consuming the `MemoryStore` **connection secret** (uniform shape, ADR 0041) rather than
  hand-wiring Letta to a database.

### 2.2 Out of scope (and where it lives instead)
- **Letta install / operation / Helm** → **A10** (Letta memory backend).
- **The `MemoryStore` XR + per-substrate Compositions + `XPostgres`/`XSearchIndex`/`XObjectStore`**
  → **B4** (Crossplane Compositions, ADR 0041). B11 *consumes* the composed store + connection secret.
- **The `Memory` CRD definition** → **ARK / A5** (`Memory.memoryStoreRef`); B11 implements the
  backend behind it.
- **The SDK `memory.*` method surface** → **B6** (Platform SDK). B11 is the backend the SDK targets.
- **The detailed memory namespace / sharing / eviction / cross-store-join semantics** → deferred
  per architecture-backlog §4 (ADR 0025 "fixes only the three-mode access contract").
- **OPA policy *content*** for the RBAC/OPA-controlled mode → B16; B11 only invokes the decision point.
- **Audit endpoint / adapter library implementation** → A18 / ADR 0034; B11 *links* the adapter.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A10 (Letta)** — the memory service B11 wraps. B11 binds to a **pinned, tested Letta version**
  (per the §9 ARK/Letta "pin a tested version" posture and the SDK compatibility matrix, §6.13).

**Implied consumed (not in the CSV `upstream` column but architecturally required):**
- **B4** — the `MemoryStore` XR and the uniform **connection secret** B11 reads. (CSV lists B4's
  downstream as including B11.) Treated as a HARD dependency in the plan.
- **A5 (ARK)** — the `Memory` CRD whose `memoryStoreRef` selects the store B11 backs.

**Downstream consumers:** none listed in `piece-index.csv`. (At runtime, every Platform Agent
that uses memory reaches B11 *through* the SDK/`Memory` CRD, but no build piece depends on B11.)

**ADRs honored:**
- **ADR 0005** — Letta is the backend; B11 is the thin wrapper; migration off Letta = re-implement B11.
- **ADR 0025** — three access modes declared on `MemoryStore`; backend honors the declared mode;
  no platform-wide override.
- **ADR 0014** — Postgres is the system of record; OpenSearch retrieval indexes must be reproducible.
- **ADR 0016 / 0018** — namespace-as-tenant; RBAC-floor / OPA-restrictor for the RBAC/OPA mode.
- **ADR 0041** — substrate abstraction: B11 consumes the composed store + uniform connection
  secret (`host`/`port`/`user`/`password`/`dbname`), never branching on kind-vs-AWS.
- **ADR 0030** — versioning of any adapter-exposed interface.
- **ADR 0034** — audit emission via the adapter/endpoint for memory operations.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B11 **defines no new CRD/XRD**. It is the backend behind existing Canon resources:
- **`Memory`** (ARK, namespaced) — `memoryStoreRef`. Access mode lives on `MemoryStore`, not here
  (interface-contract §1.2; ADR 0025).
- **`MemoryStore`** (XR, namespaced; B4-composed) — `accessMode` (private / namespace-shared /
  RBAC-OPA), `backendType`. B11 reads `accessMode` and enforces it; reads `backendType` to select
  the Letta path. (interface-contract §1.6.)
B11 does **not** add fields to either resource. Any adapter-internal config beyond these fields is
`[PROPOSED — not in source]` and must be carried in adapter config (e.g. a ConfigMap), not by
extending the CRD schema.

### 4.2 APIs / SDK surfaces
- **Platform-facing contract (consumed by B6):** the SDK's `memory.*` surface terminates, on the
  backend, at B11. The **specific `memory.*` method signatures are not specified in source**
  (interface-contract §3.1); B11 implements whatever B6 defines and MUST NOT leak Letta-shaped
  types across that boundary.
- **Letta-facing API:** Letta's own service API (vendor surface, not Canon). B11 is the only
  component permitted to call it. Concrete Letta endpoints are `[PROPOSED — not in source]`.
- **Versioning (ADR 0030):** if B11 exposes any HTTP surface of its own it uses URL-path
  versioning (`/v1/...`, interface-contract §3.3); `[PROPOSED — not in source]` whether B11 has
  its own HTTP surface at all vs. being an in-process library — design-time decision.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emits** under `platform.lifecycle.*` — `MemoryStore` lifecycle (created/started/paused/
  resumed/completed/failed/deleted) is explicitly in this namespace (interface-contract §2;
  ADR 0005 "Letta lifecycle events flow under `platform.lifecycle.*` via `MemoryStore` lifecycle").
- **Emits** under `platform.audit.*` — memory read/write audit attributed to agent identity +
  store mode (ADR 0025, ADR 0034).
- **Consumes**: none source-stated. Per-event-type schemas live in B12. `[PROPOSED — not in
  source]` whether the RBAC/OPA-controlled mode emits a `platform.policy.*` decision event — the
  *decision* event is emitted by the OPA decision point (B2/A7), not by B11.

### 4.4 Data schemas / connection-secret contracts
- **Connection secret (ADR 0041, interface-contract §4):** B11 consumes the uniform shape
  `host`, `port`, `user`, `password`, `dbname` (or per-primitive equivalents) written by the
  `XPostgres`/`XSearchIndex`/`XObjectStore` Compositions that back the `MemoryStore`. B11 MUST NOT
  branch on substrate (kind vs AWS) — it reads the secret and connects.
- **State of record:** Letta state persists to **Postgres** (`audit_events` is the audit table;
  memory state tables are Letta's own schema — `[PROPOSED — not in source]` for table names).
  OpenSearch retrieval indexes MUST be **rebuildable** from Postgres/object storage (ADR 0014).
- **Capability-parity caveat (ADR 0041):** on kind, object-storage archive may be reduced/no-op;
  B11 MUST tolerate the documented reduced-capability path without losing the system of record.

## 5. OSS-vs-Custom Decision
**Custom (build-new) thin adapter wrapping OSS Letta.** Upstream OSS: **Letta** (A10; ADR 0005;
"pin a tested version" per §9 — exact pinned version **not stated in source**, `[PROPOSED — not
in source]` to pin in the chart/compat matrix). Approach: **wrap**, not fork — B11 is a thin
translation layer so the platform's `memory.*` / `Memory` CRD contract is backend-agnostic and a
future Letta replacement is a B11 re-implementation, not a platform-wide change (ADR 0005
consequence). Rationale: ADR 0005 explicitly designates the B11 adapter as the Letta isolation
seam; Letta's multi-tenancy gaps (§9) are mitigated by wrapping it behind `Memory` + namespace-
scoped instances rather than forking Letta.

## 6. Functional Requirements
- **REQ-B11-01:** B11 MUST present the platform memory contract behind ARK's `Memory` CRD and the
  SDK `memory.*` surface such that **agent code never calls Letta directly** (§6.2, §6.3, ADR 0005).
- **REQ-B11-02:** B11 MUST be the **only** component that calls Letta's API; no Letta-shaped type
  may cross the SDK/`Memory` boundary (the isolation-seam invariant, ADR 0005).
- **REQ-B11-03:** B11 MUST read the `MemoryStore.accessMode` and **enforce** the declared mode:
  private (writer-only read), namespace-shared (same-namespace read), RBAC/OPA-controlled
  (RBAC floor + OPA restrict per request) (ADR 0025).
- **REQ-B11-04:** Private and namespace-shared modes MUST be enforced **structurally** (writer
  identity, namespace membership) without per-request OPA evaluation (ADR 0025).
- **REQ-B11-05:** The RBAC/OPA-controlled mode MUST follow RBAC-floor / OPA-restrictor — OPA may
  only further restrict, never grant beyond RBAC (ADR 0018).
- **REQ-B11-06:** B11 MUST persist Letta state to **Postgres** (system of record) and keep any
  OpenSearch retrieval indexes **reproducible** from Postgres/object storage (ADR 0014, §6.3).
- **REQ-B11-07:** B11 MUST consume the **uniform connection secret** from the B4-composed
  `MemoryStore` and MUST NOT branch on substrate (kind vs AWS) (ADR 0041).
- **REQ-B11-08:** B11 MUST run Letta **namespace-scoped** (one instance per the multi-tenancy
  mitigation, §9), respecting the namespace-as-tenant boundary (ADR 0016).
- **REQ-B11-09:** B11 MUST emit **audit events** for memory reads/writes attributed to the agent
  identity and the store's declared mode, through the audit adapter/endpoint (ADR 0025, ADR 0034).
- **REQ-B11-10:** B11 MUST emit `MemoryStore` lifecycle under `platform.lifecycle.*` (ADR 0031).
- **REQ-B11-11:** Any interface B11 exposes MUST be versioned per ADR 0030; a backend swap MUST be
  achievable by re-implementing B11 without changing the `Memory`/`memory.*` contract (ADR 0005).
- **REQ-B11-12:** B11 MUST ship the **version pin** of Letta it is tested against and surface it
  in the SDK compatibility matrix (§6.13). Exact version `[PROPOSED — not in source]`.

## 7. Non-Functional Requirements
- **Security:** no direct Letta access from agents; access-mode enforcement at the backend
  boundary; audit on every read/write enabling later access-excess detection (the v1.0 threat-model
  focus, ADR 0027).
- **Multi-tenancy (§6.9):** namespace-scoped Letta instances; cross-namespace sharing only via the
  RBAC/OPA-controlled mode, never a separate flag (ADR 0025).
- **Observability (§6.5):** OTel spans on memory operations (correlated by `trace_id`, ADR 0015);
  `MemoryStore` lifecycle + audit events emitted.
- **Scale:** adapter overhead must be thin (the "thin adapter" mandate of ADR 0005); it MUST NOT
  add a system-of-record of its own — Postgres remains authoritative, OpenSearch derived.
- **Versioning (ADR 0030):** B11 owns the adapter↔Letta compatibility lifecycle and the pin matrix.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (namespace-scoped Letta wiring + adapter deployment/config).
- Per-product docs (10.5) — **applicable** (how memory access modes map to Letta behavior; the
  "wrap, don't bypass" rule; backend-migration note).
- Runbook (10.7) — **applicable** (Letta restore from Postgres, OpenSearch index rebuild, pin bump).
- Alerts — **applicable** (Letta unreachable, OpenSearch-index drift, audit-emission failure).
  `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** (memory op latency, per-mode read/write
  counts) via `GrafanaDashboard` XR (ADR 0021). `[PROPOSED — not in source]` for exact panels.
- Headlamp plugin — **N/A** — no dedicated UI; memory stores are inspected via standard CRD views.
- OPA/Rego integration — **applicable** (RBAC/OPA-controlled mode invokes the OPA decision point;
  policy *content* is B16).
- Audit emission (ADR 0034) — **applicable** (read/write attribution per ADR 0025).
- Knative trigger flow — **N/A** — B11 reacts to `Memory`/`MemoryStore` reconcile, not to a
  CloudEvent trigger flow; it *emits* lifecycle events.
- HolmesGPT toolset — **N/A** — no diagnostic toolset contributed by the adapter.
- 3-layer tests — **applicable** (Chainsaw: apply `MemoryStore` per mode, assert enforcement +
  lifecycle/audit events; PyTest: adapter mapping + access-mode logic + dual-substrate connection
  secret; Playwright: N/A — no UI).
- Tutorials & how-tos — **applicable** ("declare a memory store with access mode X for your agent").

## 9. Acceptance Criteria
- **AC-B11-01:** An agent using `memory.*` reaches Letta only through B11; a test asserting a direct
  Letta call from agent code fails the architecture lint. (→ REQ-B11-01, REQ-B11-02)
- **AC-B11-02:** A `MemoryStore` with `accessMode: private` denies read to any agent other than the
  writer. (→ REQ-B11-03, REQ-B11-04)
- **AC-B11-03:** A `MemoryStore` with `accessMode: namespace-shared` allows read by another agent in
  the same namespace and denies a cross-namespace read. (→ REQ-B11-03, REQ-B11-04)
- **AC-B11-04:** A `MemoryStore` with the RBAC/OPA-controlled mode denies an access RBAC did not
  grant even when OPA would allow (floor cannot be exceeded). (→ REQ-B11-05)
- **AC-B11-05:** After writing memory, the state is present in Postgres; deleting and rebuilding the
  OpenSearch index reproduces retrieval results from Postgres/object storage. (→ REQ-B11-06)
- **AC-B11-06:** The adapter connects using the B4 connection secret on both kind and AWS substrates
  with no substrate-specific code path. (→ REQ-B11-07)
- **AC-B11-07:** Letta runs namespace-scoped; a second namespace gets an isolated instance.
  (→ REQ-B11-08)
- **AC-B11-08:** A memory read and a memory write each produce an audit event attributed to the
  agent identity and the store mode at the audit endpoint. (→ REQ-B11-09)
- **AC-B11-09:** `MemoryStore` create/delete emit `platform.lifecycle.*` events on the broker.
  (→ REQ-B11-10)
- **AC-B11-10:** Swapping the backend behind B11 (fake backend in test) leaves the `Memory` /
  `memory.*` contract unchanged for agents. (→ REQ-B11-11)
- **AC-B11-11:** The shipped Letta version pin is recorded in the compatibility matrix; CI fails on
  an untested pin. (→ REQ-B11-12)

## 10. Risks & Open Questions
- **R1 (high):** The **`memory.*` method signatures are not specified in source** (B6 owns them).
  B11 cannot freeze its backend contract until B6's surface is fixed. Mitigation: co-design the
  seam with B6; B11 tests against a fake until B6 lands. Blast radius: high (every memory consumer).
- **R2 (med):** **Letta version pin** is `[PROPOSED — not in source]`. ARK/Letta are
  technical-preview-ish (§9); a pin bump may break the adapter mapping. Mitigation: regression
  suite + compat matrix gate.
- **R3 (med):** **Letta's own Postgres schema / table names** are `[PROPOSED — not in source]`;
  the "reproducible OpenSearch from primaries" invariant must be verifiable against whatever Letta
  actually stores. Mitigation: an explicit rebuild test (AC-B11-05).
- **R4 (med):** **B4 dependency is not in the CSV `upstream` column** for B11 but is architecturally
  required (B4's downstream lists B11). Treated as HARD in the plan; if B4 slips, B11 needs a fake
  `MemoryStore` + connection secret. Blast radius: med.
- **R5 (low):** Whether B11 is an **in-process library vs. a sidecar/service** is `[PROPOSED — not
  in source]`; affects deployment shape and the `/v1` HTTP-versioning question.
- **OQ1:** Does the RBAC/OPA-controlled mode call OPA in B11 directly, or via the B2 LiteLLM callback
  path? `[PROPOSED]` direct decision-point call at the backend boundary; OPA *content* is B16.
- **OQ2:** Cross-store joins / eviction / namespace-naming are explicitly **deferred** (ADR 0025,
  backlog §4) — B11 must not bake assumptions that foreclose the later model.

## 11. References
- architecture-overview.md §6.3 (line 294; storage roles, access modes, Postgres/OpenSearch
  dual-mode, substrate abstraction, audit pipeline), §6.2 (line 247; `memory.*`+`rag.*` SDK surface,
  agents never call backend directly), §9 (line 1390; Letta multi-tenancy mitigation, "wrap behind
  `Memory` CRD; namespace-scoped instances"), §6.13 (compat matrix).
- ADR 0005 (Letta backend; B11 thin adapter as isolation seam), ADR 0025 (memory access modes),
  ADR 0014 (Postgres primary / OpenSearch derived), ADR 0016 (namespaces), ADR 0018 (RBAC-floor /
  OPA-restrictor), ADR 0041 (substrate abstraction + connection secret), ADR 0030 (versioning),
  ADR 0034 (audit).
- interface-contract §1.2 (`Memory`), §1.6 (`MemoryStore`), §2 (taxonomy), §3.1 (`memory.*`), §4
  (connection secret). Glossary: Letta, `MemoryStore`, connection secret.
- Related pieces: A10 (Letta), B4 (`MemoryStore` XR + Compositions), A5 (ARK `Memory`), B6 (SDK).

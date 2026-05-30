# SPEC B13 — Custom Python kopf operator for LiteLLM

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [A1] · downstream: [A17, B16, B17] · adrs: [0006, 0013, 0032, 0030, 0031, 0034, 0018, 0002] · views: [6.8, 6.6, 6.12, 6.13]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B13 is the **custom Python kopf operator** that reconciles the LiteLLM-facing capability, key, and
budget CRDs into LiteLLM's admin API (and adjacent components) so the platform's capability surface is
declarative, GitOps-reconciled, and OPA-gated like every other surface. LiteLLM (A1) is the single
gateway for every LLM, MCP, and A2A call and exposes an admin API rather than a Kubernetes-native
interface (§6.1); ADR 0006 fixes the decision to bridge that gap with a kopf operator (Python, the
platform default) rather than a Go Crossplane provider, packaged as a **subchart of the LiteLLM Helm
chart** so a single ArgoCD-applied release deploys both with automatic version pinning.

B13 owns reconciliation for **`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`,
`CapabilitySet`, `VirtualKey`, `BudgetPolicy`** (§6.8, §6.12; ADR 0013), including the dependency
edges between them and LiteLLM's runtime registry. It is the controller that makes "approved
capability" a real architectural primitive: the same CRD set is the source of truth for LiteLLM
registry contents, Envoy egress allowlists, OPA admission/runtime decisions, Headlamp's per-agent
view, audit tagging, and observability labels. B13 is T0 foundation because the initial MCP services
(A17), initial OPA content (B16), and the agent profile library (B17) all depend on it.

## 2. Scope

### 2.1 In scope
- A **Python kopf operator** reconciling the eight Canon CRDs (`MCPServer`, `A2APeer`, `RAGStore`,
  `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) into **LiteLLM's admin API**
  and into **OPA data where applicable** (§6.6, §6.8; ADR 0006/0013).
- The **CRD definitions** for those eight kinds (namespaced) with the Canon field sets (§1.4), under
  the platform CRD versioning policy (ADR 0030).
- **CapabilitySet layering resolution** per the committed Helm-style overlay model (§6.8): referenced
  sets processed in declared order, field-level add-if-absent / replace-if-present, lists restated not
  concatenated, per-Agent overrides applied last (detailed sub-semantics deferred to ADR 0032).
- Emitting the **`platform.capability.changed` CloudEvent** on any capability CRD add/update/delete
  (ADR 0013) onto the broker.
- **`VirtualKey` issuance reconciliation**: binding a key to a `CapabilitySet` + identity + budget +
  environment + allowed models + ttl, supporting the Headlamp-driven, OPA-gated self-serve flow (§6.6).
- **`BudgetPolicy`** reconciliation into both LiteLLM (enforcement) and OPA data (decisioning) (§1.4).
- Wiring `EgressTarget`s into the **Envoy egress proxy allowlist** and `RAGStore`/`Skill`/`MCPServer`/
  `A2APeer` into LiteLLM as the gateway-applied state.
- An **HTTP admin API** for the operator versioned under **`/v1/...`** (URL-path versioning, §6.13).
- **Audit emission** of capability-registry reconcile events via the audit adapter (ADR 0034) and
  the standard custom-component deliverables (HolmesGPT toolset, OPA contribution, observability,
  Knative trigger flow, tests, docs — ADR 0006).
- Packaging as a **subchart of the LiteLLM Helm chart** (single ArgoCD release, version-pinned).

### 2.2 Out of scope (and where it lives instead)
- **LiteLLM gateway install/config itself** → **A1**; B13 reconciles into A1's admin API.
- **OPA/Gatekeeper engine + admission** → **A7**; B13 writes OPA *data* and CRDs that Gatekeeper
  admits, but does not own the engine.
- **OPA policy framework + content** → **B3 (framework) / B16 (content)**; B13 supplies data (budgets),
  not policy rules.
- **Cloud-shaped XRs** (`AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, `GrafanaDashboard`,
  substrate XRDs) → **Crossplane / B4**; the kopf/Crossplane split is an invariant (ADR 0006).
- **ARK CRDs** (`Agent`, `AgentRun`, …) → **A5**; the `Agent.capabilitySetRefs[]`/`overrides` reference
  CapabilitySets B13 reconciles, but `Agent` reconciliation is ARK's.
- **Detailed CapabilitySet overlay sub-semantics** (recursive includes, missing-ref validation,
  cross-namespace rules, override-grant question) → **ADR 0032 / design specs**.
- **CloudEvent per-event-type schemas** (incl. `platform.capability.changed` schema) → **B12**.
- **Envoy egress proxy itself / Skill gateway internals** → **A6 (ADR 0003) / LiteLLM skill gateway**.
- **OpenAPI→MCP conversion** (`SyntheticMCPServer`) → **A12**; B13 reconciles the resulting `MCPServer`.

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM)** — the gateway whose admin API B13 reconciles into; B13 ships as its Helm subchart.

**Downstream consumers:**
- **A17 (initial MCP services integration)** — declared as `MCPServer` CRDs B13 reconciles.
- **B16 (initial OPA policy content)** — consumes the OPA data (budgets, capability facts) B13 writes
  and gates the CRDs/flows B13 drives.
- **B17 (agent profile library)** — profiles ship as `CapabilitySet`s B13 reconciles + layers.

**ADRs honored:**
- **ADR 0006** — Python kopf operator for LiteLLM reconciliation; kopf for app-API, Crossplane for
  cloud-shaped (invariant); subchart of LiteLLM; `/v1/...` admin API; standard custom-component
  deliverables; emits reconcile events to the audit pipeline.
- **ADR 0013** — owns the eight capability/key/budget CRDs and their dependency edges; emits
  `platform.capability.changed`; namespace-private default with opt-in OPA-checked cross-tenant share.
- **ADR 0032** — CapabilitySet overlay semantics; B13 implements the committed model, defers
  sub-semantics.
- **ADR 0030 / §6.13** — Kubernetes API versioning on its CRDs (B13 is the versioning owner); `/v1/...`
  URL-path versioning on the admin API.
- **ADR 0031** — `platform.capability.changed` and any other emission fall under exactly one Canon
  namespace; concrete schema in B12.
- **ADR 0034** — reconcile/audit events flow under `platform.audit.*` via the audit adapter; no direct
  writes to Postgres/S3/OpenSearch.
- **ADR 0018 / 0002** — RBAC-as-floor / OPA-as-restrictor; Gatekeeper admits the capability CRDs; OPA
  gates dynamic registration + virtual-key issuance against `capability_set_refs` claims.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B13 **owns and defines** these eight namespaced CRDs with the Canon field sets (interface-contract
§1.4). It is the versioning owner (ADR 0030). All fields below are **Canon (source-stated)**; no field
is invented. Any field beyond these is flagged.

- **`MCPServer`** — `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`,
  `visibility`.
- **`A2APeer`** — `endpoint`, `direction` (internal/external), `auth`, `tags`.
- **`RAGStore`** — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`.
- **`EgressTarget`** — `fqdn`, `port`, `scheme`, `allowedMethods`.
- **`Skill`** — `gitRef`, `versionPin`, `schemaRef` (managed via LiteLLM's skill gateway).
- **`CapabilitySet`** — `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`,
  `llmProviders[]`, `opaPolicyRefs[]` (layerable; overlay per §6.8 / ADR 0032).
- **`VirtualKey`** — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`,
  `ttl`.
- **`BudgetPolicy`** — `scope` (key/agent/team/tenant), `period`, `limits`, `thresholdActions[]`
  (consumed by OPA + LiteLLM).

All are **namespaced** (no cluster-scoped platform CRDs in v1.0). `[PROPOSED — not in source]` for any
status/condition subresource field names — source states the field *sets* above but does not enumerate
`status` fields; a `status` with reconcile state + applied-revision is proposed and flagged.

### 4.2 APIs / SDK surfaces
- **Reconciliation into LiteLLM's admin API** (A1) — B13 translates each CRD into LiteLLM admin-API
  calls; the LiteLLM admin-API surface is A1's, not redefined here.
- **OPA data writes** — `BudgetPolicy` limits and capability facts written into OPA `data` for
  runtime decisions (consumed by B16 content, B2 callbacks). The `data`-document namespacing follows
  B3's convention; concrete data paths `[PROPOSED — not in source]`.
- **Envoy egress allowlist** — `EgressTarget` (`fqdn`/`port`/`scheme`/`allowedMethods`) reconciled into
  the egress proxy allowlist (A6/ADR 0003). The wiring mechanism is `[PROPOSED — not in source]`.
- **Operator HTTP admin API** — versioned under **`/v1/...`** (Canon, §6.13/ADR 0006); deprecated
  versions reachable ≥1 platform release after replacement. Concrete endpoints `[PROPOSED — not in
  source]`.
- **CapabilitySet resolution** — implements the §6.8 committed overlay (declared-order, field-level
  add/replace, lists restated, per-Agent `overrides` last). Resolved-view output feeds Headlamp
  (B5/A22) and LiteLLM applied state.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** **`platform.capability.changed`** (Canon, ADR 0013) on any capability CRD
  add/update/delete (MCPServer/A2APeer/RAGStore/EgressTarget/Skill/CapabilitySet). Under
  `platform.capability.*`. Consumed by the platform SDK (B6) → agents.
- **Emitted (mediated):** capability-registry reconcile/audit events under **`platform.audit.*`** via
  the audit adapter (ADR 0034).
- `[PROPOSED — not in source]` whether B13 also emits `platform.gateway.*` (e.g. MCP health changes —
  §6.7 attributes "MCP health changes" to gateway events, which may be LiteLLM's own emission, A1, not
  B13). Concrete schemas for all the above live in **B12**; B13 mints no event-type names beyond
  `platform.capability.changed`.

### 4.4 Data schemas / connection-secret contracts
- B13 reads **`credentialsRef`** (`MCPServer`) and `auth` (`A2APeer`) via **ESO / Kubernetes Secrets**
  — ADR 0006 states the operator implements equivalent secret wiring (ESO / Secrets) since Crossplane's
  connection-secret handling is unavailable for LiteLLM-specific CRDs. It does **not** consume the
  substrate connection-secret shape (ADR 0041) — that is for cloud-shaped XRs (B4), out of B13's scope.
- B13 owns **no datastore of its own**; state lives in the CRDs (Kubernetes API) and LiteLLM's registry.

## 5. OSS-vs-Custom Decision
**Build-new** custom controller. Upstream: **kopf** (Python operator framework) — **wrap** (build the
operator on kopf), not fork. Rejected alternative: a **Go Crossplane provider for LiteLLM** (ADR 0006 —
no Go experience; kopf keeps Python default; bounded loss of Crossplane's secret/package/Composition
integration for LiteLLM-specific CRDs only). Packaging: **subchart of the LiteLLM Helm chart** (A1) —
single ArgoCD release, automatic gateway↔operator version pinning. kopf version `[PROPOSED — not in
source]` (pin in the subchart). ADR linkage: 0006 (the decision), 0013 (the CRD set), 0030 (versioning).

## 6. Functional Requirements
- **REQ-B13-01:** B13 MUST define and reconcile the eight namespaced CRDs — `MCPServer`, `A2APeer`,
  `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy` — with the Canon
  field sets (§1.4).
- **REQ-B13-02:** B13 MUST reconcile each CRD into **LiteLLM's admin API** so capabilities are
  declarative and GitOps-reconciled, not configured directly on the gateway.
- **REQ-B13-03:** B13 MUST resolve **CapabilitySet layering** per the §6.8 committed model: referenced
  sets in declared order, field-level **add-if-absent / replace-if-present**, lists restated (not
  concatenated), per-Agent **`overrides` applied last**.
- **REQ-B13-04:** B13 MUST emit a **`platform.capability.changed`** CloudEvent on every capability CRD
  add/update/delete (ADR 0013), under `platform.capability.*`.
- **REQ-B13-05:** B13 MUST reconcile **`VirtualKey`** issuance (binding `capabilitySetRef`, identity,
  `budgetRef`, `environment`, `allowedModels[]`, `ttl`) supporting the Headlamp-driven, OPA-gated,
  RBAC-aware self-serve flow (§6.6).
- **REQ-B13-06:** B13 MUST reconcile **`BudgetPolicy`** into both **LiteLLM** (enforcement) and **OPA
  data** (decisioning), per its `scope`/`period`/`limits`/`thresholdActions[]` (§1.4).
- **REQ-B13-07:** B13 MUST reconcile **`EgressTarget`** into the **Envoy egress proxy allowlist**
  (`fqdn`/`port`/`scheme`/`allowedMethods`) (ADR 0003).
- **REQ-B13-08:** B13 MUST read secrets for `MCPServer.credentialsRef` / `A2APeer.auth` via **ESO /
  Kubernetes Secrets** (ADR 0006), never embedding credentials in CRD spec or LiteLLM config plaintext.
- **REQ-B13-09:** B13 MUST expose an **HTTP admin API versioned under `/v1/...`** with deprecated
  versions reachable ≥1 platform release after replacement (§6.13).
- **REQ-B13-10:** B13 MUST follow **Kubernetes CRD versioning** (ADR 0030) as the versioning owner for
  its eight CRDs (conversion webhooks on a new `vN` group; `vN-1` deprecated ≥1 minor release).
- **REQ-B13-11:** B13 MUST **emit capability-registry reconcile/audit events** under `platform.audit.*`
  via the **audit adapter** (ADR 0034) — no direct writes to Postgres/S3/OpenSearch.
- **REQ-B13-12:** B13 MUST be packaged as a **subchart of the LiteLLM Helm chart** so one ArgoCD
  release deploys gateway + operator with automatic version pinning (ADR 0006).
- **REQ-B13-13:** B13 MUST NOT reconcile cloud-shaped XRs (Crossplane/B4 domain) — the kopf/Crossplane
  split invariant (ADR 0006).
- **REQ-B13-14:** Cross-namespace CapabilitySet references MUST default to **namespace-private** and be
  reachable only via explicit **OPA-checked publication** (ADR 0013, §6.9).
- **REQ-B13-15:** B13 MUST surface the **resolved per-Agent capability view** (from CRDs + LiteLLM
  applied state) for the Headlamp unified view (§6.8).

## 7. Non-Functional Requirements
- **Security:** RBAC-floor / OPA-restrictor (ADR 0018) — Gatekeeper admits capability CRDs; OPA gates
  dynamic registration + virtual-key issuance against `capability_set_refs`; secrets via ESO only;
  the operator never grants beyond RBAC.
- **Multi-tenancy (§6.9):** all CRDs namespaced = tenancy boundary; cross-namespace sharing is
  opt-in, OPA-checked; reconcile actions stay within the resource's namespace scope.
- **Observability (§6.5):** reconcile outcomes emit audit + metrics; operator exposes health/readiness;
  the resolved-view feeds Headlamp; `platform.capability.changed` is observable.
- **Scale:** reconciliation is idempotent and convergent; capability-change events are best-effort
  (enforcement stays at LiteLLM/OPA/Envoy); layering resolution is deterministic and bounded.
- **Versioning (ADR 0030):** B13 owns its CRD versioning lifecycle (conversion webhooks, deprecation
  windows) and `/v1/...` admin-API versioning; ships with LiteLLM as a pinned subchart.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** — packaged as a **subchart of the LiteLLM Helm chart** (ADR 0006).
- Per-product docs (10.5) — **applicable** (CRD reference, layering model, admin-API `/v1`, how-to-add
  a capability).
- Runbook (10.7) — **applicable** (reconcile-failure, LiteLLM-admin-API drift, stuck-finalizer recovery).
- Alerts — **applicable** (reconcile failure, admin-API unreachable, capability-event publish failure).
  `[PROPOSED — not in source]` for concrete alert thresholds.
- Grafana dashboard (Crossplane XR) — **applicable** — operator reconcile/health metrics via a
  `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — **applicable (feeds)** — B13 produces the resolved per-Agent view the Headlamp
  capabilities plugin (B5/A22) renders; the plugin itself is B5/A22.
- OPA/Rego integration — **applicable** — writes OPA data (budgets, capability facts); CRDs OPA-gated.
- Audit emission (ADR 0034) — **applicable** — reconcile/audit events via the adapter.
- Knative trigger flow — **applicable** — emits `platform.capability.changed`; designs its own flow
  (capability change → SDK notification) per standard deliverables.
- HolmesGPT toolset — **applicable** — a toolset for capability-registry inspection/diagnosis (ADR 0006).
- 3-layer tests — **applicable** (Chainsaw for CRD reconcile + layering + capability-event emission;
  PyTest for resolution/secret-wiring logic; Playwright N/A here — virtual-key self-serve UI is
  Headlamp/B5).
- Tutorials & how-tos — **applicable** (register an MCP server / build a CapabilitySet how-tos).

## 9. Acceptance Criteria
- **AC-B13-01:** Applying each of the eight CRDs reconciles to the expected LiteLLM admin-API / OPA-data
  / Envoy-allowlist state. (→ REQ-B13-01, REQ-B13-02)
- **AC-B13-02:** A two-CapabilitySet + overrides example resolves exactly to the §6.8 pseudocode result
  (declared-order, add/replace, lists restated, overrides last). (→ REQ-B13-03)
- **AC-B13-03:** Adding/updating/deleting any capability CRD emits a `platform.capability.changed`
  event under `platform.capability.*`. (→ REQ-B13-04)
- **AC-B13-04:** A `VirtualKey` reconciles to a LiteLLM key bound to its `capabilitySetRef`, identity,
  budget, environment, allowed models, and ttl. (→ REQ-B13-05)
- **AC-B13-05:** A `BudgetPolicy` appears in both LiteLLM enforcement config and OPA data with its
  scope/period/limits/threshold actions. (→ REQ-B13-06)
- **AC-B13-06:** An `EgressTarget` reconciles into the Envoy allowlist; a request to a non-allowlisted
  FQDN is blocked. (→ REQ-B13-07)
- **AC-B13-07:** `MCPServer.credentialsRef` is resolved via ESO/Secret; no credential appears in CRD
  spec or gateway config plaintext. (→ REQ-B13-08)
- **AC-B13-08:** The operator admin API serves under `/v1/...`; a deprecated prior version remains
  reachable per policy. (→ REQ-B13-09)
- **AC-B13-09:** A breaking CRD change ships as a new `vN` group with a conversion webhook; `vN-1`
  remains for ≥1 minor release. (→ REQ-B13-10)
- **AC-B13-10:** A reconcile action produces a `platform.audit.*` event via the adapter and no direct
  store write. (→ REQ-B13-11)
- **AC-B13-11:** Installing the LiteLLM Helm chart deploys the operator subchart in one ArgoCD release
  with pinned versions. (→ REQ-B13-12)
- **AC-B13-12:** A test confirms B13 reconciles none of the Crossplane/B4 XRs. (→ REQ-B13-13)
- **AC-B13-13:** A cross-namespace CapabilitySet reference is denied by default and admitted only with
  OPA-checked publication. (→ REQ-B13-14)
- **AC-B13-14:** The operator exposes a resolved per-Agent capability view sourced from CRDs + LiteLLM
  applied state. (→ REQ-B13-15)

## 10. Risks & Open Questions
- **R1 (high):** CRD `status`/condition field names are `[PROPOSED — not in source]`; Headlamp (B5/A22)
  and operators bind to them. Mitigation: B13 owns + freezes the status subresource; flag until
  confirmed. Blast radius: med-high (UI + ops).
- **R2 (high):** OPA-data document paths and the Envoy-allowlist wiring mechanism are `[PROPOSED — not
  in source]`. B16 (policy content) and A6 (egress) bind to them. Mitigation: align OPA-data paths with
  B3's `data`-namespacing convention; coordinate egress wiring with A6. Blast radius: high.
- **R3 (med):** CapabilitySet overlay **sub-semantics** (recursive includes, missing/deleted-ref
  validation, cross-namespace rules, whether overrides may grant beyond referenced sets) are deferred
  to ADR 0032. B13 implements only the committed model now; premature implementation risks rework.
  Mitigation: gate sub-semantic behavior behind ADR 0032; keep resolution pluggable.
- **R4 (med):** Whether B13 emits `platform.gateway.*` (MCP health changes) vs that being LiteLLM's own
  emission is `[PROPOSED]`. Mitigation: default to LiteLLM/A1 owning gateway events; B13 emits only
  `platform.capability.changed` + audit.
- **R5 (med):** `platform.capability.changed` payload schema is owned by **B12** but B13 must emit it;
  if B12 lands later, B13 needs an interim fixture. Mitigation: contract test against a B12 fixture;
  flag interim schema.
- **R6 (low):** kopf is single-process by default; HA/leader-election for the operator is `[PROPOSED —
  not in source]`. Mitigation: standard kopf peering/leader-election; flag.
- **OQ1:** Does B13 reconcile budgets into OPA data directly or hand off to B3/B2? §6.6 says OPA + LiteLLM
  consume `BudgetPolicy`. `[PROPOSED]` B13 writes OPA data; B16 authors the rules over it.
- **OQ2:** Does `VirtualKey` issuance call OPA inline at reconcile, or does Gatekeeper/the Headlamp flow
  gate before the CRD is created? §6.6 implies OPA gates issuance. `[PROPOSED]` OPA gates at the
  Headlamp request + Gatekeeper admission; B13 reconciles admitted `VirtualKey`s.

## 11. References
- architecture-overview.md §6.8 (lines 632–718; capability CRDs reconciled by the kopf operator into
  LiteLLM, layering model lines 679–714, Headlamp resolved view, `platform.capability.changed` lines
  718), §6.6 (lines 491–495; self-serve `VirtualKey` issuance, OPA-gated), §6.12 (CRD inventory),
  §6.13 (lines 987, 992; CRD versioning owner = kopf, `/v1/...` admin API).
- interface-contract.md §1.4 (the eight CRDs + Canon fields), §1.1 (CRD versioning), §2
  (`platform.capability.changed`), §5 (audit adapter), §6 (overlay semantics deferred to ADR 0032).
- ADR 0006 (Python kopf operator; subchart; `/v1`; deliverables), ADR 0013 (capability CRD model +
  `platform.capability.changed`), ADR 0032 (overlay semantics), ADR 0030 (versioning), ADR 0031
  (taxonomy), ADR 0034 (audit), ADR 0018 / 0002 (RBAC-floor / OPA-restrictor, Gatekeeper).
- Related pieces: A1 (LiteLLM), A6 (Envoy egress), A7 (OPA), A17 (MCP services), B3/B16 (OPA framework/
  content), B17 (profiles), B5/A22 (Headlamp), B12 (event schemas), B6 (SDK consumes capability events).

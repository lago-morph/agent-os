# SPEC B5 — Cross-cutting Headlamp plugins

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A23] · downstream: [B19] · adrs: [0040, 0021, 0017, 0039, 0029, 0034, 0006] · views: [6.12, 11, 6.9, 7.5]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

Many platform OSS components lack adequate per-product admin UIs; Headlamp plugins are the platform's unified admin surface to compensate (§9). B5 ships the **cross-cutting Headlamp plugins** — those that span components rather than belonging to a single one — built on the A9 Headlamp framework and the A22 graphical-editor framework. The cross-cutting set comprises: a **capability inspector** (view the capability registries — `MCPServer`/`A2APeer`/`RAGStore`/`EgressTarget`/`Skill`/`CapabilitySet`/`VirtualKey`/`BudgetPolicy` and their LiteLLM-registry state), a **virtual key admin** UI (the self-serve, OPA-gated `VirtualKey` issuance experience, §6.6), the **approval-queue UI** (the **B19 UI** — the Headlamp approval-queue plugin from §7.5), and the **Kargo plugin** (the **A23** cross-cutting plugin — Stage/Warehouse/promotion status + triggering, ADR 0040).

Per the B19 → A23 → B5 cycle resolution (waves.md), B19 is split: **B19-core (W3)** ships the `Approval` CRD + OPA elevation + Argo Workflow integration + decision CloudEvents; **B19-ui (W4)** is the approval-queue plugin, which **B5 owns and co-lands with the Kargo plugin in W4**. So although piece-index.csv shows `B5 → B19` (downstream) and `A23 → B5` (upstream), the build order is acyclic: B19-core → A23 → (B5/B19-ui co-land). B5 is therefore the home of two plugins nominally attributed elsewhere: the B19 approval-queue UI and the A23 Kargo plugin.

## 2. Scope

### 2.1 In scope
- **Capability inspector plugin** — read views over the capability registries (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) and their reconciled LiteLLM-registry / health state (§6.1, §6.8). Includes MCP server health status surfaced by LiteLLM (§6.1).
- **Approval queue UI plugin (the B19 UI / B19-ui)** — the workqueue surfaced from §7.5: lists `Approval` requests filtered by the **resolved** approval level, shows evidence refs, and lets an authorized approver approve/reject, resuming the backing Argo Workflow (§7.5; ADR 0017).
- **Virtual key admin plugin** — the self-serve `VirtualKey` issuance UI: a requester picks scope (CapabilitySet binding, budget, environment); OPA decides; approved keys returned via the secure mechanism (§6.6). Includes admin override (force-register / force-deregister of dynamic A2A/MCP, §6.9).
- **Kargo plugin (the A23 plugin)** — status visualization (Stages, Warehouses, in-flight promotions) and triggering promotions from Headlamp; one deep-link per Stage (ADR 0040, §11).
- Plugin **gating logic in plugin code** keyed on §6.9 Keycloak claims (`platform_roles`, `platform_namespaces`, `tenant_roles`, resolved approval level) — Headlamp OIDC works but plugin gating is DIY (§9).
- Auth handoff via the A9 framework; editor widgets via the A22 framework where a plugin includes edit (not just read).
- Audit emission of plugin-driven actions (key issuance, approve/reject, force-register, promote) via the platform audit adapter (ADR 0034).

### 2.2 Out of scope (and where it lives instead)
- **Headlamp install + framework** (common libraries, shared widgets, auth handoff) — Component **A9**. B5 consumes it.
- **Graphical-editor framework + the per-CRD editor set** (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `Agent`, `TenantOnboarding`, `AuditLog`, `LogLevel`, `GrafanaDashboard`) — Component **A22** (ADR 0039). B5's inspector/admin plugins reuse these editors; B5 does not re-author them. (`BudgetPolicy`, `Memory`, `XAgentDatabase` editors are excluded from A22's v1.0 set — §14.1 A22 — so the virtual-key/capability plugins surface those read-only or via raw view.)
- **B19-core** — the `Approval` CRD, OPA elevation logic, Argo Workflow integration, decision CloudEvent emission — Component **B19** (W3). B5 ships only the approval-queue **UI** (B19-ui).
- **Kargo install** + Stages + Warehouse + verification configs — Component **A23** (ADR 0040). B5 ships only its Headlamp plugin.
- **Per-component plugins** owned by their own components (e.g., A20 policy-simulator panel, A21 tenant-onboarding editor, OPA policy-review plugin if owned by A7) — not B5 unless explicitly cross-cutting.
- **kopf reconciliation** of capability CRDs into LiteLLM — Component **B13** (ADR 0006). The inspector reads reconciled state; it does not reconcile.
- The **policy simulator service** + Headlamp panel — Component **A20** (ADR 0038); B5 plugins may deep-link to it (preview-before-commit) but do not implement it.

## 3. Context & Dependencies

**Upstream consumed:**
- **A23 (Kargo)** — the Kargo API/CRDs the Kargo plugin visualizes and drives; per-Stage deep-links (ADR 0040). (piece-index `A23 → B5`.)
- **A9 (Headlamp framework)** — install, shared widgets, auth handoff (implicit; required for any plugin).
- **A22 (editor framework)** — schema-driven form widgets, validation hooks, policy-simulator integration, GitOps-diff preview, reused by edit-capable plugins (ADR 0039; implicit upstream).
- **B19-core** — the `Approval` CRD shape (`requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`) and the resolved-level + Argo-Workflow resume contract the queue UI drives (ADR 0017).

**Downstream consumers:**
- **B19** — the approval system needs B5's approval-queue UI to be usable end-to-end (piece-index `B5 → B19`; resolved by the B19-core/B19-ui split — B5 supplies B19-ui).

**ADRs honored:**
- **ADR 0040** — Kargo plugin covers Stage/Warehouse/promotion status + triggering; one deep-link per Stage; extends the Headlamp deep-link inventory.
- **ADR 0017** — approval-queue UI is the §7.5 Headlamp approval UI; filters by resolved level; approve/reject resume the Argo Workflow; no M-of-N / escalation / delegation in v1.0.
- **ADR 0039** — edit-capable plugins build on the A22 editor framework + policy-simulator preview; B5 does not duplicate the framework.
- **ADR 0029** — plugin gating consumes only the §6.9 claim schema.
- **ADR 0034** — plugin-driven actions emit audit via the adapter.
- **ADR 0021** — any dashboards a plugin surfaces follow the `GrafanaDashboard` XR model; visibility RBAC + OPA.
- **ADR 0006** — capability inspector reads kopf-reconciled (B13) LiteLLM registry state; it does not reconcile.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A (owns none) — B5 **reads / drives** existing CRDs: `Approval` (B19), the capability CRDs `MCPServer`/`A2APeer`/`RAGStore`/`EgressTarget`/`Skill`/`CapabilitySet`/`VirtualKey`/`BudgetPolicy` (B13-reconciled), and Kargo's resources (A23). All are namespaced (§6.12). Plugins must respect each CRD's per-component versioning (ADR 0030).

### 4.2 APIs / SDK surfaces
- **Consumes** the Headlamp plugin API / A9 framework (TypeScript — the explicit Headlamp/TypeScript exception to Python-default, ADR 0006 context) and the A22 editor-widget API.
- **Consumes** the Kubernetes API (via Headlamp) for CRD reads/writes; the LiteLLM admin/registry surface for health/registry state (read); the Kargo API for promotion status/trigger; the audit adapter (action emission).
- **Consumes** the §6.9 claims for gating. `[PROPOSED — not in source]` exact Headlamp plugin extension-point names / widget signatures (framework-defined by A9/A22, not Canon).
- The virtual-key issuance "secure key delivery mechanism" is design-time (§6.6). `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed (for display):** `platform.approval.*` (approval requested/elevated/decided/timed-out) drives the queue; `platform.capability.*` (registry changes incl. `platform.capability.changed`) refreshes the inspector; Kargo promotion events for the Kargo plugin.
- **Emitted (action audit):** key issuance, approve/reject, force-register/deregister, promote → `platform.audit.*` via adapter (ADR 0034). Approve/reject decisions themselves are emitted by **B19-core** (the Argo Workflow), not the UI — the UI resumes the workflow which emits `platform.approval.*` (§7.5). `[PROPOSED — not in source]` concrete event names (B12 registry).

### 4.4 Data schemas / connection-secret contracts
- No connection-secret XRD owned. Reads CRD specs/status as defined by their owning components. The approval-queue UI consumes the `Approval` `evidenceRefs[]` (links to traces/audit/RAG) for display (§7.5). Dashboards (if surfaced) via `GrafanaDashboard` XR (ADR 0021).

## 5. OSS-vs-Custom Decision

**Build-new plugins on the OSS Headlamp framework.** Upstream: **Headlamp** (A9 install + framework) provides the plugin host; **A22** provides editor widgets. B5 builds custom TypeScript plugins because (a) the OSS components lack unified admin UIs (§9 — "Headlamp plugins as the unified admin UI to compensate"), (b) Headlamp OIDC works but plugin gating is DIY (§9), and (c) the Kargo plugin and approval-queue UI are explicit cross-cutting B5 deliverables (ADR 0040 Consequences; waves.md cycle resolution). No fork of Headlamp. ADR linkage: ADR 0040 (Kargo plugin), ADR 0017/0039 (approval + editor reuse).

## 6. Functional Requirements

- **REQ-B5-01:** B5 MUST ship a **capability inspector** plugin presenting read views over `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, and `BudgetPolicy`, including reconciled LiteLLM-registry and MCP-server health state (§6.1, §6.8).
- **REQ-B5-02:** B5 MUST ship the **approval-queue UI** (B19-ui): a workqueue of `Approval` requests filtered by the **resolved** approval level, showing `actionAttributes` and `evidenceRefs[]`, with approve/reject that resumes the backing Argo Workflow (§7.5, ADR 0017).
- **REQ-B5-03:** B5 MUST ship a **virtual key admin** plugin implementing the self-serve `VirtualKey` issuance flow: requester selects scope (CapabilitySet binding, budget, environment); OPA gates; approved keys are returned via the secure delivery mechanism; no admin manually issues keys (§6.6).
- **REQ-B5-04:** B5 MUST ship the **Kargo plugin** visualizing Stages, Warehouses, and in-flight promotions, and allowing promotion triggering from Headlamp, with one deep-link per Stage (ADR 0040, §11).
- **REQ-B5-05:** Every plugin MUST gate visibility and actions on the §6.9 Keycloak claims (`platform_roles`, `platform_namespaces`, `tenant_roles`, resolved approval level) in plugin code (§9, ADR 0029).
- **REQ-B5-06:** Approve/reject and promote actions MUST be authorized by OPA (RBAC-as-floor / OPA-as-restrictor) before the plugin issues them; the approval-queue UI MUST only show requests to approvers at or above the resolved level (§7.5, ADR 0018).
- **REQ-B5-07:** Plugin-driven actions (key issuance, approve/reject, force-register/deregister, promote) MUST emit audit events via the platform audit adapter (ADR 0034); the UI MUST NOT write audit directly.
- **REQ-B5-08:** Edit-capable plugins MUST build on the A22 editor framework (schema-driven forms, validation, policy-simulator preview, GitOps-diff) rather than re-implementing editing (ADR 0039).
- **REQ-B5-09:** The virtual-key/capability plugins MUST handle CRDs whose A22 editors are excluded from the v1.0 set (`BudgetPolicy`, `Memory`, `XAgentDatabase`) by surfacing them read-only / via raw view (§14.1 A22).
- **REQ-B5-10:** The Kargo plugin and the approval-queue UI MUST co-land in W4 per the B19-core/B19-ui split; B5 MUST NOT re-implement B19-core (`Approval` CRD, OPA elevation, Argo integration, decision events) — it consumes them (waves.md).
- **REQ-B5-11:** The capability inspector MUST support the §6.9 admin-override force-register / force-deregister of dynamic A2A/MCP exposings through the plugin (§6.9).
- **REQ-B5-12:** Plugins MUST respect each consumed CRD's per-component API version and degrade gracefully across version skew (ADR 0030).

## 7. Non-Functional Requirements

- **Security:** plugin gating is the §6.9-claim DIY layer (§9); no action bypasses OPA; the secure key-delivery mechanism must not expose keys in logs/URLs (§6.6). Force-register is an audited admin override.
- **Multi-tenancy (§6.9):** every view is namespace/tenant-scoped from claims; tenant A's capabilities/approvals/keys are invisible to tenant B unless policy permits; cross-tenant visibility is OPA-driven, not implicit.
- **Observability (§6.5):** plugin actions traced/audited; any surfaced dashboards are `GrafanaDashboard` XRs with RBAC+OPA visibility (ADR 0021); Kargo plugin reflects per-Stage verification results (ADR 0040).
- **Scale:** read views paginate over potentially many CRDs; queue UI filters server-side by resolved level to avoid leaking out-of-level requests.
- **Versioning (ADR 0030):** plugins pin to consumed CRD versions; the plugin bundle is versioned and pinned to the Headlamp/A22 framework version.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests — **applicable** (plugin bundle packaging/deploy into Headlamp; manifests in `envs/<stage>/`).
- Per-product docs (10.5) — **applicable** (plugin usage docs: inspector, approval queue, virtual-key, Kargo).
- Runbook (10.7) — **applicable** (plugin-broken / claim-gating-misfire / stuck-approval recovery).
- Alerts — **applicable** (plugin load failure, approval-queue staleness, promote-action errors) — mostly UI/health signals.
- Grafana dashboard (Crossplane XR) — **applicable** where a plugin surfaces metrics (e.g., approval throughput) — `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — **applicable** — this component *is* the cross-cutting Headlamp plugins (capability inspector, approval-queue UI, virtual-key admin, Kargo plugin).
- OPA/Rego integration — **applicable** (approve/reject, promote, key issuance, force-register all OPA-gated; plugins consume A20 simulator for preview).
- Audit emission (ADR 0034) — **applicable** (every plugin-driven action).
- Knative trigger flow — **applicable** (consumes `platform.approval.*` / `platform.capability.*` / Kargo events for live refresh; emits action audit). `[PROPOSED — not in source]` concrete names.
- HolmesGPT toolset — **applicable** (e.g., "list pending approvals" / "capability inventory" query helpers). `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (Playwright primary — UI flows for queue/key/inspector/Kargo; PyTest for any gating/claim-mapping helper logic; Chainsaw for the CRD-state the plugins read, e.g., create `Approval` → expect it in the queue).
- Tutorials & how-tos — **applicable** ("approve an agent action", "issue a virtual key", "promote via Kargo from Headlamp" how-tos).

## 9. Acceptance Criteria

- **AC-B5-01** (REQ-B5-01): The inspector lists all eight capability CRD kinds with reconciled registry status and shows MCP-server health. (Playwright/Chainsaw)
- **AC-B5-02** (REQ-B5-02): An `Approval` created at a resolved level appears in the queue for an approver at that level with its evidence refs; approve resumes the Argo Workflow and the decision is recorded. (Playwright + Chainsaw)
- **AC-B5-03** (REQ-B5-03): A requester issues a `VirtualKey` for a chosen CapabilitySet/budget/environment scope; OPA-allowed issuance succeeds and the key is delivered securely; no admin manual step occurs. (Playwright)
- **AC-B5-04** (REQ-B5-04): The Kargo plugin shows Stages/Warehouses/in-flight promotions and can trigger a promotion; each Stage has a deep-link. (Playwright)
- **AC-B5-05** (REQ-B5-05): A user lacking the required `platform_roles`/namespace claim cannot see or act on the gated view. (Playwright)
- **AC-B5-06** (REQ-B5-06): An OPA-denied approve/reject or promote is blocked by the plugin; the queue never surfaces a request below the approver's resolved level. (Playwright/PyTest)
- **AC-B5-07** (REQ-B5-07): Key issuance, approve/reject, force-register, and promote each emit an audit event via the adapter; no direct audit write. (Chainsaw/PyTest)
- **AC-B5-08** (REQ-B5-08): An edit-capable plugin renders the A22 schema-driven form with policy-simulator preview, not a bespoke editor. (Playwright)
- **AC-B5-09** (REQ-B5-09): `BudgetPolicy`/`Memory`/`XAgentDatabase` appear read-only / raw (no A22 editor) without error. (Playwright)
- **AC-B5-10** (REQ-B5-10): B5 contains no `Approval` reconciler / OPA-elevation / Argo-integration code (those are B19-core); the queue UI only drives the existing workflow. (code review / PyTest)
- **AC-B5-11** (REQ-B5-11): An admin force-registers and force-deregisters a dynamic A2A/MCP exposing via the inspector, audited. (Playwright/Chainsaw)
- **AC-B5-12** (REQ-B5-12): The plugins operate against a CRD served at its current API version and warn (not crash) on an unexpected version. (PyTest)

## 10. Risks & Open Questions

- **R-B5-1** (high): The B19→A23→B5 cycle — if B19-core or A23 slips, B5/B19-ui cannot co-land. Reconciliation: build B19-ui against the B19-core `Approval` contract with a mock workflow-resume; build the Kargo plugin against a mock Kargo API; integrate when W3 lands. Blast radius high (W4 critical path).
- **OQ-B5-1** (med): Exact Headlamp/A22 plugin extension-point and widget signatures are framework-defined, not Canon. `[PROPOSED — not in source]`; bind to A9/A22 as they land.
- **OQ-B5-2** (med): The "secure key delivery mechanism," rotation, and scope grammar for virtual keys are design-time (§6.6). `[PROPOSED — not in source]`.
- **R-B5-2** (med): Plugin gating is DIY in plugin code (§9) — a gating bug leaks cross-tenant data. Mitigated by AC-B5-05/-06 + namespace-scoped reads; blast radius per-tenant.
- **OQ-B5-3** (low): Whether `BudgetPolicy` editing (excluded from A22 v1.0) is needed in the virtual-key plugin before A22 adds it (deferred per ADR 0039). Reconciliation: read-only for v1.0 (REQ-B5-09).
- **R-B5-3** (low): Drift between the inspector's view and live LiteLLM registry if the kopf reconcile (B13) lags; surface reconcile status, don't assert truth.

## 11. References

- architecture-overview.md §6.12 CRD inventory (capability CRDs, `Approval`; ~947–977).
- architecture-overview.md §11 Grafana dashboards / Headlamp deep-link inventory incl. Kargo per-Stage deep-link (~1550).
- architecture-overview.md §7.5 Approvals — generalized (workqueue / Headlamp plugin / resolved level / approve-reject resume; ~1294–1362).
- architecture-overview.md §6.6 virtual-key self-serve issuance (OPA-gated, no manual issuance; ~under 436), force-register/deregister (§6.9, ~762).
- architecture-overview.md §6.9 visibility model + admin override; required claims (~726–762).
- architecture-overview.md §9 Headlamp-plugins-as-unified-admin-UI; plugin gating DIY (~1399, ~1410, ~1415–1416).
- architecture-overview.md §14.1 A9 (framework, per-component plugins NOT in scope there), A22 (editor framework + initial editor set, exclusions), A23 (Kargo plugin via B5).
- _meta/waves.md — B19 → A23 → B5 cycle resolution (B19-core W3 / B19-ui W4 co-land with B5).
- ADR 0040 (Kargo plugin), ADR 0017 (approval system), ADR 0039 (Headlamp editors), ADR 0029 (claim schema), ADR 0034 (audit adapter), ADR 0021 (GrafanaDashboard XR), ADR 0006 (kopf reconciles; inspector reads).
- Interface-contract §1.4 (capability CRDs), §1.5 (`Approval`), §2 (`platform.approval.*`, `platform.capability.*`, `platform.audit.*`).
- Related pieces: A9, A22, A23 (upstream/framework); B19 (downstream / B19-core); B13, A20, A7.

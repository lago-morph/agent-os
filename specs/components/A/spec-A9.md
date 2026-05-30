# SPEC A9 — Headlamp install + framework

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [A21, A22, B1, B19] · adrs: [0008, 0039, 0030, 0034, 0031] · views: [6.12]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A9 installs **Headlamp** — the platform's primary Kubernetes admin/developer UI (architecture
overview §5, §10) — and ships the **cross-cutting framework code** that makes plugin development
easier: common libraries, shared widgets, and the auth handoff (§14.1 A9 row). It does **not**
ship per-component plugins or per-CRD editors; those are delivered by their owning Workstream A/B
components (e.g., A22 the editor framework, A21 the `TenantOnboarding` editor, B5 the cross-cutting
plugins, B19 the approval-queue UI).

Headlamp is the single editing surface the architecture leans on to compensate for missing
per-product admin UIs (§9): one place where edits flow through Git (PR-then-ArgoCD), not through
individual tool consoles. A9 is the foundation that every downstream plugin builds on — it provides
the shared reference pickers, diff viewer, simulator panel host, and the Keycloak/JWT auth handoff
that plugins reuse rather than each reimplementing. A9 is a W0 foundation component because A21,
A22, B1, and B19 all depend on the framework existing.

## 2. Scope

### 2.1 In scope
- Headlamp Helm install + values; branding for the platform environment.
- **Auth handoff:** Keycloak SSO into Headlamp; making the platform JWT and its claim schema
  (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`,
  `capability_set_refs`) available to plugins for UI scoping (§6.9, §6.11).
- **Shared framework libraries / widgets** that ADR 0039 names as framework-provided: reference
  pickers, the **diff-against-Git viewer**, and the **policy-simulator panel** host — built once in
  A9, consumed by every editor/plugin.
- The plugin-development scaffolding/conventions and the integration contract per-component plugins
  must satisfy (so a component "owns writing the plugin … and ensuring it integrates correctly with
  the Headlamp framework" — §14.1).
- The hard rule plumbing that editors **do not write to the cluster** — every submission produces a
  Git PR (ADR 0039); A9 provides the shared submit-to-Git mechanism.
- `LogLevel` honoring for Headlamp's own logs/traces (ADR 0035); audit emission of Headlamp plugin
  actions via the adapter (ADR 0034, §6.6 audit-emission points).
- Standard §14.1 Workstream-A deliverable set as applicable to a UI framework (see §8).

### 2.2 Out of scope (and where it lives instead)
- **Per-component plugins** — owned by each component (explicit §14.1 A9 exclusion).
- The **schema-driven graphical-editor framework** (form generation, validation hooks,
  simulator/diff integration as an editor layer) and the **initial editor set** — **A22** (ADR
  0039). A9 ships base install + shared widgets; A22 ships the editor-specific layer on top.
- **Cross-cutting plugins** (capability inspector, approval-queue UI, virtual-key admin, Kargo
  plugin) — **B5**.
- `TenantOnboarding` editor — **A21**; approval-queue UI — **B19-ui / B5**.
- The policy simulator **service** — **A20 (ADR 0038)**; A9 only hosts its panel.
- The OPA bundles / Git source of truth — OPA library (B3/B16) / ArgoCD.
- SSO proxy config in front of other UIs — **B1** (A9 provides Headlamp's own auth handoff).

## 3. Context & Dependencies

**Upstream consumed:** none (W0 foundation). Binds to Keycloak (baseline) for SSO and the
Kubernetes API for resource views.

**Downstream consumers:**
- **A22** — graphical-editor framework builds on A9's shared widgets and auth handoff.
- **A21** — `TenantOnboarding` editor builds on the framework.
- **B1** — SSO/auth proxy layer integrates with Headlamp's auth handoff.
- **B19** — approval-queue UI (B19-ui) is a plugin on the A9 framework.
- (Also A23/B5 Kargo plugin, and every per-component plugin, via the framework.)

**ADR decisions honored:**
- **ADR 0039** — Headlamp graphical editors for platform CRDs; A9 owns the **framework (A9)** /
  per-component-plugin split, and provides the shared widgets (reference pickers, diff viewer,
  simulator panel) the editors render with. PR-only submission (no cluster writes) is a hard rule
  A9's submit mechanism enforces.
- **ADR 0008** — Material for MkDocs is the docs portal; catalog-style views remain Headlamp +
  Crossplane's responsibility (Backstage explicitly not adopted). A9 is the catalog/navigation
  surface this implies.
- **ADR 0034** — Headlamp plugin actions (admin operations, incl. virtual-key issuance, budget
  edits) emit audit via the adapter (§6.6).
- **ADR 0031** — Headlamp-action audit events fall under `platform.audit.*`; per-event names B12.
- **ADR 0030** — A9 ships no platform CRD; framework JS/TS libraries are semver'd
  `[PROPOSED — not in source: framework lib versioning scheme is design-time]`.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- A9 defines **no platform CRD**. The framework **reads** every platform CRD/XRD (§6.12) to render
  views and provide reference pickers; it must render against the **served version's OpenAPI
  schema** (ADR 0039 schema-evolution note) and need not special-case conversion (webhooks keep
  older stored versions readable).
- A9 **hosts** (does not own) editors for: `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`,
  `Skill`, `CapabilitySet`, `Agent`, `TenantOnboarding`, `AuditLog`, `LogLevel`,
  `GrafanaDashboard` — these editors are A22-owned; A9 provides the widgets.

### 4.2 APIs / SDK surfaces
- **Headlamp plugin framework API** (A9-owned, `[PROPOSED — not in source: exact widget/library
  surface is design-time]`): shared reference-picker, diff-against-Git viewer, simulator-panel
  host, auth-context provider exposing platform JWT claims, and a submit-to-Git helper.
- **Auth handoff:** consumes the Keycloak-issued **Platform JWT** claim schema (glossary) for
  plugin-side UI scoping; RBAC remains the authoritative floor (ADR 0018).
- Reads Kubernetes API and (for capability views) LiteLLM current applied state (§6.8) — the
  *capability inspector* itself is B5, A9 provides the read plumbing.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.audit.*` for Headlamp plugin actions (via the audit adapter).
  `[PROPOSED — not in source: specific event names deferred to B12.]`
- **Consumed:** none required for core framework function.

### 4.4 Data schemas / connection-secret contracts
- None owned. A9 is stateless beyond Headlamp's own session handling; the source of truth for any
  edit is **Git** (ADR 0039), reconciled by ArgoCD — A9 writes PRs, not databases or the cluster.

## 5. OSS-vs-Custom Decision
- **Named project:** Headlamp (glossary: "Kubernetes UI + plugin framework (A9)").
- **Mode:** install + configure (Helm + branding) **plus build-new framework code** (shared
  TypeScript/React libraries + widgets + auth handoff) — §14.1 / §1 "all of them are code".
- **Version:** pin a tested Headlamp version `[PROPOSED — not in source]`.
- **ADR linkage:** ADR 0039 (editor framework split), ADR 0008 (Headlamp as the catalog surface
  since Backstage is not adopted).
- **Rationale:** Headlamp is the chosen admin/dev UI; OIDC is supported but plugin gating is DIY
  (§9), so the gating/auth-handoff and shared widgets are custom framework code owned here.

## 6. Functional Requirements
- **REQ-A9-01:** A9 SHALL install Headlamp via Helm with platform branding as a single
  ArgoCD-synced release.
- **REQ-A9-02:** A9 SHALL SSO Headlamp through Keycloak and expose the Platform JWT claim schema to
  plugins via an auth-context provider, with RBAC as the authoritative floor.
- **REQ-A9-03:** A9 SHALL provide shared framework widgets named by ADR 0039 — reference pickers,
  a diff-against-Git viewer, and a policy-simulator panel host — as reusable libraries.
- **REQ-A9-04:** A9 SHALL provide a submit-to-Git helper such that any plugin/editor submission
  produces a Git PR and **never** a direct cluster write (ADR 0039 hard rule).
- **REQ-A9-05:** A9 SHALL provide plugin-development scaffolding and the integration contract that
  per-component plugins must satisfy to load into the framework — **without** shipping any
  per-component plugin itself.
- **REQ-A9-06:** A9 SHALL render resource views against the **served** OpenAPI schema version of
  each CRD/XRD (ADR 0039 schema-evolution), requiring no editor migration beyond a schema rebuild.
- **REQ-A9-07:** A9 SHALL emit audit events for Headlamp plugin/admin actions via the platform
  audit adapter (ADR 0034) under `platform.audit.*`.
- **REQ-A9-08:** A9 SHALL honor a `LogLevel` CR for Headlamp's own log/trace granularity (ADR 0035).
- **REQ-A9-09:** A9 SHALL deliver a per-component Grafana dashboard (`GrafanaDashboard` XR) and
  alert rules for Headlamp availability.
- **REQ-A9-10:** A9 SHALL host the policy-simulator panel (service owned by A20) so editors can
  surface per-layer dry-run results inline, without A9 implementing simulator logic.

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** plugin UI scoping is claim-driven; RBAC floor authoritative
  (ADR 0018); no editor may bypass Git (ADR 0039). Plugin action gating logic lives in plugin code
  using Keycloak claims (§9 Headlamp gap row).
- **Observability (§6.5):** Headlamp emits OTel through the standard collector path; self-traces
  carry `trace_id`.
- **Scale:** framework supports many concurrently-loaded plugins without per-plugin auth
  reimplementation; concrete plugin-count target deferred.
- **Versioning (ADR 0030):** pin tested Headlamp version; framework libraries semver'd so editors
  rebuild against schema changes (ADR 0039).

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable**.
- Per-product docs (10.5) — **applicable** (+ maintainer/extender docs for the framework API, C7).
- Runbook (10.7) + backup/restore — **applicable** (A9 is largely stateless; runbook covers
  install/upgrade/SSO recovery; backup/restore N/A — no A9-owned state).
- Alerts — **applicable** (Headlamp availability).
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **N/A by design** — A9 ships the *framework*, not plugins (explicit §14.1
  exclusion). It ships shared widgets, not a component plugin.
- OPA/Rego integration — **applicable** (admission on Headlamp workloads; plugin-action gating uses
  OPA + claims, §6.6).
- Audit emission (ADR 0034) — **applicable** (plugin/admin actions).
- Knative trigger flow — **N/A** — the framework emits audit but has no event-routing flow of its
  own; downstream plugins design their flows. `[PROPOSED — minimal/none for the framework layer]`
- HolmesGPT toolset — **N/A** — a UI framework exposes no agent-callable tool.
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable** (heavy Playwright for the framework
  UI; PyTest for the submit-to-Git helper; Chainsaw for install).
- Tutorials & how-tos — **applicable** ("navigate the platform in Headlamp"; extender how-to "build
  a Headlamp plugin on the framework").

## 9. Acceptance Criteria
- **AC-A9-01 (REQ-A9-01):** ArgoCD sync brings Headlamp to Ready with platform branding as one
  release. *(Chainsaw)*
- **AC-A9-02 (REQ-A9-02):** Unauthenticated access redirects to Keycloak; after login a plugin can
  read the Platform JWT claims via the auth-context provider; an action denied by RBAC floor is
  blocked. *(Playwright + PyTest)*
- **AC-A9-03 (REQ-A9-03):** A test plugin renders a reference picker, a Git diff, and a simulator
  panel using only A9-provided widgets. *(Playwright)*
- **AC-A9-04 (REQ-A9-04):** A submission through the helper opens a Git PR and makes **no** cluster
  write (verified: no CR created/modified directly). *(PyTest + Chainsaw)*
- **AC-A9-05 (REQ-A9-05):** A sample per-component plugin built only against the published scaffolding
  loads and integrates; no per-component plugin ships inside A9. *(Playwright + repo-content check)*
- **AC-A9-06 (REQ-A9-06):** Given a CRD served at `v1alpha1` and later `v1`, the view renders
  against the served schema without code changes beyond a schema rebuild. *(PyTest)*
- **AC-A9-07 (REQ-A9-07):** A Headlamp admin action emits a `platform.audit.*` record via the
  adapter. *(PyTest)*
- **AC-A9-08 (REQ-A9-08):** A `LogLevel` CR raises/lowers Headlamp verbosity. *(Chainsaw + PyTest)*
- **AC-A9-09 (REQ-A9-09):** The `GrafanaDashboard` XR renders; a Headlamp-down condition fires the
  alert. *(Chainsaw + PyTest)*
- **AC-A9-10 (REQ-A9-10):** The simulator panel host renders A20's per-layer dry-run result inline
  (with A20 mocked until landed). *(Playwright)*

## 10. Risks & Open Questions
- **R1 (med):** The exact framework widget/library API surface is `[PROPOSED — not in source]`;
  over- or under-scoping it ripples into A22/B5/B19-ui/A21. *Open: lock the framework API contract
  early since four downstream pieces bind to it.*
- **R2 (med):** Overlap with A22 (editor framework) and B5 (cross-cutting plugins) is deliberate
  (§14.1 A22 note) — boundary must be drawn so A9 ships widgets and A22 ships the editor layer.
  Mis-drawn boundary = duplicated work. Blast radius med.
- **R3 (low):** Plugin gating is DIY per §9; gating logic placement (framework vs plugin) is a
  convention A9 sets — must be consistent or plugins drift.
- **R4 (low):** Simulator panel depends on A20 being available/accurate (ADR 0039 simulator-coupling
  consequence); degraded-simulator UX is a B5/A22 design-time decision, not A9's.

## 11. References
- architecture-overview.md §5 (software added to baseline, Headlamp row ~line 145); §9 (OSS-gap
  table, Headlamp/Crossplane rows ~1410–1416); §6.6 (audit/OPA hook points — Headlamp action
  gating, lines ~526, 539); §6.8 (capability Headlamp plugin reads, lines ~659, 716); §14.1 (A9 row,
  line ~1675; A22 row ~1688).
- ADR 0039 (Headlamp graphical editors — framework/plugin split, shared widgets, PR-only rule);
  ADR 0008 (Headlamp as catalog surface, Backstage not adopted); ADR 0034 (audit); ADR 0031
  (CloudEvents); ADR 0030 (versioning).
- View V6-12 (CRD inventory — what the framework renders).
- Related pieces: A22 (editor framework), A21 (`TenantOnboarding` editor), B5 (cross-cutting
  plugins), B19 (approval-queue UI), B1 (SSO), A20 (policy simulator service).

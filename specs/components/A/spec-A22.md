# SPEC A22 — Headlamp graphical-editor framework for platform CRDs

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [A9, A20] · downstream: [A21] · adrs: [0039, 0038, 0032, 0013, 0034, 0035, 0021, 0030, 0017] · views: [6.6, 6.9, 6.12, 6.13]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

The platform's primary configuration surface is its CRD inventory (§6.12). Authoring those CRDs as raw YAML is error-prone in three ways that hurt v1.0 operability (ADR 0039): cross-file overlay layering (a `CapabilitySet` references many other CRs; an `Agent` stacks `capabilitySetRefs[]` + `overrides` resolved by ADR 0032 replace-then-override); schema-heavy constrained fields (`BudgetPolicy`, `EgressTarget`, `Skill`, `TenantOnboarding`, `AuditLog`, `LogLevel`) whose validation only fires at admission; and destructive hand-edits (deleted `egressTargets[]` entries, blank-set `allowedMethods`, replace-instead-of-patch on lists) that ArgoCD applies before OPA can reject them.

A22 delivers the **cross-cutting framework and shared editor widgets** — schema-driven form generation, validation hooks, policy-simulator integration, and GitOps diff-against-Git preview — that per-component teams build their CRD editors on top of, plus the **initial editor set** mandated by ADR 0039. It is the editor-specific layer of the shared Headlamp framework: it sits above the A9 base install/branding/shared libraries and overlaps deliberately with B5 cross-cutting plugins. Every editor it ships integrates with the A20 policy simulator for preview-before-commit, and every editor submits **PR-only** against the GitOps manifest repo — editors never write to the cluster.

This is a T0 component: it is the authoring chokepoint for the platform's entire declarative surface and the foundation A21 (and per-component editor teams) depend on, so it is specified exhaustively.

## 2. Scope

### 2.1 In scope

- The **editor framework** layered on the A9 Headlamp framework: schema-driven form generation from a served CRD/XRD OpenAPI schema; shared widgets (reference pickers, enum dropdowns, FQDN/regex-validated inputs, list add/patch controls, diff viewer, simulator panel); validation hooks; GitOps diff-against-Git preview; PR submission flow.
- The **initial editor set per ADR 0039 / §14.1 A22**: `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `Agent` (and Agent templates), `TenantOnboarding`, `AuditLog`, `LogLevel`, `GrafanaDashboard`.
- **Overlay-aware editors** for `CapabilitySet`, `Agent`, and Agent templates: render the **effective resolved capability set** per ADR 0032 (unified per-agent view of MCP servers, A2A peers, RAG stores, egress targets, skills, OPA policy refs) so the author sees the post-merge result.
- **OPA-bundle policy edits** covered as a policy artifact alongside the platform CRDs: the same schema-aware validation, diff-against-Git, simulator preview, and PR-then-ArgoCD-apply discipline (§6.10, ADR 0039). OPA bundles are policy artifacts, not CRDs.
- **A20 policy-simulator integration**: inline per-layer decision trace (admission, runtime LiteLLM, egress, RBAC-floor / OPA-restrictor, approval elevation) shown alongside the diff before submission; degraded-simulator UX (banner + allow-submit-with-warning).
- **Reference pickers** scoped by Keycloak claims (`platform_namespaces`, `platform_tenants`) so multi-selects only offer in-scope resources.
- Audit emission of editor actions, Headlamp-plugin OPA gating, tests at all three layers, tutorials/how-tos, runbook, alerts, Grafana dashboard, HolmesGPT toolset contribution.

### 2.2 Out of scope (and where it lives instead)

- The **A21 `TenantOnboarding` editor logic specific to that CRD** — A21 owns its own CRD's editor; A22 ships only the framework plus the `TenantOnboarding` editor as part of the initial set. (§14.1 states A21 owns its editor and A22 owns only the cross-cutting framework / shared widgets. Reconciliation: A22 ships the initial-set `TenantOnboarding` editor *built on the framework*; the tenant-specific behaviour/validation is A21's. See §10 OQ-A22-1.)
- **Headlamp base install, branding, shared libraries, auth handoff** — component A9.
- **Cross-cutting non-editor plugins** (capability inspector, approval queue UI, virtual key admin UI, Kargo plugin) — component B5.
- **The policy simulator service itself** (aggregator + per-layer dry-run fan-out) — component A20 / ADR 0038. A22 only consumes its aggregator API.
- **The OPA bundle content / Rego library** — B3 framework, B16 initial content. A22 edits bundles as artifacts; it does not author policy.
- **CRD schema ownership / reconcilers** — each owning component (kopf B13 for capability CRDs, ARK A5 for `Agent`, Crossplane B4 for XRDs).
- **Excluded from the v1.0 editor set:** `BudgetPolicy` (deferred per ADR 0039), `Memory`, `XAgentDatabase` (not in ADR 0039's initial list), and read-only CRDs (`AgentRun`, `Sandbox`, `Approval`, `VirtualKey`) which ship read views only.
- **ArgoCD apply / reconcile** — the GitOps deployer; A22 stops at opening the PR.

## 3. Context & Dependencies

**Upstream consumed:**

- **A9 (Headlamp install + framework)** — the base Headlamp install, branding, shared libraries, and **auth handoff** (Keycloak claims into the plugin). A22 builds its editor framework on top; A9 provides shared widgets and the plugin-loading mechanism (§14.1 A9). Plugin gating logic against Keycloak claims is DIY in plugin code (§9 Headlamp row).
- **A20 (Policy simulator service)** — the aggregator API that, given a synthetic request (subject = Keycloak claims, action, resource, context), returns the per-layer decision trace (ADR 0038). A22 editors call this same aggregator for preview-before-commit; Headlamp and HolmesGPT consume the same API (ADR 0038).

**Downstream consumers:**

- **A21 (Tenant onboarding reconciler)** — builds its `TenantOnboarding` editor on the A22 framework / shared widgets.
- Per-component editor teams (any Workstream A/B component shipping a CRD editor) consume the framework.

**ADR decisions honored:**

- **ADR 0039** — schema-driven editors; diff-against-Git preview reading desired state from Git (not cluster); inline simulator; effective-resolved-set for overlay editors; **PR-only submission (hard rule)**; editors render against the *served version's* OpenAPI schema; the initial editor set.
- **ADR 0038** — simulator is one aggregator API with two consumers (Headlamp + HolmesGPT); degraded-simulator UX (banner + allow-submit-with-warning) is a B5/this-component design-time decision; simulator runs are audited under `platform.policy.*`, never enter enforcement.
- **ADR 0032** — overlay editors must show the replace-then-override resolved effective set.
- **ADR 0013** — the capability CRD shapes (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`) the core editors render.
- **ADR 0034 / 0035 / 0021** — the `AuditLog`, `LogLevel`, `GrafanaDashboard` editor targets.
- **ADR 0030** — editors render against the served version's OpenAPI schema; conversion webhooks keep older stored versions readable; no special editor migration beyond rebuilding against the new schema (ADR 0039 consequence).
- **ADR 0017** — for approval-bearing changes the PR review (and any approval) is the second gate after the diff preview.
- **§6.9** — reference pickers and submission are claim-scoped (`platform_namespaces`, `platform_tenants`, `platform_roles`).

## 4. Interfaces & Contracts

Names below are Canon (§6.12 inventory, interface-contract §1) unless tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A22 **defines no CRD of its own.** It renders editors for existing Canon CRDs/XRDs against their served OpenAPI schema. The targets and their Canon key fields (interface-contract §1, §6.12):

| Target | Owner / reconciler | Canon key fields rendered |
|---|---|---|
| `MCPServer` | kopf (B13) | `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility` |
| `A2APeer` | kopf (B13) | `endpoint`, `direction` (internal/external), `auth`, `tags` |
| `RAGStore` | kopf (B13) | `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef` |
| `EgressTarget` | kopf (B13) | `fqdn`, `port`, `scheme`, `allowedMethods` |
| `Skill` | kopf (B13) | `gitRef`, `versionPin`, `schemaRef` |
| `CapabilitySet` | kopf (B13) | `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]` (overlay-aware) |
| `Agent` + templates | ARK (A5) | `capabilitySetRefs[]`, `overrides`, `sdk` (`langgraph`/`deep-agents`), `image`, `sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes` (overlay-aware) |
| `TenantOnboarding` (XRD) | Crossplane (B4) | `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` (A21 owns CRD-specific behaviour) |
| `AuditLog` (XRD) | Crossplane (B4) | `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec`, `endpointReplicas` |
| `LogLevel` | per-component | `componentSelector`, `level`, `traceGranularity`, `scope` (component/tenant/eventClass), `expiresAt` |
| `GrafanaDashboard` (XR) | Crossplane (B4) | `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled) |

> Field sets beyond those above are NOT invented here; the editor renders whatever the served OpenAPI schema declares (ADR 0039). Where ADR 0032 overlay-resolution detail is needed it lives in ADR 0032 / B13 design.

OPA-bundle edits: OPA bundles are **policy artifacts, not CRDs** (§6.10); the editor treats a bundle as a Git-tracked artifact with schema-aware validation, not a Kubernetes resource.

### 4.2 APIs / SDK surfaces

- **Policy-simulator aggregator API (consumed)** — A20 / ADR 0038. Input: synthetic request `{subject (Keycloak claims), action, resource, context}`; output: per-layer decision trace with matching rule cited. Exact request/response schema is owned by A20; `[PROPOSED — not in source]` for any field name beyond those ADR 0038 states (subject, action, resource, context; layers: admission, runtime, egress, RBAC, approval elevation).
- **Git/PR submission surface (consumed)** — opens a PR against the appropriate manifest repo (GitHub per ADR 0033). Reads current desired state from the ArgoCD Git source of truth for the diff. The specific Git host API is GitHub (ADR 0033). `[PROPOSED — not in source]` for any internal PR-helper service shape.
- **Editor framework SDK (provided to per-component teams)** — the TypeScript/React widget + form-generation library per-component editors bind to. ADR 0039 names "shared widgets (reference pickers, diff viewer, simulator panel)" as A9-provided and the editor layer as A22's. Specific method signatures are **not specified in source** — `[PROPOSED — not in source]` for any concrete API; surface follows §3.1/§3.3 versioning where it becomes an exposed API.
- This is a Headlamp plugin (TypeScript/React), not a service with an HTTP API of its own; URL-path versioning (§6.13) applies only if it exposes an HTTP surface (it does not in v1.0).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Per-event-type names within each namespace are deferred to B12's registry (interface-contract §2); A22 commits only to namespaces:

- **Emitted:** editor actions that submit a PR or run a simulation emit under `platform.audit.*` (audit emission of the authoring action, ADR 0034) and, for simulator invocations, `platform.policy.*` (simulator runs audited under the policy taxonomy, ADR 0038). `[PROPOSED — not in source]` for any concrete event-type name.
- **Consumed:** none required for core function. (The editor is a UI; it reads Git + simulator API synchronously, not via the event mesh.)

### 4.4 Data schemas / connection-secret contracts

- A22 provisions no substrate primitive and writes no connection secret (interface-contract §4 N/A — it is a UI plugin).
- It reads the **served OpenAPI schema** of each target CRD/XRD (Kubernetes API discovery) and the **current desired-state manifest** from Git.
- Keycloak claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`) are consumed for scoping (§6.9) via the A9 auth handoff.

## 5. OSS-vs-Custom Decision

- **Base:** Headlamp (A9 install) — OSS Kubernetes UI + plugin framework (glossary; §14.1 A9).
- **Decision:** **build-new** plugin code on top of Headlamp's plugin SDK. Per §9, "Headlamp — OIDC supported but plugin gating is DIY"; the schema-driven editors, overlay-resolution view, diff-against-Git, and simulator panel are custom TypeScript/React (ADR 0039 consequence: "Each editor is a TypeScript/React component bound to a CRD schema").
- **Rationale:** No OSS plugin provides schema-driven graphical editors with effective-overlay resolution, diff-against-Git, and inline policy-simulator integration for these platform CRDs; ADR 0039 mandates them. The A9 framework provides shared widgets; A22 is the editor layer; B5 owns non-editor cross-cutting plugins.
- **Version pinning:** pin a tested Headlamp version (consistent with the §9 "pin a tested version" discipline used for ARK).

## 6. Functional Requirements

- **REQ-A22-01:** The framework SHALL generate an editor form from a target CRD/XRD's served OpenAPI schema (ADR 0030 served version), rendering enum fields as dropdowns, regex/FQDN-constrained fields with inline validation, and reference fields as reference pickers.
- **REQ-A22-02:** Reference pickers SHALL be multi-selects over existing in-scope resources of the referenced kind (e.g. `Agent.capabilitySetRefs[]` over existing `CapabilitySet` resources), scoped by the author's `platform_namespaces` / `platform_tenants` claims (§6.9, ADR 0039).
- **REQ-A22-03:** Before submission, the editor SHALL render a **diff against current Git** — reading desired state from the ArgoCD Git source of truth (NOT the cluster) — as a YAML diff of the proposed change (ADR 0039).
- **REQ-A22-04:** List-valued fields SHALL be edited via add/remove/patch controls that prevent accidental whole-list replacement and surface removals explicitly in the diff (ADR 0039 destructive-edit mitigation).
- **REQ-A22-05:** Inline validation SHALL fire against the schema **before** submission (not only at admission), blocking PR creation on hard schema violations (ADR 0039).
- **REQ-A22-06:** The editor SHALL surface the A20 policy simulator inline: given the proposed change it SHALL display the per-layer decision trace (admission, runtime LiteLLM, egress, RBAC-floor / OPA-restrictor, approval elevation) alongside the diff, with simulator failures shown next to the diff (ADR 0039, ADR 0038).
- **REQ-A22-07:** For overlay-aware editors (`CapabilitySet`, `Agent`, Agent templates) the editor SHALL compute and display the **effective resolved capability set** per ADR 0032 replace-then-override semantics — the unified view of MCP servers, A2A peers, RAG stores, egress targets, skills, and OPA policy refs after the proposed change is merged.
- **REQ-A22-08:** Every submission SHALL produce a **Git PR** against the appropriate manifest repository; the editor SHALL NOT write to the cluster under any path (ADR 0039 hard rule).
- **REQ-A22-09:** The framework SHALL ship the initial editor set: `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `Agent` (+ templates), `TenantOnboarding`, `AuditLog`, `LogLevel`, `GrafanaDashboard` (§14.1 A22, ADR 0039).
- **REQ-A22-10:** The framework SHALL provide an **OPA-bundle editor** that treats a bundle as a Git-tracked policy artifact with schema-aware validation, diff-against-Git, simulator preview, and PR-then-ArgoCD submission — sharing the same authoring discipline as the CRD editors (§6.10, ADR 0039).
- **REQ-A22-11:** When the simulator is unavailable/degraded, the editor SHALL show a banner and allow submit-with-warning rather than hard-blocking (ADR 0038/0039 degraded-simulator UX).
- **REQ-A22-12:** Editors SHALL render against the **served** API version's OpenAPI schema; on a CRD version bump (ADR 0030) the editor requires only a rebuild against the new schema (no special migration) (ADR 0039).
- **REQ-A22-13:** The framework SHALL expose a reusable widget/form-generation library so per-component editor teams (e.g. A21) build their CRD editor on it rather than re-implementing form generation, validation, diff, or simulator panels.
- **REQ-A22-14:** Plugin actions SHALL be gated by the author's Keycloak claims (`platform_roles`, `tenant_roles`, `platform_namespaces`) in plugin code (§9 Headlamp gating is DIY).
- **REQ-A22-15:** Every editor authoring action (PR open, simulation run) SHALL emit a structured audit event through the platform audit adapter (ADR 0034) — simulator runs under `platform.policy.*`, PR-submission actions under `platform.audit.*`.
- **REQ-A22-16:** The `Agent` editor SHALL constrain the `sdk` field to the v1.0 allowed values `langgraph` and `deep-agents` (ADR 0019).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** Reference pickers, editor visibility, and PR-target selection are claim-scoped; an author can only edit/reference resources within their `platform_namespaces`/`platform_tenants`. Plugin gating is enforced in plugin code against Keycloak claims (§9). The editor never bypasses Git → no cluster-write path exists (ADR 0039), preserving GitOps as the single source of truth.
- **Observability (§6.5):** Authoring actions and simulator runs emit audit + are dashboarded; a `LogLevel` toggle (ADR 0035) can raise editor verbosity per-tenant. HolmesGPT toolset exposes editor/authoring telemetry.
- **Policy correctness:** Simulator preview is "what would happen for this case" only — not formal verification (ADR 0038); the editor must not present it as a guarantee.
- **Scale:** Overlay resolution and Git-diff rendering must remain responsive for `CapabilitySet`/`Agent` graphs of realistic depth; degraded simulator must not block authoring (REQ-A22-11).
- **Versioning (ADR 0030):** Editors track served CRD versions; the framework widget library, if exposed as an API to per-component teams, follows semantic versioning (§6.13) — `[PROPOSED — not in source]` for the concrete version surface.
- **Availability coupling:** Editor usefulness depends on simulator availability and accuracy (ADR 0039 simulator-coupling consequence); the degraded-UX path is the mitigation.

## 8. Cross-Cutting Deliverable Checklist

Per §14.1 standard set:

- **Helm values / manifests in Git** — applicable: the plugin's packaging/registration manifests and Headlamp plugin config.
- **Per-product architecture docs (10.5)** — applicable: editor framework reference + how-to-write-a-CRD-editor (extends §10.2 "How to write a Headlamp plugin").
- **Per-product operator runbook (10.7)** — applicable: editor degraded-simulator handling, schema-version-bump rebuild procedure, PR-flow troubleshooting.
- **Backup / restore** — N/A — the plugin is stateless; desired state lives in Git (source of truth) and the cluster.
- **Alert rules** — applicable: plugin load failure, simulator-unreachable rate, PR-submission failure rate.
- **Grafana dashboard (Crossplane `GrafanaDashboard` XR)** — applicable: authoring-action volume, simulator latency/error, PR success rate.
- **Headlamp plugin** — applicable: this component **is** Headlamp plugin work (the editor framework + initial editor set).
- **OPA / Rego integration** — applicable: plugin-action gating + the OPA-bundle editing surface; Rego for who-may-edit-which-CRD contributed to the policy library (B3/B16).
- **Audit emission (ADR 0034)** — applicable: PR-open + simulation-run events (REQ-A22-15).
- **Knative trigger flow** — N/A — the editor is a synchronous UI; it does not consume/produce event-mesh triggers in v1.0 (audit emission is via the adapter, not a trigger flow).
- **HolmesGPT toolset** — applicable: expose authoring telemetry / "what editors exist + recent authoring activity" tool; the simulator skill itself is A20's (ADR 0038).
- **3-layer tests (Chainsaw / Playwright / PyTest)** — applicable: Playwright for editor e2e (form, diff, simulator panel, PR submission), PyTest for any helper logic, Chainsaw where editor output is applied to a CRD to verify admission.
- **Tutorials & how-tos** — applicable: "How to write a Headlamp plugin" (§10.2) extended to "author a CRD via the graphical editor"; tutorial coverage for the initial editor set.

## 9. Acceptance Criteria

- **AC-A22-01** (REQ-A22-01): Opening an editor for a CRD with an enum field renders a dropdown over the schema's enum values and rejects out-of-enum input before submission.
- **AC-A22-02** (REQ-A22-02): The `Agent` editor's `capabilitySetRefs[]` picker lists exactly the `CapabilitySet` resources in the author's in-scope namespaces and no others.
- **AC-A22-03** (REQ-A22-03): Editing a field and opening the preview shows a YAML diff computed against the current Git desired state, not the live cluster object (verified by a Git/cluster divergence fixture).
- **AC-A22-04** (REQ-A22-04): Removing one entry from `EgressTarget.allowedMethods` (or an `egressTargets[]` list) surfaces a removal in the diff and does not silently replace the whole list.
- **AC-A22-05** (REQ-A22-05): A change violating the schema (e.g. invalid FQDN in `EgressTarget.fqdn`) blocks PR creation with an inline error before submission.
- **AC-A22-06** (REQ-A22-06): A proposed change displays the per-layer simulator trace (admission/runtime/egress/RBAC/approval) alongside the diff; a simulated denial is shown next to the diff.
- **AC-A22-07** (REQ-A22-07): Editing one `CapabilitySet` overlay shows the post-merge effective resolved set (replace-then-override per ADR 0032), differing from the single edited layer when overlays stack.
- **AC-A22-08** (REQ-A22-08): Submitting any editor opens a Git PR and performs zero cluster writes (verified by API-server audit showing no write from the plugin identity).
- **AC-A22-09** (REQ-A22-09): All eleven initial-set editors are present and loadable in Headlamp.
- **AC-A22-10** (REQ-A22-10): The OPA-bundle editor produces a diff-against-Git + simulator preview and submits a PR; it does not write the bundle directly to the bundle store.
- **AC-A22-11** (REQ-A22-11): With the simulator API forced unavailable, the editor shows a degraded banner and still permits submit-with-warning.
- **AC-A22-12** (REQ-A22-12): Rebuilding the plugin against a bumped CRD version (`v1alpha1`→`v1`) renders the new schema with no editor code change beyond the rebuild.
- **AC-A22-13** (REQ-A22-13): A22's widget library is consumed by the A21 `TenantOnboarding` editor build (cross-component integration test) without A21 re-implementing form/diff/simulator code.
- **AC-A22-14** (REQ-A22-14): A viewer-role principal cannot open a write editor; gating is enforced from Keycloak claims.
- **AC-A22-15** (REQ-A22-15): Opening a PR emits a `platform.audit.*` event and running a simulation emits a `platform.policy.*` event, both via the audit adapter.
- **AC-A22-16** (REQ-A22-16): The `Agent` editor `sdk` dropdown offers only `langgraph` and `deep-agents`.

## 10. Risks & Open Questions

- **OQ-A22-1** (blast-radius: med): §14.1 says A21 owns its `TenantOnboarding` editor while A22 owns "only the cross-cutting framework / shared widgets," yet ADR 0039's initial set (which A22 ships) lists `TenantOnboarding`. **Reconciliation note:** A22 ships the framework + a baseline `TenantOnboarding` editor built on it; A21 owns the CRD-specific validation/behaviour and may extend/replace the form. Confirm split with A21 owner. `[PROPOSED]` boundary.
- **OQ-A22-2** (low): The framework widget library's concrete API/method signatures are **not specified in source** (interface-contract §3 names no such surface). Tagged `[PROPOSED — not in source]`; to be designed with A9 since A9 provides shared widgets.
- **OQ-A22-3** (med): The policy-simulator aggregator request/response schema is owned by A20/ADR 0038 and not field-enumerated in source; A22 must pin to A20's contract. Mock A20 until it lands (§10 mock-out commitment). `[PROPOSED — not in source]` for any field names beyond subject/action/resource/context + the five layers.
- **OQ-A22-4** (low): ADR 0032 overlay-resolution *semantics* are deferred to ADR 0032 + design specs; A22's effective-set view must consume B13/ADR 0032's resolver output rather than re-implement resolution. Risk if the resolver is not exposed as a reusable surface.
- **OQ-A22-5** (low): Whether OPA-bundle editing shares the manifest repo or a dedicated bundle store affects the diff-against-Git source; §6.10 says bundles are Git-tracked artifacts but does not fix the repo. `[PROPOSED — not in source]`.

## 11. References

- architecture-overview.md §6.6 (security/policy, editing-surface, lines 436–460); §6.9 (multi-tenancy + claims, lines 720–772); §6.10 (HolmesGPT + Headlamp policy editor + simulator, lines 821–835); §6.12 (CRD inventory, lines 942–979); §6.13 (versioning, lines 981–1001); §9 (OSS limitations — Headlamp gating DIY, lines 1390–1423); §14.1 (Workstream A deliverables + A9/A20/A21/A22 rows, lines 1647–1689).
- ADR 0039 (Headlamp graphical editors — schema-driven, diff-against-Git, inline simulator, overlay effective set, PR-only, initial editor set).
- ADR 0038 (policy simulators — aggregator API, two consumers, degraded UX, audited under `platform.policy.*`).
- ADR 0032 (CapabilitySet overlay semantics — replace-then-override).
- ADR 0013 (capability CRD model). ADR 0034 (audit pipeline / `AuditLog`). ADR 0035 (`LogLevel`). ADR 0021 (`GrafanaDashboard` XR). ADR 0030 (CRD/API versioning). ADR 0017 (approval as second gate). ADR 0019 (`Agent.sdk` values). ADR 0033 (GitHub target).
- Related pieces: A9 (Headlamp framework), A20 (policy simulator), A21 (TenantOnboarding editor), B5 (cross-cutting plugins), B13 (capability CRD reconciler), B4 (XRDs).
- _meta/interface-contract.md §1 (CRD/XRD fields), §2 (CloudEvent taxonomy), §3 (SDK surfaces), §4 (connection-secret — N/A here).

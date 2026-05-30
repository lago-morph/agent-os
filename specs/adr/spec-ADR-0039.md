# SPEC ADR-0039 — Headlamp graphical editors for platform CRDs `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [A9; A20; B5] · downstream: [A22; A21; A23; B5] · adrs: [0039] · views: [6.12]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0039 is a settled decision: the platform ships **schema-driven graphical editors in Headlamp** for
the platform CRDs most prone to authoring error. Each editor renders fields from the CRD's OpenAPI
schema with inline validation and reference pickers, shows a **diff against current Git**, surfaces the
**OPA simulator (ADR 0038) inline**, and for overlay-aware editors shows the **effective resolved
capability set (ADR 0032)**. **Editors do not write to the cluster** — every submission produces a Git
PR. This SPEC states what honoring the decision requires of each editor — it does not re-argue the
graphical-editor approach.

The problem the decision solves: authoring CRDs in raw YAML is error-prone in three specific ways —
cross-file layering (a user editing one overlay can't see the resolved effective set), schema-heavy
constrained fields (validation only fires at admission), and destructive edits (list replacements /
blanked fields land before ArgoCD/OPA can reject). Graphical editors with a Git diff and inline
simulation move those errors before the PR exists.

## 2. Scope

### 2.1 In scope
- Schema-driven editors in Headlamp for the initial editor set, rendering from each CRD's OpenAPI schema with inline validation, enum dropdowns, and reference pickers.
- The "diff against current Git" preview (read desired state from Git, the ArgoCD source of truth — not from the cluster).
- Inline OPA simulator (ADR 0038) preview across all enforcement layers before submission.
- Overlay-aware editors (`CapabilitySet`, `Agent`, Agent templates) showing the effective resolved set per ADR 0032.
- The hard rule: PR-only submission; editors never write to the cluster.

### 2.2 Out of scope (and where it lives instead)
- The simulator itself — owned by ADR 0038 / component A20; degraded-simulator UX (banner + allow-submit-with-warning) is a B5 design-time decision, not this ADR.
- Overlay resolution semantics — owned by ADR 0032.
- The Headlamp install + plugin framework / shared widgets — owned by component A9.
- The CRD schemas the editors render — owned by each CRD's owning component (ADR 0030 versioning).
- The editor component builds — owned by Workstream B5 (cross-cutting + overlay-aware editors) and per-component teams (CRD-specific editors); the framework-level editor surface is A22.

## 3. Context & Dependencies

Upstream consumed: A9 (Headlamp install + framework — provides shared widgets: reference pickers, diff
viewer, simulator panel); A20 (policy simulator aggregator — surfaced inline); B5 (cross-cutting
Headlamp plugin work — owns the cross-cutting + overlay-aware editors). Downstream consumers: A22
(Headlamp graphical-editor framework for platform CRDs), A21 (tenant onboarding uses the
`TenantOnboarding` editor), A23 (Kargo authoring flows upstream of promotion), B5 (the editors).

ADR decisions honored:
- **ADR 0039** (this): schema-driven editors; diff-against-Git; inline simulator; effective-resolved-set for overlays; PR-only submission.
- **ADR 0032**: overlay-aware editors render the effective resolved capability set (replace-then-override).
- **ADR 0038**: the OPA simulator is surfaced inline; simulator failures shown alongside the diff.
- **ADR 0013**: the core LiteLLM-side capability CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`) are in the initial editor set.
- **ADR 0037 / 0034 / 0035 / 0021**: `TenantOnboarding`, `AuditLog` XRD config, `LogLevel`, `GrafanaDashboard` XR editors are in the initial set.
- **ADR 0017**: PR review (and any approval) is the second gate after the diff-against-Git preview.
- **ADR 0030**: editors render against the served version's OpenAPI schema; conversion webhooks keep older stored versions readable; no special editor migration beyond rebuilding against the new schema.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
This ADR imposes no new CRD; it consumes the OpenAPI schemas of existing CRDs/XRDs. Initial editor set:
- `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill` — core LiteLLM-side capability CRDs (ADR 0013).
- `CapabilitySet`, `Agent`, and Agent templates — overlay-aware, showing the effective resolved set (ADR 0032).
- `TenantOnboarding` (ADR 0037). `AuditLog` XRD config (ADR 0034). `LogLevel` (ADR 0035). `GrafanaDashboard` XR (ADR 0021).
Other CRDs (`AgentRun`, `Sandbox`, `Approval`, `VirtualKey`, `BudgetPolicy`) may receive editors later;
v1.0 ships read views + the existing capability inspector + the editors above.

### 4.2 APIs / SDK surfaces
- Each editor is a TypeScript/React component bound to a CRD schema. The A9 framework provides shared
  widgets (reference pickers, diff viewer, simulator panel). The editor reads desired state from Git
  (ArgoCD source of truth), renders a YAML diff, and on submission opens a Git PR against the
  appropriate manifest repository. It calls the ADR 0038 aggregator API for the inline simulator.
- Concrete widget/editor method signatures are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — editors submit Git PRs, not cluster writes; they emit no CloudEvent of their own. Downstream
admission/policy events on the eventual ArgoCD apply fall under `platform.policy.*` / `platform.capability.*`
per the owning components. Inline-simulator runs are audited under `platform.policy.*` per ADR 0038.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface. The editor's inputs are CRD OpenAPI schemas + current Git desired
state; its output is a Git PR (a YAML diff).

## 5. OSS-vs-Custom Decision
N/A — ADR. The editors are custom TypeScript/React plugins on the OSS Headlamp framework (A9); the
shared widgets are framework-provided. Those OSS-vs-custom calls live in the A9 / A22 / B5 SPECs.

## 6. Functional Requirements

- REQ-ADR-0039-01: Each editor MUST render fields from the CRD's OpenAPI schema with inline validation, dropdowns over enum values, and reference pickers (e.g. `Agent.capabilitySetRefs[]` as a multi-select over existing `CapabilitySet` resources in scope).
- REQ-ADR-0039-02: Each editor MUST show a "diff against current Git" preview before submission, reading desired state from Git (the ArgoCD source of truth) — NOT from the cluster.
- REQ-ADR-0039-03: Each editor MUST surface the OPA simulator (ADR 0038) inline, previewing the change's effect at every enforcement layer (admission, runtime LiteLLM, egress, RBAC-floor / OPA-restrictor) against the current policy bundle; simulator failures MUST be shown alongside the diff.
- REQ-ADR-0039-04: Overlay-aware editors (`CapabilitySet`, `Agent`, Agent templates) MUST show the effective resolved capability set per ADR 0032 — the unified per-agent view — so the author sees the post-merge result, not just the layer being edited.
- REQ-ADR-0039-05: Editors MUST NOT write to the cluster; every submission MUST produce a Git PR against the appropriate manifest repository, reconciled by ArgoCD. The diff-against-Git preview is the first gate; PR review (and any ADR 0017 approval) is the second.
- REQ-ADR-0039-06: The initial editor set MUST cover `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `Agent` + Agent templates, `TenantOnboarding`, `AuditLog` XRD config, `LogLevel`, and `GrafanaDashboard` XR.
- REQ-ADR-0039-07: Editors MUST render against the served version's OpenAPI schema (ADR 0030); conversion webhooks keep older stored versions readable; no special editor migration is required beyond rebuilding against the new schema.

## 7. Non-Functional Requirements
- Safety: the diff-against-Git preview catches list replacements, accidental field deletions, and overlay surprises before a PR exists (lower destructive-edit risk).
- Consistency: editing flows through Headlamp, not per-product admin UIs (reinforces §6.6).
- GitOps invariant: Git is the single source of desired state; PR-only submission is a hard rule (REQ-05).
- Simulator coupling: the simulator must remain available and accurate for editors to be useful; degraded-simulator UX is a B5 design-time decision.
- Versioning (ADR 0030): editors track the served schema version; no migration beyond a rebuild.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The editors are §14.4 / Workstream D + B5 deliverables; A9 provides the framework, B5 owns
the cross-cutting + overlay-aware editors, per-component teams own CRD-specific editors. Conformance to
this decision is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0039-01: Honored when an editor renders a CRD from its OpenAPI schema with enum dropdowns, inline validation, and a reference picker (e.g. `capabilitySetRefs[]` multi-select). (→ REQ-01)
- AC-ADR-0039-02: Honored when an edit shows a YAML diff computed against current Git desired state, not the live cluster. (→ REQ-02)
- AC-ADR-0039-03: Honored when an edit shows inline simulator results across all enforcement layers, and a simulator failure is rendered alongside the diff. (→ REQ-03)
- AC-ADR-0039-04: Honored when editing a `CapabilitySet` / `Agent` overlay shows the post-merge effective resolved set per ADR 0032, not just the edited layer. (→ REQ-04)
- AC-ADR-0039-05: Honored when submitting any editor change produces a Git PR and performs no direct cluster write. (→ REQ-05)
- AC-ADR-0039-06: Honored when all initial-set CRDs/XRDs have a working editor and the non-set CRDs show read views + the capability inspector. (→ REQ-06)
- AC-ADR-0039-07: Honored when an editor rebuilt against a new served schema version renders the new fields with older stored versions still readable. (→ REQ-07)

## 10. Risks & Open Questions
- (med) Plugin work is non-trivial — each editor is a schema-bound React component; the overlay-aware Agent / CapabilitySet editor is the heaviest. Effort is a real B5 budget item.
- (med) Simulator coupling: an unavailable or inaccurate simulator degrades editor value; the degraded-simulator UX (banner + allow-submit-with-warning) is a B5 design decision, not fixed here.
- (low) `[PROPOSED]` — widget/editor method signatures are not specified in source; flagged for A9 / B5 design.

## 11. References
- ADR 0039 (this decision). Enforcing/realizing components: A22 (graphical-editor framework surface), A9 (Headlamp framework + shared widgets), B5 (cross-cutting + overlay-aware editors), A20 (inline simulator), A21 (`TenantOnboarding` editor), A23 (Kargo authoring upstream).
- architecture-overview.md §6.12 (CRD inventory), §6.6, §10, §14.4 (Workstream D), Workstream B5. ADR 0013, 0017, 0021, 0030, 0032, 0034, 0035, 0037, 0038.

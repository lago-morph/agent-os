# ADR 0039: Headlamp graphical editors for platform CRDs

- Status: Accepted
- Date: 2026-05-10

## Context

The platform exposes its primary configuration surface through CRDs (architecture overview §6.12). Authoring those CRDs in raw YAML is error-prone in three specific ways that hurt v1.0 operability:

- **Cross-file layering.** A `CapabilitySet` references many other CRs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`), and an `Agent` may stack multiple `capabilitySetRefs[]` plus `overrides`. Per ADR 0032, the effective set is computed by replace-then-override resolution across these layers. A user editing one overlay file in isolation cannot easily see what the resolved effective set will be after their change is merged.
- **Schema-heavy fields.** `BudgetPolicy`, `EgressTarget`, `Skill`, `TenantOnboarding` (ADR 0037), the `AuditLog` XRD config (ADR 0034), and `LogLevel` (ADR 0035) all carry structured fields with constrained value domains (enums, regex-validated FQDNs, scope grammars). Schema validation only fires at admission; raw-YAML authors see errors late.
- **Destructive edits.** Hand-edited YAML in operator hands has, repeatedly in the industry, deleted `egressTargets[]` entries, blank-set `allowedMethods`, or replaced rather than patched a list. By the time ArgoCD applies and OPA admits or rejects, the PR may already be merged.

Headlamp is already the platform's primary admin/developer UI (overview §5, §10, §14.4 / Workstream D), with the framework owned by component A9 and per-component plugins owned by their respective components. Workstream B5 is explicitly responsible for cross-cutting Headlamp plugins. We need to formalize what those editors must do.

## Decision

The platform ships **schema-driven graphical editors in Headlamp** for the platform CRDs that are most prone to authoring error. Each editor:

- Renders fields from the CRD's OpenAPI schema, with inline validation, dropdowns over enum values, and reference pickers (e.g., when editing an `Agent`, the `capabilitySetRefs[]` field is a multi-select over existing `CapabilitySet` resources in scope).
- Shows a **"diff against current Git"** preview before submission. The editor reads the current desired state from Git (the ArgoCD source of truth), not from the cluster, and renders a YAML diff of the proposed change.
- Surfaces the **OPA simulator (ADR 0038) inline**. The author can preview, before submitting, what the change does at every enforcement layer (admission, runtime LiteLLM, egress, RBAC-floor / OPA-restrictor) given the current policy bundle. Simulator failures are shown alongside the diff.
- For overlay-aware editors (`CapabilitySet`, `Agent`, Agent templates), shows the **effective resolved capability set** per ADR 0032 — the unified per-agent view of MCP servers, A2A peers, RAG stores, egress targets, skills, and OPA policy refs — so the author sees the post-merge result, not just the layer they are editing.

**Editors do not write to the cluster.** Every submission produces a Git PR against the appropriate manifest repository. ArgoCD reconciles the merged change. The "diff against Git" preview is the gate before submission; the PR review (and any approval per ADR 0017) is the second gate.

### Initial editor set (Workstream B Headlamp plugin work)

- `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill` — the core LiteLLM-side capability CRDs (ADR 0013).
- `CapabilitySet`, `Agent`, and Agent templates — overlay-aware editor showing the effective resolved set per ADR 0032.
- `TenantOnboarding` — per ADR 0037.
- `AuditLog` XRD config — per ADR 0034.
- `LogLevel` — per ADR 0035.
- `GrafanaDashboard` XR — per ADR 0021 (already implied by Workstream D).

Other CRDs (`AgentRun`, `Sandbox`, `Approval`, `VirtualKey`, `BudgetPolicy`, etc.) may receive editors later; v1.0 ships read views and the existing capability inspector, plus the editors above.

## Consequences

- **Lower destructive-edit risk.** The diff-against-Git preview catches list replacements, accidental field deletions, and overlay surprises before a PR exists.
- **One editing surface across components.** This reinforces the "edits flow through Headlamp, not per-product admin UIs" property already stated in overview §6.6.
- **Plugin work is non-trivial.** Each editor is a TypeScript/React component bound to a CRD schema. The framework (A9) provides shared widgets (reference pickers, diff viewer, simulator panel); Workstream B5 owns the cross-cutting editors and the overlay-aware Agent / CapabilitySet editor; per-component teams own editors specific to their CRD.
- **GitOps remains the source of truth.** Editors that bypass Git would split the desired-state model. This ADR forecloses that: PR-only submission is a hard rule.
- **Simulator coupling.** The simulator must remain available and accurate for editors to be useful. ADR 0038 owns the simulator; degraded-simulator UX (banner + allow-submit-with-warning) is a design-time decision in B5, not in this ADR.
- **Schema evolution.** Per ADR 0030, CRDs may move through `v1alpha1` → `v1`. Editors render against the served version's OpenAPI schema; conversion webhooks keep older stored versions readable. No special editor migration is required beyond rebuilding against the new schema.

## References

- Architecture overview §6.12 (CRD inventory), §10, §14.4, Workstream B5
- ADR 0013 — Capability CRD and CapabilitySet layering
- ADR 0017 — Generalized approval system
- ADR 0021 — `GrafanaDashboard` XRs
- ADR 0032 — CapabilitySet overlay semantics
- ADR 0034 — Audit pipeline (`AuditLog` XRD)
- ADR 0035 — Dynamic log and trace toggle (`LogLevel`)
- ADR 0037 — Tenant onboarding (`TenantOnboarding`)
- ADR 0038 — Policy simulators

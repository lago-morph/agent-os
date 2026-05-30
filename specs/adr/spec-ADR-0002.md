# SPEC ADR-0002 — OPA + Gatekeeper as the policy engine [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [A17;A18;A20;A23;B2;B3;B16;B19;B20] · adrs: [0002] · views: [6.6]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0002 fixes a single policy decision engine — OPA, with Gatekeeper as its Kubernetes admission integration — across admission, LiteLLM runtime callbacks, Envoy egress, Headlamp action gating, approval-level elevation, and virtual-key issuance. This SPEC states what honoring that single-engine decision requires: one policy language (Rego), one decision API at every enforcement point, Rego-as-code (Git-versioned, SHA-pinned, CI-tested), and no second policy toolchain. The decision is settled; the value here is the enforcement map and the conformance gates that prove "exactly one engine."

## 2. Scope
### 2.1 In scope
- OPA as the decision engine; Gatekeeper as the only admission path.
- The obligation that every policy enforcement point (admission, gateway callback, egress, Headlamp, approvals, virtual-key issuance) evaluates against OPA.
- Rego bundles in Git, ArgoCD-reconciled, SHA-pinned, CI unit-tested.

### 2.2 Out of scope (and where it lives instead)
- Rego library framework — B3; initial content — B16.
- LiteLLM callback chain implementation — B2.
- Envoy egress hook wiring — A6 (ADR 0003).
- Approval elevation logic — B19 (ADR 0017).
- Policy simulator — A20 (ADR 0038); image-signature verification — explicitly out (revisit backlog §3.9).

## 3. Context & Dependencies
Upstream: none (foundation). Downstream: A17, A18, A20, A23, B2, B3, B16, B19, B20 consume OPA decisions.
ADR decisions honored: **0002** — single engine; **0018** — OPA may only restrict, never grant beyond RBAC floor; **0003** — Envoy consults OPA at egress; **0017** — OPA elevates approval level; **0013** — capability CRDs are admission-gated.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs
Gatekeeper admission-gates all platform CRDs/XRs (`Agent`, `AgentRun`, `Sandbox`, `MCPServer`, `A2APeer`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`, `Approval`, and Crossplane XRs). `BudgetPolicy` is consumed by OPA + LiteLLM. No new CRD introduced by this ADR. Constraint templates are Gatekeeper-native (not platform CRDs).

### 4.2 APIs / SDK surfaces
Single OPA decision API consumed by LiteLLM callbacks (B2), Envoy egress, Headlamp, approvals. Rego is the single authoring language. No new SDK surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
OPA decisions, policy violations, and dynamic-registration accept/deny under `platform.policy.*`. Policy-bypass attempts under `platform.security.*`. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
N/A — engine decision; introduces no substrate primitive. OPA data inputs (e.g. `BudgetPolicy`) sourced from CRDs, not a connection secret.

## 5. OSS-vs-Custom Decision
Upstream projects: **OPA** (engine) + **Gatekeeper** (admission). Mode: install + config; policy content is custom (B3/B16). Rationale per ADR 0002: OPA's runtime-decision capability is required for LiteLLM callbacks regardless of admission choice; Kyverno rejected to avoid two policy languages. Accepted gap: no out-of-box image-signature verification (revisit backlog §3.9).

## 6. Functional Requirements
- REQ-ADR-0002-01: All Kubernetes admission policy MUST be enforced via Gatekeeper; there is no separate OPA-only admission path.
- REQ-ADR-0002-02: All runtime authorization (LiteLLM per-request tool/model auth, dynamic MCP/A2A registration, budget enforcement) MUST evaluate against OPA.
- REQ-ADR-0002-03: Envoy egress, Headlamp action gating, approval-level elevation, and virtual-key issuance checks MUST evaluate against OPA.
- REQ-ADR-0002-04: All policy MUST be authored as Rego bundles, Git-versioned, ArgoCD-reconciled, and SHA-pinned.
- REQ-ADR-0002-05: Rego unit tests MUST run as a required CI check.
- REQ-ADR-0002-06: No second policy engine/language may be introduced for the governed surface.
- REQ-ADR-0002-07: The OPA-as-restrictor invariant (ADR 0018) MUST hold — OPA may further restrict but never grant beyond the RBAC floor.

## 7. Non-Functional Requirements
- Security/tenancy: single decision API is the defense-in-depth backbone (admission + gateway + egress + UI + approvals).
- Observability: decisions auditable; emitted under `platform.policy.*`; Gatekeeper audit-mode feeds the policy simulator (ADR 0038).
- Versioning: bundles SHA-pinned; per-component contribution conventions deferred to B3/B16.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. Deliverable enforcement owned by A7 (install), B3 (framework), B16 (content); conformance in §9 / PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0002-01: Admission of a non-conformant platform CRD is denied by a Gatekeeper constraint; no non-Gatekeeper admission webhook gates the same resources. (REQ-01)
- AC-ADR-0002-02: A LiteLLM request for an unauthorized tool/model is denied by an OPA decision in the callback chain. (REQ-02)
- AC-ADR-0002-03: Envoy, Headlamp, approval elevation, and virtual-key issuance each produce a verifiable OPA decision for a test input. (REQ-03)
- AC-ADR-0002-04: Every active bundle is SHA-pinned and reconciled from Git via ArgoCD; an unpinned bundle fails CI. (REQ-04)
- AC-ADR-0002-05: Rego unit tests are a required check; a failing policy test blocks merge. (REQ-05)
- AC-ADR-0002-06: A repo scan finds no Kyverno/second-engine policy artifacts for the governed surface. (REQ-06)
- AC-ADR-0002-07: An OPA policy attempting to grant beyond RBAC is rejected by the restrictor conformance test. (REQ-07)

## 10. Risks & Open Questions
- Image-signature verification gap (blast radius: med) — accepted for v1.0; reconciled per ADR 0002, revisit trigger backlog §3.9.
- OPA-bridge callback ordering vs A1 (low) — ships with A7 if it lands before A1; settle in A7/B2 plans.
- Central policy-management UI absent (low) — compensated by Headlamp plugin; `[PROPOSED]` plugin scope in A9/ADR 0039.

## 11. References
- ADR 0002 (`adr/0002-opa-gatekeeper-policy-engine.md`) — the decision.
- Enforcing components: A7 (OPA/Gatekeeper install, owner), B3 (Rego framework), B16 (initial Rego content), B2 (LiteLLM callbacks), A6 (Envoy egress hook), A9/A22 (Headlamp gating), B19 (approval elevation), A20 (policy simulator).
- architecture-overview.md §6.6, §11.7, §14.2; architecture-backlog.md §2.2, §3.9, §6.
- Related: ADR 0003, 0013, 0017, 0018, 0038.

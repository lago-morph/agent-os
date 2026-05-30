# SPEC V6-06 — Security and policy architecture `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0002, 0003, 0017, 0018, 0027, 0034, 0038] · views: [6.6]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the security and policy slice: the platform-wide **defense-in-depth** model and the **RBAC-as-floor / OPA-as-restrictor** enforcement model that every component must honor. Enforcement is layered — capability scoping at admission (Gatekeeper), OPA runtime restriction at the gateway, FQDN allowlist at the egress proxy, gVisor/Kata kernel isolation in the sandbox, and complete audit emission. It is not a buildable component; it is realized by OPA/Gatekeeper (A7), the OPA policy library framework (B3), the initial OPA policy content (B16), the security threat-model design (B22), and the audit endpoint/adapter (A18).

The problem it solves: an agent platform's primary v1.0 threat is **users or Platform Agents exceeding their authorized access, with a focus on unintentional excess**. Misconfigured capabilities, prompt-injection-driven actions, careless OPA policies, and over-broad RBAC grants are the dominant concern. This view fixes the enumerated audit and OPA decision points, the self-serve OPA-gated virtual-key issuance invariant, the OPA-driven budget invariant, the dry-run policy-simulator invariant, and the threat-model-first ordering so every component lands with enforcement wired in from the start.

## 2. Scope
### 2.1 In scope
- The v1.0 threat-model stance: unintentional excess as primary modeled threat; the defense-in-depth model designed against it.
- The dedicated adversarial threat-model design deliverable (B22) that ships before the first A/B implementation wave and feeds standards forward (ADR 0027).
- The defense-in-depth invariant: capability scope at admission (Gatekeeper), OPA at gateway, FQDN allowlist + OPA at egress, gVisor/Kata at kernel, complete audit.
- The RBAC-as-floor / OPA-as-restrictor enforcement model platform-wide (ADR 0018).
- The enumerated audit-emission points and OPA decision points (§6.6) as the canonical hook-point set.
- Self-serve virtual-key issuance: `VirtualKey` CRD, Headlamp-driven, OPA-gated, RBAC-aware (no manual issuance, no tickets).
- Cost budgets through OPA: `BudgetPolicy` CRD reconciled into OPA data + LiteLLM tracking; Headlamp is the editing surface.
- The policy simulator (ADR 0038): structured dry-run at each layer, `simulated: true`, audited under `platform.policy.*`, no enforcement-path side effects.

### 2.2 Out of scope (and where it lives instead)
- Gateway internals (callback chain mechanics) — view V6-01 / components A1, B2.
- Sandbox/egress runtime implementation — view V6-02 / component A6 / ADR 0003.
- Audit pipeline store topology — views V6-03 / V6-05 / component A18 / ADR 0034.
- Capability-registry CRD model / CapabilitySet layering — view V6-08 / ADR 0013 / ADR 0032.
- Multi-tenancy / namespacing model — view V6-09 / ADR 0016.
- Identity federation chain / JWT claim schema — view V6-11 / ADR 0028 / ADR 0029.
- Approval-system mechanics (`Approval` CRD + Argo Workflow) — component B19 / ADR 0017 (referenced here only as an OPA decision point and audit point).
- Policy-simulator service implementation — component A20 / ADR 0038 (consumed here).

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A7 OPA / Gatekeeper** — the policy engine: Gatekeeper for admission, OPA for runtime decisions (ADR 0002).
- **B3 OPA policy library (framework)** — the reusable policy framework the decision points consume.
- **B16 Initial OPA policy library content** — the concrete v1.0 policy content for the enumerated decision points.
- **B22 Security threat model design specification** — the design deliverable (not code) that ships before the first A/B wave and feeds standards forward (ADR 0027).
- **A18 Audit endpoint + audit adapter library** — the audit plane that records every enumerated audit point (ADR 0034).

ADR decisions honored:
- **ADR 0002** — OPA + Gatekeeper as the policy engine (admission + runtime).
- **ADR 0003** — Envoy egress proxy as the egress enforcement layer (FQDN allowlist + OPA).
- **ADR 0017** — generalized approval system via `Approval` CRD + Argo Workflows + OPA elevation (escalation out of v1.0).
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor enforcement model platform-wide.
- **ADR 0027** — threat-model scope: v1.0 stance and B22 prerequisite (ships before first implementation wave).
- **ADR 0034** — audit emission via the adapter/endpoint; every enumerated audit point uses it.
- **ADR 0038** — policy simulators for OPA and RBAC (structured dry-run at each layer).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Consumed (owned by B13):
- `VirtualKey` — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`.
- `BudgetPolicy` — `scope` (key/agent/team/tenant), `period`, `limits`, `thresholdActions[]` (consumed by OPA + LiteLLM).
- `CapabilitySet` — `opaPolicyRefs[]` (the OPA-policy attachment surface).

Consumed (owned by Argo Workflow + B19):
- `Approval` — `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt` (OPA elevation per ADR 0017).

Decision points apply to the full admission set (Gatekeeper): `Agent`, `AgentRun`, `Sandbox`, `SandboxTemplate`, `Memory`, `MemoryStore`, `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`, `Approval`, `LogLevel`, and the Crossplane XRs (`AgentEnvironment`, `SyntheticMCPServer`, `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `XAgentDatabase`, `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`).

All namespaced; versioning per ADR 0030; owners per the interface contract.

### 4.2 APIs / SDK surfaces
- OPA decision API at each enumerated decision point (LiteLLM callbacks, dynamic registration, LibreChat, Envoy egress, Headlamp action gates, approval elevation, `agent-platform` CLI promotion gates, HolmesGPT autonomy gate).
- Structured **dry-run mode** at each layer (ADR 0038): same decision shape as the live path plus `simulated: true`, no enforcement-path side effects.
- Audit adapter library (A18) — every enumerated audit point links it; no direct store writes.
- Headlamp plugin actions — the editing surface for OPA bundles, virtual keys, budgets (one editing surface across components).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the security/policy slice (per-event-type names deferred to B12 registry):
- `platform.policy.*` — OPA decisions, policy violations, dynamic-registration accept/deny, and **all simulator dry-run decisions** (audited even though they never enforce).
- `platform.security.*` — security events distinct from audit: sandbox-escape signal, repeated authn failures, policy-bypass attempts.
- `platform.audit.*` — every enumerated audit point (admission, gateway, egress, ESO, ArgoCD, LibreChat, Headlamp actions, approval, Coach, HolmesGPT, kopf registry changes) via the adapter.
- `platform.approval.*` — approval requested, OPA-elevated, decided, timed out (ADR 0017).

### 4.4 Data schemas / connection-secret contracts
- Budgets: `BudgetPolicy` reconciled into OPA data + LiteLLM spend tracking; OPA returns allow/deny on each request given current spend.
- Secrets: ESO handles MCP-service secrets; secret access is itself an audit-emission point (built in).
- Simulator: per-layer dry-run inputs compose into a single response; every dry-run is recorded under `platform.policy.*` with `simulated: true`.

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom for OPA/Gatekeeper, the policy framework/content, and the audit endpoint lives in component SPECs A7, B3, B16, A18; B22 is a design spec).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-06-01:** The v1.0 modeled primary threat MUST be treated as users or Platform Agents exceeding authorized access (focus on unintentional excess), and the defense-in-depth model MUST be designed primarily against this class. (§6.6, line ~440)
- **REQ-V6-06-02:** A dedicated adversarial threat-model design (B22) MUST ship before the first wave of component implementation lands and MUST feed standards, acceptance criteria, mandatory test cases, and dashboard signals forward into every component (ADR 0027). (§6.6, lines ~442–453)
- **REQ-V6-06-03:** Per-agent allowed capabilities MUST be enforced at every layer of defense in depth: declaratively (`Agent` CRD + `CapabilitySet`), at the gateway (virtual-key claims + OPA), at the egress proxy (FQDN allowlist + OPA), and at the kernel (gVisor/Kata). (§6.6, line ~489)
- **REQ-V6-06-04:** Enforcement MUST follow RBAC-as-floor / OPA-as-restrictor platform-wide: RBAC grants the permission floor and OPA may only further restrict per-decision (ADR 0018). (§6.6, line ~440; glossary)
- **REQ-V6-06-05:** Every enumerated audit-emission point (§6.6) MUST emit audit through the adapter/endpoint; no enumerated point may be silently omitted. (§6.6, lines ~515–530)
- **REQ-V6-06-06:** Every enumerated OPA decision point (§6.6) MUST consult OPA at the stated layer; the admission decision point MUST cover the full CRD/XR admission set. (§6.6, lines ~532–542)
- **REQ-V6-06-07:** Virtual-key issuance MUST be self-serve via Headlamp, RBAC-aware and OPA-gated against the requestor's Keycloak identity; no admin manually issues keys and users MUST NOT file tickets; keys are `VirtualKey` CRDs reconciled by the kopf operator. (§6.6, lines ~491–495)
- **REQ-V6-06-08:** Cost budgets MUST be configured through OPA (not LiteLLM's admin UI): a `BudgetPolicy` CRD reconciled into OPA data + LiteLLM tracking, with the per-request OPA callback returning allow/deny given current spend; Headlamp is the editing surface. (§6.6, lines ~497–509)
- **REQ-V6-06-09:** The policy simulator MUST answer "what would happen at each enforcement layer?" via a structured dry-run at each layer (RBAC, Gatekeeper audit-mode, LiteLLM OPA callback, Envoy egress OPA, approval elevation, CLI promotion gates), emitting the same decision shape with `simulated: true` and NO enforcement-path side effects (ADR 0038). (§6.6, lines ~544–551)
- **REQ-V6-06-10:** Simulator dry-run decisions MUST be audited under `platform.policy.*` so policy-edit history is observable, while never producing enforcement-path effects. (§6.6, line ~551)
- **REQ-V6-06-11:** RBAC MUST be reported as a separately-attributed layer alongside the OPA-driven layers in the simulator, so admins can distinguish denial at the floor from restriction by policy. (§6.6, line ~548)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** defense-in-depth is the security model; tenant ↔ tenant is a named trust boundary in the B22 threat model; capability scope is enforced at admission/gateway/egress/kernel.
- **Observability (§6.5):** all policy decisions emit `platform.policy.*`; security events emit `platform.security.*`; every enumerated audit point flows via the adapter; simulator runs are audited.
- **Scale:** OPA decisions on the request hot path (gateway/egress) must be low-latency; dry-run must add no live-path side effects.
- **Versioning (ADR 0030):** policy bundles and CRDs versioned per-component; OPA decision shape stable across the live and dry-run paths.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A7, B3, B16, A18, and the B22 design spec).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-06-01:** The defense-in-depth controls are present and demonstrably target unintentional-excess scenarios (misconfigured capability, prompt-injection action, over-broad RBAC). (→ REQ-01)
- **AC-V6-06-02:** B22 is delivered and merged before the first A/B implementation wave, and its standards/ACs/test-cases/dashboard-signals are referenced by component deliverable lists. (→ REQ-02)
- **AC-V6-06-03:** An agent's attempt to use a capability outside its set is blocked at admission, gateway, egress, and would be contained at the kernel — each layer independently denies. (→ REQ-03)
- **AC-V6-06-04:** A request RBAC permits but OPA denies is denied; a request RBAC denies cannot be allowed by OPA (OPA cannot widen the floor). (→ REQ-04)
- **AC-V6-06-05:** Each enumerated audit point produces an audit record via the adapter under load; a scan finds no enumerated point missing emission. (→ REQ-05)
- **AC-V6-06-06:** Each enumerated OPA decision point is exercised and consults OPA; admission covers the full CRD/XR set. (→ REQ-06)
- **AC-V6-06-07:** A developer self-issues a `VirtualKey` through Headlamp gated by OPA on their Keycloak identity; no manual/ticketed path exists. (→ REQ-07)
- **AC-V6-06-08:** A `BudgetPolicy` reconciles into OPA + LiteLLM; a request over budget is denied by the OPA callback; the budget is editable in Headlamp, not LiteLLM's admin UI. (→ REQ-08)
- **AC-V6-06-09:** The simulator returns a per-layer decision composite for a sample request with `simulated: true` and causes no state change or live denial. (→ REQ-09)
- **AC-V6-06-10:** Each simulator dry-run produces a `platform.policy.*` audit record with no enforcement-path effect. (→ REQ-10)
- **AC-V6-06-11:** The simulator response attributes RBAC as a distinct layer from the OPA-driven layers. (→ REQ-11)

## 10. Risks & Open Questions
- **OQ-1 (med):** Some LiteLLM budget features (alert routing, automated suspension) may be enterprise-only; if so, extended via callbacks (B2). OPA-as-policy / LiteLLM-as-spend-tracker invariant is unchanged. (§6.6, line ~509)
- **OQ-2 (low):** Approval escalation is out of v1.0 (ADR 0017); only request/elevation/decision/timeout are in scope as audit/approval events.
- **R-1 (high):** A missed audit or OPA decision point creates a silent enforcement/observability gap; mitigated by REQ-05/06 enumerated-point conformance and B22 mandatory test cases.
- **R-2 (med):** A dry-run that leaks an enforcement-path side effect would corrupt the simulator's safety guarantee; mitigated by REQ-09 "no side effects" and `simulated: true` shape parity.
- **R-3 (med):** Careless OPA policy can over-restrict (break a legitimate flow) or under-restrict (admit excess); mitigated by the simulator (REQ-09) as the primary editing companion and by RBAC-as-floor bounding.

## 11. References
- architecture-overview.md §6.6 Security and policy architecture (lines ~436–551); §6.2 (sandbox/egress controls); §6.5 (audit/observability); §6.8 (capability scoping).
- ADRs: 0002 (OPA + Gatekeeper), 0003 (Envoy egress), 0017 (approval system), 0018 (RBAC-as-floor / OPA-as-restrictor), 0027 (threat-model scope / B22 prerequisite), 0034 (audit pipeline), 0038 (policy simulators).
- Realizing components: A7 (OPA/Gatekeeper), B3 (policy framework), B16 (policy content), B22 (threat-model design), A18 (audit endpoint/adapter); policy simulator A20; budgets/keys via B13; approval via B19.
- Related views: V6-01, V6-02, V6-03, V6-05, V6-08, V6-09, V6-11.

# SPEC A7 — OPA / Gatekeeper

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A17, A18, A20, A23, B2, B3, B19, B20] · adrs: [0002, 0018, 0038, 0031, 0034, 0030, 0044, 0017] · views: [6.6]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05

## 1. Purpose & Problem Statement

A7 installs and operates the platform's policy engine: **Open Policy Agent** as the decision engine and **Gatekeeper** as its Kubernetes admission integration (ADR 0002). It is the single decision API and single policy language (Rego) consulted at every enforcement point in the defense-in-depth model: **Kubernetes admission** (Gatekeeper), **LiteLLM gateway callbacks**, **Envoy egress**, **Headlamp action gating**, **approval-level elevation**, and **virtual-key issuance** (§6.6). Cost budgets are enforced **through OPA**, not LiteLLM's admin UI.

The problem A7 solves is **uniform, reviewable governance**. With one engine and one language, reviewers can audit RBAC for the outer permission bound and OPA bundles for per-decision narrowing, with the guarantee that **OPA may only restrict, never grant** (RBAC-as-floor / OPA-as-restrictor, ADR 0018). A7 owns the engine + Gatekeeper install and the structured **dry-run mode** each decision point exposes for the policy simulator (ADR 0038). It also carries the mandatory **OPA-bridge callback** for LiteLLM if A7 lands before A1 (§14.2 B2 note).

A7 is **T0, contract-owning** — almost every governed component depends on the OPA decision contract: MCP services (A17), audit (A18), policy simulator (A20), Kargo (A23), LiteLLM callbacks (B2), the policy-library framework (B3), the approval system (B19), and PV access (B20).

## 2. Scope

### 2.1 In scope

- **OPA** decision engine install + **Gatekeeper** admission-controller install (Helm).
- The Gatekeeper admission integration for **all v1.0 platform CRDs + Crossplane XRs** (the §6.6 OPA-decision-points admission list).
- The **single decision API** consumed by LiteLLM callbacks, Envoy egress, Headlamp gating, approvals, virtual-key issuance.
- The **RBAC-as-floor / OPA-as-restrictor** enforcement contract (ADR 0018) — restrict-or-deny only, never grant.
- **Budget enforcement contract** — OPA consulted with current spend + `BudgetPolicy` (ADR 0006/0002).
- **Structured dry-run mode** for admission (Gatekeeper audit-mode + dry-run admission) feeding the policy simulator (ADR 0038); dry-run decisions emit the same shape with `simulated: true`.
- **Substrate-XR admission** — reject any Crossplane XR with no matching Composition for the cluster's `platform.io/environment` label (ADR 0044 + 0002).
- Rego treated as code: SHA-pinned bundles, CI unit tests, GitOps-reconciled.
- The mandatory **OPA-bridge LiteLLM callback** if A7 lands before A1 (§14.2 B2).
- Audit emission of admission decisions + `platform.policy.*` events.
- The §14.1 standard deliverable set.

### 2.2 Out of scope (and where it lives instead)

- **Rego policy content** — framework in **B3**; initial library content (admission for all v1.0 CRDs, runtime decisions, egress, RBAC-floor helpers, Headlamp gating) in **B16**; per-component policies land in their own components.
- **LiteLLM callbacks** (PII, audit, guardrails) — **B2** / **A1** (A7 owns only the OPA-bridge callback contract, and carries it if A1 lands first).
- **Envoy egress proxy** — **A6** (A7 provides the decision; A6 calls it).
- **The policy simulator service** (per-layer composition) — **A20** (A7 provides the per-layer dry-run; A20 composes).
- **Approval CRD + elevation logic** — **B19** (A7 provides the elevation decision; B19 owns the `Approval` workflow).
- **Audit endpoint / adapter** — **A18**.
- **Image signature verification** — explicitly **not** available (Kyverno path rejected, ADR 0002); deferred to backlog.
- **Central policy-management UI** — none ships with OPA; Headlamp plugin compensates (review/bundle mgmt) — framework A9/A22, content per component.

## 3. Context & Dependencies

**Upstream consumed:** None (W0 foundation). OPA + Gatekeeper OSS on the k8-platform baseline.

**Downstream consumers (what they consume):**
- **A17** — MCP-service access decisions (web-search/scrape access-layer gating).
- **A18** — admission/audit of the audit pipeline's XRs; policy on audit access.
- **A20** — the per-layer structured dry-run decisions to compose the simulator trace.
- **A23 (Kargo)** — promotion-action policy decisions.
- **B2** — the OPA-bridge callback target at LiteLLM.
- **B3 / B16** — the engine the policy library targets.
- **B19** — required-approval-level elevation decisions (ADR 0017).
- **B20** — PV-access RBAC+OPA decisions.
- Cross-cutting: A1 (gateway runtime authz, budget, dynamic-registration), A6 (egress), A4 (Trigger filtering), Headlamp action gates.

**ADR decisions honored:**
- **ADR 0002** — OPA = decision engine, Gatekeeper = admission; one Rego language + one decision API across admission/gateway/egress/Headlamp/approvals; Gatekeeper *is* OPA's admission controller ("OPA admission" = Gatekeeper); Rego tested in CI, SHA-pinned; **no image-signature verification** in v1.0.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor invariant enforced in one place: OPA returns a result equal to or more restrictive than RBAC, never grants; "restrict" includes raising approval level and outright deny.
- **ADR 0038** — Gatekeeper audit-mode + dry-run admission are data sources for the policy simulator; every decision point exposes a dry-run emitting the same decision shape with `simulated: true`; simulator runs are audited under `platform.policy.*` with no enforcement side effects.
- **ADR 0031** — OPA decisions / violations / dynamic-registration accept-deny emit under `platform.policy.*`.
- **ADR 0034** — admission decision audit goes through the platform audit adapter; no direct store writes.
- **ADR 0044** — Gatekeeper rejects substrate XRs with no matching Composition for the cluster's `platform.io/environment` label.
- **ADR 0017** — OPA elevates required approval level for `Approval` (can elevate, never lower; never make approvable by someone RBAC would forbid).
- **ADR 0030** — Gatekeeper Constraint/ConstraintTemplate and any A7 config follow versioning policy.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A7 installs Gatekeeper's own CRDs (`ConstraintTemplate`, `Config`, and generated Constraint kinds) — these are **upstream Gatekeeper CRDs**, not platform CRDs in the §6.12 inventory. A7 owns no *platform* CRD reconciler.

A7's admission applies to (the §6.6 OPA-decision-points admission set; owners noted):

| Admitted resource | Owner of reconciler |
|---|---|
| `Agent`, `AgentRun`, `Memory` | ARK (A5) |
| `Sandbox`, `SandboxTemplate` | agent-sandbox (A6) |
| `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy` | kopf operator (B13) |
| `Approval`, `LogLevel` | B19 / per-component |
| `MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer`, `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `AgentDatabase`, `Postgres`, `SearchIndex`, `ObjectStore`, `MongoDocStore` (XRs/XRDs) | Crossplane (B4) |

Platform CRDs owned here: **N/A — A7 installs upstream OPA/Gatekeeper; it introduces no platform CRD. Gatekeeper Constraint/ConstraintTemplate are upstream kinds.**

### 4.2 APIs / SDK surfaces

- **OPA decision API** — the single query interface consumed by LiteLLM callbacks, Envoy egress, Headlamp gating, the approval system, virtual-key issuance. Returns a decision (allow/deny + obligations) bounded by RBAC (restrictor only).
- **Gatekeeper admission webhook** — admit/deny on CR create/update; audit-mode continuous re-evaluation.
- **Structured dry-run interface** — each decision point's dry-run input + the same decision shape with `simulated: true`; consumed by A20. `[PROPOSED — not in source]` exact decision-shape field names (subject/action/target/decision/reason/`simulated`) — reconcile with A20/ADR 0038.
- No bespoke HTTP service beyond OPA/Gatekeeper's own; the OPA-bridge LiteLLM callback (if carried here) is Python in the LiteLLM callback chain (A1 contract).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Emitted (per-event-type names deferred to B12):
- `platform.policy.*` — OPA decisions, policy violations, dynamic-registration accept/deny, simulator dry-run records.
- `platform.security.*` — security-relevant policy signals (policy-bypass attempts, break-glass activation, repeated denials).
- `platform.audit.*` — every CR admit/deny (the §6.6 "Kubernetes admission via Gatekeeper" audit point), via the audit adapter; emission is gated on the audit-adapter freeze-gate (D-05).
- `platform.approval.*` — OPA-elevation decision contributes to the approval flow (emitted in concert with B19). `[PROPOSED — not in source]` whether A7 or B19 emits the elevation event — reconcile with B19 (ADR 0017).

**Namespace ownership (QN-03):** A7 is the **single owner of both `platform.policy` and `platform.security`** — A7 authors and registers the schemas for both namespaces in B12 (the event catalogue). All other components — including every component that emits a security-relevant event under `platform.security` — take an explicit dependency on A7 as the schema owner and do not co-own either namespace. **Cross-cutting security contract:** every component that detects a security-relevant event MUST (1) perform its existing local handling AND (2) additionally emit the event to the event bus under `platform.security` against the schema A7 owns.

Consumed: none directly required for enforcement (decisions are pull/query, not event-driven).

Every event carries `specversion` + `schemaVersion` (ADR 0031/0030).

### 4.4 Data schemas / connection-secret contracts

- OPA **data** is loaded from Rego bundles + reconciled CRD-derived data (e.g. `BudgetPolicy` → OPA data, reconciled by B13). A7 holds no system-of-record state; bundles are in Git (SHA-pinned), reconciled by ArgoCD.
- No connection-secret XRD. `[PROPOSED — not in source]` the exact OPA-data document shape for budgets / capability data — not specified; reconcile with B13/B16.

## 5. OSS-vs-Custom Decision

- **OPA + Gatekeeper:** OSS. **Decision: config** — install both via Helm; configure Gatekeeper admission for the v1.0 resource set; load Rego bundles (content from B3/B16). No fork.
- **ADR 0002 linkage:** OPA over Kyverno because OPA's runtime decision capability is required for LiteLLM callbacks regardless of admission choice — one language avoids two toolchains. Accepted gap: **no out-of-the-box image-signature verification** (Kyverno's strength); deferred per backlog §3.9.
- **Custom code:** the dry-run decision-shape contract, the substrate-XR admission rule (ADR 0044), and the OPA-bridge LiteLLM callback (carried here if A7 precedes A1). Rego content is B3/B16, not A7.

## 6. Functional Requirements

- **REQ-A7-01:** A7 SHALL install OPA (decision engine) and Gatekeeper (admission integration) via Helm.
- **REQ-A7-02:** Gatekeeper SHALL admit/deny all v1.0 platform CRDs and Crossplane XRs in the §6.6 admission set.
- **REQ-A7-03:** Every OPA decision (admission, gateway, egress, Headlamp, approvals, virtual-key issuance) SHALL be a **restrictor** over an RBAC-bounded identity and SHALL NOT grant any permission RBAC has not given (ADR 0018).
- **REQ-A7-04:** OPA SHALL provide a single decision API and single policy language (Rego) across all enforcement points (ADR 0002).
- **REQ-A7-05:** The budget-enforcement contract SHALL evaluate current spend + `BudgetPolicy` and return allow/deny; budgets SHALL be configured through OPA, not LiteLLM's admin UI.
- **REQ-A7-06:** Each OPA decision point SHALL expose a **structured dry-run** input emitting the same decision shape with `simulated: true` and no enforcement side effects; Gatekeeper audit-mode + dry-run admission SHALL be available as simulator data sources (ADR 0038). A7 **runs the OPA engine**; it does **not** own the OPA decision-document format — that format is owned by **B3** (D-03). A7 consumes/produces decisions in B3's format, and A1/A6 are consumers of the same B3-owned format.
- **REQ-A7-07:** Gatekeeper SHALL **reject any Crossplane XR with no matching Composition** for the cluster's `platform.io/environment=kind|aws` label (ADR 0044).
- **REQ-A7-08:** Rego bundles SHALL be SHA-pinned, GitOps-reconciled, and unit-tested in CI (ADR 0002).
- **REQ-A7-09:** Every CR admit/deny SHALL emit an audit event via the platform audit adapter; A7 SHALL NOT write audit directly to any store (ADR 0034). Audit emission is gated on the audit-adapter freeze-gate (D-05) — A7 cannot emit audit events until the frozen audit-adapter interface and `audit_events` schema pass the freeze-gate.
- **REQ-A7-14 (distinct RBAC granularity, #23):** The platform SHALL provide distinct RBAC permissions at the proposed granularity — separately for *set agent-type default budget* vs *per-instance budget override* vs *budget-maintainer*, and for *policy-change* vs *policy-maintain*. Permissions native RBAC can express are granted via Roles/ClusterRoles; field-level rules native RBAC cannot express SHALL be enforced via Gatekeeper (OPA) admission (already deployed by A7).
- **REQ-A7-15 (decision-document format ownership, D-03):** A7 runs the OPA decision engine but SHALL NOT own or define the OPA decision-document format; that format is owned by **B3**. A7 (engine), A1 (gateway), and A6 (sandbox proxy) are consumers of the B3-owned format.
- **REQ-A7-16 (cross-cutting security emission, QN-03):** A7 SHALL author and register the schemas for the `platform.policy` and `platform.security` namespaces in B12, and any component detecting a security-relevant event SHALL both handle it locally AND emit it to the bus under `platform.security` against A7's schema.
- **REQ-A7-10:** OPA decisions / violations / dynamic-registration accept-deny / simulator dry-runs SHALL emit under `platform.policy.*` (ADR 0031).
- **REQ-A7-11:** For `Approval`, OPA SHALL be able to elevate the required approval level and never lower it, and never make an action approvable by an identity RBAC would forbid (ADR 0017/0018).
- **REQ-A7-12:** If A7 lands before A1, the mandatory **OPA-bridge LiteLLM callback** SHALL ship with A7 (§14.2 B2 note).
- **REQ-A7-13:** A7 SHALL NOT provide image-signature verification in v1.0 (accepted gap, ADR 0002).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** namespace-scoped RBAC defines the tenant boundary; OPA layers cross-tenant restrictions on top, never bridging tenants RBAC separated (ADR 0016/0018). Decisions are deterministic and reviewable.
- **Availability:** the admission webhook is in the critical path of every CR write; A7 must be HA and fail-safe (admission failure policy must not silently admit policy-violating resources). `[PROPOSED — not in source]` fail-open vs fail-closed default — not specified; recommend fail-closed for security-relevant constraints.
- **Observability (§6.5):** OTel + metrics on decision latency, deny rates, constraint violations; per-component Grafana dashboard via `GrafanaDashboard` XR.
- **Performance:** runtime OPA decisions (LiteLLM/Envoy callbacks) are per-request — decision latency must be bounded.
- **Versioning (ADR 0030):** Gatekeeper Constraint/ConstraintTemplate + A7 config versioned; bundles SHA-pinned.

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | applicable — OPA + Gatekeeper charts |
| Per-product docs (10.5) | applicable |
| Operator runbook (10.7) | applicable — webhook-down recovery, constraint debugging, fail-policy |
| Backup / restore | applicable-with-caveat — no SoR state; bundles + constraints in Git (rebuild by re-sync) |
| Alert rules | applicable — webhook down/unreachable, constraint-violation spike, decision-latency breach |
| Grafana dashboard (`GrafanaDashboard` XR) | applicable |
| Headlamp plugin | applicable — policy review + bundle management (compensates for no OPA UI; framework A9/A22, A7 ensures integration) |
| OPA/Rego integration | applicable — A7 **is** the engine; ships the dry-run contract + substrate-XR constraint; content from B3/B16 |
| Audit emission (ADR 0034) | applicable — every admit/deny |
| Knative trigger flow | applicable — `platform.policy.*` violation flow |
| HolmesGPT toolset | applicable — "why was this denied" (policy-simulator skill via A20), constraint-state queries |
| 3-layer tests | applicable — Chainsaw (admission admit/deny), PyTest (Rego unit + decision/dry-run shape), Playwright (Headlamp policy view) |
| Tutorials & how-tos | applicable — "add an admission constraint", "use the policy simulator" |

## 9. Acceptance Criteria

- **AC-A7-01** (REQ-A7-01): OPA + Gatekeeper pods are Ready after Helm install. *(Chainsaw)*
- **AC-A7-02** (REQ-A7-02): Creating each v1.0 CRD/XR triggers a Gatekeeper admission decision; a policy-violating instance of each is denied. *(Chainsaw)*
- **AC-A7-03** (REQ-A7-03): For a constructed case, OPA returns deny/restrict for an RBAC-permitted action but never returns allow for an action RBAC forbids. *(PyTest, Rego unit)*
- **AC-A7-04** (REQ-A7-04): The same Rego decision API answers an admission query and a runtime (gateway/egress) query with consistent semantics. *(PyTest)*
- **AC-A7-05** (REQ-A7-05): With a `BudgetPolicy` over limit, the budget decision returns deny; budgets are sourced from OPA data, not LiteLLM admin UI. *(PyTest)*
- **AC-A7-06** (REQ-A7-06): A dry-run query at each decision point returns the same shape with `simulated: true`, no enforcement side effect; Gatekeeper audit-mode reports current violations. *(PyTest + Chainsaw)*
- **AC-A7-07** (REQ-A7-07): On a `platform.io/environment=kind` cluster, an XR whose only Composition targets `aws` is rejected at admission. *(Chainsaw)*
- **AC-A7-08** (REQ-A7-08): Rego unit tests run in CI; bundles are SHA-pinned; an unpinned/tampered bundle fails the gate. *(PyTest/CI)*
- **AC-A7-09** (REQ-A7-09): A sampled admit/deny produces an audit record at the audit endpoint via the adapter; no direct store write from A7. *(PyTest)*
- **AC-A7-10** (REQ-A7-10): Decisions/violations/dynamic-registration/simulator-runs emit under `platform.policy.*`. *(PyTest)*
- **AC-A7-11** (REQ-A7-11): OPA can elevate an `Approval` required level but a policy attempting to lower it (or approve via an RBAC-forbidden identity) has no effect / is rejected. *(PyTest)*
- **AC-A7-12** (REQ-A7-12): If A1 is absent, the OPA-bridge callback artifact ships with A7 and registers into the LiteLLM callback chain when A1 lands. *(PyTest)*
- **AC-A7-13** (REQ-A7-13): No image-signature-verification constraint is shipped; the gap is documented. *(doc check)*

## 10. Risks & Open Questions

- **R1 (high):** Admission webhook is in the write path of every CR; an outage or a fail-open misconfig either blocks all CR writes or silently admits policy violations. *Open question:* fail-open vs fail-closed default is `[PROPOSED — not in source]` — recommend fail-closed for security constraints, documented per-constraint. Blast radius high.
- **R2 (high):** No image-signature verification (ADR 0002 accepted gap) leaves a supply-chain hole (skill artifacts, images). *Reconciliation:* B22 threat model must cover; revisit per backlog §3.9 (Kyverno-for-images / Sigstore / Connaisseur).
- **R3 (med):** Decision-shape field names for the dry-run contract are `[PROPOSED — not in source]`; A20, A1, A6 all depend on a single agreed shape — reconcile before any of those freeze.
- **R4 (med):** OPA-bridge callback ownership oscillates between A1/A7/B2 depending on landing order (§14.2 B2). Must be an explicit handoff or a gap opens at the gateway. *Reconciliation:* A7 carries it if A1 lands first; document the swap.
- **R5 (med):** Approval-elevation event emitter (A7 vs B19) is `[PROPOSED]`; reconcile with B19/ADR 0017.
- **R6 (low):** OPA-data document shape for budgets/capabilities is `[PROPOSED — not in source]`; reconcile with B13/B16.

## 11. References

- architecture-overview.md §6.6 Security and policy architecture (line ~436 — defense-in-depth, audit/OPA hook points, budgets-through-OPA, self-serve key issuance, policy simulator), §6.5 (~370, observability), §6.13 (~981, versioning), §14.2 (B2/B3/B16 policy split).
- ADR 0002 (OPA/Gatekeeper), ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0038 (policy simulators), ADR 0031 (CloudEvent taxonomy), ADR 0034 (audit pipeline), ADR 0044 (substrate abstraction — XR admission), ADR 0017 (approval elevation), ADR 0030 (versioning), ADR 0016 (multi-tenancy).
- Related pieces: A1 (gateway callbacks), A6 (egress), A18 (audit), A20 (policy simulator), A23 (Kargo), B2 (callbacks), B3 (policy framework), B16 (policy content), B19 (approval), B20 (PV access), B13 (BudgetPolicy→OPA data).

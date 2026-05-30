# SPEC ADR-0003 — Envoy egress proxy as CNI-agnostic egress control [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [A5;A20;B7;B16;B20] · adrs: [0003] · views: [6.1;6.6;6.8;6.9]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0003 fixes an Envoy-based egress proxy as the single chokepoint for all non-LiteLLM agent outbound HTTP, CNI-agnostically across EKS, AKS, and kind. This SPEC states what honoring that decision requires: every Platform Agent outbound call traverses either LiteLLM or the Envoy egress proxy — no other path to external services exists; Envoy enforces the FQDN allowlist derived from each agent's `EgressTarget` capabilities, calls OPA for runtime decisions, and emits audit on every connection. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- Envoy egress proxy as component A6's egress chokepoint for all non-LiteLLM outbound HTTP.
- The invariant: outbound traffic traverses LiteLLM **or** Envoy egress only.
- Per-agent-class FQDN allowlist enforcement derived from `EgressTarget` capabilities (via CapabilitySet composition).
- OPA hook at egress; structured audit emission per connection through the audit adapter.
- CNI-agnostic portability across EKS, AKS, kind.

### 2.2 Out of scope (and where it lives instead)
- Envoy/agent-sandbox install mechanics + deliverables — component A6 SPEC.
- `EgressTarget` CRD reconciliation into config — B13 (kopf operator, ADR 0006/0013).
- CapabilitySet overlay/override resolution for per-Agent `EgressTarget` — ADR 0032.
- OPA decision engine itself — ADR 0002; egress audit destination — ADR 0034.
- Graphical `EgressTarget` editor — ADR 0039; runtime Envoy log-level — ADR 0035.

## 3. Context & Dependencies
Upstream consumed: none directly (A6 is foundation; reads `EgressTarget`/CapabilitySet as data). Downstream consumers: A5, A20, B7, B16, B20 depend on the egress chokepoint and its OPA/audit hooks.
ADR decisions honored: **0003** — Envoy is the single non-LiteLLM egress chokepoint, config is declarative from `EgressTarget`; **0002** — Envoy consults OPA at egress; **0034** — egress audit flows through the audit adapter; **0026** — CNI-agnostic for independent-cluster install; **0032** — per-Agent `EgressTarget` overlays; **0013** — `EgressTarget` is the declarative input.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Consumes `EgressTarget` (namespaced; `fqdn`, `port`, `scheme`, `allowedMethods`) and `CapabilitySet` (`egressTargets[]`). Envoy config is derived; nothing is written to the cluster by hand — edits flow Git → ArgoCD. No new CRD introduced by this ADR.

### 4.2 APIs / SDK surfaces
No new SDK surface. Envoy queries the single OPA decision API (ADR 0002) at the egress hook for runtime decisions.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Egress audit events flow under `platform.audit.*` via the audit adapter (never a direct Envoy → OpenSearch sink). OPA egress decisions/violations may surface under `platform.policy.*`. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
N/A — no substrate primitive. Egress audit is mediated solely by the audit adapter (ADR 0034); OpenSearch is at most advisory fanout off that pipeline.

## 5. OSS-vs-Custom Decision
Upstream project: **Envoy**, installed via Helm and configured per agent class (A6). Mode: install + per-agent-class config. Rationale per ADR 0003: Cilium L7 NetworkPolicy is unavailable across EKS/AKS/kind, so it would constrain the cluster set or fork egress policy per environment; Envoy is CNI-agnostic with first-class OPA + audit hooks. New operational surface: an Envoy install + per-agent-class config to own and upgrade.

## 6. Functional Requirements
- REQ-ADR-0003-01: Every Platform Agent outbound call MUST traverse LiteLLM or the Envoy egress proxy; no other external path exists.
- REQ-ADR-0003-02: Envoy MUST enforce the FQDN allowlist derived from each agent's `EgressTarget` capabilities (per agent class).
- REQ-ADR-0003-03: Envoy MUST call OPA for a runtime decision on egress connections.
- REQ-ADR-0003-04: Envoy MUST emit a structured audit event per egress connection through the platform audit adapter.
- REQ-ADR-0003-05: Egress enforcement MUST be identical on EKS, AKS, and kind with no CNI dependency.
- REQ-ADR-0003-06: Adding/removing a destination MUST be a declarative `EgressTarget`/CapabilitySet change via Git → ArgoCD, not a hand-edit of Envoy config.
- REQ-ADR-0003-07: Cross-tenant A2A MUST be blocked at Envoy even if gateway authorization fails (defense-in-depth).

## 7. Non-Functional Requirements
- Security/tenancy: network-layer defense-in-depth backstops gateway authorization (§6.9); FQDN allowlist is per-agent-class.
- Observability: per-connection audit under `platform.audit.*`; Envoy log level runtime-toggleable (ADR 0035).
- Portability: no CNI dependency; identical runbooks across substrates (ADR 0026).
- Versioning: Envoy pinned; `EgressTarget` CRD versioned per ADR 0030 (owner B13).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1 deliverable set is owned by A6 (Envoy install) and B13 (`EgressTarget` reconcile); conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0003-01: An agent attempting outbound HTTP to a non-allowlisted FQDN bypassing LiteLLM is blocked at Envoy. (REQ-01, REQ-02)
- AC-ADR-0003-02: An `EgressTarget` not in the agent's CapabilitySet is unreachable; adding it via Git → ArgoCD makes it reachable. (REQ-02, REQ-06)
- AC-ADR-0003-03: An egress connection produces an OPA query and a deny is enforced. (REQ-03)
- AC-ADR-0003-04: Every egress connection produces exactly one audit event via the audit adapter; no direct OpenSearch write. (REQ-04)
- AC-ADR-0003-05: The same egress test suite passes on kind and on an EKS cluster with a different CNI. (REQ-05)
- AC-ADR-0003-06: A hand-edited Envoy config drift is reverted by ArgoCD; only `EgressTarget`/CapabilitySet changes take effect. (REQ-06)
- AC-ADR-0003-07: With gateway authorization stubbed to allow, a cross-tenant A2A call is still blocked at Envoy. (REQ-07)

## 10. Risks & Open Questions
- New Envoy operational surface to own/upgrade (blast radius: med) — accepted per ADR 0003 consequences.
- OpenAI↔A2A and per-class config interaction with overlays (low) — overlay semantics owned by ADR 0032; settle in A6/B13 plans.
- Open question: does the OPA egress hook ship with A6 or consume the A7 bundle directly? (low) — ordering note for A6/A7 plans.

## 11. References
- ADR 0003 (`adr/0003-envoy-egress-proxy.md`) — the decision.
- Enforcing components: A6 (agent-sandbox + Envoy egress, owner), B13 (`EgressTarget` reconcile), A7 (OPA decision), A18 (audit adapter), B16 (egress Rego), A9 (Headlamp `EgressTarget` editor).
- architecture-overview.md §6.1, §6.6, §6.8, §6.9; architecture-backlog.md §2.3, §6, §7.
- Related: ADR 0002, 0013, 0026, 0032, 0033, 0034, 0035, 0039.

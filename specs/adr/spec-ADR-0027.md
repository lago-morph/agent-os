# SPEC ADR-0027 — Threat-model scope: v1.0 stance and B22 prerequisite [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [B22] · downstream: [B16, B3, A7, A6, A18, A20] · adrs: [0027] · views: [6.6]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0027 is a settled decision: for v1.0 the platform optimizes its built-in controls against **unintentional access excess** as the primary modeled threat, and the **adversarial threat model** is component **B22** — a required pre-implementation design specification that ships before the first wave of A/B component implementation lands and feeds forward into every component's deliverable list, OPA policy targets, mandatory test cases, and mandatory dashboard signals. This SPEC states what honoring that decision obliges: B22 exists as a fan-out input before component security hardening; every component completes a security review against B22's standards before being "complete"; B22 covers the §6.6 minimum adversary/asset/boundary/attack set; the existing defense-in-depth chain (ADR 0002/0003/0018) is refined by B22, not replaced. It does not re-argue the v1.0 scope stance.

The problem the decision solves: the existing defense-in-depth model is designed primarily against the everyday-mistake class; a dedicated adversarial threat model is a different artifact whose **output is the set of security standards every component must meet**. That analysis must exist before components harden around it, or each invents its own threat assumptions and the controls cannot be validated end-to-end.

## 2. Scope

### 2.1 In scope
- The obligation that B22 is a **required pre-implementation** design spec, published as a fan-out input before the first A/B implementation wave.
- The obligation that B22's output sets per-component security standards: acceptance criteria, mandatory test cases, mandatory dashboard signals, OPA policy targets.
- The obligation that B22 updates the per-component deliverable lists (Workstream A) and the OPA policy library targets (B16).
- The obligation that each component completes a **security review against B22's standards** before its implementation is considered complete.
- The obligation that B22 covers at minimum the §6.6 adversary classes, asset inventory, trust boundaries, and attack patterns.

### 2.2 Out of scope (and where it lives instead)
- The threat-model design content itself — component **B22** SPEC (this ADR makes it a prerequisite; B22 authors it).
- Per-component security checklists / reviews — each component SPEC §8 + Workstream **F** security review (F4).
- OPA policy library content realizing B22's targets — components **B3** (framework) / **B16** (content).
- Continuous runtime posture (image re-scanning, admission signing/verification, runtime drift detection) — out of v1.0 (future-enhancements §2).
- The defense-in-depth layers themselves — ADR 0002 (OPA/Gatekeeper), ADR 0003 (Envoy egress), ADR 0018 (RBAC-floor/OPA-restrictor).

## 3. Context & Dependencies

Upstream consumed: **B22** (security threat model design specification) is the artifact this ADR makes a prerequisite; it is a Workstream-B, W0 piece per the CSV.
Downstream consumers: **B16**/**B3** (OPA policy library content/framework) absorb B22's policy targets; **A7** (OPA/Gatekeeper), **A6** (agent-sandbox + Envoy egress) harden against B22's attack patterns; **A18** (audit) emits the signals B22's per-attack mapping requires; **A20** (policy simulator) validates against B22's targets.

ADR decisions honored:
- **ADR 0027** (this) — unintentional-access-excess is the v1.0 primary modeled threat; B22 is a required pre-implementation fan-out input setting per-component security standards.
- **ADR 0002** — OPA + Gatekeeper layer that B22 refines/extends.
- **ADR 0003** — Envoy egress FQDN-allowlist layer that B22 refines/extends.
- **ADR 0018** — RBAC-floor / OPA-restrictor model that B22 builds on.
- **ADR 0034** — audit emission is the substrate for B22's mandatory observability signals (per-attack detection mapping).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — ADR 0027 introduces no CRD/XRD. It governs a design artifact (B22) and a process gate (security review before complete). It constrains the targets of existing surfaces — `CapabilitySet` scoping, OPA policy refs, `EgressTarget` allowlists, `Sandbox` runtime isolation — without adding a new kind.

### 4.2 APIs / SDK surfaces
N/A — no API/SDK surface introduced. B22's output is a standards document consumed at design time, not a callable surface.

### 4.3 CloudEvents emitted / consumed
- B22's per-attack mapping designates which signals the observability stack must surface; security events distinct from audit ride `platform.security.*` (sandbox-escape signal, repeated authn failures, policy-bypass attempts), and audit rides `platform.audit.*` (ADR 0031, §2). Concrete per-event-type names are B12/component design — `[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts
N/A — no data schema or connection secret introduced. B22's outputs are alert-rule/dashboard-panel specifications layered over existing audit/observability data.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: B22 is a custom design specification, not software; it refines the existing upstream-backed defense-in-depth chain — **OPA/Gatekeeper** (A7), **Envoy** egress (A6), **gVisor/Kata** sandbox isolation — rather than introducing new tooling.)

## 6. Functional Requirements
- REQ-ADR-0027-01: v1.0 built-in controls MUST be optimized against unintentional access excess as the primary modeled threat.
- REQ-ADR-0027-02: B22 MUST ship as a design specification BEFORE the first wave of A/B component implementation lands (pre-implementation prerequisite, not code).
- REQ-ADR-0027-03: B22's output MUST set per-component security standards — acceptance criteria, mandatory test cases, mandatory dashboard signals, and OPA policy targets — and MUST update the Workstream-A deliverable lists and the B16 OPA policy targets.
- REQ-ADR-0027-04: Each component implementation MUST complete a security review against B22's standards before it is considered complete; components beginning before B22's first wave publishes MUST wait on B22 to close their security checklist.
- REQ-ADR-0027-05: B22 MUST cover at minimum the §6.6 adversary classes, asset inventory, trust boundaries, and attack patterns: capability escape, audit tampering, OPA bypass, sandbox escape, A2A lateral movement, exfiltration via approved tools, prompt-injection secret extraction, and budget/capability-exhaustion DoS.
- REQ-ADR-0027-06: B22 MUST refine and extend the ADR 0002/0003/0018 defense-in-depth layers rather than replace them.
- REQ-ADR-0027-07: B22's per-attack mapping (controls applied, residual risk, detecting observability) MUST be the canonical source for the alert rules and dashboard panels the observability stack surfaces.

## 7. Non-Functional Requirements
- Security: continuous runtime posture (re-scanning running images, admission signing/verification, runtime drift) is explicitly out of v1.0 scope (future-enhancements §2); B22 may add gaps to that backlog.
- Process: because B22 is a fan-out input (not a sequenced predecessor), its publication unblocks security sign-off across both workstreams simultaneously; delays directly delay every dependent component's security-complete state.
- Posture evolution: the v1.0 stance is a scope decision, not permanent; as adversarial controls mature the primary modeled threat shifts and the §6.6 narrative updates.
- Observability (§6.5): B22-mandated signals must be surfaced through the standard observability stack, not a parallel pipeline.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set — and specifically the security-review item — applies to every enforcing component, with B22 supplying the standard each is reviewed against.

## 9. Acceptance Criteria
- AC-ADR-0027-01: Honored when v1.0 built-in controls (CapabilitySet scoping, OPA restriction, egress allowlist, sandbox isolation, audit, RBAC-floor/OPA-restrictor) are documented as optimized against unintentional access excess. (REQ-01)
- AC-ADR-0027-02: Honored when B22 is published before the first A/B implementation wave merges. (REQ-02)
- AC-ADR-0027-03: Honored when component deliverable lists and B16 policy targets reflect B22's standards (ACs, mandatory tests, dashboard signals, OPA targets). (REQ-03)
- AC-ADR-0027-04: Honored when no component is marked implementation-complete without a passing security review against B22's standards. (REQ-04)
- AC-ADR-0027-05: Honored when B22 covers all eight §6.6 attack patterns plus adversary classes, assets, and trust boundaries. (REQ-05)
- AC-ADR-0027-06: Honored when B22's controls map onto (extend, not replace) the ADR 0002/0003/0018 layers. (REQ-06)
- AC-ADR-0027-07: Honored when the observability stack's security alert rules / dashboard panels trace back to B22's per-attack mapping. (REQ-07)

## 10. Risks & Open Questions
- OQ-1 (med): The `platform_roles` catalog and exact OPA policy targets are partly design-time; B22's targets sharpen but do not fully enumerate them until B16 content lands. `[PROPOSED]`
- R-1 (high): B22 is a single fan-out dependency for security sign-off across both workstreams; a delay in B22 delays the security-complete state of every dependent component — blast radius is platform-wide.
- R-2 (low): The v1.0 stance under-models adversarial actors by design; mitigated by it being an explicit, revisable scope decision with continuous-security gaps tracked in future-enhancements §2.

## 11. References
- ADR 0027 (`adr/0027-threat-model-scope-and-b22.md`) — the decision enforced here.
- architecture-overview.md §6.6 (security and policy architecture, threat model v1.0 stance), §14.2 (Workstream B, B22).
- architecture-backlog.md §4; future-enhancements.md §2 (continuous container security).
- Enforcing/related components: B22 (threat model), B16/B3 (OPA policy library), A7 (OPA/Gatekeeper), A6 (sandbox + Envoy egress), A18 (audit), A20 (policy simulator).
- ADR 0002 (OPA/Gatekeeper), 0003 (Envoy egress), 0018 (RBAC-floor/OPA-restrictor), 0034 (audit).

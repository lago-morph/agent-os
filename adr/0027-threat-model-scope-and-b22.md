# ADR 0027: Threat-model scope — v1.0 stance and B22 prerequisite

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Two distinct security risk classes shape this platform.

**Unintentional access excess** is the everyday-mistake class: misconfigured CapabilitySets, careless OPA policies, accidental over-broad RBAC grants, prompt-injection-driven actions that wander outside intended scope, virtual keys issued with too much reach. The actor here is well-meaning but the configuration or the agent's behavior exceeds what was authorized.

**Adversarial actors** are a different problem: malicious agent authors, compromised tenants, compromised admins, compromised LLM or MCP providers, supply-chain compromise of skill artifacts or container images, sandbox-escape attempts, audit tampering, A2A-based lateral movement, exfiltration via legitimately-approved tools. The actor here is hostile and is actively probing for weaknesses in the controls themselves.

The existing defense-in-depth model — admission-time CapabilitySet scoping, OPA runtime restriction at the gateway, FQDN allowlist at the egress proxy, gVisor / Kata kernel isolation, complete audit emission, RBAC-as-floor / OPA-as-restrictor — is designed primarily against the everyday-mistake class. A dedicated adversarial threat model is a different artifact: a structured analysis of adversary classes, assets, trust boundaries, and attack patterns whose **output is a set of security standards every component must meet**. That analysis must exist before component implementation hardens around it; otherwise each component invents its own threat assumptions and the controls cannot be validated end-to-end.

## Decision

For v1.0 the platform optimizes its built-in controls against **unintentional access excess** as the primary modeled threat. The **adversarial threat model** is component **B22** in Workstream B and is a **required pre-implementation design specification**: it ships before the first wave of A and B component implementation lands, and its output feeds forward into every component's deliverable list, OPA policy targets, mandatory test cases, and mandatory dashboard signals. B22 is design, not code.

## Consequences

- B22's output **sets the security standards for every component** — acceptance criteria, mandatory test cases, mandatory dashboard signals, OPA policy targets — and updates the per-component deliverable lists in Workstream A and the OPA policy library targets in B16.
- Each component implementation must complete a **security review against B22's standards before its implementation is considered complete**; components that begin before B22's first wave publishes wait on the design to close out their security checklist.
- The defense-in-depth chain established by ADR 0002 (OPA + Gatekeeper), ADR 0003 (Envoy egress), and ADR 0018 (RBAC-floor / OPA-restrictor) remains the architectural commitment; B22 refines and extends those layers rather than replacing them.
- B22 must cover at minimum the adversary classes, asset inventory, trust boundaries, and attack patterns listed in architecture-overview.md § 6.6: capability escape, audit tampering, OPA bypass, sandbox escape, A2A lateral movement, exfiltration via approved tools, prompt-injection secret extraction, and budget / capability exhaustion DoS.
- Threats that depend on continuous runtime posture — re-scanning running images, signing and verification at admission, runtime drift detection on running workloads — are explicitly out of v1.0 scope and tracked in future-enhancements.md § 2 (continuous container security); B22 may identify additional gaps that get added to that backlog.
- The v1.0 stance is a **scope decision, not a permanent posture**: as adversarial controls mature (image signing, continuous re-scanning, runtime verification), the primary modeled threat will shift and the defense-in-depth narrative in § 6.6 will be updated accordingly.
- Because B22 is a fan-out input rather than a sequenced predecessor, its publication unblocks security sign-off across both workstreams simultaneously; delays in B22 directly delay the security-complete state of every dependent component.
- B22's per-attack mapping (which controls apply, what residual risk remains, what observability detects exploitation) becomes the canonical source for the alert rules and dashboard panels that the observability stack must surface, tying the threat model directly into the operability work in Workstream A.

## References

- architecture-overview.md § 6.6 (security and policy architecture, threat model — v1.0 stance)
- architecture-overview.md § 14.2 (Workstream B, component B22)
- architecture-backlog.md § 4 (topics that need further design before implementation)
- architecture-backlog.md § 7 item 27 (this ADR's backlog entry)
- future-enhancements.md § 2 (continuous container security)
- ADR 0002 (OPA + Gatekeeper)
- ADR 0003 (Envoy egress proxy)
- ADR 0018 (RBAC-floor / OPA-restrictor)

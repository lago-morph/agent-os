# SPEC ADR-0038 — Policy simulators for OPA and RBAC `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [A7; A1; A6] · downstream: [A20; A22; A14] · adrs: [0038] · views: [6.6]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0038 is a settled decision: the platform ships a **policy simulator** surface that, given a
synthetic request (subject = Keycloak claims, action, resource, context), returns the decision at
**every applicable enforcement layer** (admission, runtime, egress, RBAC, approval elevation) with the
matching rule cited. It is exposed as a **Headlamp simulator panel** and a **HolmesGPT skill** over one
**aggregator service**. This SPEC states what honoring the decision requires of each policy-decision
component and the two consumers — it does not re-argue the simulator's value.

The problem the decision solves: the platform stacks multiple enforcement layers on every privileged
action (Gatekeeper admission, LiteLLM runtime, Envoy egress, RBAC floor, OPA elevation per ADR 0017),
which has a predictable failure mode — a rule in one layer interacts unexpectedly with another, and the
failure is hard to debug after the fact. The simulator answers "what would happen for this exact
request" before an incident, with the layer and rule cited.

## 2. Scope

### 2.1 In scope
- The simulator semantics: synthetic request in → per-layer decision out, with the matching rule cited at each applicable layer.
- The two exposures over one aggregator API: a Headlamp simulator panel (interactive) and a HolmesGPT skill (diagnostic, no human in the loop).
- The per-component obligation: every component hosting a policy decision point MUST expose a structured dry-run.
- RBAC reported as a separate layer (not collapsed into OPA), preserving the RBAC-floor / OPA-restrictor contract.
- Approval elevation reported as resolved required level WITHOUT creating an `Approval`.
- Simulator runs audited under `platform.policy.*` but never entering the enforcement path.

### 2.2 Out of scope (and where it lives instead)
- Formal verification / policy-space enumeration / cross-layer conflict analysis / drift detection — explicitly out of v1.0 scope (future enhancements).
- The security review ADR 0027 requires — the simulator does not replace it.
- The Headlamp editor framework the panel embeds in — owned by ADR 0039 / components A9, A22.
- The OPA policy content the layers evaluate — owned by B16 over B3.
- The aggregator service and per-component dry-run builds — owned by component A20 (and each decision-point component).

## 3. Context & Dependencies

Upstream consumed: A7 (OPA/Gatekeeper — `data.allow` evaluation + Gatekeeper audit-mode dry-run); A1
(LiteLLM — runtime decision dry-run via callbacks / dynamic registration); A6 (agent-sandbox + Envoy —
egress ext-authz dry-run path). Downstream consumers: A20 (the aggregator service), A22 (Headlamp
simulator panel + inline use in editors per ADR 0039), A14 (HolmesGPT skill).

ADR decisions honored:
- **ADR 0038** (this): one aggregator, two consumers; per-layer decision with rule citation; point-query "what would happen" only.
- **ADR 0002**: Gatekeeper/OPA is the admission + runtime engine the simulator dry-runs.
- **ADR 0003**: the Envoy egress proxy exposes an ext-authz dry-run path.
- **ADR 0018**: RBAC is reported as a separate layer (the ceiling/floor), OPA as the restrictor; citations name which layer denied.
- **ADR 0017**: for approval-bearing actions the simulator reports the resolved required level (post-OPA-elevation) without creating an `Approval`.
- **ADR 0012**: HolmesGPT consumes the same aggregator as a skill, so its explanation is reproducible in Headlamp.
- **ADR 0027**: the simulator does not replace the required security review; "OPA bypass" / "capability escape" are the attack patterns it helps reason about.
- **ADR 0039**: Headlamp surfaces the simulator inline alongside the policy-editing surfaces.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — this ADR imposes no CRD/XRD. The simulator dry-runs existing policy decision points; it never
creates an `Approval` (ADR 0017) and never mutates state.

### 4.2 APIs / SDK surfaces
- A single **aggregator service** API: takes a synthetic request (subject = Keycloak claims, action,
  resource, context); fans it out to layer-specific dry-run paths — OPA-backed components evaluate
  `data.allow` against the supplied input bundle; Gatekeeper exposes its existing audit-mode evaluation;
  Envoy exposes an ext-authz dry-run path; the approval system evaluates elevation without creating an
  `Approval` — and composes the verdicts with the matching rule cited per layer.
- Two consumers of the same API: the **Headlamp simulator panel** and the **HolmesGPT skill**, with
  identical semantics so a HolmesGPT explanation is reproducible in Headlamp.
- Concrete aggregator request/response signatures are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Simulator runs are audited under **`platform.policy.*`** but never enter the enforcement path — a
  simulator call MUST NOT cause a real admission, runtime, or egress decision. Concrete event names
  deferred to B12.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface. The simulator's input is a synthetic request bundle (claims /
action / resource / context); its output is a per-layer verdict set with rule citations.

## 5. OSS-vs-Custom Decision
N/A — ADR. The realizing component A20 builds the custom aggregator (thin fan-out over existing
OPA/Gatekeeper/Envoy/approval dry-run paths); the Headlamp panel (A22) is custom on the A9 framework;
the HolmesGPT skill is custom. Those calls live in the A20 / A22 / A14 SPECs.

## 6. Functional Requirements

- REQ-ADR-0038-01: The simulator MUST accept a synthetic request (subject = Keycloak claims, action, resource, context) and return the decision at every applicable enforcement layer (admission, runtime, egress, RBAC, approval elevation) with the matching rule cited.
- REQ-ADR-0038-02: Every component hosting a policy decision point (Gatekeeper, LiteLLM callbacks + dynamic registration, Envoy egress, the approval system, Headlamp plugin actions, agent-platform CLI gates) MUST expose a structured dry-run alongside its audit emission and OPA hookup.
- REQ-ADR-0038-03: The simulator MUST be exposed both as a Headlamp panel and as a HolmesGPT skill, both calling the same aggregator API with identical semantics.
- REQ-ADR-0038-04: RBAC MUST be reported as a separate layer (not collapsed into OPA), preserving the RBAC-floor / OPA-restrictor contract (ADR 0018); citations MUST name which layer denied.
- REQ-ADR-0038-05: For approval-bearing actions the simulator MUST report the resolved required level (post-OPA-elevation per ADR 0017) WITHOUT creating an `Approval`.
- REQ-ADR-0038-06: A simulator call MUST NOT enter the enforcement path — it MUST NEVER cause a real admission, runtime, or egress decision.
- REQ-ADR-0038-07: Simulator runs MUST be audited under the `platform.policy.*` taxonomy.
- REQ-ADR-0038-08: The simulator MUST be a point-query "what would happen for this case" surface only; it MUST NOT claim to be formal verification, enumerate the policy space, or replace the ADR 0027 security review.
- REQ-ADR-0038-09: A candidate policy bundle's cryptographic signature MUST verify (ADR 0002, #25) before its simulation result is accepted by the promotion gate; an unsigned or signature-invalid bundle MUST NOT be promotable.
- REQ-ADR-0038-10: The simulate→audit-mode→enforce sequence MUST be staged: after simulation a new/changed bundle runs in audit mode first and flips to enforce only after review, via the Kargo-promoted transition (ADR 0002 #26, ADR 0040).

## 7. Non-Functional Requirements
- Security (§6.6): the simulator reasons about "OPA bypass" / "capability escape" (ADR 0027) before an incident; it is read-only and side-effect-free by REQ-06.
- Reproducibility: one aggregator, two consumers ⇒ a HolmesGPT explanation is reproducible in Headlamp.
- Multi-tenancy (§6.9): the synthetic subject carries Keycloak claims, so tenant-scoped policy is exercised faithfully.
- Observability: simulator runs are auditable (`platform.policy.*`) without polluting enforcement audit.
- Scope discipline: point queries only; cross-layer conflict analysis and drift detection are future enhancements.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The per-component structured-dry-run obligation (REQ-02) becomes a deliverable embedded in
each decision-point component's own §14.1 set; the aggregator (A20), the Headlamp panel (A22), and the
HolmesGPT skill (A14) carry their deliverables in their own SPECs. Conformance is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0038-01: Honored when a synthetic request returns a per-layer verdict set (admission, runtime, egress, RBAC, approval) each with the matching rule cited. (→ REQ-01)
- AC-ADR-0038-02: Honored when a component hosting a policy decision point is shown to expose a structured dry-run, and a component lacking one fails the deliverable check. (→ REQ-02)
- AC-ADR-0038-03: Honored when the Headlamp panel and the HolmesGPT skill return identical verdicts for the same synthetic request against the same policy bundle. (→ REQ-03)
- AC-ADR-0038-04: Honored when a denial is shown attributed to a named layer (e.g. RBAC vs OPA), with RBAC reported separately from OPA. (→ REQ-04)
- AC-ADR-0038-05: Honored when an approval-bearing action returns its resolved required level and no `Approval` resource is created by the simulation. (→ REQ-05)
- AC-ADR-0038-06: Honored when a simulator run is shown to produce no real admission/runtime/egress decision (side-effect-free). (→ REQ-06)
- AC-ADR-0038-07: Honored when every simulator run appears as an audit event under `platform.policy.*`. (→ REQ-07)
- AC-ADR-0038-08: Honored when a bundle with no/invalid signature is rejected by the promotion gate even if its simulation passed, and a validly signed simulated bundle is promotable. (→ REQ-09)
- AC-ADR-0038-09: Honored when a newly simulated bundle is shown entering audit mode first and flipping to enforce only after review via the Kargo-promoted transition. (→ REQ-10)

## 10. Risks & Open Questions
- (med) The simulator's accuracy depends on every decision-point component exposing a faithful dry-run (REQ-02); a stale or divergent dry-run path silently misleads — per-component dry-run fidelity is an A20-coordinated concern.
- (low) Point-query scope means cross-layer conflicts that only manifest across many requests are not caught; cross-layer conflict analysis is an explicit future enhancement.
- (low) `[PROPOSED]` — the aggregator request/response signatures are not specified in source; flagged for A20 design.

## 11. References
- ADR 0038 (this decision). Enforcing/realizing components: A20 (aggregator service + each decision-point's dry-run), A22 (Headlamp simulator panel), A14 (HolmesGPT skill), A7 (Gatekeeper/OPA dry-run), A1 (LiteLLM runtime dry-run), A6 (Envoy ext-authz dry-run), B19 (approval elevation eval).
- architecture-overview.md §6.6 (security/policy), §6.10 (HolmesGPT). ADR 0002, 0003, 0012, 0017, 0018, 0027, 0039.

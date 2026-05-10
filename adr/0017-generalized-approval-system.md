# ADR 0017: Generalized approval system

## Status

Accepted

## Context

Several Platform Agent flows require human review before action: Coach Component–suggested skill changes and HolmesGPT–suggested remediations are the initial v1.0 cases, and more are anticipated. An earlier stance (see backlog §2.18, §2.22) was to wire each flow bespoke and only refactor when a third use case appeared. During this design pass that stance was reversed: building two parallel approval paths and then rebuilding on the third use is more expensive than generalizing once, since each bespoke path drags its own UI, audit, and policy wiring.

The platform already has the building blocks. Argo Workflows is in the stack (ADR 0021 candidate; see overview §6) for orchestration, including suspend/resume. OPA is the runtime policy engine (ADR 0002) and the platform-wide rule is RBAC-as-floor / OPA-as-restrictor (ADR 0018) — OPA can only restrict or raise required levels, never grant. Headlamp is the admin/developer UI surface and already hosts cross-component plugins. Reusing these for approvals avoids new infrastructure.

## Decision

Adopt one approval mechanism for all human-in-the-loop reviews of Platform Agent actions:

- **Approval CRD.** A request for action carries the requesting agent, the action attributes (full metadata sufficient to act on the proposal independently if approved), a **default approval level** (e.g., `operator`), and links to evidence (traces, audit, RAG references).
- **OPA elevation only.** When an `Approval` is created, OPA is consulted and may **raise the required level** (e.g., elevate `operator` → `org-admin` for sensitive data). OPA cannot lower the bar. This is the same RBAC-floor / OPA-restrictor pattern applied to approvals.
- **Argo Workflow mechanics.** An Argo Workflow underlies each `Approval` and drives the suspend/resume, approve/reject transitions, and basic timeout behavior. Workflow templates live in Git.
- **Headlamp plugin.** A cross-component Headlamp plugin (B5-owned) presents the approval queue to anyone with the resolved permission level, surfaces the evidence, and records the decision.
- **Reuse, don't rebuild.** New approval needs are added by emitting an `Approval` from the requesting agent. No parallel approval path is allowed.

Initial use cases in v1.0 are Coach skill changes and HolmesGPT remediation (ADR 0012). Both are wired through the same CRD, the same workflow shape, and the same UI.

Several specifics are deferred to design (backlog §1.5): the exact `Approval` CRD schema (which attributes are first-class vs generic metadata), timeout and escalation behavior beyond basic ask-and-decide, whether the Argo Workflow template is shared across approval types or per-type, and the audit detail level for approve vs reject decisions.

## Consequences

Positive:

- One mechanism, one UI, one audit shape for every approval. New use cases are cheap to add.
- OPA-elevation-only preserves the platform-wide invariant (overview §7.5; backlog §6) — approval policy doesn't introduce a new authorization model.
- Argo Workflows' existing suspend/resume and Headlamp's existing plugin surface mean no new long-lived infrastructure.
- Decisions emit CloudEvents like every other audit-relevant action, so approvals participate in the standard observability path.

Negative / accepted trade-offs:

- The mechanism is intentionally lean. M-of-N approvers, delegation, complex routing, and escalation timers are explicitly out of scope for v1.0. Triggers for adding these features are tracked in backlog §3.3 — they get added when concrete use cases reveal gaps, and they get added inside the existing mechanism (Argo Workflow behaviors, OPA rules, Headlamp UI), not by forking.
- Generalizing now costs slightly more upfront than wiring two bespoke flows, but avoids the rebuild-on-third-use cost the earlier stance accepted.
- Schema, timeout/escalation behavior, template sharing, and audit detail are not yet pinned down — implementation cannot start until the design specification (backlog §1.5) lands.

## Alternatives considered

- **Two bespoke flows now, generalize on third use case.** Originally chosen, reversed during this design pass (backlog §2.18, §2.22). Rebuilding two production flows into a generic one is more expensive than generalizing the first time.
- **OPA grants approval permissions directly.** Rejected: violates the RBAC-floor / OPA-restrictor invariant (ADR 0018). OPA may only restrict or elevate; it never grants.
- **Custom approval controller instead of Argo Workflows.** Rejected: Argo Workflows already provides suspend/resume and is in the stack. A bespoke controller would duplicate it.
- **Rich approval features in v1.0** (M-of-N, delegation, escalation timers, complex routing). Deferred to evolution path §3.3 — added when a concrete use case demands them.

## Related

- ADR 0002 — OPA + Gatekeeper as the policy engine (provides the elevation decision point).
- ADR 0012 — HolmesGPT as a Platform Agent (one of the two initial approval consumers; autonomous remediations are gated through this mechanism).
- ADR 0018 — RBAC-as-floor / OPA-as-restrictor (the platform-wide pattern that approval elevation instantiates).
- Architecture overview §7.5 — Approvals — generalized.
- Backlog §1.5 — deferred design details (schema, timeouts, template sharing, audit detail).
- Backlog §2.18, §2.22 — reversed stance from "do not generalize" to "generalize once now."
- Backlog §3.3 — triggers for richer approval features.
- Backlog §6 — invariant: approvals flow through one mechanism.

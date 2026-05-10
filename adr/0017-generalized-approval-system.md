# ADR 0017: Generalized approval system via Approval CRD + Argo Workflows + OPA elevation

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Platform Agents periodically propose actions that require human review before proceeding — initially Coach Component–suggested skill changes and HolmesGPT–suggested remediation actions. An earlier stance in this architecture explicitly said "do not generalize": build the two specific flows, refactor only when a third use case appeared. During this design pass we reversed that stance. Two parallel approval implementations already create governance and UX divergence, and we expect more approval surfaces (capability registration, budget exceptions, dynamic MCP registration) to emerge soon enough that one mechanism is cheaper than three. The generalized mechanism is intentionally lean so the reversal does not pull in v1.0 scope we cannot afford.

## Decision

The platform ships one approval mechanism with four parts. (1) An `Approval` CRD represents the request — requesting agent, action attributes and metadata sufficient to act on the proposal independently, default approval level, and evidence references. (2) Argo Workflows runs underneath each `Approval`, managing suspend/resume mechanics, timeout, and decision-event emission. (3) OPA is consulted on creation and may **only elevate** the required approval level — never lower it; RBAC remains the floor. (4) A Headlamp plugin provides the approval queue UI, filtered to anyone holding the resolved level. Initial use cases remain Coach skill changes and HolmesGPT remediation; new approval needs are added by writing an `Approval` from the requesting agent, not by building parallel machinery.

## Alternatives considered

- **Rebuild on need (refactor when third use case appears)** — the original "do not generalize" stance from backlog 2.22. Rejected because we already have two independent flows in flight and a generic CRD + Argo template + Headlamp plugin is a small enough increment to build once. Refactoring two live, audit-bearing flows later costs more than generalizing now, and parallel mechanisms invite divergence in audit, OPA hooks, and approver UX that is hard to undo.

## Consequences

- Establishes the architectural invariant (backlog §6) that **approvals flow through one mechanism** — `Approval` CRD + Argo Workflows + Headlamp plugin. New approval types reuse it rather than building parallel ones.
- Sub-decisions are deferred to design (backlog §1.5): exact CRD schema (first-class fields vs generic metadata blob), timeout/escalation behavior, whether the Argo template is shared across approval types or per-type, and audit detail level for approve vs reject.
- Commits the platform to "request a level, OPA may elevate, anyone with that level decides." M-of-N approvers, delegation, escalation timers, and complex routing are explicitly out of v1.0 scope (future-enhancements.md §4).
- Revisit triggers for richer approval features (backlog §3.3): a concrete use case that cannot be expressed as OPA-elevation-only — e.g., multi-party sign-off, time-bounded escalation, or routing rules that depend on requester history. Added where the mechanism already lives (Argo behaviors, OPA rules, Headlamp UI), not by rebuilding.
- Creates ongoing reuse pressure: any future approval surface (capability registration, budget exceptions, dynamic MCP registration, virtual key issuance overrides) must justify why it would not flow through this mechanism. The default is "use the `Approval` CRD."
- Decisions emit a CloudEvent on the `platform.approval.*` taxonomy back to the requesting agent and an audit event through the platform audit adapter (ADR 0034) — Postgres + S3 are the system of record (Postgres only on kind), with OpenSearch as advisory fanout — so approval traceability is uniform regardless of action type.
- Depends on Argo Workflows (ADR 0003-equivalent install — see §A3 in architecture-overview.md) and OPA (ADR 0002); couples the OPA policy library (B16) to a new policy surface (`approval.elevation`).
- The HolmesGPT three-state permission model (ADR 0012) has an "upon-approval" state whose entire mechanic is creating an `Approval` CRD here — a third concrete consumer alongside Coach skill changes and HolmesGPT remediation, reinforcing the "use one mechanism" invariant.
- Admins can preview the approval impact of an OPA policy change before submitting it via the Headlamp policy simulator (ADR 0038), which dry-runs `approval.elevation` rules against recent and synthetic `Approval` requests.

## References

- [architecture-overview.md § 7.5](../architecture-overview.md#75-approvals--generalized)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.5](../architecture-backlog.md#15-approval-crd-details), [2.18](../architecture-backlog.md#218-generalized-approval-system-defer-vs-do-now), [2.22](../architecture-backlog.md#222-approval-mechanism-rebuild-on-need-vs-generalize-once), [3.3](../architecture-backlog.md#33-more-approval-flow-features), [6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [future-enhancements.md § 4](../future-enhancements.md#4-other-deferred-items-cross-references) (M-of-N approvers)
- [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC-floor / OPA-restrictor)
- [ADR 0012](./0012-holmesgpt-first-class-agent.md) (HolmesGPT three-state permission model — "upon-approval" consumer)
- [ADR 0038](./0038-policy-simulators.md) (Headlamp OPA policy simulator)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline — Postgres + S3 system of record, OpenSearch advisory fanout)

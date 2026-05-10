# ADR 0012: HolmesGPT as a first-class Platform Agent

## Status
Accepted

## Context
The platform needs a diagnostic and self-management capability from day one — something that can investigate failing pods, sustained latency, policy violations, and cluster-level issues, and that improves as the platform itself grows. HolmesGPT is the obvious fit, but the question is *how* it lives on the platform.

The shape of that question matters because the platform already commits to a specific runtime model for agents: every Platform Agent invocation traverses LiteLLM (every-call-through-LiteLLM), every agent runs in a sandbox, every action is OPA-gated, and every audit-relevant event is emitted to the audit index. If HolmesGPT runs outside that model — as an external SaaS, a sidecar tool, or a one-off operator — it becomes a special case that bypasses the controls the rest of the platform depends on.

Two further observations push toward keeping HolmesGPT inside the model:

- The architecture invariant (backlog §6) requires every component to contribute a HolmesGPT toolset, an OPA policy contribution, audit emission, observability, a Knative trigger flow design, tests, and tutorial / how-to documentation. That invariant only makes sense if HolmesGPT is a platform-internal artifact whose toolsets are co-evolved with each component.
- The AlertManager → HolmesGPT trigger flow (overview §5.5) routes diagnostic-relevant alerts as CloudEvents through Knative into HolmesGPT. This is the same eventing path other Platform Agents use; treating HolmesGPT specially would require a parallel path.

## Decision
HolmesGPT runs as a first-class **Platform Agent**, declared as an **Agent CRD** and administered by ARK like any other agent. It is subject to the same constraints:

- **Every-call-through-LiteLLM.** Model calls, A2A handoffs into HolmesGPT, and skill/RAG access all traverse LiteLLM.
- **OPA-gated.** Reads and writes are policy-decided. Broad read access is granted (OPA policies, audit, traces, metrics, Knowledge Base) so HolmesGPT can diagnose effectively, including reading the policies that gate it.
- **Sandbox.** Runs in a Sandbox like any other agent pod.
- **Per-component HolmesGPT toolset contributions.** Each component (operator, MCP server, callback, glue service) ships its own toolset definition that registers with HolmesGPT when the component is installed. HolmesGPT's diagnostic surface area grows monotonically as the platform matures.
- **AlertManager → HolmesGPT trigger flow.** AlertManager (and other monitoring sources) emit CloudEvents into the Knative broker; a Trigger filters for diagnostic-relevant alerts and dispatches to HolmesGPT.
- **Read-only with auto-PR for narrow safe changes.** HolmesGPT starts strictly read-only for execution: findings and remediation suggestions land as auto-PRs against config repositories or as Headlamp suggestion cards. Autonomous execution expands per ADR 0017 only when a remediation pattern is trusted enough to be gated behind OPA and executed without a human.
- **A2A interface.** Other Platform Agents can hand off troubleshooting questions; calls in are authenticated, traced, and audited like any other agent call.

The Knowledge Base (`platform-knowledge-base` RAGStore, ADR 0022) is referenced via HolmesGPT's CapabilitySet; it is not embedded in HolmesGPT.

## Consequences
**Positive.**
- HolmesGPT inherits the platform's governance for free: audit trail, observability, OPA, virtual keys, sandbox isolation, egress control. No special case to maintain.
- The "every component contributes a toolset" invariant becomes mechanically enforceable — the contribution is a tracked component deliverable.
- The same `Approval` mechanism (ADR 0017) handles HolmesGPT-suggested remediations as it handles Coach-suggested skill changes — one mechanism, two use cases.
- HolmesGPT is useful from very early in platform implementation, including for diagnosing the platform's own bring-up issues.

**Negative / accepted trade-offs.**
- HolmesGPT having read access to the policies that gate it creates a self-referential risk: it could in principle identify a way around its own constraints. We accept this — if such a path exists we want it found and fixed.
- Toolset contributions become a per-component obligation; components that skip them leave diagnostic blind spots. Mitigated by the standard component-deliverables checklist.
- Running HolmesGPT through LiteLLM and OPA on every call adds latency relative to a direct-to-cluster diagnostic tool. Acceptable at v1.0 scale.

## Alternatives considered
- **External / SaaS HolmesGPT.** Rejected: bypasses LiteLLM, OPA, audit, and sandbox; cannot use per-component toolsets registered as Kubernetes resources; does not fit the AlertManager → Knative trigger model.
- **One-off operator outside the agent runtime** (HolmesGPT as a bespoke controller with its own LLM and policy paths). Rejected: re-implements governance the agent runtime already provides, and parallel mechanisms drift.
- **Defer HolmesGPT until the platform is observable end-to-end.** Rejected: HolmesGPT is most useful precisely during bring-up, when observability is partial and diagnostic capacity is scarce.

## Related
- ADR 0001 — ARK as the agent operator (HolmesGPT runs as an Agent CRD reconciled by ARK).
- ADR 0002 — OPA + Gatekeeper as the policy engine (gates HolmesGPT reads and writes).
- ADR 0017 — Generalized approval flow (gates autonomous remediations as autonomy expands).
- ADR 0022 — Knowledge Base as a separate primitive (HolmesGPT references it via CapabilitySet, not built-in).
- Architecture overview §4 (high-level placement), §5 (HolmesGPT row), §6.4 (Knowledge Base), §6.10 (platform self-management), §14 (component deliverables).
- Architecture backlog §3.11 (autonomous remediation expansion trigger), §6 (every-component-contributes-a-toolset invariant), §7 (ADR candidate 12).

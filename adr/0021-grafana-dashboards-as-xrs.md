# ADR 0021: Dashboards as namespaced Crossplane-composed GrafanaDashboard XRs

## Status

Accepted

## Context

Grafana is the cross-stack pane of glass for the platform: metrics from Mimir,
logs from Loki, traces from Tempo (correlated with Langfuse via `trace_id` per
ADR 0015), and SLO views all surface through it. Dashboards exist for several
audiences — per-component (mostly vendor-provided with tweaks), integrated
operator views, developer-facing prompt/eval/cost views, test-framework trend
views, and dashboards Platform Agents publish about themselves (e.g. a Coach
"agent X performance trend", a HolmesGPT "incident pattern" view).

Without a declarative model, dashboards drift toward click-built one-offs in the
Grafana UI: not in Git, not reviewable, not promotable between environments,
and — most importantly — globally visible. That last point breaks tenancy. The
platform's tenancy model is namespace-scoped with RBAC as the floor and OPA as
the restrictor (ADR 0016, ADR 0018); a global Grafana folder of dashboards
sidesteps both.

The platform already runs Crossplane v2 Compositions for cloud-shaped
declarative resources (`AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`,
etc.) under Component B4. A Grafana dashboard is cloud-shaped in exactly this
sense: a spec rendered by a provider into an external system's API. It is not
an application-API reconciliation problem (which is what kopf is reserved for
per ADR 0006).

## Decision

All dashboards on the platform are first-class declarative resources, delivered
as **Crossplane v2 Compositions of a `GrafanaDashboard` XR**. The XR is
**namespace-scoped**. Visibility and edit rights are governed by Kubernetes
**RBAC + OPA access control**, consistent with the platform-wide
RBAC-as-floor / OPA-as-restrictor model. Dashboards reconcile via the standard
**GitOps reconcile** path (ArgoCD applies the XR; Crossplane composes it; the
Grafana provider — or equivalent reconciliation path — pushes it into Grafana).

This applies uniformly to per-component dashboards (shipped by Workstream A
components), integrated and developer-facing dashboards (Workstream D), test
framework dashboards, and any dashboard a Platform Agent publishes about
itself. The `GrafanaDashboard` Composition lives in Component B4 alongside the
other platform XRs.

This reinforces the architecture invariant: **all dashboards are namespaced and
access-controlled (Crossplane-composed `GrafanaDashboard` XRs).**

## Consequences

- One declarative GitOps path for dashboards, identical to every other platform
  resource. Same review, same promotion, same rollback.
- Per-tenant scoping for free: tenant A's dashboards live in tenant A's
  namespace and are invisible to tenant B unless explicitly published.
- Platform Agents can publish dashboards as part of their spec or as standalone
  XRs without inventing a new mechanism.
- Promotion across environments uses standard Crossplane composition selectors.
- Dashboard authors must produce a `GrafanaDashboard` spec (Jsonnet, Grafonnet,
  or rendered JSON wrapped in the XR) — there is no supported "edit in the
  Grafana UI and save" path. Click-built changes in Grafana are ephemeral and
  will be reconciled away.
- Component B4 owns and maintains the Composition and the Grafana
  provider/reconciliation path. A provider regression blocks dashboard updates
  platform-wide; this risk is accepted and is no different from the risk
  carried by any other Crossplane-managed resource.

## Alternatives considered

- **Store dashboard JSON in Git, apply via the Grafana sidecar provisioner.**
  Rejected: bypasses namespace tenancy and OPA entirely — the sidecar drops
  every dashboard into a global Grafana folder, breaking the namespacing
  invariant and the RBAC-as-floor / OPA-as-restrictor model.
- **Click-built dashboards in the Grafana UI, exported manually.** Rejected:
  not GitOps, not namespaced, not reviewable, not promotable, drifts
  immediately.
- **A bespoke kopf operator for dashboards.** Rejected: a Grafana dashboard is
  a cloud-shaped resource pushed into an external system's API, which is
  exactly what Crossplane Compositions are for. ADR 0006 reserves kopf for
  application-API reconciliation (LiteLLM admin); using it here would blur the
  controller-framework boundary the architecture deliberately maintains.

## Related

- ADR 0006 — Python kopf operator for LiteLLM (the kopf-vs-Crossplane boundary
  this decision sits on the Crossplane side of).
- ADR 0015 — Tempo + Langfuse correlated by `trace_id` (Grafana is the
  cross-stack pane of glass that surfaces this correlation).
- ADR 0016 — Multi-tenancy via namespaces with RBAC + OPA + network
  enforcement (the tenancy model dashboard namespacing inherits).
- ADR 0018 — RBAC-as-floor / OPA-as-restrictor enforcement model.
- Architecture overview §11 (Grafana dashboards), §14.4 (Workstream D),
  Component B4.
- Architecture backlog §6 (invariant: all dashboards are namespaced and
  access-controlled), §7 item 21.

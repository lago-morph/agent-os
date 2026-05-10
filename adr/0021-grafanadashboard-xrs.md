# ADR 0021: Dashboards as namespaced Crossplane-composed GrafanaDashboard XRs

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform ships many Grafana dashboards: per-component dashboards (mostly vendor-provided with tweaks), cross-cutting operator dashboards, developer-facing dashboards, and dashboards Platform Agents publish about themselves (e.g., a Coach-published "agent X performance trend" or a HolmesGPT-published "incident pattern" dashboard). Treating these as ad-hoc JSON imported into Grafana would diverge from the rest of the platform, which is GitOps-driven end to end and uses Kubernetes namespaces as the tenancy boundary (ADR 0016) governed by RBAC-as-floor / OPA-as-restrictor (ADR 0018). Without a CRD shape, dashboards would also escape Gatekeeper admission and the platform-wide visibility model. Crossplane v2 is already the chosen reconciliation path for cloud-shaped resources via Compositions, and the same model fits dashboards cleanly because a dashboard is a declarative resource composed of vendor pieces (Grafana folders, dashboard JSON, datasource bindings).

## Decision

All dashboards — per-component, integrated, developer-facing, and any dashboards Platform Agents publish — are delivered as Crossplane v2 Compositions of a `GrafanaDashboard` XR. The XR is **namespaced**, with `dashboardJson`, `folder`, and `visibility` fields, and visibility is controlled by Kubernetes RBAC plus OPA per the platform-wide RBAC-as-floor / OPA-as-restrictor model. The `GrafanaDashboard` Composition (and the Grafana provider or equivalent reconciliation path) is owned by Component B4 alongside `AgentEnvironment`, `MemoryStore`, and `SyntheticMCPServer`.

## Consequences

- Dashboards are GitOps-managed on the same ArgoCD reconciliation path as every other platform manifest; promotion across environments uses standard Crossplane composition selectors.
- Dashboard access is namespace-scoped by default — tenant A's dashboards are not visible to tenant B unless explicitly shared — and both Kubernetes RBAC and OPA policy decisions can gate read/write on `GrafanaDashboard` resources, consistent with ADR 0016 and ADR 0018.
- Gatekeeper admission applies to `GrafanaDashboard` XRs (per overview §6 admission scope), so dashboard creation can be policy-checked the same way other CRDs are.
- B4 ships the `GrafanaDashboard` XRD and Composition as part of its scope (architecture-backlog.md § 4); per-component dashboards are then authored as instances of this XR by each Workstream A component, and the integrated/developer dashboards by Workstream D.
- Platform Agents (Coach, HolmesGPT, others) can publish dashboards declaratively as part of their Agent CRD spec or as standalone resources, without bypassing platform governance.
- Crossplane stays scoped to cloud-shaped and composition-shaped resources; LiteLLM-specific reconciliation remains on the kopf operator path per ADR 0006. Dashboards qualify for Crossplane because they are composed declarative resources, not application-API state.
- SLO dashboards, error-budget tracking, and operations-metrics summaries are out of scope here and deferred to future enhancements (per overview §11); when they ship, they ship as additional `GrafanaDashboard` instances under the same model.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 11](../architecture-overview.md#11-grafana-dashboards), [§ 14.4](../architecture-overview.md#144-workstream-d--dashboards)
- [architecture-backlog.md](../architecture-backlog.md) [§ 4](../architecture-backlog.md#4-topics-that-need-further-design-before-implementation) (B4 Compositions), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0006](./0006-python-kopf-litellm-operator.md) (kopf vs Crossplane split), [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (namespace tenancy), [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC-floor / OPA-restrictor)

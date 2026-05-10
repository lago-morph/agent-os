# ADR 0040: Kargo as the v1.0 promotion fabric for GitOps environments

- Status: Accepted
- Date: 2026-05-10

## Context

v1.0 starts on a single dev cluster (initially kind), scales to dev+staging as agent teams converge on the platform, and grows to dev+staging+prod as the platform reaches pilot use. ArgoCD (overview §5) is the GitOps deployer; pre-merge static checks run in the GitHub Actions pipeline (ADR 0010) — kubeval on manifests, conftest against the OPA Rego library, `helm template` / `kustomize build`, and ArgoCD server-side dry-run. Those checks catch syntax and schema problems before merge.

What they don't address is **promotion**. Without an explicit promotion layer, "what's deployed in dev right now and how does it get to staging" is buried in branch-and-path conventions and ad-hoc human handoff. Audit trails for environment progression are reconstructed from git log archaeology rather than read from a system of record. The generalized Approval CRD (ADR 0017) provides human gates as a primitive, but it does not itself orchestrate environment-to-environment progression — it answers "did a human say yes," not "what gets promoted from where to where, in what order, with what verification."

Even on a single cluster, a promotion-shaped workflow (single source of truth for what is deployed, decoupling of authoring from deploy, audit trail for the progression itself) earns its weight. And the muscle has to be built before it is load-bearing.

## Decision

Adopt **Kargo** as the v1.0 promotion fabric, deployed early under the same phased trajectory as HolmesGPT (ADR 0012). Kargo is installed in single-cluster mode initially with one Stage; additional Stages come online as additional environments are added.

### GitOps repo layout — path-based, single mainline

A single mainline branch with environment directories: `envs/dev/`, `envs/staging/`, `envs/prod/`. Promotion is "Kargo updates the kustomization in `envs/<stage>/` to reference a new commit hash from the candidate Kargo Warehouse." There are **no long-lived per-environment branches**; environment differences live in path overlays, not branch divergence.

### Authoring → checks → Warehouse → promotion

The flow is:

1. Author through Headlamp graphical editors (ADR 0039), or directly in Git for power users.
2. Pre-merge static checks (kubeval, conftest, `helm template` / `kustomize build`, ArgoCD dry-run) run in GitHub Actions (ADR 0010).
3. On merge, the candidate commit lands in the Kargo Warehouse.
4. Kargo promotes through Stages: dev (always) → staging (when the second Stage is added) → prod (when the third Stage is added).

### Stage = environment, with verification

Kargo Stages map 1:1 to environments. Each Stage carries a verification step (smoke suite) that must pass before Kargo will promote onward.

### Approval CRD as Kargo's human gate (composition)

When a Stage requires human approval, Kargo creates an **Approval CRD** (ADR 0017); the Argo Workflow that already backs the Approval CRD reports the decision back to Kargo. The two systems compose — Kargo orchestrates the progression, the Approval CRD records the human decision, neither subsumes the other.

### OPA gates promotion actions

OPA (ADR 0018) gates Kargo promotion actions automatically: who can promote what, where, when, fine-grained including time-of-day windows where needed. Promotion-action policies live in the OPA bundle alongside the rest of the platform's Rego (B16).

### Identity

Keycloak OIDC (ADR 0028) authenticates the Kargo UI and API for human users; cluster-OIDC → Keycloak workload identity (ADR 0028) authenticates the Kargo controller's outbound calls.

### Headlamp surface

Headlamp deep-links into the Kargo UI per Stage. This extends the existing deep-link inventory tracked in architecture-backlog.md § 1.15.

### Substrate-asymmetric resources flow through Crossplane Compositions

Kargo promotes **claim shapes** that work identically on every substrate; substrate-asymmetric resources are reconciled by Crossplane Compositions (ADR 0041). Promotion across kind→AWS is invisible to Kargo — the claim shape is uniform; the Composition is what differs per substrate.

### Recovery model

Recovery is "promote a previous Warehouse commit through the affected Stages" through Kargo, **not** git-revert on main. Kargo's promotion record is the rollback handle; git history stays append-only.

### Test setup

A two-kind integration test setup (one kind playing dev, one playing staging) exercises promotion mechanics without AWS, suitable for CI.

## Alternatives considered

- **ArgoCD ApplicationSets only.** Works for fan-out but has no promotion model, no batched gating, no clean "what's in dev right now versus staging" view. Reconstructs promotion from generator templating rather than recording it.
- **Custom promotion scripts.** Tech debt, not auditable, every team would build something different.
- **Argo Rollouts.** Workload-level canary, not environment-level config promotion. Solves a different problem; complementary, not a substitute.
- **Defer Kargo to v1.1.** The earlier user-facing position. Reversed because the workflow wins (single source of truth for promotion, audit trail, decoupling authoring from deploy) accrue even in single-cluster mode, and building the muscle before it is load-bearing is cheaper than retrofitting it.

## Consequences

- **New Workstream A component** for Kargo install + Helm values + GitOps wiring + per-Stage verification configs. Standard component deliverables (overview §14.1) apply: HolmesGPT toolset contribution, OPA policy contribution, audit emission through the platform adapter (ADR 0034), per-component Grafana dashboards, Knative trigger flow design, tests at all three layers, tutorial(s) and how-to guide(s), per-product runbook.
- **New Workstream B5 plugin** — the cross-cutting Kargo plugin for Headlamp covers status visualization (Stages, Warehouses, in-flight promotions) and triggering promotions from the platform's primary admin UI.
- **Per-service OIDC role-mapping config** (Grafana, OpenSearch, ArgoCD, LiteLLM, LibreChat, Mattermost, Langfuse) promotes through Kargo like any other config — mappings live in Helm values, ConfigMaps, or per-service config files in Git, in `envs/<stage>/`.
- **Single-cluster phase deliberately runs Kargo with one Stage.** The workflow muscle is built before it is load-bearing.
- **Trigger to add the third Stage (prod)** — when v1.0 is feature-complete and ready for real workloads.
- **Audit.** Kargo promotion actions emit audit through the platform adapter (ADR 0034); promotion is a first-class auditable event, not implicit in git history.
- **Cross-substrate.** Promotion across kind→AWS is invisible to Kargo because Crossplane Compositions (ADR 0041) absorb the substrate asymmetry below the claim layer. AWS+GitHub remain the initial implementation targets (ADR 0033).
- **Future enhancement candidate.** A unifying `OidcRoleMapping` CRD if N service mappings get out of sync more than once. Tracked separately; not in scope here.

## References

- [ADR 0010](./0010-github-actions-only-v1.md) — GitHub Actions only for v1.0 (pre-merge checks complement Kargo promotion)
- [ADR 0012](./0012-holmesgpt-first-class-agent.md) — HolmesGPT phased trajectory (same deploy-early pattern)
- [ADR 0017](./0017-generalized-approval-system.md) — Approval CRD as Kargo's human gate
- [ADR 0018](./0018-rbac-floor-opa-restrictor.md) — RBAC floor / OPA restrictor (OPA gates promotion actions)
- [ADR 0026](./0026-independent-cluster-install-no-federation.md) — Independent-cluster install (Kargo handles cross-cluster promotion)
- [ADR 0028](./0028-identity-federation.md) — Identity federation (Keycloak OIDC for the Kargo UI; cluster-OIDC for the controller)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) — Initial implementation targets AWS + GitHub
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) — Audit pipeline (Kargo promotion actions emit audit through the platform adapter)
- [ADR 0039](./0039-headlamp-graphical-editors-for-platform-crds.md) — Headlamp graphical editors (authoring surface upstream of Kargo)
- [ADR 0041](./0041-substrate-abstraction-crossplane.md) — Substrate abstraction via Crossplane Compositions (enabler for cross-substrate promotion)
- [architecture-overview.md](../architecture-overview.md) [§5](../architecture-overview.md#5-software-added-to-baseline), [§7.4](../architecture-overview.md#74-developer-iteration-via-gitops), [§14.1](../architecture-overview.md#141-workstream-a--platform-installation-and-operations), [§14.2](../architecture-overview.md#142-workstream-b--custom-platform-development)
- [architecture-backlog.md § 1.15](../architecture-backlog.md#115-documentation-portal-vs-headlamp-linking-strategy) — Headlamp deep-link strategy (Kargo joins the deep-link inventory)

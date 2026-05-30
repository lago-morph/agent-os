# SPEC ADR-0040 — Kargo as the v1.0 promotion fabric for GitOps environments `[PROPOSED]`

> kind: ADR · workstream: — · tier: T0
> upstream: [A7; B19; B4; A9] · downstream: [A23; B5; A18] · adrs: [0040] · views: [6.6]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0040 is a settled decision: **Kargo** is the v1.0 promotion fabric, deployed early in single-cluster
mode with **one Stage**, growing to dev→staging→prod as environments come online. The GitOps repo is a
**single mainline with path-based environment directories** (`envs/dev/`, `envs/staging/`, `envs/prod/`)
— no long-lived per-environment branches. A merged candidate lands in a Kargo **Warehouse**; Kargo
promotes through **Stages** (each carrying a verification step); the **`Approval` CRD** is Kargo's human
gate; **OPA gates promotion actions**; recovery is **promote-a-previous-Warehouse-commit**, not git-revert.
This SPEC states what honoring the decision requires — it does not re-argue adopting Kargo.

The problem the decision solves: without an explicit promotion layer, "what's deployed in dev and how it
gets to staging" is buried in branch/path conventions and ad-hoc handoff, and the audit trail for
environment progression is reconstructed from git-log archaeology. The `Approval` CRD provides human
gates but does not orchestrate environment-to-environment progression. Kargo records the progression as
a system of record.

## 2. Scope

### 2.1 In scope
- Kargo as the v1.0 promotion fabric, installed early, single-cluster with one Stage; Stages added as environments come online.
- The GitOps layout: single mainline + `envs/<stage>/` path overlays; no long-lived per-environment branches.
- The flow: author (Headlamp editors ADR 0039 / direct Git) → pre-merge static checks (GitHub Actions ADR 0010) → merge → Warehouse → Kargo promotes through Stages.
- Stage = environment, each with a verification step (smoke suite) that must pass before promoting onward.
- `Approval` CRD as Kargo's human gate (composition — neither subsumes the other); OPA gates promotion actions; Keycloak OIDC authenticates UI + controller.
- Substrate-asymmetric resources flow through Crossplane Compositions (ADR 0041): Kargo promotes uniform claim shapes; cross-substrate promotion is invisible to Kargo.
- Recovery = promote a previous Warehouse commit; git history stays append-only.

### 2.2 Out of scope (and where it lives instead)
- The substrate-abstraction mechanism that makes cross-substrate promotion uniform — owned by ADR 0041 / B4.
- Pre-merge static checks (kubeval, conftest, `helm template`/`kustomize build`, ArgoCD dry-run) — owned by the GitHub Actions pipeline (ADR 0010 / B15).
- The `Approval` CRD + its Argo Workflow backing — owned by ADR 0017 / component B19.
- The authoring surface (Headlamp editors) upstream of promotion — owned by ADR 0039.
- The Kargo install component build + the Kargo Headlamp plugin — owned by component A23 (install) and B5 (plugin).
- A unifying `OidcRoleMapping` CRD — explicitly a future enhancement, not in scope.

## 3. Context & Dependencies

Upstream consumed: A7 (OPA/Gatekeeper — gates promotion actions: who can promote what, where, when,
including time-of-day windows); B19 (generalized approval system — the `Approval` CRD + Argo Workflow
that reports the decision back to Kargo); B4 (Crossplane Compositions — absorb substrate asymmetry below
the claim layer); A9 (Headlamp — deep-links into the Kargo UI per Stage). Downstream consumers: A23
(Kargo install component), B5 (cross-cutting Kargo Headlamp plugin), A18 (audit — Kargo promotion
actions emit through the audit adapter).

ADR decisions honored:
- **ADR 0040** (this): Kargo as promotion fabric; path-based single mainline; Warehouse→Stage flow; verification per Stage; recovery via re-promotion.
- **ADR 0010**: pre-merge static checks run in GitHub Actions and complement (do not replace) Kargo promotion.
- **ADR 0012**: Kargo deploys early under the same phased trajectory as HolmesGPT.
- **ADR 0017**: the `Approval` CRD is Kargo's human gate; the two compose, neither subsumes the other.
- **ADR 0018**: OPA gates promotion actions; promotion-action policies live in the OPA bundle (B16).
- **ADR 0028**: Keycloak OIDC authenticates the Kargo UI/API for humans; cluster-OIDC→Keycloak workload identity authenticates the controller's outbound calls.
- **ADR 0034**: Kargo promotion actions emit audit through the platform adapter (a first-class auditable event).
- **ADR 0041**: Kargo promotes claim shapes uniform across substrates; the Composition is what differs per substrate.
- **ADR 0033**: AWS + GitHub remain the initial implementation targets.
- **ADR 0039**: Headlamp graphical editors are the authoring surface upstream of Kargo.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- **`Approval` (CRD)** (owned by ADR 0017 / B19), namespaced. Source-stated fields: `requestingAgent`,
  `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`,
  `decidedAt`. Kargo **creates** an `Approval` when a Stage requires human approval; the backing Argo
  Workflow reports the decision back to Kargo. Kargo's own resources (Warehouse, Stage, Promotion) are
  Kargo-native objects, not platform CRDs in the §6.12 inventory `[PROPOSED — not in source]` for any
  platform-CRD treatment of them.

### 4.2 APIs / SDK surfaces
- Kargo promotes **claim shapes** (Crossplane XRC claims) that work identically on every substrate;
  substrate-asymmetric resources are reconciled by Crossplane Compositions (ADR 0041), so kind→AWS
  promotion is invisible to Kargo.
- Per-service OIDC role-mapping config (Grafana, OpenSearch, ArgoCD, LiteLLM, LibreChat, Mattermost,
  Langfuse) promotes through Kargo like any other config — mappings live in Helm values / ConfigMaps /
  per-service config files in Git under `envs/<stage>/`.
- Headlamp deep-links into the Kargo UI per Stage (extends the deep-link inventory, architecture-backlog §1.15).
- Concrete Kargo Warehouse/Stage/Promotion field shapes are upstream-Kargo's; **not specified in source** beyond the above.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Kargo promotion actions emit audit through the platform audit adapter (ADR 0034) under
  **`platform.audit.*`**; promotion is a first-class auditable event, not implicit in git history. The
  ADR also notes "Knative trigger flow design" as a standard component deliverable for A23; concrete
  per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface of its own. Kargo consumes Crossplane claim shapes (whose backends
present the ADR 0041 connection-secret) but Kargo itself promotes the *claim*, not the secret.

## 5. OSS-vs-Custom Decision
N/A — ADR. The realizing component A23 installs OSS **Kargo** (Helm values + GitOps wiring + per-Stage
verification configs); the Kargo Headlamp plugin (B5) is custom. Those OSS-vs-custom calls live in the
A23 / B5 SPECs.

## 6. Functional Requirements

- REQ-ADR-0040-01: Kargo MUST be the v1.0 promotion fabric, installed early in single-cluster mode with exactly one Stage initially; additional Stages MUST come online only as additional environments are added.
- REQ-ADR-0040-02: The GitOps repo MUST be a single mainline branch with environment directories (`envs/dev/`, `envs/staging/`, `envs/prod/`); there MUST be no long-lived per-environment branches — environment differences live in path overlays.
- REQ-ADR-0040-03: On merge, the candidate commit MUST land in the Kargo Warehouse; promotion MUST be Kargo updating the kustomization in `envs/<stage>/` to reference a new commit hash from the Warehouse.
- REQ-ADR-0040-04: Kargo Stages MUST map 1:1 to environments, each carrying a verification step (smoke suite) that MUST pass before Kargo promotes onward.
- REQ-ADR-0040-05: When a Stage requires human approval, Kargo MUST create an `Approval` CRD (ADR 0017); the backing Argo Workflow MUST report the decision back to Kargo — the two compose, neither subsumes the other.
- REQ-ADR-0040-06: OPA MUST gate Kargo promotion actions (who can promote what, where, when, including time-of-day windows); promotion-action policies MUST live in the OPA bundle alongside the rest of the platform's Rego (B16).
- REQ-ADR-0040-07: Keycloak OIDC MUST authenticate the Kargo UI/API for human users; cluster-OIDC→Keycloak workload identity MUST authenticate the Kargo controller's outbound calls (ADR 0028).
- REQ-ADR-0040-08: Kargo MUST promote claim shapes that are uniform across substrates; substrate-asymmetric resources MUST be reconciled by Crossplane Compositions (ADR 0041) so kind→AWS promotion is invisible to Kargo.
- REQ-ADR-0040-09: Recovery MUST be "promote a previous Warehouse commit through the affected Stages" via Kargo — NOT git-revert on main; git history MUST stay append-only and Kargo's promotion record MUST be the rollback handle.
- REQ-ADR-0040-10: Kargo promotion actions MUST emit audit through the platform audit adapter (ADR 0034); promotion MUST be a first-class auditable event.
- REQ-ADR-0040-11: A two-kind integration test setup (one kind playing dev, one playing staging) MUST exercise promotion mechanics without AWS, suitable for CI.

## 7. Non-Functional Requirements
- Auditability: promotion is a recorded first-class event (`platform.audit.*`), not git-log archaeology; the Warehouse promotion record is the rollback handle.
- Security: OPA gates promotion actions (RBAC-floor / OPA-restrictor, ADR 0018); Keycloak OIDC authenticates both human and controller paths.
- Portability: cross-substrate promotion is mechanically uniform because Compositions (ADR 0041) absorb asymmetry below the claim layer; AWS+GitHub are the initial targets (ADR 0033).
- Phasing: single-cluster phase deliberately runs one Stage — the workflow muscle is built before it is load-bearing; the trigger to add the prod Stage is "v1.0 feature-complete and ready for real workloads."
- Versioning (ADR 0030): promoted claim shapes follow XRD versioning; promotion of a new claim shape across Stages respects conversion/deprecation windows.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing component A23 carries the full §14.1 standard set (Helm/manifests, per-product
docs, runbook, alerts, Grafana dashboards, HolmesGPT toolset, OPA policy contribution, audit emission
via the adapter, Knative trigger flow, 3-layer tests incl. the two-kind promotion test, tutorials +
how-tos); B5 carries the Kargo Headlamp plugin. Conformance to this decision is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0040-01: Honored when Kargo runs in single-cluster mode with one Stage, and adding a second environment is shown to add a second Stage without re-architecture. (→ REQ-01)
- AC-ADR-0040-02: Honored when the repo is a single mainline with `envs/<stage>/` directories and no long-lived per-environment branches. (→ REQ-02)
- AC-ADR-0040-03: Honored when a merged candidate lands in the Warehouse and promotion updates `envs/<stage>/` to a new Warehouse commit hash. (→ REQ-03)
- AC-ADR-0040-04: Honored when a Stage's verification (smoke suite) failure blocks onward promotion. (→ REQ-04)
- AC-ADR-0040-05: Honored when an approval-gated Stage causes Kargo to create an `Approval` CRD and proceeds only after the backing Argo Workflow reports the decision back. (→ REQ-05)
- AC-ADR-0040-06: Honored when an OPA promotion-action policy (e.g. a time-of-day window) is shown to block/allow a promotion attempt. (→ REQ-06)
- AC-ADR-0040-07: Honored when the Kargo UI authenticates a human via Keycloak OIDC and the controller's outbound calls authenticate via cluster-OIDC workload identity. (→ REQ-07)
- AC-ADR-0040-08: Honored when the same claim shape is promoted across a kind→AWS boundary with no Kargo-visible substrate difference. (→ REQ-08)
- AC-ADR-0040-09: Honored when a faulty release is recovered by promoting a previous Warehouse commit (no git-revert on main; git history stays append-only). (→ REQ-09)
- AC-ADR-0040-10: Honored when every promotion action appears as an audit event via the platform adapter. (→ REQ-10)
- AC-ADR-0040-11: Honored when a two-kind setup (dev + staging) exercises promotion mechanics in CI without AWS. (→ REQ-11)

## 10. Risks & Open Questions
- (med) Substrate-asymmetric promotion correctness depends entirely on ADR 0041 Compositions absorbing the asymmetry; a missing Composition for a target's environment label would let a uniform claim silently fail — ADR 0041's Gatekeeper admission guardrail (reject claim with no matching Composition) is the mitigation.
- (med) Per-service OIDC role-mapping config drift across N services is a known pain point; a unifying `OidcRoleMapping` CRD is a tracked future enhancement, not in v1.0 scope.
- (low) `[PROPOSED]` — Kargo-native Warehouse/Stage/Promotion objects are not platform CRDs in the §6.12 inventory; any platform-CRD treatment of them is not specified in source. Concrete promotion-event names defer to B12.

## 11. References
- ADR 0040 (this decision). Enforcing/realizing components: A23 (Kargo install), B5 (Kargo Headlamp plugin), B19 (`Approval` CRD gate), A7 (OPA promotion-action gating), B4 (Crossplane Compositions), A18 (promotion-action audit), A9 (Headlamp deep-links).
- architecture-overview.md §5, §7.4 (GitOps iteration), §14.1, §14.2. architecture-backlog.md §1.15 (deep-link inventory). ADR 0010, 0012, 0017, 0018, 0026, 0028, 0033, 0034, 0039, 0041.

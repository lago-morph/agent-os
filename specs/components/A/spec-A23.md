# SPEC A23 — Kargo promotion fabric

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A7, B19] · downstream: [B5] · adrs: [0040, 0017, 0018, 0028, 0034, 0041, 0010, 0033, 0030, 0031, 0039] · views: [6.6, 7.4, 8, 11, 14]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

v1.0 starts on a single dev cluster (initially kind), scales to dev+staging, then dev+staging+prod as the platform reaches pilot use. ArgoCD is the within-environment GitOps deployer and pre-merge GitHub Actions checks (ADR 0010) catch syntax/schema/policy problems before merge — but neither addresses **promotion**: "what's deployed in dev right now and how it gets to staging" is otherwise buried in branch-and-path conventions and ad-hoc human handoff, with audit trails reconstructed from git-log archaeology. The generalized `Approval` CRD (ADR 0017) provides human gates as a primitive but does not orchestrate environment-to-environment progression.

A23 delivers **Kargo as the v1.0 promotion fabric** (ADR 0040): Helm values + install manifests, GitOps wiring against path-based environment overlays (`envs/dev/`, `envs/staging/`, `envs/prod/`) on a single mainline (no long-lived per-environment branches), Warehouse configuration for candidate commits, and per-Stage verification configs (smoke suites bound to each environment). It **composes** with the `Approval` CRD (B19) for human gates and with OPA (A7) for promotion-action policy. It lands early under the same phased trajectory as HolmesGPT: **single Stage initially**, with staging/prod Stages added as those environments come online. Substrate asymmetry (kind vs AWS) is invisible to Kargo — it promotes uniform claim shapes; Crossplane Compositions (ADR 0041) absorb the difference below the claim layer.

This is the promotion *control plane* for the platform's entire GitOps surface, so it is specified thoroughly with explicit phased-rollout, composition, and recovery semantics.

## 2. Scope

### 2.1 In scope

- **Kargo install** — Helm values + install manifests, deployed single-cluster, **single Stage initially** (ADR 0040 phased trajectory).
- **GitOps repo layout wiring** — path-based, single mainline with environment directories `envs/dev/`, `envs/staging/`, `envs/prod/`; promotion = Kargo updates the kustomization in `envs/<stage>/` to reference a new commit hash from the candidate Warehouse. **No long-lived per-environment branches.**
- **Warehouse configuration** — the construct where a candidate commit lands on merge (glossary).
- **Stage = environment, 1:1**, each carrying a **per-Stage verification step (smoke suite)** that must pass before Kargo promotes onward.
- **`Approval`-CRD composition (B19)** — when a Stage requires human approval, Kargo creates an `Approval` CRD; the Argo Workflow backing it reports the decision back to Kargo. Neither system subsumes the other.
- **OPA promotion-action gating (A7)** — OPA gates who can promote what, where, when (fine-grained, including time-of-day windows); promotion-action policies live in the OPA bundle alongside the rest of the Rego (B16).
- **Identity** — Keycloak OIDC (ADR 0028) for the Kargo UI/API (human users); cluster-OIDC → Keycloak workload identity for the Kargo controller's outbound calls.
- **Recovery model** — "promote a previous Warehouse commit through the affected Stages" via Kargo (NOT `git revert` on main); git history stays append-only; same verification + approval gates apply to rollback promotions.
- **Substrate-transparency** — Kargo promotes claim shapes; substrate-asymmetric resources are reconciled by Crossplane Compositions (ADR 0041); kind→AWS promotion is invisible to Kargo.
- **Two-kind integration test setup** (one kind = dev, one = staging) exercising promotion mechanics without AWS, suitable for CI.
- **Headlamp surface** — deep-link into the Kargo UI per Stage (extends the §11 deep-link inventory). (The Kargo Headlamp **plugin** is B5 — see §2.2.)
- Standard Workstream A deliverables (§14.1): docs, runbook, alerts, Grafana dashboard, OPA integration, audit emission, Knative trigger flow, HolmesGPT toolset, 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)

- **The Kargo Headlamp plugin** (per-Stage status visualization, promotion-triggering controls, deep-link out) — **B5** (cross-cutting Headlamp plugins, §14.2 B5 / ADR 0040 consequence). A23 owns the per-Stage deep-link *inventory entry* and the Kargo backend it points at; B5 owns the plugin.
- **The `Approval` CRD, OPA elevation logic, Argo Workflow integration, decision CloudEvents** — **B19-core** (waves.md). A23 *composes* with it; it does not implement approval.
- **The OPA engine / Gatekeeper + the Rego library framework** — **A7** (engine) + B3/B16 (policy content). A23 contributes promotion-action Rego into the B16 bundle.
- **Pre-merge static checks** (kubeval, conftest, `helm template`/`kustomize build`, ArgoCD dry-run) — the **GitHub Actions reference pipeline** (ADR 0010 / B15); these run *before* merge and complement Kargo's post-merge runtime gates (§8).
- **Within-environment deploy** (PR + ArgoCD sync) — ArgoCD; A23 handles only environment-to-environment progression on top.
- **Crossplane Compositions / substrate XRDs** — B4; A23 relies on them for substrate transparency but owns none.
- **Keycloak realm/client + cluster-OIDC issuer setup** — install admin + B1; A23 configures Kargo to consume them.
- **The `agent-platform promote` CLI subcommand** — B9 (the CLI surface, §8); A23's promotion is driven by Kargo + the CLI's `promote` calls into it. (See §10 OQ-A23-4.)
- **Adding staging/prod Stages on day one** — explicitly deferred; v1.0 runs a single Stage (ADR 0040).
- **`OidcRoleMapping` unifying CRD** — future enhancement, not in scope (ADR 0040).

## 3. Context & Dependencies

**Upstream consumed:**

- **A7 (OPA / Gatekeeper)** — the policy engine that gates Kargo promotion actions (who/what/where/when, time-of-day windows). A23 contributes promotion-action Rego into the bundle (B16) and consumes A7's decision path.
- **B19 (Generalized approval system / B19-core)** — the `Approval` CRD + Argo Workflow integration Kargo composes with for human gates. B19-core lands in W3 ahead of A23 (W4) per waves.md (the B19→A23→B5 cycle resolution).

**Downstream consumers:**

- **B5 (Cross-cutting Headlamp plugins)** — the Kargo plugin (status viz + promotion controls + deep-link) is built against A23's Kargo install/API (CSV downstream A23→B5).

**ADR decisions honored:**

- **ADR 0040** — Kargo as the v1.0 promotion fabric; path-based single-mainline layout; Warehouse→Stage flow; Stage=environment with verification; `Approval`-CRD human gate composition; OPA gates promotion actions; Keycloak OIDC + cluster-OIDC identity; recovery = promote-previous-Warehouse-commit (not git-revert); substrate transparency via Crossplane; two-kind CI test; phased single-Stage start; Headlamp deep-link per Stage.
- **ADR 0017** — the `Approval` CRD primitive Kargo composes with (answers "did a human say yes", not "what gets promoted").
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor; OPA gates promotion actions.
- **ADR 0028** — identity federation: Keycloak OIDC (UI/API), cluster-OIDC workload identity (controller).
- **ADR 0034** — Kargo promotion actions emit audit through the platform adapter; promotion is a first-class auditable event.
- **ADR 0041** — substrate asymmetry absorbed by Crossplane Compositions; Kargo promotes claim shapes only.
- **ADR 0010 / 0033** — GitHub Actions pre-merge checks complement Kargo; AWS + GitHub are the initial targets.
- **ADR 0030 / 0031** — versioning + CloudEvent taxonomy.
- **ADR 0039** — authoring is upstream of Kargo via Headlamp editors; Kargo promotes what was authored/merged.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

- A23 **defines no platform CRD/XRD of its own.** Kargo's own CRDs (Stage, Warehouse, Promotion, etc.) are **upstream Kargo constructs** — `Stage` and `Warehouse` are Canon glossary terms; the others are vendor CRDs. Any Kargo field/kind not in glossary is `[PROPOSED — not in source]` and resolved against upstream Kargo at implementation (consult upstream Kargo docs).
- **`Approval` (consumed, owner Argo Workflow + B19; namespaced)** — Canon key fields (§6.12, interface-contract §1.5): `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`. Kargo *creates* an `Approval` when a Stage needs a human gate and reads `decision`/`decidedBy`/`decidedAt` back. A23 invents no `Approval` fields.
- **Crossplane claim shapes (consumed, not owned)** — Kargo promotes uniform claim shapes; substrate XRDs (`XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`) and higher-level XRDs are B4-owned (interface-contract §1.6).

### 4.2 APIs / SDK surfaces

- **Kargo API/UI (provided by the vendor product, configured here)** — authenticated via Keycloak OIDC for humans, cluster-OIDC workload identity for the controller (ADR 0028/0040). B5's plugin and the `agent-platform promote` CLI (B9) drive it.
- **Promotion-action OPA decision (consumed)** — A7; promotion actions evaluate against the B16 bundle.
- **HTTP API versioning (§6.13):** any custom service A23 adds (none anticipated beyond Kargo config) would use URL-path versioning; Kargo's own API versioning is the vendor's. `[PROPOSED — not in source]` for any A23-owned HTTP surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Per-event-type names deferred to B12 (interface-contract §2):

- **Emitted:** promotion actions emit audit under **`platform.audit.*`** (ADR 0034 — promotion is a first-class auditable event). Promotion progression/Stage-health signals plausibly fall under **`platform.lifecycle.*`** (Workflow/lifecycle) and/or **`platform.observability.*`** (threshold/alert routing for verification failures) — the precise mapping is design-time per B12. `[PROPOSED — not in source]` for any concrete event-type name and for the exact namespace of promotion-progression events.
- **Consumed:** approval decisions surfaced by B19 under **`platform.approval.*`** (approved/rejected/timed-out) — Kargo reacts to the decision the Argo Workflow reports back (ADR 0040). The two initial v1.0 trigger flows (AlertManager→HolmesGPT, budget-exceeded→email) are not A23's; A23 designs its own Knative trigger flow (§8).

### 4.4 Data schemas / connection-secret contracts

- **N/A — connection secret:** Kargo is a control-plane install; it provisions no substrate primitive and writes no host/port/user/password/dbname connection secret (interface-contract §4 applies to substrate XRDs). It *promotes* claim shapes whose connection secrets are written by the Crossplane Compositions it triggers (B4), not by Kargo.
- **GitOps repo contract:** single mainline; `envs/dev/`, `envs/staging/`, `envs/prod/` path overlays; promotion mutates the kustomization in `envs/<stage>/` to a new commit hash from the Warehouse (ADR 0040). Per-environment Kustomize overlays handle sizing/replicas only — substrate selection is the Composition's job (§14).
- **Identity material:** Keycloak OIDC config (UI/API) + cluster-OIDC workload identity (controller); consumes the platform claim schema (§6.9) for OPA promotion-action decisions.

## 5. OSS-vs-Custom Decision

- **Named upstream project:** **Kargo** (glossary; ADR 0040). Version: pin a tested release (consistent with the §9 pin-a-tested-version discipline; exact pin `[PROPOSED — not in source]`).
- **Decision:** **config / wrap** — install + configure Kargo (Helm values, Stage/Warehouse/verification config, GitOps wiring, OIDC), wrap it into the platform's composition surfaces (compose `Approval` for gates, OPA for promotion-action gating, audit adapter for emission). No fork.
- **Rationale:** ADR 0040 selects Kargo over ArgoCD ApplicationSets (no promotion model), custom scripts (tech debt, not auditable), and Argo Rollouts (workload-level canary, different problem). The platform-specific value is the *composition* (Approval + OPA + audit + Crossplane substrate transparency), which is configuration/integration around the OSS product, not a rebuild.

## 6. Functional Requirements

- **REQ-A23-01:** A23 SHALL install Kargo single-cluster with Helm values + manifests, configured with a **single Stage** initially; the Stage list SHALL be treated as evolving (additional Stages added as environments come online) (ADR 0040 phased trajectory).
- **REQ-A23-02:** Promotion SHALL operate on a single mainline with path-based overlays `envs/dev/`, `envs/staging/`, `envs/prod/`; a promotion SHALL update the kustomization in `envs/<stage>/` to reference a new commit hash from the Warehouse, with **no long-lived per-environment branches** (ADR 0040).
- **REQ-A23-03:** On merge, the candidate commit SHALL land in the Kargo **Warehouse**; Kargo SHALL promote candidates from the Warehouse through the configured Stage(s) (ADR 0040).
- **REQ-A23-04:** Each Stage SHALL carry a **verification step (smoke suite)** that MUST pass before Kargo promotes onward (ADR 0040).
- **REQ-A23-05:** When a Stage requires human approval, Kargo SHALL create an **`Approval` CRD** (B19) and SHALL proceed only after the backing Argo Workflow reports the decision back; A23 SHALL NOT implement the approval/elevation logic itself (ADR 0040/0017).
- **REQ-A23-06:** Promotion actions SHALL be gated by **OPA** (A7) — who can promote what, where, when, including time-of-day windows where configured; promotion-action Rego SHALL live in the platform OPA bundle (B16) (ADR 0040/0018).
- **REQ-A23-07:** The Kargo UI/API SHALL authenticate human users via **Keycloak OIDC**; the Kargo controller's outbound calls SHALL use **cluster-OIDC → Keycloak workload identity** (ADR 0028/0040).
- **REQ-A23-08:** Recovery SHALL be performed by **promoting a previous Warehouse commit through the affected Stages** via Kargo — NOT `git revert` on main; the same per-Stage verification + approval gates SHALL apply to a rollback promotion (ADR 0040).
- **REQ-A23-09:** Kargo SHALL promote **uniform claim shapes** only; substrate-asymmetric resources SHALL be reconciled by Crossplane Compositions (ADR 0041), making kind→AWS promotion invisible to Kargo (ADR 0040/0041).
- **REQ-A23-10:** Every promotion action SHALL emit a structured audit event through the **platform audit adapter** (ADR 0034) — promotion is a first-class auditable event.
- **REQ-A23-11:** A23 SHALL provide a **two-kind integration test setup** (one kind = dev, one = staging) exercising promotion mechanics without AWS, suitable for CI (ADR 0040).
- **REQ-A23-12:** A23 SHALL register a **Headlamp deep-link per Stage** into the Kargo UI surfacing current promotion state, verification results, and manual-promotion controls (the deep-link inventory in §11), while the Kargo Headlamp *plugin* is delivered by B5 (ADR 0040/§14.2).
- **REQ-A23-13:** Kargo's runtime promotion gates SHALL run **after** a candidate has merged into the Warehouse, complementing (not duplicating) the pre-merge GitHub Actions static checks (ADR 0010, §8).
- **REQ-A23-14:** Per-environment Kustomize overlays SHALL handle **sizing and replicas only**; substrate selection SHALL remain the Composition's responsibility, not the overlay's (§14).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9, §6.6):** The CI/CD + promotion path is a privileged component (merge-trigger access to GitOps reconciliation). Promotion actions are OPA-gated (RBAC-as-floor / OPA-as-restrictor, ADR 0018); controller uses scoped workload identity (no long-lived static creds, consistent with §8 security-first design). Promotion across tenants/environments respects claim scope.
- **Observability (§6.5):** Promotion actions audited (REQ-A23-10); Grafana dashboard surfaces Stage health, verification pass/fail, in-flight promotions; `LogLevel` (ADR 0035) toggles Kargo-component verbosity per the platform pattern.
- **Auditability:** Kargo's promotion record is the system of record for "what's deployed where" and the rollback handle (ADR 0040) — git history stays append-only.
- **Scale / phasing:** Single-Stage v1.0 deliberately builds the muscle before it is load-bearing; the design must accept additional Stages additively without rework (ADR 0040). The trigger to add the third (prod) Stage is "v1.0 feature-complete and ready for real workloads."
- **Versioning (ADR 0030):** Kargo version pinned + tested; Kargo's own CRD/API versioning is the vendor's; any A23-owned config surface follows the per-component model.
- **Substrate parity caveat:** promotion across substrates is *shape*-uniform but runtime behaviour may differ per substrate (ADR 0041 capability-parity caveat) — the design must not assume identical runtime behaviour kind vs AWS.

## 8. Cross-Cutting Deliverable Checklist

Per §14.1 (ADR 0040 explicitly states standard deliverables apply):

- **Helm values / manifests in Git** — applicable: Kargo install Helm values + Stage/Warehouse/verification config + GitOps wiring.
- **Per-product architecture docs (10.5)** — applicable: promotion model, phased-Stage trajectory, recovery model.
- **Per-product operator runbook (10.7)** — applicable: promote a candidate, add a Stage, recover via previous-Warehouse-commit, handle a failed verification.
- **Backup / restore** — applicable: Kargo promotion state/config; GitOps repo is the source of truth (restore = re-apply config).
- **Alert rules** — applicable: verification-suite failure, stuck/aborted promotion, Stage unhealthy, controller-OIDC auth failure.
- **Grafana dashboard (Crossplane `GrafanaDashboard` XR)** — applicable: Stage health, verification pass/fail, promotion throughput, in-flight promotions.
- **Headlamp plugin** — **delivered by B5** (the Kargo plugin); A23 owns the per-Stage deep-link inventory entry + ensuring the Kargo backend integrates (§14.1 Headlamp-plugin clause: the component ensures it works in our environment).
- **OPA / Rego integration** — applicable: promotion-action policies (who/what/where/when + time-of-day windows) contributed to the B16 bundle.
- **Audit emission (ADR 0034)** — applicable: every promotion action (REQ-A23-10).
- **Knative trigger flow** — applicable: A23 designs its own flow — emits promotion/verification lifecycle events, consumes `platform.approval.*` decisions from B19; defines the Triggers routing between them. (Distinct from the two initial platform trigger flows.)
- **HolmesGPT toolset** — applicable: "what's deployed in each Stage / last verification result / promotion history" read tool.
- **3-layer tests (Chainsaw / Playwright / PyTest)** — applicable: Chainsaw for Stage/Warehouse/Approval-composition reconcile, the two-kind promotion integration test; Playwright for the Kargo UI deep-link / promotion-trigger e2e (with B5); PyTest for promotion-action OPA-gating + audit-emission logic.
- **Tutorials & how-tos** — applicable: "promote a change through Stages" how-to; tutorial covering the promotion gesture.

## 9. Acceptance Criteria

- **AC-A23-01** (REQ-A23-01): Kargo installs single-cluster with exactly one configured Stage; adding a second Stage is a config change requiring no rework of the first.
- **AC-A23-02** (REQ-A23-02): A promotion updates the kustomization under `envs/<stage>/` to a new Warehouse commit hash on the single mainline; no per-environment branch is created.
- **AC-A23-03** (REQ-A23-03): A merged candidate appears in the Warehouse and is promotable through the configured Stage(s).
- **AC-A23-04** (REQ-A23-04): A Stage with a failing verification smoke suite does not promote onward; a passing one does.
- **AC-A23-05** (REQ-A23-05): A Stage configured to require approval creates an `Approval` CRD and proceeds only after the Argo Workflow reports `decision` back; A23 contains no approval-elevation logic.
- **AC-A23-06** (REQ-A23-06): A promotion action denied by an OPA promotion-action policy (e.g. outside a time-of-day window) is blocked; an allowed one proceeds.
- **AC-A23-07** (REQ-A23-07): The Kargo UI authenticates a human via Keycloak OIDC; the controller's outbound call uses cluster-OIDC workload identity (no static credential present).
- **AC-A23-08** (REQ-A23-08): Rolling back by promoting a previous Warehouse commit re-runs the Stage verification + approval gates and does not require a `git revert` on main; git history remains append-only.
- **AC-A23-09** (REQ-A23-09): Promoting a claim that resolves to different substrates (kind vs AWS) succeeds with an identical claim shape; the Composition absorbs the difference.
- **AC-A23-10** (REQ-A23-10): Every promotion action emits a `platform.audit.*` event via the adapter, reconstructable as a promotion record.
- **AC-A23-11** (REQ-A23-11): The two-kind setup (dev + staging) promotes a candidate dev→staging in CI without AWS.
- **AC-A23-12** (REQ-A23-12): A Headlamp deep-link per Stage opens the Kargo UI showing current promotion state + verification results for that Stage.
- **AC-A23-13** (REQ-A23-13): Kargo's verification/approval gates fire only after Warehouse merge; pre-merge GitHub Actions checks remain the earlier, separate gate.
- **AC-A23-14** (REQ-A23-14): A per-environment overlay changes sizing/replicas only; changing the substrate is not expressible in the overlay (it is the Composition's job).

## 10. Risks & Open Questions

- **OQ-A23-1** (blast-radius: med): Kargo's own CRD kinds/fields beyond Canon `Stage`/`Warehouse` (e.g. Promotion, Freight, verification config shape) are vendor constructs not in source. Tagged `[PROPOSED — not in source]`; resolve against the pinned upstream Kargo version at implementation.
- **OQ-A23-2** (med): The exact CloudEvent namespace for promotion-*progression* events (vs the audit emission, which is clearly `platform.audit.*`) is design-time per B12 — plausibly `platform.lifecycle.*` and/or `platform.observability.*`. `[PROPOSED — not in source]`; confirm with B12.
- **OQ-A23-3** (med): B19-core (W3) must expose the create-`Approval` + report-decision-back surface Kargo composes with. If B19-core's interface for *external orchestrators* (Kargo) isn't a first-class surface, A23 needs a fixture and a coordination point. The B19→A23→B5 cycle is resolved by B19-core landing first (waves.md); verify the composition contract exists.
- **OQ-A23-4** (low): The `agent-platform promote` CLI subcommand (B9, §8) and Kargo's promotion API overlap — confirm `promote` drives Kargo rather than a parallel path. `[PROPOSED]` boundary.
- **OQ-A23-5** (low): The pinned Kargo version + whether its OSS feature set covers OPA-gated promotion-action hooks and Keycloak-OIDC + workload-identity natively, or needs wrapping. `[PROPOSED — not in source]`; validate against upstream at install.
- **OQ-A23-6** (low): B5 (Kargo plugin) and A23 co-land in W4; the deep-link inventory entry (A23) and the plugin (B5) must be coordinated so the surface is built once (ADR 0040 / §14.2).

## 11. References

- architecture-overview.md §6.6 (security/policy); §7.4 (developer iteration via GitOps, lines 1223–1225 — Kargo on top of within-env loop, recovery model); §8 (CI/CD integration, lines 1364–1388 — pre-merge checks complement Kargo's runtime gates); §11 (Headlamp deep-link inventory, line 1550 — Kargo deep-link per Stage); §14 promotion model (lines 1645 — promotion via Kargo, phased Stage trajectory, substrate via Crossplane, overlays = sizing/replicas only); §14.1 A23 (line 1689); §14.2 B5 (line 1699); §5 (Kargo install + config, line 160).
- ADR 0040 (Kargo as v1.0 promotion fabric — primary). ADR 0017 (`Approval` CRD). ADR 0018 (RBAC-floor / OPA-restrictor — OPA gates promotion). ADR 0028 (identity federation — Keycloak OIDC + cluster-OIDC). ADR 0034 (audit emission). ADR 0041 (substrate abstraction). ADR 0010 (GitHub Actions pre-merge). ADR 0033 (AWS + GitHub targets). ADR 0030 (versioning). ADR 0031 (CloudEvent taxonomy). ADR 0039 (Headlamp editors upstream of Kargo).
- Related pieces: A7 (OPA), B19 (approval system), B5 (Kargo Headlamp plugin), B4 (Crossplane Compositions), B16 (OPA policy content), B9 (`agent-platform` CLI / `promote`), B15 (CI/CD reference pipeline), A18 (audit adapter).
- _meta/interface-contract.md §1.5 (`Approval` fields), §1.6 (substrate XRDs), §2 (CloudEvent taxonomy), §4 (connection-secret — N/A here). _meta/waves.md (B19→A23→B5 cycle resolution, A23 = W4).

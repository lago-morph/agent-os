# SPEC OPS5 — Day-2 Operations architecture

> kind: COMPONENT (cross-cutting NFR layer) · workstream: — · tier: T1
> upstream: [A23, B9, B4, A5, A6, B13, B19, A14, A13, A18, A7, A4, D1, F1, F2, F4, F5, F6] · downstream: [] · adrs: [0030, 0040, 0041, 0035, 0034, 0012, 0011, 0010, 0026] · views: [6.5, 6.6, 6.7, 6.10, 6.13]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

OPS5 is the **Day-2 operations** layer of the platform: how the running platform itself is upgraded, observed, kept healthy, and recovered after v1.0 is installed. The functional specs build the platform; OPS5 says how an operator *runs* it across its lifetime — platform/vendor-chart upgrades, CRD/XRD version migrations, observability and on-call, incident response, configuration-drift detection and remediation, and capacity/lifecycle management of the platform's own control plane. This is **cross-cutting NFR architecture**: it owns no new product and almost no new CRD; it states the operational invariants that constrain many existing component pieces and maps each invariant onto the mechanism (ArgoCD/Kargo, ADR 0030 versioning discipline, HolmesGPT, the D-series dashboards, the F-series readiness pieces, the `agent-platform` CLI) that already realizes or must realize it.

The problem OPS5 solves is that the source treats operational/NFR architecture thinly and scatters it: CRD versioning lives in ADR 0030, promotion/recovery in A23, drift in ArgoCD's GitOps stance, runbooks in F6, capacity baselines in F5, DR in F2, retention in F1, self-diagnosis in A14. No single piece states the **Day-2 operating model** — what an upgrade is, what "drift" means and how it is remediated, what is monitored and by whom, how an incident flows from signal to remediation, and where the genuinely open decisions are (SLO targets, CRD migration strategy, the Crossplane `Operation` type). OPS5 is that statement, with each obligation made verifiable and each open decision surfaced as a PROPOSED-ADR rather than silently decided.

A defining constraint inherited from Canon: **v1.0 commits no SLOs or error budgets** (future-enhancements.md §3 — deferred); F5 produces *measured baselines*, not SLO conformance. OPS5 therefore specifies the **observability and capacity-ops machinery** and a baseline-watch posture, and surfaces SLO-target ownership as an open ADR — it does not invent numeric SLOs that Canon declines to commit.

## 2. Scope

### 2.1 In scope

- **Platform/vendor upgrade operations** — the operating model for upgrading the platform's own components and their pinned vendor charts (every Workstream A component installs at a single pinned, tested upstream version, ArgoCD-synced, no hand-config drift) through the GitOps + Kargo promotion path, with verification gates per Stage.
- **CRD/XRD version-migration operations** — the *Day-2 procedure* for executing an ADR 0030 breaking change in production: new `vN` API group, conversion webhook, stored-version migration, `vN-1` deprecation ≥1 platform release, identical XRD/Composition handling on both substrate Compositions (ADR 0041). OPS5 owns the **migration runbook and the safety gates**, not the per-CRD schema (owned by each reconciler component).
- **Observability & on-call** — the operator observability surface (D1 integrated dashboards over Mimir/Tempo/Langfuse/OpenSearch), the alert-to-on-call path (AlertManager), and the `LogLevel` dynamic-diagnostics toggle (ADR 0035) as the standard verbosity-raise gesture during an incident.
- **Incident response** — the signal→triage→remediation flow: AlertManager → HolmesGPT (one of the two initial v1.0 trigger flows), three-state OPA-gated remediation (allowed / upon-approval / denied) with `Approval`-gated writes, and the compiled cross-cutting incident runbooks (F6).
- **Drift detection & remediation** — the GitOps invariant that Git is the source of truth and running config MUST NOT drift from it; ArgoCD reconciliation as the standing drift-remediation mechanism; Crossplane Composition drift reconciliation for substrate resources (B4); recovery-by-promoting-a-previous-Warehouse-commit (NOT `git revert`) as the Kargo rollback model (A23/ADR 0040).
- **Capacity & lifecycle management of the platform itself** — watching the F5 measured baselines (gateway throughput, sandbox cold-start, broker backlog, audit-batch keep-up) against capacity signals on the D1 Capacity dashboard, and the lifecycle posture for single-instance defaults + the ADR-0035 LiteLLM `replicas >= 2` carve-out.
- **The operational deliverable obligations OPS5 places on existing pieces** — a cross-reference matrix naming, per obligation, the owning component spec that must absorb or already absorbs it (realized in the PLAN's dependency map).

### 2.2 Out of scope (and where it lives instead)

- **The promotion fabric itself** (Kargo install/config, Warehouse/Stage, verification gates, OPA promotion-action gating) → **A23** (ADR 0040). OPS5 specifies the *operating model* on top; A23 builds the fabric.
- **The CRD/XRD versioning *policy*** (the rules: new `vN` group, conversion webhook, deprecation window) → **ADR 0030**. OPS5 specifies the *Day-2 execution procedure and gates*; ADR 0030 sets the policy; each reconciler component (A5/A6/B13/B19/B4) owns its own schema and migration.
- **The integrated operator dashboards** (the six §11.1 `GrafanaDashboard` XRs) → **D1**. OPS5 names which dashboard verifies which obligation; D1 authors them.
- **SLO/SLI definitions, error budgets, burn-rate alerting, reliability dashboards** → **future-enhancements.md §3** (deferred). OPS5 surfaces *SLO-target ownership* as a PROPOSED-ADR and specifies the baseline-watch posture; it commits no numeric SLO.
- **Scale measurement** (gateway throughput / cold-start / broker backlog baselines) → **F5**. OPS5 consumes the baselines for capacity ops; F5 measures them.
- **DR design/testing** (RTO/RPO, backup topology, restore ordering) → **F2** (testing) and the sibling **OPS2** (DR architecture). OPS5 references restore as an incident path; it does not own DR.
- **Audit retention / redaction** → **F1**. OPS5 references audit availability for incident forensics; F1 owns retention.
- **Security drills / threat model** → **F4 / B22 / ADR 0027** and the sibling **OPS4** (compute isolation). OPS5 covers operational incident response, not the threat model.
- **Runbook *authoring* + compilation** → each Workstream A component (per-component 10.7 runbook), **C6** (cross-cutting), **F6** (compilation + exercise verification). OPS5 specifies *which* Day-2 runbooks must exist and be exercised; F6 compiles and verifies them.
- **The `agent-platform` CLI surface** (`promote`, `--debug`/`--trace`, etc.) → **B9**. OPS5 uses the CLI as an operator entry point; B9 owns the subcommand API.
- **HolmesGPT itself / its toolsets / rewiring** → **A14** (+ per-component toolsets). OPS5 uses HolmesGPT as the incident-triage agent; A14 builds it.

## 3. Context & Dependencies

**Upstream consumed (and exactly what):**
- **A23 (Kargo)** — the promotion + recovery control plane: per-Stage verification gates, the Warehouse-commit-as-rollback-handle recovery model, OPA-gated promotion actions, promotion audit. OPS5's upgrade and rollback operating model rides A23.
- **B9 (`agent-platform` CLI)** — the operator entry point: `promote` (drives Kargo), `--debug`/`--trace` (drives the ADR-0035 `LogLevel` toggle for a run), `validate`/`scan` pre-merge checks. OPS5 names these as the operator gestures; it defines none.
- **B4 (Crossplane Compositions)** — Day-2 Composition drift reconciliation and the XRD/Composition side of CRD migration (both substrate Compositions, ADR 0041).
- **A5 / A6 / B13 / B19** — the reconcilers that own the CRDs/XRDs OPS5's migration procedure operates on (ARK CRDs; `Sandbox`/`SandboxTemplate`; capability/key/budget CRDs; `Approval`).
- **A14 (HolmesGPT)** — the incident-triage agent; AlertManager→HolmesGPT trigger flow; three-state OPA-gated remediation; `LogLevel` driver.
- **A13 (Tempo + Mimir) / A18 (audit) / D1 (dashboards)** — the observability + audit surfaces OPS5's monitoring/forensics rest on; D1 is the operator dashboard layer.
- **A7 (OPA) / B19 (`Approval`)** — remediation gating: allowed / upon-approval / denied; `Approval`-backed human gate for risky Day-2 actions.
- **A4 (Knative + NATS JetStream)** — broker backlog is a primary capacity signal and incident class.
- **F1/F2/F4/F5/F6** — retention, DR testing, security review, scale baselines, runbook compilation: OPS5 is the operating-model umbrella that consumes their outputs.

**Downstream consumers:** none as a build edge (OPS5 is a terminal NFR layer). Operationally, every operator and HolmesGPT consume the operating model at runtime; several component specs must *absorb* an OPS5 obligation (see PLAN §3.2).

**ADR decisions honored (and the constraint each imposes):**
- **ADR 0030** — CRD/API versioning policy with per-component ownership; new `vN` group + conversion webhook + `vN-1` deprecated ≥1 platform release; **no central lockstep version**. → OPS5's migration procedure is per-component and cannot assume a coordinated platform-wide bump.
- **ADR 0041** — XRDs version identically to CRDs; conversion webhooks + deprecation windows on **both** substrate Compositions; connection-secret shape is a versioned contract. → migration gates apply to kind and AWS Compositions equally.
- **ADR 0040** — Kargo promotion fabric; per-Stage verification; recovery = promote-previous-Warehouse-commit (NOT `git revert`); git history append-only. → OPS5's upgrade/rollback model is Kargo-mediated.
- **ADR 0035** — rolling-restart-as-reload; `LogLevel` dynamic toggle; LiteLLM `replicas >= 2` for SSE-safe restarts. → upgrades are staged-restart, not hot-reload; the toggle is the verbosity gesture.
- **ADR 0034** — audit via the single adapter to Postgres+S3 system of record (OpenSearch advisory). → incident forensics read the system of record; OpenSearch is non-authoritative.
- **ADR 0012** — HolmesGPT first-class Platform Agent; three-state OPA model; AlertManager→HolmesGPT trigger flow. → incident triage is on-platform and governed, not a parallel surface.
- **ADR 0011** — three-layer testing (Chainsaw/Playwright/PyTest), CLI-orchestrated; stress probing via concurrency (no custom harness). → upgrade/migration verification reuses the test framework, not new tooling.
- **ADR 0010** — GitHub Actions pre-merge static checks complement runtime gates. → upgrade changes pass pre-merge checks before promotion.
- **ADR 0026** — independent single-cluster topology; runbooks are per-cluster, no federation. → Day-2 ops are per-cluster; no cross-cluster failover flow in v1.0.

## 4. Interfaces & Contracts

Use ONLY Canon names; anything not in Canon is tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

OPS5 **introduces no new CRD/XRD.** It operates on existing ones as a Day-2 layer:
- `LogLevel` (owner: per-component, ADR 0035) — fields `componentSelector`, `level`, `traceGranularity`, `scope`, `expiresAt`. The standard verbosity-raise during an incident; driven by HolmesGPT (OPA-gated) or by the CLI `--debug`/`--trace` toggle. OPS5 invents no field.
- `Approval` (owner: B19) — fields `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`. The human gate for upon-approval Day-2 remediations and risky migrations.
- `GrafanaDashboard` (XR, owner: B4; authored by D1) — fields `dashboardJson`, `folder`, `visibility`. The operator observability surface OPS5 obligations map onto.
- `AuditLog` (XRD, owner: B4) / audit endpoint (A18) — the system-of-record pipeline read for incident forensics.
- Migration operates on every versioned CRD/XRD (`Agent`, `Sandbox`, capability CRDs, `XPostgres`/`XSearchIndex`/`XObjectStore`/`XMongoDocStore`, higher-level XRDs) per ADR 0030/0041 — owned by A5/A6/B13/B19/B4 respectively.
- A declarative **`ScheduledTest` / `HealthCheck` XRD** for recurring capacity baselining / synthetic health probing is **out of scope** (deferred per F5 / future-enhancements.md §5); any such kind is `[PROPOSED — not in source]`.
- The Crossplane **`Operation` type** (a declarative Day-2 imperative-action primitive) — its adoption is an **open decision** (backlog §2.20 / §3.6); referenced only as a PROPOSED-ADR in §10, `[PROPOSED — not in source]`.

### 4.2 APIs / SDK surfaces

OPS5 owns no API/SDK surface. It references, as operator gestures owned elsewhere:
- `agent-platform promote` (B9) — invokes Kargo promotion (and rollback-by-previous-Warehouse-commit), honoring OPA promotion gates.
- `agent-platform --debug` / `--trace` (B9) — activates the ADR-0035 `LogLevel` toggle for a run and restores on exit.
- HolmesGPT A2A interface (A14) — incident handoff from other Platform Agents.
- Kargo UI/API (A23, Keycloak OIDC) — manual promotion / Stage status.
Per-subcommand flags beyond `--debug`/`--trace` are `[PROPOSED — not in source]` (B9-owned).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

OPS5 mints no event type; per-event-type names are deferred to B12. The Day-2 operating model rides existing namespaces:
- **Consumed (operationally):** `platform.observability.*` (threshold crossings, alert routing — the on-call trigger surface, incl. AlertManager→HolmesGPT); `platform.lifecycle.*` (upgrade/restart/migration progression of Agent/AgentRun/Sandbox/Workflow); `platform.policy.*` (OPA decisions on remediation actions); `platform.approval.*` (decisions on `Approval`-gated Day-2 actions); `platform.audit.*` (promotion/remediation forensics); `platform.security.*` (sandbox-escape / repeated-authn signals as incident triggers); `platform.capability.*` (capability-registry changes as drift/incident context).
- **Emitted:** none of OPS5's own; the *components* it constrains emit on these namespaces (e.g. A23 promotion audit under `platform.audit.*`, A14 remediation under `platform.audit.*`/`platform.policy.*`). Whether an explicit "migration started/completed" or "drift detected/remediated" lifecycle event is minted is design-time per the owning reconciler / B12 — `[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts

N/A — OPS5 provisions no substrate primitive and writes no connection secret. Migration of a substrate XRD touches the uniform connection-secret shape (`host`, `port`, `user`, `password`, `dbname`, ADR 0041), which versions with the owning XRD (B4) under the conversion-webhook + deprecation-window discipline on both substrate Compositions. OPS5 reads the audit system of record (Postgres `audit_events` + S3 archive, A18/ADR 0034) for incident forensics; it alters no schema.

## 5. OSS-vs-Custom Decision

N/A — OPS5 is a **cross-cutting NFR architecture layer**, not a deployable. It builds no service and forks nothing. Its realization is **config + operating-model + verification** on top of already-selected OSS/custom pieces: ArgoCD (GitOps drift reconciliation), Kargo (A23, promotion/rollback), Crossplane (B4, Composition drift + XRD migration), Grafana/Mimir/Tempo (A13/D1, observability), AlertManager + HolmesGPT (A14, incident triage), the `agent-platform` CLI + three-layer test framework (B9/B14, upgrade/migration verification), and the F-series readiness pieces. The single architectural value OPS5 adds is the *operating model that binds these into a Day-2 discipline* plus the verification map — no new tooling is mandated. (Enforcement/verification map lives in the PLAN.)

## 6. Functional Requirements

Numbered, testable. (Cross-cutting: many REQs are realized by an existing component; the PLAN maps each to its mechanism + owning piece.)

**Upgrade / migration safety**
- **REQ-OPS5-01:** Every platform/vendor-chart upgrade MUST flow through the GitOps + Kargo promotion path: a pinned-version change merges, lands in the Warehouse, and promotes through each configured Stage; each Stage's verification step (smoke suite) MUST pass before promotion onward (ADR 0040). No platform component is upgraded by hand-applied config (no drift; §6.5/A1 invariant).
- **REQ-OPS5-02:** A platform/vendor upgrade MUST pass the pre-merge GitHub Actions static checks (ADR 0010) before merge AND the Kargo per-Stage runtime verification after merge; the two gates are complementary, not redundant.
- **REQ-OPS5-03:** A CRD/XRD breaking-change migration MUST be executed as the ADR-0030 procedure: a new `vN` API group with a working conversion webhook, planned stored-version migration, and `vN-1` retained as deprecated for at least one platform release before removal — never an in-place destructive schema change.
- **REQ-OPS5-04:** A migration of a substrate XRD MUST ship conversion webhooks + deprecation windows on **both** the kind and AWS Compositions (ADR 0041); a connection-secret-shape change MUST follow the same discipline so neither substrate breaks consumers.
- **REQ-OPS5-05:** Every upgrade/migration MUST have a defined **rollback** that is exercised as part of its verification: for promotion state, rollback is **promoting a previous Warehouse commit** through the affected Stages (NOT `git revert` on main), re-running the same verification + approval gates (ADR 0040); for a CRD migration, rollback is the documented reverse/halt path while `vN-1` is still served.
- **REQ-OPS5-06:** A migration or high-blast-radius upgrade action MUST be gated: OPA evaluates the promotion/remediation action (allowed / upon-approval / denied), and an upon-approval action MUST create an `Approval` CRD and proceed only on a recorded human decision (ADR 0018/0017/0040).
- **REQ-OPS5-07:** Upgrade/migration verification MUST reuse the three-layer test framework (Chainsaw/Playwright/PyTest, ADR 0011) as the Stage smoke suites and conversion-webhook tests — no bespoke upgrade-test harness.
- **REQ-OPS5-08:** Upgrades MUST assume **staged rolling-restart, not hot-reload** (ADR 0035): SSE/long-lived-connection components (LiteLLM at `replicas >= 2`) MUST drain SSE-safe; the operating model MUST NOT assume zero-restart config application.

**Observability, SLOs & on-call**
- **REQ-OPS5-09:** The operator observability surface for Day-2 MUST be the D1 integrated dashboards (Platform overview, request-flow, Cost, Audit & security, Capacity, Test-framework health) over Mimir/Tempo/Langfuse/OpenSearch; OPS5 MUST NOT stand up a parallel telemetry stack.
- **REQ-OPS5-10:** Every Day-2 alert MUST name the signal that triggers it (a metric threshold / dashboard panel / CloudEvent under `platform.observability.*`) AND the runbook it routes to, so each alert is actionable and on-call has a defined response (per F6 §6.5 obligation).
- **REQ-OPS5-11:** OPS5 MUST adopt a **measured-baseline watch** posture for v1.0 — the F5 baselines (gateway throughput, sandbox cold-start, broker backlog, audit-batch keep-up) are surfaced on the D1 Capacity dashboard and watched for regression — and MUST NOT assert numeric SLOs or error budgets, which are deferred (future-enhancements.md §3). The ownership and targets of any future SLO are surfaced as a PROPOSED-ADR (§10), not decided here.
- **REQ-OPS5-12:** The standard incident verbosity-raise MUST be the ADR-0035 `LogLevel` toggle (scoped per component / tenant / event-class, with `expiresAt`), driven either by HolmesGPT through its OPA-gated path or by the `agent-platform --debug`/`--trace` run toggle, and MUST auto-restore prior levels (no permanent verbosity creep).

**Incident response**
- **REQ-OPS5-13:** The primary incident-triage flow MUST be AlertManager → HolmesGPT (one of the two initial v1.0 trigger flows, ADR 0012): diagnostic-relevant alerts arrive as CloudEvents, a Knative Trigger filters and dispatches them to HolmesGPT, which produces findings / suggestion cards / auto-PRs.
- **REQ-OPS5-14:** Every HolmesGPT remediation action MUST resolve to exactly one of allowed / upon-approval / denied at the central OPA decision point; read-only diagnostics and auto-PR / suggestion cards are the default write surface; autonomous remediation occurs only for OPA-graduated patterns (ADR 0012). All remediation actions MUST emit audit under `platform.audit.*`.
- **REQ-OPS5-15:** Cross-cutting incident runbooks MUST exist and be exercised at least once for, at minimum: correlated gateway/audit/OPA failure, broker backlog, a failed/stuck promotion, a failed CRD/XRD migration (conversion-webhook failure / stuck stored-version migration), configuration drift, and DR invocation — folded into the F6 compiled, Knowledge-Base-indexed runbook set so HolmesGPT and operators can retrieve them.
- **REQ-OPS5-16:** Incident forensics MUST read the audit **system of record** (Postgres `audit_events` + S3 archive, ADR 0034); the OpenSearch advisory index MUST be treated as non-authoritative and its unavailability MUST NOT block forensics.

**Drift detection & remediation**
- **REQ-OPS5-17:** Git MUST be the single source of truth for all platform configuration; running config MUST NOT drift from Git. ArgoCD reconciliation MUST be the standing drift-detection + remediation mechanism for GitOps-managed resources; an out-of-band manual change MUST be detected and reconciled back (or surfaced) rather than silently persisting.
- **REQ-OPS5-18:** Substrate-resource drift MUST be reconciled by Crossplane Composition reconciliation (B4); a Day-2 drift in a composed resource MUST be remediated by the Composition, not by hand-editing the managed resource.
- **REQ-OPS5-19:** Recovery from a bad deployed state MUST use the Kargo promote-previous-Warehouse-commit model (ADR 0040), keeping git history append-only; OPS5 MUST NOT prescribe `git revert` on main or direct cluster mutation as the recovery path.

**Capacity & lifecycle**
- **REQ-OPS5-20:** Capacity ops MUST watch the F5 measured baselines against the D1 Capacity dashboard signals (sandbox warm-pool utilization, cold-start rate, memory-backend headroom, broker backlog, queue depth) and route a baseline-regression to the owning component + the capacity runbook (F5/F6); OPS5 commits no autoscaling SLO (deferred, future §1/§3).
- **REQ-OPS5-21:** The lifecycle posture MUST record the v1.0 single-instance defaults and the ADR-0035 LiteLLM `replicas >= 2` carve-out, so operators know which components are single-instance (and thus a chokepoint / planned-maintenance concern) and which are restart-resilient.
- **REQ-OPS5-22:** OPS5 MUST produce a **cross-reference matrix** naming, per Day-2 obligation above, the owning component spec that realizes or must absorb it (e.g. upgrade=A23, migration=ADR 0030 + reconciler owners, dashboards=D1, triage=A14, runbooks=F6), so no obligation is unowned.

## 7. Non-Functional Requirements

- **Security / least-privilege:** Day-2 actions are privileged (merge-trigger, promotion, remediation). They MUST be OPA-gated (RBAC-as-floor / OPA-as-restrictor, ADR 0018), use scoped workload identity (no long-lived static creds — Kargo controller cluster-OIDC, HolmesGPT phase-1 read-only SA/IAM), and `Approval`-gate high-blast-radius operations. Remediation MUST default to read-only/auto-PR, expanding to autonomous only as OPA graduates a pattern.
- **Multi-tenancy (§6.9):** Day-2 operations that touch tenant resources MUST state namespace/tenant scope and required RBAC; cross-tenant operations are called out explicitly; D1 dashboard visibility is RBAC-floored + OPA-restricted so on-call does not gain cross-tenant data without grant.
- **Observability (§6.5):** OPS5 adds no parallel telemetry; it consumes Mimir/Tempo/Langfuse/OpenSearch via D1 and the audit system of record via A18. Every alert is tied to a signal and a runbook (REQ-OPS5-10).
- **Scale:** Capacity ops is baseline-watch, not SLO-conformance (future §3 deferred); the watched numbers are F5's measured baselines, recorded with their substrate (AWS-representative vs kind non-representative) and component versions.
- **Versioning (ADR 0030):** every Day-2 change to a versioned surface obeys per-component versioning; there is **no central lockstep platform version**, so "upgrade the platform" is a coordinated set of independently-versioned component upgrades, each with its own compatibility matrix, not a single monolithic bump.
- **Reversibility:** every upgrade/migration step is independently reversible (REQ-OPS5-05/19); read-only-default remediation never silently regresses (mirrors A14 phasing).
- **Resilience to advisory-component loss:** OpenSearch (advisory) or the OTel collector being down MUST degrade Day-2 surfaces gracefully (forensics from the system of record; CLI soft-dependency warn-and-continue) rather than block operations (ADR 0034 / B9 soft-dependency stance).

## 8. Cross-Cutting Deliverable Checklist

Cross-cutting NFR layer — OPS5 ships no product; most §14.1 product deliverables are realized by the components it constrains. Marked applicable / N/A accordingly:

- Helm / manifests — **N/A** — OPS5 deploys nothing; the GitOps manifests it relies on are owned per component.
- Per-product docs (10.5) — **applicable (content)** — the Day-2 operating model document + the REQ-OPS5-22 cross-reference matrix.
- Runbook (10.7) — **applicable (specifies + verifies)** — OPS5 specifies *which* Day-2 runbooks must exist (upgrade, CRD migration, drift remediation, incident flows, capacity regression, rollback) and that F6 compiles/exercises them; it does not re-author per-component runbooks.
- Alerts — **applicable (obligation)** — OPS5 requires every Day-2 alert name a signal + runbook (REQ-OPS5-10); the alert *rules* are owned per component / surfaced via D1/D3.
- Grafana dashboard (Crossplane XR) — **N/A (consumes)** — the operator dashboards are D1's `GrafanaDashboard` XRs; OPS5 maps obligations onto them.
- Headlamp plugin — **N/A** — Day-2 surfaces use existing Headlamp deep-links (A9/A22) + the Kargo/approval plugins (B5/B19-ui); OPS5 authors none.
- OPA / Rego integration — **applicable (obligation)** — Day-2 actions OPA-gated (promotion-action + remediation policies live in the B16 bundle); OPS5 sets the gating obligation, B16/A7 own the Rego.
- Audit emission (ADR 0034) — **applicable (obligation)** — every promotion/remediation/migration action is auditable via the adapter; the emission is by the acting component (A23/A14/reconcilers).
- Knative trigger flow — **N/A (uses)** — the AlertManager→HolmesGPT flow is A14's; OPS5 designs no new flow.
- HolmesGPT toolset — **N/A (uses)** — incident triage uses HolmesGPT (A14) + per-component toolsets; OPS5 authors none.
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable** — OPS5 mandates upgrade/migration/drift/rollback verification via the existing framework (REQ-OPS5-07); the conformance tests live with the owning components, orchestrated by B9/B14.
- Tutorials & how-tos — **applicable (content)** — "run a platform upgrade," "execute a CRD migration," "respond to drift," "roll back via previous Warehouse commit" how-tos (owned by Workstream C/E, specified here).

## 9. Acceptance Criteria

Each maps to a REQ in §6 (verifying mechanism named in the PLAN §5).

- **AC-OPS5-01** (REQ-OPS5-01/02): A vendor-chart version bump promoted through Kargo passes each Stage's smoke suite before promotion; the pre-merge GitHub Actions checks ran before merge; no platform component shows config drift between Git and running state after sync. *(Chainsaw / CI)*
- **AC-OPS5-02** (REQ-OPS5-03): A simulated CRD breaking change is realized as a new `vN` API group with a conversion webhook that converts `vN-1` resources, and `vN-1` is recorded deprecated (not removed) for ≥1 platform release. *(Chainsaw conversion-webhook test)*
- **AC-OPS5-03** (REQ-OPS5-04): A substrate-XRD claim-shape change ships conversion webhooks + deprecation windows on both the kind and AWS Compositions; the connection-secret shape change does not break a consumer on either substrate. *(Chainsaw, two-substrate)*
- **AC-OPS5-04** (REQ-OPS5-05/19): A bad deploy is recovered by promoting a previous Warehouse commit through the affected Stages (re-running verification + approval gates); no `git revert` on main occurred; git history is append-only. A CRD migration's documented rollback halts/reverses while `vN-1` is still served. *(Chainsaw / Playwright)*
- **AC-OPS5-05** (REQ-OPS5-06): A migration/high-blast-radius action configured to require approval creates an `Approval` CRD and proceeds only on a recorded human `decision`; an OPA-denied action is blocked. *(Chainsaw / PyTest)*
- **AC-OPS5-06** (REQ-OPS5-07): Stage smoke suites and the conversion-webhook test are Chainsaw/Playwright/PyTest cases in the B14 framework — no bespoke upgrade harness exists. *(repo/CI inspection)*
- **AC-OPS5-07** (REQ-OPS5-08): A config change to LiteLLM applies via staged rolling restart at `replicas >= 2` with SSE-safe drain; the operating model documents no hot-reload assumption. *(Chainsaw + load drain check, reuses F5 AC-F5-07)*
- **AC-OPS5-08** (REQ-OPS5-09/11): The six D1 dashboards are the Day-2 operator surface; the Capacity dashboard renders the F5 baseline signals; no SLO/error-budget panel is asserted in v1.0 and no parallel telemetry stack exists. *(Playwright / inspection)*
- **AC-OPS5-09** (REQ-OPS5-10): Every Day-2 alert in the set names a triggering signal and a target runbook; an alert with no runbook is flagged as a gap. *(doc/link-check, via F6 verification matrix)*
- **AC-OPS5-10** (REQ-OPS5-12): An incident-driven verbosity raise creates a scoped `LogLevel` with `expiresAt` (or a `--trace` run) and restores prior levels on expiry/exit; no permanent verbosity creep remains. *(Chainsaw / PyTest with kill injection)*
- **AC-OPS5-11** (REQ-OPS5-13/14): A diagnostic-relevant AlertManager alert arrives as a CloudEvent, is filtered by a Trigger, dispatched to HolmesGPT, and yields a finding/suggestion; a non-graduated remediation produces an auto-PR/suggestion card (not autonomous execution) and emits `platform.audit.*`. *(Chainsaw / PyTest)*
- **AC-OPS5-12** (REQ-OPS5-15): The cross-cutting incident runbooks (correlated gateway/audit/OPA failure, broker backlog, stuck promotion, failed migration, drift, DR invocation) exist in the F6 compiled set, are exercise-verified, and are retrievable from `platform-knowledge-base`. *(F6 verification matrix + KB retrieval)*
- **AC-OPS5-13** (REQ-OPS5-16): A forensic query is answered from the Postgres+S3 system of record with OpenSearch unavailable. *(PyTest / integration)*
- **AC-OPS5-14** (REQ-OPS5-17/18): An out-of-band manual change to a GitOps-managed resource is detected and reconciled back by ArgoCD; a drifted composed substrate resource is reconciled by its Crossplane Composition, not by hand-edit. *(Chainsaw)*
- **AC-OPS5-15** (REQ-OPS5-20/21): A simulated baseline regression (e.g. broker backlog beyond the F5 baseline) surfaces on the Capacity dashboard and routes to the owning component + capacity runbook; the lifecycle posture doc lists every single-instance component and the LiteLLM `replicas >= 2` carve-out. *(Playwright / inspection)*
- **AC-OPS5-16** (REQ-OPS5-22): The cross-reference matrix maps every §6 Day-2 obligation to an owning component spec; no obligation is unowned. *(inspection / link-check)*

## 10. Risks & Open Questions

Each with blast-radius and a `[PROPOSED]` / reconciliation note. **§10 also surfaces PROPOSED-ADR TITLES for genuinely open decisions — titles + one-line rationale; these are NOT auto-decided.**

### 10.1 PROPOSED-ADR TITLES (open decisions — author only if asked)

- **PROPOSED-ADR — "CRD/XRD production migration strategy (stored-version migration cadence, conversion-webhook availability/cert-rotation, deprecation-calendar unit)."** Rationale: ADR 0030 fixes the *policy* (new `vN` + webhook + ≥1-release deprecation) but the *Day-2 execution mechanics* — how stored-version migration is driven in production, webhook HA/cert rotation during the window, and the calendar unit of "platform release" — are deferred (backlog §1.18) and genuinely open; OPS5 must not invent them. **(blast radius: high)**
- **PROPOSED-ADR — "Platform SLO/SLI targets and ownership (who owns the numbers, which paths get an error budget, burn-rate alerting)."** Rationale: v1.0 deliberately commits no SLOs (future-enhancements.md §3); OPS5 provides the baseline-watch machinery but cannot set targets or assign ownership without a decision — the F5 baselines are candidate SLIs, not committed SLOs. **(blast radius: high)**
- **PROPOSED-ADR — "Adoption of the Crossplane `Operation` type for declarative Day-2 imperative actions (migrations, one-shot remediations, scheduled maintenance)."** Rationale: backlog §2.20 / §3.6 raises whether the Crossplane `Operation` primitive should back Day-2 imperative actions vs. ad-hoc Jobs/CLI; this directly shapes how migrations and remediations are expressed and is currently undecided. **(blast radius: med)**
- **PROPOSED-ADR — "Declarative recurring health/synthetic-probe + capacity-baseline mechanism (`ScheduledTest`/`HealthCheck` XRD) for continuous Day-2 assurance."** Rationale: F5/future §5 defer a recurring capacity/health probing primitive; OPS5's capacity-watch posture would be far stronger with one, but introducing a new XRD kind is an open architectural decision, not a v1.0 given. **(blast radius: med)**
- **PROPOSED-ADR — "ArgoCD drift-remediation stance: auto-heal+prune vs. detect-and-alert per resource class."** Rationale: REQ-OPS5-17 requires drift be detected and remediated, but whether ArgoCD `selfHeal`/`prune` is enabled automatically (vs. surfacing drift for operator action) for sensitive resource classes (e.g. secrets, capability CRDs) is a real operational trade-off the source does not pin. **(blast radius: med)**

### 10.2 Risks & open questions

- **R1 (high):** No central lockstep version (ADR 0030) means "upgrade the platform" is N independently-versioned component upgrades; a missed compatibility-matrix entry (e.g. SDK vs gateway skew) silently breaks at runtime. Mitigation: per-component compatibility matrices (ADR 0030 REQ-10) gate promotion smoke suites; CI matrix enforcement is itself deferred (backlog §1.18). `[PROPOSED]`
- **R2 (high):** A botched CRD migration (failed conversion webhook / stuck stored-version migration) can wedge a reconciler cluster-wide. Mitigation: REQ-OPS5-03/05 mandate webhook tests + a rollback path while `vN-1` is served; resolved fully only by the first PROPOSED-ADR above. Blast radius: high.
- **R3 (med):** Drift auto-remediation could fight a legitimate emergency manual change (operator breaks glass mid-incident, ArgoCD reverts it). Reconciliation: the fifth PROPOSED-ADR (auto-heal stance) plus a documented break-glass procedure in the F6 incident runbooks. `[PROPOSED]`
- **R4 (med):** "Expected capacity baseline" is not numerically pinned in source (inherited from F5 R2); a regression alert needs a baseline to compare against. Mitigation: watch F5's *recorded* baselines, not invented numbers; route to owner for confirmation. `[PROPOSED]`
- **R5 (med):** OPS5 obligations must be *absorbed* by component specs (REQ-OPS5-22 matrix) but OPS5 does not edit them (constraint); if a component never absorbs its obligation, the matrix flags a gap rather than enforcing it. Reconciliation: the matrix makes holes explicit (mirrors F6's flag-don't-hide stance). Blast radius: med.
- **R6 (low):** HolmesGPT autonomous remediation is narrow in v1.0 (read-only/auto-PR default, ADR 0012); most incident response is human-in-the-loop. Accepted: this is the intended v1.0 posture, not a gap. Blast radius: low.
- **OQ1 (low):** Should a "migration started/completed" and "drift detected/remediated" lifecycle CloudEvent be minted (under `platform.lifecycle.*` / `platform.observability.*`), or is audit emission sufficient? Design-time per the owning reconciler / B12. `[PROPOSED — not in source]`
- **OQ2 (low):** Single-cluster (ADR 0026) means no failover Day-2 flow; cross-cluster/region operations are out of v1.0. Confirmed by Canon; revisit only with the DR sibling (OPS2 / future §10).

## 11. References

- architecture-overview.md §6.5 (observability), §6.6 (OPA/audit decision points, policy simulator), §6.7 (eventing / AlertManager→HolmesGPT), §6.10 (HolmesGPT self-management), §6.13 (versioning policy), §11 (Grafana dashboards / deep-link inventory), §14 (promotion via Kargo, GitOps, single-instance defaults), §14.1 (standard deliverables), §14.6 (F-series production readiness).
- future-enhancements.md §1 (single-instance defaults + ADR-0035 two-replica carve-out; HA deferred), §3 (SLO / error-budget / reliability framework deferred), §5 (recurring health/scheduled-test primitive deferred), §10 (cross-cluster deferred).
- architecture-backlog.md §1.18 (versioning specifics / platform-release calendar deferred), §2.20 / §3.6 (Crossplane `Operation` type question).
- ADR 0030 (CRD/API versioning, per-component ownership), 0041 (substrate XRDs / both-Composition migration), 0040 (Kargo promotion + recovery), 0035 (rolling-restart-as-reload / `LogLevel` / LiteLLM `replicas>=2`), 0034 (audit system of record), 0012 (HolmesGPT), 0011 (three-layer testing), 0010 (pre-merge checks), 0026 (single-cluster), 0018/0017 (OPA-gate / `Approval`).
- Related pieces: A23 (Kargo), B9 (CLI), B4 (Compositions), A5/A6/B13/B19 (reconcilers / migration owners), A14 (HolmesGPT), A13/A18/D1 (observability/audit/dashboards), A7/B16 (OPA/Rego), A4 (broker), F1 (retention), F2 (DR), F4 (security), F5 (scale baselines), F6 (runbook compilation), C6 (cross-cutting runbooks), D3 (test dashboards). Sibling OPS pieces: OPS1 (scale/cost), OPS2 (DR), OPS3 (secrets), OPS4 (compute isolation), OPS6 (policy lifecycle).
- _meta/glossary.md (Kargo, Warehouse, Stage, `Approval`, HolmesGPT, `LogLevel`, substrate, connection secret, ArgoCD); _meta/interface-contract.md §1.1 (versioning), §1.5–1.6 (`LogLevel`/`Approval`/XRDs), §2 (CloudEvent taxonomy), §4 (connection-secret), §5 (audit adapter); _meta/waves.md (post-W4 cross-cutting layer); _meta/pending-operational-nfr-layer.md (OPS5 scope).

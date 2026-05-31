# SPEC OPS1 — Scale & Cost

> kind: COMPONENT (cross-cutting NFR layer) · workstream: — · tier: T0 (NFR)
> upstream: [A1, A4, A6, A18, B4, B13, A5] · downstream: [F5, D1, D2, D3] · adrs: [0035, 0006, 0013, 0041, 0034, 0030, 0002, 0018, 0033, 0011] · views: [6.1, 6.2, 6.5, 6.7, 6.9]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

OPS1 is the platform's **scale & cost architecture layer**: a cross-cutting NFR piece that fixes
the capacity model, the autoscaling stance, per-tenant cost attribution / showback, and the
interplay between LLM-spend budgets (`BudgetPolicy`) and Kubernetes-compute quotas
(`AgentEnvironment.quotas`). It does **not** build a new component. It constrains and is realized
by existing pieces — LiteLLM (A1) as the cost-and-throughput chokepoint, the agent-sandbox
runtime (A6) as the cold-start / warm-pool surface, the NATS JetStream broker (A4) as the backlog
surface, the audit pipeline (A18) as the highest-volume internal flow, Crossplane (B4) and the
kopf operator (B13) as the quota/budget reconcilers, and Scale evaluation (F5) as the measurement
arm. (`pending-operational-nfr-layer.md` §OPS1; architecture-overview.md §6.5 line 434.)

The problem OPS1 solves is that the source architecture treats operational scale and cost
**thinly and dispersed**: the gateway "controls cost" (§6.1 line 12) and a Cost dashboard exists
(§11.1 line 1556), but per-tenant **attribution / showback** is not specified; the platform ships
**single-instance defaults** with one carve-out — ADR 0035 raised LiteLLM to `replicas >= 2`
(future-enhancements.md §1) — yet there is **no committed autoscaling strategy**; tenant-scoped
**resource quotas** (max agents, max sandboxes, max virtual keys, cost budget, rate limit) are an
explicit deferred design decision (architecture-backlog §1.2). OPS1 gives this layer a single
coherent spec: an explicit capacity model with measurable targets, an autoscaling stance that
honours the chokepoint/fail-closed posture, a cost-attribution model that reuses Canon primitives
(virtual key → CapabilitySet → identity → tenant), and the quota↔budget interplay — surfacing as
**PROPOSED-ADR titles** every place a decision is genuinely open (e.g. *which* autoscaler, whether
SLOs are committed) rather than silently deciding.

OPS1 is treated **T0 in the NFR sense**: it is contract-shaping for many pieces. The capacity
targets it sets are *measured* by F5 (which produces baselines, asserts no SLO); the cost and
capacity dashboards it requires are realized as `GrafanaDashboard` XRs (B4/D-series); the quota
enforcement it requires lands as `AgentEnvironment.quotas`, native `ResourceQuota`/`LimitRange`,
and `BudgetPolicy`. Where v1.0 has explicitly deferred a deliverable (SLOs, HA clustering, the
custom load harness), OPS1 records the **target and the realization seam** without re-committing
the deferred work — it states the architecture, not the future enhancement.

## 2. Scope

### 2.1 In scope

- A **capacity model** with named load-bearing paths and per-tier measurable targets:
  gateway throughput/latency (A1), sandbox cold-start (A6), broker backlog/lag (A4), audit
  in-flight + batch keep-up (A18) — expressed as *capacity baselines the platform commits to
  measure* (F5), not SLOs (which §3 / future-enhancements.md §3 defers).
- An **autoscaling strategy / stance** per load-bearing component: which components scale
  horizontally, which stay single-instance by ADR 0035, the fail-closed constraint on scaling the
  gateway chokepoint, and the trigger signal class (CPU / RPS / queue-depth) — naming the
  *mechanism choice* as a PROPOSED-ADR where Canon does not fix it.
- A **cost attribution / showback model**: how LLM spend is attributed down the Canon identity
  chain (`VirtualKey.ownerIdentity` → `capabilitySetRef` → Platform-JWT `platform_tenants`) and
  surfaced per tenant / team / agent / model on the Cost dashboard (§11.1 line 1556), reusing
  LiteLLM per-key spend tracking (A1) — **showback** (visibility), not chargeback (billing).
- The **quota↔budget interplay**: how compute quotas (`AgentEnvironment.quotas`, native
  `ResourceQuota`/`LimitRange`) and LLM-spend budgets (`BudgetPolicy`) compose into one
  per-tenant capacity envelope, including the precedence/interaction rules and the deferred
  tenant-quota defaults (architecture-backlog §1.2).
- **Right-sizing** guidance: warm-pool sizing (`SandboxTemplate.warmPoolSize`/`hibernationEnabled`),
  resource-limit defaults (`SandboxTemplate.resourceLimits`), and the inputs F5's measured
  baselines feed back into those knobs.
- The **realization map** of cross-cutting obligations onto owning component specs and the dashboards
  (Cost, Capacity) — i.e. which existing spec must absorb each OPS1 obligation (detailed in PLAN-OPS1).
- The set of **PROPOSED-ADR titles** for genuinely-open scale/cost decisions (§10).

### 2.2 Out of scope (and where it lives instead — owning piece)

- **Actual measurement of throughput / cold-start / backlog** — **F5** (Scale evaluation); OPS1 sets
  the targets/paths, F5 measures them and reports gaps. OPS1 asserts **no SLO** (future-enhancements.md §3).
- **SLO/SLI definitions, error budgets, burn-rate alerting, reliability dashboards** — deferred,
  **future-enhancements.md §3**. OPS1 may name *candidate* SLIs but commits none.
- **HA topologies / clustering beyond the ADR-0035 two-replica baseline** — **future-enhancements.md §1**.
- **The `BudgetPolicy` CRD + reconcile into OPA/LiteLLM** — **B13** (kopf) / **A1** (enforcement) /
  **B3/B16** (Rego). OPS1 specifies the attribution/showback *use* of spend data, not the CRD.
- **The `AgentEnvironment` XRD + `quotas` field shape + Composition** — **B4** (Crossplane). OPS1
  specifies how quotas compose into the capacity envelope, not the Composition.
- **The Cost / Capacity dashboards' rendering** — `GrafanaDashboard` XR (B4) authored by the
  **D-series** (D1/D2 operator dashboards, D3 test-framework health). OPS1 specifies what they must show.
- **A custom load-testing harness** — deferred, **future-enhancements.md §14 / backlog 3.8**; v1.0
  uses Chainsaw/Playwright concurrency (ADR 0011) via B14.
- **Chargeback / billing integration** (invoicing, external cost systems) — not in v1.0; OPS1 is
  **showback only**. `[PROPOSED — not in source]` that chargeback is out of scope (no source commits it).
- **Cross-region / cross-AZ capacity** — **future-enhancements.md §1/§10**; ADR 0026 single-cluster.
- **DoS-class budget/capability-exhaustion defence** — security framing is **F4** / B22 threat model
  (§6.6 attack patterns line 449); OPS1 covers the capacity envelope, not the attack response.

## 3. Context & Dependencies

**Upstream pieces consumed (what OPS1 binds to — none are HARD build edges; this is a post-build NFR layer):**
- **A1 (LiteLLM)** — the cost-and-throughput chokepoint: per-key/per-model spend tracking, the
  `BudgetPolicy` two-layer enforcement, `platform.observability.*` threshold events, the
  `replicas >= 2` baseline (ADR 0035). OPS1's attribution model rides A1's spend data.
- **A6 (agent-sandbox + Envoy)** — `SandboxTemplate.warmPoolSize`/`hibernationEnabled`/`resourceLimits`
  are the cold-start / right-sizing knobs OPS1 governs.
- **A4 (Knative + NATS JetStream)** — broker backlog/lag is a named capacity path; stream sizing/retention
  is declarative (A4 REQ-A4-03).
- **A18 (audit endpoint + adapter)** — highest-volume internal flow; in-flight `audit_events` growth and
  the ~5-min batch keep-up (ADR 0034) are a named capacity path.
- **B4 (Crossplane Compositions)** — owns `AgentEnvironment.quotas` and the `GrafanaDashboard` XR the
  Cost/Capacity dashboards are realized as.
- **B13 (kopf operator)** — sole writer of `BudgetPolicy`/`VirtualKey` into LiteLLM+OPA; the attribution
  chain's reconcile path.
- **A5 (ARK)** — `Agent`/`AgentRun` are the unit of compute whose count quotas bound.

**Downstream consumers:** **F5** (measures OPS1's capacity targets), **D1/D2** (Cost + Capacity operator
dashboards realize OPS1's showback/capacity views), **D3** (test-framework health). F1/F4/F6 consume F5's
numbers, which trace to OPS1's targets.

**ADR decisions honored:**
- **ADR 0035** — single-instance defaults with the LiteLLM `replicas >= 2` SSE-safe carve-out; OPS1's
  autoscaling stance starts from this baseline and does not silently exceed it (broader HA is future §1).
- **ADR 0006 / 0013** — `BudgetPolicy` is the budget primitive, reconciled by B13 into OPA + LiteLLM;
  cost budgets are configured *through OPA*, not the LiteLLM UI (§6.6 line 499). OPS1's attribution reuses this.
- **ADR 0041** — `AgentEnvironment` (with `quotas`) and substrate sizing are Crossplane-composed; resource
  sizing/replicas/region are a documented *non-wrapped* exception handled by Kustomize (interface-contract §4).
- **ADR 0034** — audit is durable-async; audit volume is a capacity concern but never a system-of-record
  availability concern (Postgres+S3 are SoR; OpenSearch advisory).
- **ADR 0030** — any CRD/field OPS1 leans on is versioned by its owning component; OPS1 introduces no new CRD.
- **ADR 0002 / 0018** — quota and budget enforcement are RBAC-as-floor / OPA-as-restrictor; Gatekeeper admits
  quota-bearing resources. OPS1's envelope never *grants* beyond RBAC.
- **ADR 0033** — production-representative capacity numbers are measured on AWS/EKS (kind numbers are non-representative).
- **ADR 0011** — capacity probing is Chainsaw/Playwright-concurrency via B14; no custom harness in v1.0.

## 4. Interfaces & Contracts

Use ONLY Canon names. Anything not in Canon is tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

OPS1 **owns no CRD/XRD reconciler**. It constrains the use of existing Canon primitives (source-stated fields only):

| Primitive | Owner (reconciler) | Field(s) OPS1 constrains / consumes | OPS1 obligation |
|---|---|---|---|
| `BudgetPolicy` | B13 | `scope` (key/agent/team/tenant), `period`, `limits`, `thresholdActions[]` | LLM-spend half of the per-tenant capacity envelope; `scope: tenant` is the showback aggregation key |
| `VirtualKey` | B13 | `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment` | the cost-attribution chain root (identity → CapabilitySet → tenant) |
| `AgentEnvironment` (XR) | B4 | `quotas`, `region`, `defaultCapabilitySetRef` | compute-quota half of the per-tenant capacity envelope |
| `SandboxTemplate` | A6 | `warmPoolSize`, `hibernationEnabled`, `resourceLimits` | the right-sizing / cold-start knobs |
| `Sandbox` | A6 | `state` | counted against per-tenant max-sandboxes quota |
| `Agent` / `AgentRun` | A5 (ARK) | (count) | counted against per-tenant max-agents quota |
| `GrafanaDashboard` (XR) | B4 | `dashboardJson`, `folder`, `visibility` | the Cost + Capacity dashboards are realized as these XRs (D-series) |
| `TenantOnboarding` (XRD) | B4 | `tenantId`, `namespaces[]` | the boundary the quota envelope attaches to |

- **Native Kubernetes `ResourceQuota` / `LimitRange`** — per-namespace compute caps. These are *upstream
  Kubernetes* objects, not platform CRDs (not in §6.12); OPS1 requires them as the namespace-level realization
  of compute quotas alongside `AgentEnvironment.quotas`. `[PROPOSED — not in source]` whether tenant compute
  quotas are expressed via `AgentEnvironment.quotas`, native `ResourceQuota`, or both — Canon names the field
  but not the realization split (architecture-backlog §1.2 lists the *intent*, not the mechanism). See §10 R1 / PROPOSED-ADR.
- `[PROPOSED — not in source]` the **internal schema of `AgentEnvironment.quotas`** (whether it enumerates
  max-agents / max-sandboxes / max-virtual-keys / rate-limit, per backlog §1.2's list) — the field name is
  Canon; its sub-shape is design-time, owned by B4.

### 4.2 APIs / SDK surfaces

- **No new API/SDK surface.** OPS1 consumes existing telemetry surfaces: LiteLLM per-key/per-model spend
  (A1), Prometheus metrics → Mimir, OTel traces → Tempo, Langfuse generations+cost (§6.5; ADR 0015). The
  showback computation reads Mimir/Langfuse/the LiteLLM spend store; it adds no method signature.
- `[PROPOSED — not in source]` whether a "tenant cost rollup" is a stored aggregate or a dashboard-time query —
  not specified; OPS1 prefers dashboard-time aggregation over Cost-dashboard panels (§11.1) to avoid a new store. See §10 R4.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

OPS1 emits no events itself; it relies on existing emitters and the closed ten-namespace taxonomy:
- `platform.observability.*` — budget-threshold crossings (A1) and capacity/threshold crossings (A4 broker
  backlog, A6 warm-pool exhaustion). The budget-exceeded → email flow (interface-contract §2) is the v1.0
  example. `[PROPOSED — not in source]` a "quota-exhausted" or "capacity-baseline-exceeded" event type —
  per-event-type names are deferred to B12; OPS1 mints none and notes SLO-style alerting is deferred (future §3).
- `platform.tenant.*` — tenant onboarding / namespace association changes that change the quota envelope.
- OPS1 introduces **no new top-level namespace** (ADR 0031 invariant). Every event carries `specversion` + `schemaVersion`.

### 4.4 Data schemas / connection-secret contracts

- OPS1 introduces **no data schema and no connection secret**. Spend data lives in LiteLLM's backing Postgres
  (A1, via `XPostgres`); capacity metrics live in Mimir; cost-per-generation lives in Langfuse. OPS1 reads these;
  it writes none.
- `[PROPOSED — not in source]` the exact column/label set used to tag a spend record with `tenant` — LiteLLM's
  internal spend schema is upstream (A1 §4.4 treats it as opaque); attribution must derive `tenant` from the
  `VirtualKey.ownerIdentity` ↔ Platform-JWT `platform_tenants` mapping rather than assume a native tenant column. See §10 R3.

## 5. OSS-vs-Custom Decision

**Constrain + reuse — no new build.** OPS1 is an NFR/architecture layer; it builds no deployable. It reuses:
LiteLLM native per-key spend tracking (A1; basic budgets are OSS, advanced alert-routing/auto-suspension may be
enterprise — §6.6 line 509 / backlog §1.4, gap filled via callbacks if needed), `BudgetPolicy`+OPA as the policy
layer (ADR 0006), native Kubernetes `ResourceQuota`/`LimitRange` + `AgentEnvironment.quotas` (B4) for compute caps,
`SandboxTemplate` warm-pool knobs (A6) for right-sizing, Mimir/Tempo/Langfuse (A13/A2) for measurement, and the
`GrafanaDashboard` XR (B4) for the Cost/Capacity dashboards. The only *new work* OPS1 mandates is **wiring and
authoring** distributed across owning specs: the tenant-attribution join, the Cost/Capacity dashboard panels, the
quota-envelope precedence rules, and the right-sizing defaults — plus the F5 measurement of its targets. Rationale:
every primitive the layer needs already exists in Canon; the missing thing is a coherent cross-cutting *contract*, not code.

## 6. Functional Requirements

(Cross-cutting NFR requirements. Each is a constraint OPS1 places on the platform, realized by named owning pieces — see PLAN-OPS1 §3 for the realization map. "MUST measure" obligations are discharged by F5; OPS1 asserts no SLO.)

### Capacity model & targets
- **REQ-OPS1-01:** OPS1 SHALL define the platform's load-bearing capacity paths as exactly: gateway
  throughput/latency (A1), sandbox cold-start (A6), broker backlog/lag (A4), and audit in-flight + batch
  keep-up (A18); each SHALL have a *measured capacity baseline* (not an SLO), measured by F5 on the AWS/EKS
  substrate (ADR 0033) at the ADR-0035 `replicas >= 2` gateway baseline.
- **REQ-OPS1-02:** OPS1 SHALL state, for each capacity path, the **expected v1.0 capacity target** that F5's
  measurement is compared against; where the source does not pin a number, the target SHALL be recorded as
  `[PROPOSED — not in source]` and confirmed with the path's owning component before any gap is declared (mirrors F5 OQ1).
- **REQ-OPS1-03:** OPS1 SHALL NOT assert any SLO, SLI, or error budget; capacity numbers are baselines, and the
  spec SHALL state explicitly that SLO management is deferred (future-enhancements.md §3).

### Autoscaling strategy
- **REQ-OPS1-04:** OPS1 SHALL define, per load-bearing component, an explicit autoscaling **stance**: which
  components remain single-instance by ADR 0035 default, which are eligible for horizontal scaling, and the
  trigger-signal class (CPU / request-rate / queue-depth) for each eligible component.
- **REQ-OPS1-05:** Any autoscaling of the LiteLLM gateway SHALL preserve the fail-closed chokepoint posture and
  the SSE-safe rolling-restart constraints of ADR 0035 (readiness gate, `preStop` drain,
  `terminationGracePeriodSeconds` 60–300s); scaling SHALL NOT introduce an egress path that bypasses the chokepoint.
- **REQ-OPS1-06:** OPS1 SHALL NOT commit broader HA clustering beyond the ADR-0035 two-replica baseline (that is
  future-enhancements.md §1); the autoscaling stance SHALL stay within the committed deployment shape and SHALL
  flag the specific *autoscaler mechanism* choice as a PROPOSED-ADR (§10) rather than fixing it.
- **REQ-OPS1-07:** The autoscaling stance for stateful/chokepoint dependencies (OPA, the kopf operator, the NATS
  broker, Postgres) SHALL be documented as **out of scope for autoscaling in v1.0** (hard-error-on-failure per
  future §1), so no scaling rule is silently assumed for them.

### Cost attribution / showback
- **REQ-OPS1-08:** OPS1 SHALL define a cost-attribution chain that maps every LLM spend record to a tenant via
  `VirtualKey.ownerIdentity` → `VirtualKey.capabilitySetRef` → the Platform-JWT `platform_tenants` claim, with
  no spend left unattributed to a tenant/team/agent/model dimension.
- **REQ-OPS1-09:** OPS1 SHALL require a **Cost dashboard** (the §11.1 line 1556 dashboard, realized as a
  `GrafanaDashboard` XR, authored by the D-series) showing spend per **tenant, team, agent, and model** with
  budget-vs-actual and burn rate, scoped per the XR's `visibility` (RBAC + OPA), so tenant A's cost view is not
  visible to tenant B.
- **REQ-OPS1-10:** Cost attribution SHALL be **showback (visibility) only** in v1.0, not chargeback/billing;
  OPS1 SHALL state chargeback as out of scope `[PROPOSED — not in source]`.
- **REQ-OPS1-11:** OPS1 SHALL reuse LiteLLM's native per-virtual-key / per-model spend tracking (A1) as the spend
  source of record for attribution and SHALL NOT introduce a parallel spend store; where a needed budget feature
  is LiteLLM-enterprise-only (§6.6 line 509 / backlog §1.4), the gap SHALL be filled via A1's callback surface (B2), not a new store.

### Quota ↔ budget interplay
- **REQ-OPS1-12:** OPS1 SHALL define the per-tenant **capacity envelope** as the composition of (a) compute quotas —
  `AgentEnvironment.quotas` and native `ResourceQuota`/`LimitRange` — and (b) LLM-spend budgets —
  `BudgetPolicy` with `scope: tenant`; and SHALL state the precedence/interaction rule when one binds before the other.
- **REQ-OPS1-13:** OPS1 SHALL enumerate the v1.0 tenant-scoped quota dimensions as: max agents, max sandboxes,
  max virtual keys, cost budget, and rate limit (architecture-backlog §1.2), and SHALL mark the **default values**
  for each as `[PROPOSED — not in source]` (defaults are an explicit deferred decision).
- **REQ-OPS1-14:** Compute-quota and budget enforcement SHALL follow RBAC-as-floor / OPA-as-restrictor (ADR 0018)
  and SHALL be admitted via Gatekeeper (ADR 0002); a quota/budget SHALL never *grant* capacity beyond what RBAC permits.
- **REQ-OPS1-15:** Exhausting a compute quota SHALL deny new `Agent`/`Sandbox`/`VirtualKey` creation at admission;
  exhausting a spend budget SHALL apply the `BudgetPolicy.thresholdActions[]` (block / require-approval) on the next
  call (A1 REQ-A1-12) — the two SHALL be independently enforced, not collapsed into one mechanism.

### Right-sizing
- **REQ-OPS1-16:** OPS1 SHALL define right-sizing guidance for the sandbox runtime via
  `SandboxTemplate.warmPoolSize` / `hibernationEnabled` / `resourceLimits` (A6), and SHALL specify that F5's
  measured cold-start-vs-warm-pool numbers (F5 REQ-F5-02) feed those defaults back.
- **REQ-OPS1-17:** OPS1 SHALL require a **Capacity dashboard** (§11.1 line 1558, `GrafanaDashboard` XR) showing
  warm-pool utilization, cold-start rate, memory-backend headroom, broker backlog, and queue depth, so right-sizing
  decisions are evidence-based.

### Realization & non-coining
- **REQ-OPS1-18:** OPS1 SHALL introduce **no new CRD, XRD, CloudEvent top-level namespace, or SDK surface**; every
  obligation SHALL be realized through an existing Canon primitive owned by a named component, and every non-Canon
  detail SHALL be tagged `[PROPOSED — not in source]`.
- **REQ-OPS1-19:** For every cross-cutting obligation, OPS1 (via PLAN-OPS1 §3) SHALL name the **owning spec that
  must absorb it** (e.g. the Cost-dashboard panel set in D-series, the `quotas` sub-shape in B4), so no obligation
  is orphaned.
- **REQ-OPS1-20:** Where a scale/cost decision is genuinely open (autoscaler mechanism, quota realization split,
  whether to commit candidate SLIs), OPS1 SHALL surface it as a **PROPOSED-ADR title** in §10 and SHALL NOT auto-decide it.

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9; ADR 0016/0018):** the capacity envelope is a tenancy boundary — quotas and
  budgets are per-tenant (namespace), enforced RBAC-floor/OPA-restrictor, admitted by Gatekeeper. Cost showback is
  visibility-scoped by the `GrafanaDashboard` XR `visibility` (RBAC+OPA) so cost data never crosses a tenant boundary.
  Capacity/budget exhaustion is also a DoS surface (§6.6 line 449) — the envelope is the first line, with the
  threat response owned by F4/B22.
- **Observability (§6.5; ADR 0015):** OPS1 is realized *through* the existing three planes — Mimir metrics, Tempo
  traces, Langfuse cost — correlated by `trace_id`; the Cost and Capacity dashboards (§11.1) are the operator-facing
  surfaces. Off-mode tracing has zero cost (§6.5 line 430); raising verbosity has a recorded cost (ADR 0035).
- **Scale:** OPS1 *is* the scale architecture. Its own "cost" is near-zero (it adds no hot-path component); the
  attribution join and dashboard queries run at dashboard-time, not on the request path.
- **Versioning (ADR 0030):** OPS1 owns no versioned artifact; every primitive it constrains is versioned by its
  owning component. The capacity baselines are valid only against the measured component versions (F5 REQ-F5; report records versions).
- **Substrate (ADR 0033/0041):** production-representative capacity numbers are AWS/EKS only; kind numbers are
  labelled non-representative. Quota realization is substrate-agnostic (native K8s objects + `AgentEnvironment`).

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | **N/A** — OPS1 ships no deployable; it constrains existing components' manifests (quota defaults land in B4/A6 charts) |
| Per-product docs (10.5) | **applicable** — the scale & cost architecture doc: capacity model, autoscaling stance, attribution model, quota envelope |
| Operator runbook (10.7) | **applicable** — the cross-cutting "**cost spike traced back through virtual keys**" and capacity-planning runbooks (§10.7 line 1531, §10.7 "capacity planning") — seeds **F6** |
| Backup / restore | **N/A** — OPS1 holds no state |
| Alert rules | **applicable-with-caveat** — budget-threshold + capacity-pressure (warm-pool exhaustion, broker backlog) alerts ride existing component alerts; **SLO-style burn-rate alerting is deferred (future §3)** `[PROPOSED]` capacity-baseline alert |
| Grafana dashboard (`GrafanaDashboard` XR) | **applicable** — the **Cost** (§11.1 line 1556) and **Capacity** (line 1558) dashboards are OPS1's primary realization surface; authored by D-series as `GrafanaDashboard` XRs |
| Headlamp plugin | **applicable-thin** — budgets/quotas are edited via the existing Headlamp policy/budget editing surface (§6.6 line 505); OPS1 adds no new plugin |
| OPA/Rego integration | **applicable** — budget enforcement is *through OPA* (§6.6 line 499); quota admission via Gatekeeper; Rego content in B3/B16 |
| Audit emission (ADR 0034) | **applicable** — budget edits and quota-bearing resource admission are audited via the adapter; OPS1 emits no new audit type |
| Knative trigger flow | **applicable (existing)** — the budget-exceeded → email flow (interface-contract §2) is OPS1's exemplar; no new flow |
| HolmesGPT toolset | **applicable-thin** — a cost-posture / capacity-posture read tool rides A1's budget toolset + the Capacity dashboard `[PROPOSED]`; SLO-breach autonomous remediation is deferred (future §3) |
| 3-layer tests | **applicable** — the *capacity* probes are F5's (Chainsaw/Playwright concurrency, ADR 0011); the *attribution* logic and *quota admission* get PyTest (attribution join) + Chainsaw (quota/`ResourceQuota` admission) |
| Tutorials & how-tos | **applicable** — "how the gateway controls cost" (§10.5 line 1493), "set a tenant quota", "read a tenant's showback" |

## 9. Acceptance Criteria

- **AC-OPS1-01** (REQ-OPS1-01): The spec names exactly four load-bearing capacity paths, each bound to its owning
  component (A1/A6/A4/A18) and to an F5 measurement on AWS/EKS at `replicas >= 2`. *(doc/review + F5 cross-ref)*
- **AC-OPS1-02** (REQ-OPS1-02): Each capacity path has a stated expected target; every unpinned target carries
  `[PROPOSED — not in source]` and a named owning component to confirm it. *(review)*
- **AC-OPS1-03** (REQ-OPS1-03): The spec asserts no SLO/SLI/error-budget and cites future-enhancements.md §3 as the
  deferral. *(review/lint for the words "SLO"/"error budget" appearing only as deferred)*
- **AC-OPS1-04** (REQ-OPS1-04): For each load-bearing component there is an explicit single-instance-vs-scale stance
  and a named trigger-signal class. *(review)*
- **AC-OPS1-05** (REQ-OPS1-05): A described gateway scale-out preserves the ADR-0035 readiness/`preStop`/grace-period
  constraints and adds no chokepoint-bypassing path; a test scaling the gateway shows no agent egress path appears
  outside LiteLLM/Envoy. *(Chainsaw + cross-ref A6 AC-A6-05)*
- **AC-OPS1-06** (REQ-OPS1-06): No requirement commits HA clustering beyond two replicas; the autoscaler-mechanism
  choice appears only as a PROPOSED-ADR title, undecided. *(review)*
- **AC-OPS1-07** (REQ-OPS1-07): OPA/kopf/NATS/Postgres are each listed as not-autoscaled-in-v1.0 with the
  hard-error-on-failure note. *(review)*
- **AC-OPS1-08** (REQ-OPS1-08): A spend record from a `VirtualKey` resolves to exactly one tenant via
  `ownerIdentity`→`capabilitySetRef`→`platform_tenants`; an unattributable record is detectable as a defect. *(PyTest on the attribution join)*
- **AC-OPS1-09** (REQ-OPS1-09): A Cost `GrafanaDashboard` XR renders spend by tenant/team/agent/model with
  budget-vs-actual + burn rate; tenant B cannot load tenant A's cost view (XR `visibility`). *(Playwright + Chainsaw on the XR)*
- **AC-OPS1-10** (REQ-OPS1-10): The spec states showback-only and chargeback-out-of-scope (`[PROPOSED]`). *(review)*
- **AC-OPS1-11** (REQ-OPS1-11): Attribution reads LiteLLM spend (A1) with no second spend store introduced; an
  enterprise-only budget feature, if needed, is shown filled via a B2 callback, not a new store. *(review + PyTest fake on A1 spend surface)*
- **AC-OPS1-12** (REQ-OPS1-12): The capacity envelope is defined as compute-quota ∘ spend-budget with a stated
  precedence rule; a tenant hitting either limit is constrained without needing the other. *(review + Chainsaw)*
- **AC-OPS1-13** (REQ-OPS1-13): The five quota dimensions are enumerated; each default value carries `[PROPOSED — not in source]`. *(review)*
- **AC-OPS1-14** (REQ-OPS1-14): A quota/budget at admission denies over-limit creation but never grants beyond RBAC;
  Gatekeeper admission rejects an over-quota `Agent`/`Sandbox`/`VirtualKey`. *(Chainsaw on `ResourceQuota`/Gatekeeper)*
- **AC-OPS1-15** (REQ-OPS1-15): Quota exhaustion denies creation at admission; budget exhaustion applies
  `thresholdActions[]` on the next call — verified as two independent paths. *(Chainsaw for quota + PyTest/cross-ref A1 AC-A1-12 for budget)*
- **AC-OPS1-16** (REQ-OPS1-16): Right-sizing guidance references the three `SandboxTemplate` knobs and cites F5's
  warm-pool measurement as the feedback input. *(review + cross-ref F5 AC-F5-02)*
- **AC-OPS1-17** (REQ-OPS1-17): A Capacity `GrafanaDashboard` XR renders warm-pool util, cold-start rate, memory
  headroom, broker backlog, and queue depth. *(Playwright + Chainsaw on the XR)*
- **AC-OPS1-18** (REQ-OPS1-18): A review confirms OPS1 introduces no new CRD/XRD/event-namespace/SDK surface and
  every non-Canon detail is `[PROPOSED]`-tagged. *(review/lint)*
- **AC-OPS1-19** (REQ-OPS1-19): Every §6 obligation maps in PLAN-OPS1 §3 to a named owning spec; no obligation is orphaned. *(review against PLAN-OPS1)*
- **AC-OPS1-20** (REQ-OPS1-20): Each genuinely-open decision appears as a PROPOSED-ADR title in §10 with no
  pre-decided outcome in the body. *(review)*

## 10. Risks & Open Questions

- **R1 (high):** **Quota realization split is undecided.** Canon names `AgentEnvironment.quotas` *and* the backlog
  lists tenant quota dimensions, but whether compute quotas are enforced via `AgentEnvironment.quotas`, native
  `ResourceQuota`/`LimitRange`, or both — and where admission denial fires — is `[PROPOSED — not in source]`
  (architecture-backlog §1.2 is intent, not mechanism). Blast radius high: it touches B4, A5, A6, and every tenant
  onboarding. *Reconciliation:* spec commits to the *governed envelope*, not the syntax → see PROPOSED-ADR below.
- **R2 (high):** **No autoscaling mechanism is committed in Canon.** ADR 0035 fixes only the two-replica baseline;
  choosing HPA vs KEDA vs none-in-v1.0 is open and interacts with the fail-closed chokepoint and future §1 HA work.
  Auto-deciding it would pre-empt an architectural decision. Blast radius high (gateway hot path). → PROPOSED-ADR.
- **R3 (med):** **Tenant attribution depends on an identity join, not a native tenant column.** LiteLLM spend is
  keyed by virtual key; deriving `tenant` requires the `ownerIdentity`→`platform_tenants` mapping to be reliable.
  If a `VirtualKey` is issued with an `ownerIdentity` not resolvable to a tenant, spend is unattributed.
  `[PROPOSED — not in source]` the exact join. *Reconciliation:* B13 must guarantee `ownerIdentity` resolvability at issuance; alert on unattributed spend.
- **R4 (med):** **Showback aggregation site is open** — dashboard-time query vs a stored per-tenant rollup. A stored
  rollup risks a second cost store (violates REQ-OPS1-11 intent); dashboard-time query may not scale to many tenants.
  `[PROPOSED — not in source]`. *Reconciliation:* prefer dashboard-time; revisit if F5 shows the query is too heavy.
- **R5 (med):** **"Expected v1.0 capacity target" is not numerically pinned** in source (same gap F5 OQ1 names). If
  OPS1 states a target the owner disagrees with, the gap list is meaningless. *Reconciliation:* every target
  `[PROPOSED]` + confirmed by the path owner before sign-off.
- **R6 (med):** **Advanced budget features may be LiteLLM-enterprise-only** (§6.6 line 509 / backlog §1.4) — alert
  routing, automated suspension on threshold. If so, the showback/threshold-action half leans on B2 callbacks.
  Blast radius med (affects REQ-OPS1-11/15 realization, not the model). *Reconciliation:* confirm OSS-vs-enterprise during impl; gap fills via callback.
- **R7 (low):** **SLO temptation.** v1.0 explicitly defers SLOs (future §3); OPS1's capacity targets could be
  mistaken for SLOs. *Reconciliation:* REQ-OPS1-03 forbids SLO assertion; F5 numbers are baselines, recorded as *candidate* SLIs at most.
- **R8 (low):** **Chargeback creep.** Showback could be read as billing. REQ-OPS1-10 fixes showback-only; chargeback is `[PROPOSED]` out of scope.

### PROPOSED-ADR titles (decisions left genuinely open — titles + one-line rationale; NOT decided here)

- **PROPOSED-ADR: "Platform autoscaling mechanism and per-component scaling stance (HPA vs KEDA vs single-instance-by-ADR-0035)."**
  Canon commits only the two-replica gateway baseline; the autoscaler choice and which components scale on what signal is unfixed and hot-path-critical.
- **PROPOSED-ADR: "Tenant compute-quota realization: `AgentEnvironment.quotas` vs native `ResourceQuota`/`LimitRange` vs both, and where admission denial fires."**
  The field name is Canon but the enforcement mechanism and admission seam are undecided (architecture-backlog §1.2).
- **PROPOSED-ADR: "Tenant cost-attribution join and the showback aggregation site (dashboard-time query vs stored per-tenant rollup)."**
  Spend is virtual-key-keyed; deriving tenant and choosing where to aggregate is open and risks a second cost store.
- **PROPOSED-ADR: "Default tenant-scoped quota values (max agents / sandboxes / virtual keys / cost budget / rate limit) and their override path."**
  Explicitly deferred in architecture-backlog §1.2; OPS1 enumerates the dimensions but must not invent the defaults.
- **PROPOSED-ADR: "Whether v1.0 capacity baselines are promoted to candidate SLIs feeding future-§3 SLO management."**
  F5 measures baselines and asks the same question (F5 OQ2); committing them as candidate SLIs is a deferred-scope decision, not OPS1's to auto-make.

## 11. References

- `_meta/pending-operational-nfr-layer.md` §OPS1 (scope: capacity model, throughput/SLO targets per tier, autoscaling, cost model + per-tenant attribution, right-sizing; realized by A1/A4/A6/F5; relates future §3).
- architecture-overview.md §6.1 line 12 (chokepoint for routing/audit/cost/policy), §6.5 line 430/434 (off-mode zero-cost; ops-metrics/SLO deferral scope note), §6.6 lines 449 (DoS attack pattern), 497–509 (cost budgets through OPA; LiteLLM OSS-vs-enterprise budget note), §6.9 lines 720+ / `tenant_roles` line 739 (tenancy claims), §10.7 lines 1529/1531 (per-product + cross-cutting "cost spike traced through virtual keys" runbooks), §11 line 1541 (`GrafanaDashboard` XR, per-tenant scoping), §11.1 lines 1556 (Cost dashboard) / 1558 (Capacity dashboard) / 1561 (SLO dashboards deferred), §14.6 line 1761 (F5 scope).
- ADR 0035 (LiteLLM `replicas >= 2`, SSE-safe restart — the single committed scaling carve-out), ADR 0006/0013 (`BudgetPolicy`/`VirtualKey` reconcile; budgets through OPA), ADR 0041 (`AgentEnvironment.quotas`, substrate sizing as non-wrapped exception), ADR 0034 (audit durable-async, volume = capacity not availability), ADR 0030 (versioning), ADR 0002/0018 (Gatekeeper admission; RBAC-floor/OPA-restrictor), ADR 0033 (AWS/EKS for representative numbers), ADR 0011 (Chainsaw/Playwright concurrency, no custom harness).
- future-enhancements.md §1 (single-instance defaults; ADR-0035 carve-out; HA clustering deferred; no per-chokepoint fail-open/closed policy), §3 (SLO/SLI/error-budget/reliability framework deferred), §14 (custom load harness deferred — backlog 3.8).
- architecture-backlog.md §1.2 (tenant-scoped resource quotas — max agents/sandboxes/virtual keys/cost budget/rate limit — defaults deferred), §1.4 (cost-budget enforcement details — actions on threshold; OSS-vs-enterprise verification).
- interface-contract.md §1.3 (`SandboxTemplate` knobs), §1.4 (`BudgetPolicy`, `VirtualKey`), §1.6 (`AgentEnvironment`, `GrafanaDashboard`, `TenantOnboarding`), §2 (`platform.observability.*`/`platform.tenant.*`; budget-exceeded→email exemplar), §4 (connection-secret / sizing exception).
- glossary.md (Tenant, Platform JWT — `platform_tenants`, CapabilitySet, ESO, RBAC-as-floor / OPA-as-restrictor).
- Related pieces: A1, A4, A6, A18, A5, B4, B13, F5; D1/D2/D3 (Cost/Capacity/test dashboards); F1/F4/F6 (consume F5 numbers); B22 (DoS threat framing).

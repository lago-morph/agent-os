# PLAN OPS4 — Multi-Tenant Compute Isolation

> spec: SPEC-OPS4 · kind: COMPONENT (cross-cutting NFR layer) · tier: T1
> wave: post-W4 (cross-cutting NFR layer; authoring-parallel) · estimate: L
> upstream-pieces: [A6, A7, ADR-0016, A21, B16, B20, B4] · downstream-pieces: []

## 1. Implementation Strategy

OPS4 is realized as a **declarative isolation overlay + admission-conformance layer over upstream Kubernetes primitives**, not a new controller. The strategy: (a) define a *tenant isolation baseline* — a default-deny `NetworkPolicy`, a `ResourceQuota`/`LimitRange`, and `RuntimeClass`/node-pool conformance — provisioned at tenant onboarding so every tenant namespace inherits it from Git; (b) author Gatekeeper/OPA Rego (in B16/B3) that *admission-enforces* the four boundaries (network-policy presence, runtime-class validity vs substrate, node-pool entitlement, quota ceilings); (c) bind each SPEC REQ to a concrete K8s primitive + a test layer so the guarantee is verifiable, including per-substrate (kind vs managed-K8s) runs because enforcement is CNI/runtime-dependent; (d) publish the §7 threat-model/guarantee-tier table as platform documentation so the platform never over-claims. The genuinely-open decisions (quota schema, runtime-default policy, soft-vs-hard tenancy, NetworkPolicy substrate contract) are surfaced as PROPOSED-ADRs A–D and gate the *strength* of REQ-OPS4-04/06/09/10/02 — the plan builds the soft-tenancy v1.0 baseline and marks the hard-tenancy path as ADR-gated.

## 2. Ordered Task List

- **TASK-01:** Author the threat-model + per-boundary guarantee-tier table (SPEC §7) as published platform docs — produces: isolation-guarantees doc — depends-on: [].
- **TASK-02:** Define the tenant-isolation-baseline overlay (default-deny `NetworkPolicy`, `ResourceQuota`, `LimitRange`, `RuntimeClass` bindings, node-pool label scheme) as Git-reconciled manifests — produces: isolation-baseline manifests — depends-on: [TASK-01].
- **TASK-03:** Wire the baseline into A21's `TenantOnboarding` composition so onboarding provisions it (REQ-OPS4-13) — produces: A21-integration overlay + NFR-absorption note for A21 — depends-on: [TASK-02].
- **TASK-04:** Author Gatekeeper/OPA isolation Rego in B16/B3: network-policy presence, runtime-class-vs-substrate admission (REQ-OPS4-05), node-pool entitlement (REQ-OPS4-10), quota-ceiling admission (REQ-OPS4-07) — produces: isolation Rego bundle — depends-on: [TASK-02].
- **TASK-05:** Define audit + CloudEvent emission for isolation decisions (`platform.policy.*`/`platform.security.*`/`platform.observability.*`/`platform.audit.*`) via the adapter (REQ-OPS4-12) — produces: emission wiring — depends-on: [TASK-04].
- **TASK-06:** Author the cross-substrate conformance suite (Chainsaw/PyTest) for all 14 ACs, two substrates — produces: OPS4 conformance suite — depends-on: [TASK-03, TASK-04, TASK-05].
- **TASK-07:** Author runbook + alerts + Grafana dashboard (quota utilisation, isolation-deny rate, escape signal) — produces: ops deliverables — depends-on: [TASK-05].
- **TASK-08:** Surface PROPOSED-ADRs A–D to the user (titles + rationale, SPEC §10); do NOT author until asked — produces: ADR-decision request — depends-on: [TASK-01].

Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-06.

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed

- **A6 (W0)** — `Sandbox`/`SandboxTemplate` (`runtime`, `resourceLimits`, `warmPoolSize`) + the `RuntimeClass` kernel boundary + the `platform.security.*` escape signal. OPS4 binds REQ-OPS4-04/05/09 to these.
- **A7 (W0)** — OPA/Gatekeeper engine; the admission decision point all OPS4 isolation policies target.
- **ADR-0016 (auth)** — the namespace-tenancy invariant + four-layer model OPS4 extends to the compute plane.
- **A21 (W3)** — `TenantOnboarding` composition that must provision the isolation baseline (REQ-OPS4-13). HARD for TASK-03.
- **B16 (W2)** — the OPA policy-library content where isolation Rego lands (TASK-04). HARD.
- **B4 (W1)** — Crossplane composition runtime carrying `AgentEnvironment.quotas` / `TenantOnboarding`; consumed for the quota carrier.
- **B20 (W2)** — PV-mount admission whose tenancy invariant OPS4 reuses as a conformance target (REQ-OPS4-11).

### 3.2 Downstream pieces blocked on this — ids

None in the build graph (OPS4 is a terminal constraint layer). At runtime it constrains every tenant `Sandbox`; the obligations are absorbed back into A6, A21, B16 (NFR-absorption notes, SPEC §6) rather than new dependents.

### 3.3 Continuous (non-blocking) inputs

- **B22 (security threat model design, W0)** — the platform threat-model standard OPS4's §7 adversary model aligns with.
- **B14 (test framework, W3)** — the Chainsaw/PyTest harness the conformance suite (TASK-06) is built on.
- **A18 (audit adapter, W1)** — the adapter all isolation audit flows through (no direct store writes).
- **OPS1 (scale & cost)** — adjacency: OPS1 owns latency/throughput SLOs; OPS4 owns isolation of resource consumption. Coordinate the noisy-neighbor boundary so neither double-specifies.

## 4. Parallelizable Subtasks

- TASK-01 (docs/threat-model) and TASK-08 (surface ADRs) can run immediately, parallel to everything.
- Once TASK-02 lands, **TASK-03 (A21 wiring)** and **TASK-04 (Rego)** fan out concurrently — independent (composition vs policy).
- TASK-07 (runbook/alerts/dashboard) runs parallel to TASK-06 (conformance suite) once TASK-05 lands.
- Fan-out groups: {TASK-01, TASK-08} ‖ then {TASK-03, TASK-04} ‖ then {TASK-06, TASK-07}.

## 5. Test Strategy

Map ACs (SPEC §9) to layers. Fixtures/fakes noted for not-yet-landed upstreams.

| AC | Layer | Notes / fixtures |
|---|---|---|
| AC-OPS4-01 (boundary independence) | Chainsaw (fault-injection) | Disable one boundary, assert others hold. Needs two tenant namespaces. |
| AC-OPS4-02 (pod isolation) | Chainsaw + PyTest | NetworkPolicy enforcement; **per substrate** (CNI-dependent — see R1/PROPOSED-ADR-D). |
| AC-OPS4-03 (independent of egress) | PyTest | Fake Envoy-bypass; assert pod-policy still denies. |
| AC-OPS4-04 (runtime-class) | Chainsaw | Real gVisor/Kata `RuntimeClass`; runc-reject path. |
| AC-OPS4-05 (substrate-gap reject) | Chainsaw (kind + managed) | If managed-K8s unavailable in CI, fake the substrate-capability label; assert admission reject not downgrade. |
| AC-OPS4-06 (noisy-neighbor) | Chainsaw | `ResourceQuota`; tenant-A-saturation vs tenant-B-schedule. |
| AC-OPS4-07 (quota-deny) | Chainsaw + PyTest | Quota-exceed admission deny + event emission. Quota field set is `[PROPOSED]` → fixture stub until PROPOSED-ADR-A. |
| AC-OPS4-08 (no over-claim) | doc-lint / review | Assert guarantee-tier table present; no claim > classification. |
| AC-OPS4-09 (escape blast radius) | PyTest | Simulate `platform.security.*` escape; assert no cross-tenant reach. Reuses A6 escape-signal fixture. |
| AC-OPS4-10 (node-pool entitlement) | Chainsaw | Rogue `nodeSelector`/`toleration` admission reject. Node-pool scheme `[PROPOSED]` → PROPOSED-ADR-C. |
| AC-OPS4-11 (PV tenancy) | Chainsaw | Reuse B20 AC-B20-02 in the shared conformance suite. |
| AC-OPS4-12 (audit) | PyTest | Adapter mock; assert no direct store write. |
| AC-OPS4-13 (GitOps baseline) | Chainsaw | Extends A21 AC-A21-02; assert baseline provisioned, no `kubectl`. |
| AC-OPS4-14 (single-cluster) | review check | Guard: no cross-cluster claim. |

Note: any AC depending on the deferred quota schema (07) or node-pool scheme (10) runs against a **fixture stub** until PROPOSED-ADR-A/C are decided; the conformance test asserts the *invariant* (deny-on-exceed, deny-on-unentitled) independent of the concrete field names.

## 6. PR / Branch Mapping

### 6.1 Stack position

Base branch = `wave/ops` (the cross-cutting operational/NFR layer branch, stacked **after `wave/4`** — OPS4 consumes A21 (W3) and B16 (W2), all of which land by W4). Per pending-operational-nfr-layer.md, each OPS piece is its own stacked PR in the graph.

### 6.2 PR

`piece/OPS4-multi-tenant-compute-isolation` → base `wave/ops`; carries `specs/ops/spec-OPS4.md` + `plans/ops/plan-OPS4.md` (and, when implemented, the isolation-baseline manifests, the B16 Rego contribution, and the conformance suite). NFR-absorption notes (A6 §7, A21 composition, B16 Rego) are cross-referenced but land as small follow-on edits to those components' branches, not in this PR (per the "do not edit existing specs" constraint at authoring time — implementation PRs come later).

### 6.3 Merge order

Within the OPS layer, OPS1–OPS6 are **independent siblings** (each a separate PR onto `wave/ops`); OPS4 has no hard dependency on its OPS siblings (only the OPS1 noisy-neighbor-boundary coordination, handled by review, not by merge order). `wave/ops` rolls up to `main` after the W0–W4 component waves are merged.

## 7. Effort Estimate

| Task | Size |
|---|---|
| TASK-01 threat-model/docs | M |
| TASK-02 isolation-baseline manifests | M |
| TASK-03 A21 integration | S |
| TASK-04 isolation Rego (B16) | L |
| TASK-05 audit/event emission | S |
| TASK-06 conformance suite (2 substrates) | L |
| TASK-07 runbook/alerts/dashboard | M |
| TASK-08 surface ADRs | S |

Rollup: **L**. Critical path within the piece: TASK-01 → TASK-02 → TASK-04 → TASK-06 (≈ two of the three L/M legs are serial). The L tasks (Rego, conformance) dominate; quota-dependent slices are blocked-but-stubbable pending PROPOSED-ADR-A/C.

## 8. Rollback / Reversibility

OPS4 introduces **no new controller or CRD** — back-out is removing the isolation-baseline overlay, the B16 isolation Rego, and the A21 wiring; the underlying ADR-0016 four-layer enforcement (namespace/RBAC/Gatekeeper/OPA-at-LiteLLM/Envoy) remains intact, so reverting OPS4 **weakens the compute-plane (pod-network, node, quota) isolation but does not remove tenancy**. Downstream impact of reverting: tenants lose the default-deny pod-network policy and quota ceilings (noisy-neighbor and intra-cluster pod isolation regress to the pre-OPS4 posture); no piece is functionally broken because nothing depends on OPS4 as a build input. Reversibility is therefore high; the residual risk on revert is a *silent weakening of an isolation guarantee*, mitigated by AC-OPS4-08 (the published guarantee-tier table must be updated in lockstep so the platform never claims an isolation tier it no longer enforces).

# PLAN V6-09 — Multi-tenancy and namespacing [PROPOSED]

> spec: SPEC-V6-09 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [B4, A21, B1, A7, B16]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized by the tenancy components in wave
order. The structural root is **B4** (W1) — the `TenantOnboarding` XRD that provisions namespaces,
default ServiceAccounts, and the cluster-OIDC claim mapping. **A21** (W3) drives the Headlamp +
GitOps onboarding flow over that XRD. **B1** (W1) scopes UI sessions to the Platform JWT claims, and
the policy enforcement (A7 admission + B16 OPA content, W0/W2) realizes the three-layer
visibility model. The namespace-as-tenancy-boundary and claim-consumption invariants (SPEC §6) are
verified end-to-end against a running two-tenant cluster, not per-component.

## 2. Ordered Task List
- **TASK-01:** Bind SPEC §6 invariants to realizing piece IDs — produces: realization trace
  (REQ-01/02 → namespaced-everything across all owners; REQ-03/04 → B1+OPA claim consumption;
  REQ-05/06/07/08 → A7/B16+Envoy+LiteLLM; REQ-09/10 → B4+A21) — depends-on: [].
- **TASK-02:** Confirm the claim-schema boundary with V6-11 (this view consumes, does not define) —
  produces: cross-view deferral note — depends-on: [TASK-01].
- **TASK-03:** Define the three-layer enforcement verification path (admission / gateway-OPA /
  network) including the gateway-bypass case — produces: §5 AC→layer map — depends-on: [TASK-01].
- **TASK-04:** Cross-link to V6-08 (cross-tenant CapabilitySet publish) and V6-12 (CRD inventory) —
  produces: reference set — depends-on: [TASK-01].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
B4, A21, B1, A7, B16 bind to this contract (constraint dependency, not a build block).
### 3.3 Continuous (non-blocking) inputs
B14 test framework (Chainsaw/PyTest for namespace + policy assertions); B22 threat model (isolation
assumptions); V6-11 (claim schema, consumed); B12 registry (`platform.tenant.*` schemas, deferred).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 run concurrently after TASK-01.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by B4/A21/B1/A7/B16 deliverables, verified end-to-end:
- AC-V6-09-01/02 → **Chainsaw**: two namespaces; assert tenant-B cannot enumerate tenant-A CRDs;
  assert no cluster-scoped platform CRD; **PyTest**: OPA decision uses `platform_namespaces`.
- AC-V6-09-03 → **PyTest/Chainsaw**: cross-tenant A2A call denied at gateway; with gateway faked,
  blocked by Envoy network policy.
- AC-V6-09-04 → **PyTest**: OPA-denied dynamic registration yields no discoverable A2A/MCP exposing.
- AC-V6-09-05 → **Chainsaw**: cross-namespace CapabilitySet without OPA publication rejected.
- AC-V6-09-06 → **Chainsaw**: apply `TenantOnboarding` claim; assert namespace(s) + ServiceAccounts
  wired to cluster OIDC + claim-to-tenant binding.
- AC-V6-09-07 → **Chainsaw**: onboard tenant with zero CapabilitySets; remove a CapabilitySet without
  re-onboarding.
Fixtures: fake Keycloak JWT minter (claim schema); fake LiteLLM admin API + Envoy allowlist until
A1/A6 land; fake OPA bundle until B16.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-09-multi-tenancy` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band

## 7. Effort Estimate
S overall. TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the spanning tenancy contract B4/A21/B1/A7/B16 bind
to. No runtime blast radius; downstream component specs would lose their §6.9 isolation anchor.

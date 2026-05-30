# PLAN ADR-0029 — Required Keycloak JWT claim schema [PROPOSED]

> spec: SPEC-ADR-0029 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [] · downstream-pieces: [A1, A7, A9, A8, A21, B13]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0029 fixes the §6.9 claim schema as an invariant; it is enforced by every identity consumer reading ONLY the §6.9 claims — **A1** (LiteLLM auth + virtual-key binding), **A7** (OPA), **A9** (Headlamp gating), **A8** (LibreChat visibility) — and by the platform-shipped cluster-OIDC mapper bundles (ADR 0028) landing SA principals in that shape, tested in platform CI across kind/EKS/AKS. **A21** populates the claims at onboarding; **B13** uses `capability_set_refs` to gate `VirtualKey` issuance. Conformance is tested by schema-validating sampled human and SA JWTs, linting consumers for invented claim names, proving mapper parity across distributions, and routing a simulated schema change through ADR 0030 versioning with mapper bundles bumped in lockstep.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing piece (consumers read-only-§6.9, mapper produces shape, B13 gating, version-lockstep) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify sampled human + SA JWTs validate against the §6.9 schema (all five platform claims, correct types/required-ness) — produces: schema-validation set — depends-on: [TASK-01]
- TASK-03: Lint each identity consumer (A1/A7/A9/A8) for §6.9-only claim names; no invented names; §6.9 not duplicated — produces: consumer lint — depends-on: [TASK-01]
- TASK-04: Verify cluster-OIDC mappers produce the §6.9 shape identically on kind/EKS/AKS in platform CI — produces: mapper-parity check — depends-on: [TASK-02]
- TASK-05: Verify SA `capability_set_refs` gates dynamic A2A/MCP registration and `VirtualKey` issuance via OPA — produces: B13/A7 gating test — depends-on: [TASK-02]
- TASK-06: Verify a simulated schema change routes through ADR 0030 versioning with kind/EKS/AKS mapper bundles bumped in lockstep, pinned to the schema version — produces: versioning conformance — depends-on: [TASK-04]
- TASK-07: Verify OPA grants nothing beyond the RBAC floor given these claims — produces: RBAC-floor check — depends-on: [TASK-03]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None as platform pieces; depends on §6.9 as source of truth and ADR 0028 for JWT production.
### 3.2 Downstream pieces blocked on this
- A1, A7, A9, A8 (consumers), A21 (onboarding populates claims), B13 (VirtualKey gating).
### 3.3 Continuous (non-blocking) inputs
- ADR 0028 (mapper bundles), ADR 0030 (versioning discipline), ADR 0018 (RBAC floor), ADR 0037 (onboarding binding); B22 threat model.

## 4. Parallelizable Subtasks
TASK-02→TASK-04→TASK-06 are the version-lockstep chain; TASK-03→TASK-07 and TASK-05 fan out in parallel once TASK-01/02 land.

## 5. Test Strategy
- AC-01 → PyTest: schema-validate sampled human + SA JWTs against §6.9.
- AC-02 → PyTest doc/code-lint: consumers read only §6.9 names; no duplication of §6.9.
- AC-03 → PyTest in platform CI: mapper-parity across kind/EKS/AKS.
- AC-04 → Chainsaw+PyTest: `capability_set_refs` gates A2A/MCP registration and VirtualKey issuance via OPA.
- AC-05 → versioning test: simulated schema change → ADR 0030 path + lockstep mapper-bundle bump pinned to schema version.
- AC-06 → PyTest: OPA grants nothing beyond RBAC floor.
- Fixtures/fakes: Keycloak realm fixture issuing test JWTs; stub OPA decision endpoint + LiteLLM/Headlamp/LibreChat claim readers for not-yet-landed consumers.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0029-keycloak-jwt-claim-schema` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; coordinate with ADR-0028 (mapper bundles) and ADR-0030 (versioning); rolls up to main.

## 7. Effort Estimate
M overall. Per-task: TASK-01 S, 02 S, 03 M, 04 M, 05 M, 06 M, 07 S. Critical path: TASK-01 → 02 → 04 → 06.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, identity consumers lose the stable §6.9 contract and would drift to per-component claim names; the cluster-OIDC mapper loses its destination-shape target. High blast radius — every identity consumer plus the platform-shipped mappers depend on this contract; reversal is strongly discouraged. No runtime artifact is deleted.

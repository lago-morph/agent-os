# PLAN ADR-0037 — Tenant onboarding via Headlamp + a TenantOnboarding XRD `[PROPOSED]`

> spec: SPEC-ADR-0037 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B4; A9; A7] · downstream-pieces: [A21; A22; B4]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0037 is enforced at **B4** (the `TenantOnboarding` Composition that
creates namespace + default ServiceAccounts + claim-mapping record and creates no CapabilitySet), at
**A21** (the reconciler realizing onboarding/offboarding symmetry), and at **A22/B5** (the Headlamp
plugin that submits Git PRs, never cluster writes). The identity/capability decoupling is verified by
a zero-CapabilitySet tenant-creation test; the GitOps invariant by a PR-not-cluster-write test;
offboarding by a teardown-leaves-CapabilitySets-intact test.

## 2. Ordered Task List
- TASK-01: Verify applying a `TenantOnboarding` instance creates namespace + default ServiceAccounts + cluster-OIDC claim-mapping record — produces: Chainsaw XRD test — depends-on: []
- TASK-02: Verify the namespace carries tenancy labels and OPA/observability scoping picks it up — produces: Chainsaw/integration test — depends-on: [TASK-01]
- TASK-03: Verify a tenant is created/usable with zero CapabilitySets and none is created by reconciliation — produces: Chainsaw decoupling test — depends-on: [TASK-01]
- TASK-04: Verify Headlamp onboarding submission produces a Git PR (not a cluster write), schema-validated, with a proposed claim-mapping — produces: Playwright UI test — depends-on: []
- TASK-05: Verify the CapabilitySet recommendation is a dismissible UX prompt with no effect on tenant creation — produces: Playwright UI test — depends-on: [TASK-04]
- TASK-06: Verify offboarding tears down namespace + SAs, archives audit refs, notifies operator, leaves referenced CapabilitySets intact — produces: Chainsaw offboarding test — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B4 — Crossplane Compositions (composes the XRD). A9 — Headlamp framework (hosts the plugin). A7 — OPA/Gatekeeper (namespace-scope pickup).
### 3.2 Downstream pieces blocked on this
- A21 (reconciler build), A22/B5 (onboarding plugin build), B4 (`TenantOnboarding` Composition).
### 3.3 Continuous (non-blocking) inputs
- ADR 0039 editor framework; ADR 0036 Mattermost channel (offboarding notification); B14 test framework; F1 (retention of archived refs).

## 4. Parallelizable Subtasks
TASK-01 and TASK-04 run concurrently. After TASK-01: TASK-02, TASK-03, TASK-06 fan out. TASK-05 follows TASK-04.

## 5. Test Strategy
- AC-01 → Chainsaw XRD provisioning (TASK-01); fake B4 Composition until B4 lands.
- AC-02 → Chainsaw/integration namespace-label + scope pickup (TASK-02).
- AC-03 → Chainsaw decoupling (TASK-03).
- AC-04 → Playwright Git-PR-not-cluster-write (TASK-04); fake A22 plugin fixture until A22 lands.
- AC-05 → Playwright UX-prompt dismissibility (TASK-05).
- AC-06 → Chainsaw offboarding symmetry (TASK-06).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0037-tenant-onboarding` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 M · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S · TASK-06 M. Rollup M. Critical path: TASK-01 → TASK-06 (offboarding symmetry over the composed resources).

## 8. Rollback / Reversibility
Reverting backs out the `TenantOnboarding` XRD/Composition and the onboarding plugin; tenant creation
reverts to the manual `kubectl` + Keycloak runbook and tenant state leaves Git (losing review/rollback/audit).
Already-created tenants are unaffected (their namespaces/SAs persist). No data migration; archived
audit references survive per the deferred retention policy.

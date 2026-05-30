# PLAN ADR-0028 — Identity federation chain [PROPOSED]

> spec: SPEC-ADR-0028 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [] · downstream-pieces: [B1, A1, A7, A9, A8, A21, B13]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0028 is enforced by the platform-shipped **cluster-OIDC → platform-claim mapper bundles** (uniform across kind/EKS/AKS) and the **kind OIDC bootstrap utility**, plus the federation configuration on each cluster surface. Identity consumers — **A1** (LiteLLM), **A7** (OPA), **A9** (Headlamp), **A8** (LibreChat) — prove they trust only Keycloak-issued JWTs; **B1** (SSO/auth proxy) fronts the UI flows; **A21** (tenant onboarding) populates claims; **B13** binds `VirtualKey` to identity. Conformance is tested by minting human and SA JWTs that validate against the ADR 0029 schema, running the mapper identically across distributions in platform CI, proving the kind bootstrap exposes a reachable OIDC/JWKS endpoint, and proving `kubectl` resolves to Keycloak with no static human kubeconfig users.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing surface (mapper bundles, kind bootstrap, federation config, identity consumers) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify SSO resolves exclusively to Keycloak JWTs carrying the ADR 0029 schema (human + workload) — produces: SSO-invariant check — depends-on: [TASK-01]
- TASK-03: Verify the platform-shipped cluster-OIDC mapper lands SA principals in the §6.9 shape identically on kind/EKS/AKS in CI — produces: mapper conformance — depends-on: [TASK-01]
- TASK-04: Verify the kind bootstrap utility serves a reachable OIDC discovery + JWKS endpoint and the workload-identity flow completes — produces: kind-bootstrap check — depends-on: [TASK-03]
- TASK-05: Verify AKS provisioning passes `--enable-oidc-issuer` + `--enable-workload-identity`; EKS needs only the discovery-URL pointer — produces: provisioning check — depends-on: [TASK-01]
- TASK-06: Verify the same projected SA token federates to AWS IRSA / Azure WI for cloud credentials — produces: cloud-federation check — depends-on: [TASK-03]
- TASK-07: Verify `kubectl` resolves to a Keycloak identity with no static human kubeconfig users; verify upstream-IdP mappers absent from platform deliverables — produces: kubectl + scope-split check — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None as platform pieces; the cluster OIDC issuer + `k8-platform` baseline + Keycloak install are environmental prerequisites.
### 3.2 Downstream pieces blocked on this
- B1, A1, A7, A9, A8 (identity consumers), A21 (onboarding populates claims), B13 (VirtualKey binding).
### 3.3 Continuous (non-blocking) inputs
- ADR 0029 (claim schema), ADR 0033 (EKS exercised; AKS deferred; kind dev), ADR 0026 (single-cluster scope); B22 threat model (identity trust boundaries).

## 4. Parallelizable Subtasks
TASK-03→TASK-04 and TASK-06 are serial-ish off the mapper; TASK-02→TASK-07 and TASK-05 run in parallel once TASK-01 lands.

## 5. Test Strategy
- AC-01 → PyTest: human + SA login both yield Keycloak JWTs with ADR 0029 schema; no upstream identity trusted directly.
- AC-02 → PyTest in platform CI: mapper produces §6.9 shape identically on kind/EKS/AKS.
- AC-03 → Chainsaw on kind: bootstrap serves discovery+JWKS; workload-identity flow completes.
- AC-04 → provisioning-lint: AKS flags set; EKS discovery-URL-only.
- AC-05 → PyTest: same SA token obtains AWS IRSA (and Azure WI where applicable) credentials.
- AC-06 → Playwright/PyTest: `kubectl` resolves to Keycloak identity; no static human kubeconfig user.
- AC-07 → scope-lint: upstream-IdP mappers absent; cluster-OIDC mapper present + versioned.
- Fixtures/fakes: kind cluster with bootstrap; Keycloak realm fixture; stub IRSA/Azure WI endpoints for not-yet-landed cloud bootstrap.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0028-identity-federation-chain` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; coordinate with ADR-0029 (shared claim-schema contract); rolls up to main.

## 7. Effort Estimate
M overall. Per-task: TASK-01 S, 02 S, 03 M, 04 M, 05 S, 06 M, 07 S. Critical path: TASK-01 → 03 → 04.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the cluster-OIDC mapper and kind bootstrap lose their conformance contract and workload identity cannot reach the platform plane on kind; identity consumers lose the single-Keycloak-root guarantee. No runtime artifact is deleted, but the workload-identity dev/CI path breaks without the bootstrap — coordinate reversal with ADR-0029.

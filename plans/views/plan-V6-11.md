# PLAN V6-11 — Identity federation [PROPOSED]

> spec: SPEC-V6-11 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [B1, A15, A17, B4, A21]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized by the identity-federation pieces in
wave order. The claim-schema contract (SPEC §4.4) is the load-bearing artifact every privileged
consumer (LiteLLM, OPA, Headlamp, LibreChat) binds to. **A15** (W0) installs oauth2-proxy; **B1**
(W1) configures the Flow-1 SSO redirect and consumes the Platform JWT; **B4/A21** (W1/W3) ship the
cluster-OIDC-side mappers + default ServiceAccounts via `TenantOnboarding` (Flow-2 enrichment);
**A17/ESO** (W2) realizes the trust-bootstrap secret retrieval. As a T0 view, verification centers on
the exact claim schema and the three-flow trust topology, exercised on EKS and on kind (via the
bootstrap utility) for parity.

## 2. Ordered Task List
- **TASK-01:** Bind SPEC §6 invariants to realizing piece IDs — produces: realization trace
  (REQ-01/02/03 → claim schema consumed by LiteLLM/OPA/Headlamp/LibreChat; REQ-04 → B1;
  REQ-05/08 → B4/A21 cluster-OIDC mappers + IRSA/WI adapters; REQ-06 → API-server OIDC config;
  REQ-07 → mapper boundary; REQ-09 → kind bootstrap utility; REQ-10 → A17/ESO) — depends-on: [].
- **TASK-02:** Freeze the §4.4 claim table as the canonical JWT contract and confirm one-to-one with
  ADR 0029 / §6.9 — produces: claim-contract confirmation — depends-on: [TASK-01].
- **TASK-03:** Define the three-flow + per-environment (EKS/AKS/kind) verification path including the
  kind bootstrap-utility parity case — produces: §5 AC→layer map — depends-on: [TASK-01].
- **TASK-04:** Cross-link to V6-09 (claim semantics / tenancy) and route the Azure-deferral +
  signing-key-root risks to B22/F4 — produces: reference + risk-routing set — depends-on: [TASK-01].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
B1, A15, A17, B4, A21 bind to this contract; every Platform-JWT consumer (LiteLLM A1, OPA A7,
Headlamp A9, LibreChat A8) binds to the §4.4 claim schema (constraint dependency, not a build block).
### 3.3 Continuous (non-blocking) inputs
B14 test framework (3-flow PyTest/Chainsaw harness); B22 threat model (single-trust-root analysis);
the `k8-platform` baseline (IRSA trust established once at provisioning); B12 registry
(`platform.security.*` schemas, deferred).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 run concurrently after TASK-01.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by B1/A15/A17/B4/A21 deliverables, verified end-to-end:
- AC-V6-11-01/02 → **PyTest**: every privileged call presents a Keycloak JWT with the §4.4 claims; an
  upstream-IdP-only token is rejected; OPA resource-scope keys off `platform_namespaces`; empty
  `platform_tenants` + `platform-admin` still authorized.
- AC-V6-11-03 → **Playwright**: browser → UI → Keycloak redirect → Platform JWT.
- AC-V6-11-04 → **PyTest/Chainsaw**: SA token exchanged for scoped AWS creds (IRSA) and a Keycloak
  JWT, both anchored on the cluster OIDC issuer.
- AC-V6-11-05 → **PyTest**: `kubectl oidc-login` against a configured API server authenticates via
  Keycloak; RBAC/OPA then decide.
- AC-V6-11-06 → **review/Chainsaw**: cluster-OIDC mappers ship + are tested; no upstream-IdP mapper
  in deliverables.
- AC-V6-11-07 → **Chainsaw**: on kind with the bootstrap utility, Flow 2 + Flow 3 complete; without
  it, Flow 2 cannot complete.
Fixtures: fake Keycloak realm + signing keys; static-JWKS kind fixture; IRSA/STS stub; fake
LiteLLM/OPA/Headlamp/LibreChat JWT consumers until A1/A7/A9/A8 land.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-11-identity-federation` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band

## 7. Effort Estimate
S overall (T0 thoroughness concentrated in the §4.4 claim contract). TASK-01 S, TASK-02 S, TASK-03 S,
TASK-04 S. Critical path: TASK-01 → TASK-02 → TASK-03.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the federation contract and the canonical claim schema
B1/A15/A17/B4/A21 and every JWT consumer bind to. No runtime blast radius; downstream specs lose
their §6.11 / ADR 0029 anchor — high authoring blast radius given T0 fan-out, so revert only with a
replacement claim contract in place.

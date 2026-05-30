# SPEC ADR-0028 — Identity federation chain [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [B1, A1, A7, A9, A8, A21, B13] · adrs: [0028] · views: [6.11]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0028 is a settled decision: the platform federates identity along three flows — **users → Keycloak** (browser SSO, install-owned upstream-IdP mappers), **workloads → cluster OIDC issuer → Keycloak** (platform-owned cluster-OIDC mapper, plus the same projected SA token federating to AWS IRSA / Azure AD Workload Identity), and **`kubectl` → Keycloak** (API server `--oidc-issuer-url`, no static kubeconfig users). The cluster OIDC issuer is the single root for workload identity; Keycloak is the single root for platform identity (machine and human); no platform component trusts upstream identity directly. This SPEC states what honoring that chain obliges: the cluster-OIDC→platform-claim mapper is a platform-shipped, tested deliverable uniform across kind/EKS/AKS; kind requires the bootstrap utility; AKS provisioning passes the OIDC/workload-identity flags; upstream-IdP mappers stay install-owned. It does not re-argue the trust-root choices.

The problem the decision solves: three identity domains (Kubernetes ServiceAccounts, cloud-provider identity, Keycloak platform JWTs) must be tied together coherently. Keeping Keycloak the sole SSO root and the cluster OIDC issuer the sole workload-identity root prevents any component from trusting upstream identity directly and keeps the workload-identity flow uniform across cluster surfaces.

## 2. Scope

### 2.1 In scope
- The obligation that all privileged platform identity resolves to a Keycloak-issued JWT carrying the ADR 0029 claim schema (SSO is exclusively Keycloak — invariant).
- The obligation that the cluster-OIDC → platform-claim mapper is a platform-shipped, maintained, tested deliverable uniform across kind/EKS/AKS.
- The obligation that kind ships a bootstrap utility exposing a working OIDC issuer + JWKS reachable by Keycloak.
- The obligation that AKS provisioning passes `--enable-oidc-issuer` and `--enable-workload-identity`; EKS needs no extra cluster-side OIDC work.
- The obligation that the same projected SA token federates to AWS IRSA / Azure AD Workload Identity for cloud-resource access.
- The obligation that `kubectl` resolves to a Keycloak-authenticated identity (no static kubeconfig users for humans).
- The scope split: upstream-IdP→platform-claim mappers are install-owned (out of scope); cluster-OIDC→platform-claim mappers are platform-owned (in scope).

### 2.2 Out of scope (and where it lives instead)
- The JWT claim schema itself — **ADR 0029** (this ADR produces JWTs conforming to it).
- Upstream-IdP-side protocol mappers (AD/Okta/SAML → platform claims) — install-owned; explicitly out of architecture scope.
- SSO/auth proxy layer config — component **B1** (oauth2-proxy in front of platform UIs).
- Tenant onboarding (the moment new identifiers first appear in claims) — **ADR 0037** / component **A21**.
- RFC 8693 token-exchange shape, JWT caching/refresh in long-running agents, Azure WI bootstrap specifics, the SA→CapabilitySet binding mechanism — deferred (backlog §1.17).
- AKS bootstrap build itself — deferred (ADR 0033); this ADR records the flag requirement it inherits.

## 3. Context & Dependencies

Upstream consumed: none — this is a foundational identity-chain decision. (The cluster OIDC issuer and `k8-platform` baseline are environmental, not platform pieces.)
Downstream consumers: **B1** (SSO/auth proxy), **A1** (LiteLLM auth + virtual-key binding), **A7** (OPA identity-dependent decisions), **A9** (Headlamp UI scoping), **A8** (LibreChat endpoint visibility), **A21** (tenant onboarding populates claims), **B13** (kopf operator issues `VirtualKey`s bound to identity).

ADR decisions honored:
- **ADR 0028** (this) — three federation flows; Keycloak sole platform-identity root; cluster OIDC sole workload-identity root; cluster-OIDC mapper platform-owned; kind bootstrap required.
- **ADR 0029** — JWTs produced by this chain MUST satisfy the §6.9 claim schema; the cluster-OIDC mapper lands SA principals in that same shape.
- **ADR 0033** — EKS is the v1.0-exercised target; AKS architecturally supported (deferred); kind is the dev/CI target requiring the bootstrap.
- **ADR 0026** — identity federation is single-cluster-scoped; no inter-cluster trust path.
- **ADR 0016** — namespace-based tenancy is established through the claims this chain populates.
- **ADR 0030** — the cluster-OIDC mapper bundles version in lockstep with the claim schema (per ADR 0029).
- **ADR 0037** — tenant onboarding is the binding moment for new tenant identifiers in the claims.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
- `VirtualKey` (kopf-operator B13, namespaced) — source-stated `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`. Bound to the Keycloak-issued identity this chain produces.
- `TenantOnboarding` (XRD, namespaced, ADR 0028/0037) — source-stated `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`. The `clusterOIDCClaimMapping` field is where the platform-owned cluster-OIDC mapper configuration is expressed per tenant.

### 4.2 APIs / SDK surfaces
- Cluster OIDC issuer discovery URL + JWKS (environmental surface Keycloak federates against). On kind, the bootstrap utility provides a routable discovery/JWKS endpoint.
- Kubernetes API server `--oidc-issuer-url` family of flags configured to Keycloak (developer `kubectl` flow), supported on EKS/AKS/kind.
- The cluster-OIDC → platform-claim mapper is a platform-shipped artifact (bundle), versioned per ADR 0030; its exact token-exchange (RFC 8693) shape is deferred — `[PROPOSED — not in source]` if detailed.

### 4.3 CloudEvents emitted / consumed
- Identity/tenant lifecycle events ride `platform.tenant.*` (onboarding, namespace association changes) and authn-failure security events ride `platform.security.*` (repeated authn failures) per ADR 0031, §2. Per-event-type names deferred to B12 — `[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts
- Bootstrap secrets (Keycloak admin credentials, JWT signing keys) are fetched via an IRSA-bound ESO ServiceAccount from AWS Secrets Manager on AWS, established once by the `k8-platform` baseline; on kind, ESO uses a local backend and the kind OIDC bootstrap provides the federation surface. These follow ESO's secret handling, not a new platform connection-secret shape.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream **Keycloak** is the IdP/JWT issuer; **ESO** handles bootstrap secrets; AWS **IRSA** / Azure AD **Workload Identity** federate cloud access. The custom platform deliverables are the cluster-OIDC → platform-claim mapper bundles and the kind OIDC bootstrap utility — build-new/config, no fork of Keycloak.)

## 6. Functional Requirements
- REQ-ADR-0028-01: All privileged platform identity flows MUST resolve to a Keycloak-issued JWT carrying the ADR 0029 claim schema; SSO MUST be exclusively Keycloak.
- REQ-ADR-0028-02: The platform MUST ship and maintain a cluster-OIDC → platform-claim mapper that lands ServiceAccount-derived principals in the same §6.9 claim shape as human principals, uniformly across kind, EKS, and AKS.
- REQ-ADR-0028-03: The kind install path MUST include a bootstrap utility that exposes a working OIDC issuer + JWKS reachable by an out-of-cluster Keycloak (patch kubeadm SA issuer/keys; serve discovery + JWKS).
- REQ-ADR-0028-04: AKS provisioning MUST pass `--enable-oidc-issuer` and `--enable-workload-identity`; EKS MUST require no extra cluster-side OIDC work beyond pointing Keycloak at the managed issuer's discovery URL.
- REQ-ADR-0028-05: The same projected SA token MUST federate to AWS IRSA / Azure AD Workload Identity when cloud-resource credentials are needed.
- REQ-ADR-0028-06: Developer `kubectl` access MUST resolve to a Keycloak-authenticated identity via the API server `--oidc-issuer-url` flags; there MUST be no static long-lived kubeconfig users for humans.
- REQ-ADR-0028-07: No platform component MUST trust upstream identity directly; the cluster OIDC issuer MUST be the single workload-identity root and Keycloak the single platform-identity root.
- REQ-ADR-0028-08: Upstream-IdP → platform-claim mappers MUST remain install-owned and out of scope; cluster-OIDC → platform-claim mappers MUST be platform-owned and in scope.

## 7. Non-Functional Requirements
- Security/multi-tenancy (§6.9): tenancy is established through the claims this chain populates (ADR 0029/0016); the cluster-OIDC mapper's SA-namespace→tenant mapping is an architectural commitment testable in platform CI.
- Testability: the cluster-OIDC mapper MUST be testable in the platform's own CI against the supported distributions so a regression (e.g. in the EKS mapper) is caught upstream, not at customer install time.
- Versioning (ADR 0030): cluster-OIDC mapper bundle versions MUST be pinned to the claim-schema version they target; a schema change has a second blast radius across the kind/EKS/AKS mapper bundles.
- Single-cluster scope (ADR 0026): no inter-cluster trust path; one Keycloak realm per install.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set applies to enforcing components (B1, A1, A7, A9, A8, A21, B13) and to the mapper/bootstrap deliverables, not to this decision record.

## 9. Acceptance Criteria
- AC-ADR-0028-01: Honored when a human SSO login and a workload SA login both yield a Keycloak JWT carrying the ADR 0029 claim schema, with no component trusting upstream identity directly. (REQ-01/07)
- AC-ADR-0028-02: Honored when the platform-shipped cluster-OIDC mapper lands an SA principal in the §6.9 claim shape identically on kind, EKS, and AKS in CI. (REQ-02)
- AC-ADR-0028-03: Honored when, on kind, the bootstrap utility serves a discovery + JWKS endpoint Keycloak can reach and the workload-identity flow completes. (REQ-03)
- AC-ADR-0028-04: Honored when AKS provisioning sets the two OIDC/workload-identity flags and EKS works with only the discovery-URL pointer. (REQ-04)
- AC-ADR-0028-05: Honored when the same projected SA token obtains AWS IRSA (and, where applicable, Azure WI) cloud credentials. (REQ-05)
- AC-ADR-0028-06: Honored when a developer's `kubectl` resolves to a Keycloak identity and no static human kubeconfig user exists. (REQ-06)
- AC-ADR-0028-07: Honored when upstream-IdP mappers are absent from platform deliverables while the cluster-OIDC mapper is a platform-shipped, versioned artifact. (REQ-08)

## 10. Risks & Open Questions
- OQ-1 (med): RFC 8693 token-exchange shape, JWT caching/refresh in long-running agents, Azure WI bootstrap specifics, and the SA→CapabilitySet binding mechanism (annotation/label/CRD) are deferred (backlog §1.17); ACs depending on them are partial. `[PROPOSED]`
- R-1 (med): A bug in any single mapper bundle (e.g. EKS) silently malforms SA principals so OPA/LiteLLM/Headlamp see empty principals; mitigated by mandatory platform-CI testing of the mapper across distributions (REQ-02/§7).
- R-2 (low): The kind bootstrap utility is a required, not optional, dev-path component; if it regresses, workload-identity dev/CI breaks — blast radius is the dev/integration path only.

## 11. References
- ADR 0028 (`adr/0028-identity-federation.md`) — the decision enforced here.
- architecture-overview.md §6.11 (identity federation), §6.9 (required Keycloak JWT claims).
- architecture-backlog.md §1.17 (identity federation details), §6.
- Enforcing/related components: B1 (SSO/auth proxy), A1 (LiteLLM), A7 (OPA), A9 (Headlamp), A8 (LibreChat), A21 (tenant onboarding), B13 (kopf operator / VirtualKey).
- ADR 0029 (JWT claim schema), 0033 (EKS+GitHub; AKS deferred; kind dev), 0026 (independent cluster), 0016 (multi-tenancy), 0030 (versioning), 0037 (tenant onboarding).

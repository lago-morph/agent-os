# ADR 0028: Identity federation chain

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform spans three identity domains that must be tied together:
Kubernetes ServiceAccounts (running pods, including Platform Agents),
cloud-provider identity (for access to cloud secret stores and other cloud
resources), and Keycloak-issued platform JWTs (the authoritative identity for
OPA decisions, virtual key claims, LiteLLM, and UI access). To keep the trust
model coherent, this ADR distinguishes two flows up front and is explicit about
what the platform owns versus what it does not.

**1. User identity (humans).** Humans authenticate to **Keycloak** directly via
browser SSO / OIDC. Keycloak brokers the install's upstream IdP (Entra,
Google, Okta, etc.) and applies **Keycloak protocol mappers** to produce a
platform JWT that conforms to the claim schema in ADR 0029. The platform
commits to the claim **schema**, not to the mapper code: authoring of the
upstream-IdP-side mappers is **out of scope** and remains the responsibility
of whoever administers a specific install (architecture-backlog.md § 6
invariant; ADR 0029 unchanged on this point).

**2. Workload / machine identity (pods, Platform Agents, controllers).**
Kubernetes ServiceAccount tokens are minted and signed by the **cluster OIDC
issuer**. Keycloak federates that issuer as an Identity Provider and exchanges
projected SA tokens for platform JWTs that carry the ADR 0029 claim schema.
Unlike the user-identity case, the **cluster-OIDC-side mapper** (ServiceAccount
attributes → platform claims, including CapabilitySet binding) **is in scope**
for this architecture — this is the change in the current revision. The
mapper must work uniformly across the three supported cluster surfaces:

- **EKS** — provides a managed cluster OIDC issuer out of the box (the same
  issuer used by IRSA). No extra cluster-side work is needed beyond pointing
  Keycloak at the discovery URL. This is the v1.0 exercised target
  (cross-ref ADR 0033).
- **AKS** — provides a managed OIDC issuer, but it is **opt-in**: the cluster
  must be created (or updated) with `--enable-oidc-issuer` and
  `--enable-workload-identity`. The issuer URL is then publicly discoverable.
  Architecturally supported, not v1.0-exercised (cross-ref ADR 0033).
- **kind** — does **not** expose an external OIDC issuer out of the box. To
  make the workload-identity flow work in dev and integration, the platform
  ships a **kind-cluster bootstrap utility** that:
  1. patches kubeadm to set `--service-account-issuer` to a stable URL,
     supplies `--service-account-signing-key-file` /
     `--service-account-key-file`, and configures the JWKS URI;
  2. runs a small static mirror (or in-cluster front) for the OIDC discovery
     document and JWKS so that Keycloak — running outside the kind node — can
     reach a routable URL.
  This is required, not optional, because workload-identity development and CI
  integration testing must run on kind (per ADR 0033's kind-as-dev commitment).

**3. Developer cluster access (kubectl).** As a related flow, the **Kubernetes
API server itself** can be configured to use Keycloak as its OIDC provider,
via the API server's `--oidc-issuer-url` family of flags. This means a
developer's `kubectl` resolves to a Keycloak-authenticated identity rather
than a static long-lived kubeconfig user. This flow is in scope for the
architecture, and EKS, AKS, and kind all support it. It uses the same
Keycloak realm as flow (1); it does **not** use the cluster OIDC issuer
(which signs SA tokens, not user tokens).

The cluster OIDC issuer also continues to be the federation root for
**cloud-resource access** via AWS IRSA and Azure AD Workload Identity (used by
ESO and any other in-cluster process needing scoped cloud credentials). That
chain is unchanged from the prior revision of this ADR.

## Decision

The platform federates identity along the flows above:

- **Users → Keycloak** (browser SSO), Keycloak brokers the upstream IdP and
  applies install-owned protocol mappers to produce platform JWTs per
  ADR 0029.
- **Workloads → cluster OIDC issuer → Keycloak**, where Keycloak applies a
  **platform-owned** mapper (this ADR) to produce platform JWTs per ADR 0029.
  The same projected SA token also federates to **AWS IRSA** /
  **Azure AD Workload Identity** when cloud-resource credentials are needed.
- **`kubectl` → Keycloak** (API server `--oidc-issuer-url`) for developer
  cluster access; no static kubeconfig users for humans.

The cluster OIDC issuer is the single root for workload identity; Keycloak is
the single root for platform identity (machine and human). No platform
component trusts upstream identity directly.

## Consequences

- **SSO is exclusively Keycloak (invariant).** All privileged platform
  identity flows resolve to a Keycloak-issued JWT carrying the ADR 0029 claim
  schema (architecture-backlog.md § 6).
- **The cluster-OIDC → platform-claim mapper is now an explicit platform
  deliverable** (new in this revision). It is shipped and maintained by the
  platform, not by individual installs, and must be uniform across kind, EKS,
  and AKS.
- **kind requires a bootstrap utility** that exposes a working OIDC issuer
  + JWKS endpoint reachable by Keycloak. Without it, workload-identity dev
  and integration testing do not work; this is therefore a required component
  of the kind install path, not an optional convenience.
- **AKS provisioning must pass `--enable-oidc-issuer` and
  `--enable-workload-identity`** whenever AKS is used. Recorded here so the
  AKS bootstrap (deferred per ADR 0033) inherits this requirement when it is
  built.
- **EKS** needs no extra cluster-side OIDC work; the managed issuer is
  sufficient. v1.0 exercises this path (ADR 0033).
- **Mapper authoring split**: upstream-IdP → platform-claims mappers (user
  flow) remain **out of scope** and install-owned. Cluster-OIDC →
  platform-claims mapper (workload flow) is **in scope** and platform-owned.
- Trust bootstrap on AWS continues to use an IRSA-bound ESO ServiceAccount to
  fetch Keycloak admin credentials, JWT signing keys, and other bootstrap
  secrets from AWS Secrets Manager; established once during cluster
  provisioning by the `k8-platform` baseline. On kind, ESO uses a local
  backend and the kind OIDC bootstrap above provides the federation surface.
- **Sub-decisions remain deferred (architecture-backlog § 1.17)**: the
  specific RFC 8693 token-exchange shape (subject-token / actor-token /
  audience handling), platform-JWT caching and refresh strategy in
  long-running Platform Agents, Azure Workload Identity bootstrap specifics,
  and the ServiceAccount → CapabilitySet binding mechanism (annotation,
  label, or separate CRD) consumed by the cluster-OIDC mapper.

## References

- [architecture-overview.md § 6.11](../architecture-overview.md#611-identity-federation)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.17](../architecture-backlog.md#117-identity-federation-details), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0029](./0029-keycloak-jwt-claim-schema.md) (required Keycloak JWT claim schema)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (initial implementation targets — EKS + GitHub; AKS deferred; kind for dev)
- [ADR 0037](./0037-tenant-onboarding-xrd.md) (tenant onboarding)
- [ADR 0026](./0026-independent-cluster-install-no-federation.md) (independent-cluster install)
- [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (multi-tenancy via namespaces)

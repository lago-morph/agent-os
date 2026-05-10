# ADR 0028: Identity federation chain

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform spans three identity domains that must be tied together: Kubernetes
service accounts (running pods, including Platform Agents), cloud-provider
identity (for access to cloud secret stores and other cloud resources), and
Keycloak-issued platform JWTs (the authoritative identity for OPA decisions,
virtual key claims, LiteLLM, and UI access). These needs split into two
distinct federations with the same root: cloud-resource access on one side, and
platform-internal identity on the other. The cluster's OIDC issuer is the
natural federation root because it already signs projected ServiceAccount
tokens and is recognized as an OIDC provider by the major cloud IdPs and by
Keycloak's identity-brokering features. Without an explicit decision,
components could be tempted to mint their own credentials or trust upstream
identity directly, which would fragment the trust model.

## Decision

The platform federates identity along two chains, both rooted in the cluster
OIDC issuer. **For cloud resources**, ServiceAccount tokens federate to **AWS
IRSA** or **Azure AD Workload Identity**, which return scoped cloud credentials
used by ESO and any other in-cluster process needing cloud access. **For
platform identity**, ServiceAccount tokens are exchanged (via standard OIDC
token exchange) for a **Keycloak-issued platform JWT**, with Keycloak
federating the cluster OIDC issuer as an Identity Provider and applying
installer-configured mappers to populate the claim schema. The platform JWT is
what LiteLLM, OPA, Headlamp, and LibreChat actually consume; for human users
the chain skips the SA step and starts at Keycloak brokering the upstream IdP.

## Consequences

- **SSO is exclusively Keycloak (invariant).** No platform component trusts
  upstream identity directly; all privileged platform identity flows resolve to
  a Keycloak-issued JWT carrying the section 6.9 claim schema.
- Cluster OIDC is the single federation root: IRSA / Workload Identity is the
  cloud-resource adapter, Keycloak is the platform-identity adapter.
  Components do not invent parallel trust paths.
- **Azure (Workload Identity + Key Vault) is supported architecturally but not
  exercised in v1.0** — initial implementation targets AWS (EKS) and GitHub
  (cross-ref ADR 0033). Azure parity is a configuration concern, not an
  architectural one.
- Trust bootstrap on AWS uses an IRSA-bound ESO ServiceAccount to fetch
  Keycloak admin credentials, JWT signing keys, and other bootstrap secrets
  from AWS Secrets Manager; the IRSA trust is established once during cluster
  provisioning by the `k8-platform` baseline. On kind / dev the cluster runs
  without cloud federation and ESO is configured against a local backend.
- The chain ties directly to the **required Keycloak JWT claim schema**
  (ADR 0029): this ADR commits to the federation topology, ADR 0029 commits to
  what claims must be present in the resulting JWT.
- Mapper authoring (translating upstream IdP attributes into platform claims)
  is **out of scope** for this architecture; whoever administers a specific
  install owns it, since their choice of upstream IdP determines the source
  attributes.
- **Sub-decisions are deferred (architecture-backlog § 1.17)**: the specific
  RFC 8693 token-exchange flow shape (subject-token / actor-token / audience
  handling), caching and refresh strategy for platform JWTs in long-running
  Platform Agents, Azure Workload Identity bootstrap specifics, and the
  ServiceAccount → CapabilitySet binding mechanism (annotation, label, or
  separate CRD).

## References

- architecture-overview.md § 6.11
- architecture-backlog.md § 1.17, 6
- ADR 0029 (required Keycloak JWT claim schema), ADR 0033 (initial implementation targets), ADR 0026 (independent-cluster install)

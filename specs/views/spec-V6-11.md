# SPEC V6-11 — Identity federation [PROPOSED]

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0028, 0029, 0033, 0018, 0031] · views: [6.11]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's identity-federation slice: the
architectural commitment that **all privileged platform identity flows ultimately resolve to a
Keycloak-issued Platform JWT carrying the §6.9 claim schema** (§6.11, ADR 0028 / 0029), and that
**no platform component trusts upstream identity directly**. Three distinct flows — human user
(browser SSO), workload/machine (IRSA / Workload Identity + cluster-OIDC token exchange), and
developer `kubectl` (OIDC plugin) — are tied together through Keycloak as the platform identity
adapter, with the cluster OIDC issuer as the workload-identity federation root.

The slice is realized chiefly by the SSO/auth-proxy configuration (B1) and the federation mechanics
described in ADR 0028; this SPEC does **not** build them. It fixes the invariants, the **exact
Platform JWT claim schema** every consumer binds to, the three-flow trust topology, the
in-scope / out-of-scope mapper boundary, and the per-environment cluster-OIDC-issuer availability
contract each participating component must honor. As a T0 view it is deliberately exhaustive on the
claim schema and the JWT contract, because ~30 component specs bind to those names.

## 2. Scope

### 2.1 In scope
- The invariant that every privileged identity flow resolves to a Keycloak-issued Platform JWT
  carrying the claim schema below; no component trusts upstream identity directly.
- The **exact claim schema** Keycloak must emit (the authoritative table in §4.4) and which
  components consume it (LiteLLM, OPA, Headlamp, LibreChat).
- The three-flow trust topology (human / workload / developer) and the trust roots of each.
- The in-scope vs out-of-scope **mapper boundary**: upstream-IdP mappers (Flow 1) out of scope;
  cluster-OIDC mappers (Flow 2) in scope and shipped + tested by the platform.
- The per-environment cluster-OIDC-issuer availability contract (EKS / AKS / kind) and the kind
  bootstrap utility that brings kind to feature parity.

### 2.2 Out of scope (and where it lives instead)
- SSO/auth-proxy implementation in front of platform UIs — **component B1** (oauth2-proxy config;
  A15 install).
- The `TenantOnboarding` XRD that writes the cluster-OIDC-side claim mapping + ServiceAccounts —
  **components B4 / A21** (V6-09 / ADR 0037).
- The tenancy semantics of the claims (namespace = tenancy boundary, visibility model) — **V6-09**.
- Upstream-IdP mapper authoring (AD/Okta/Google → claim schema) — **out of scope, install owner**.
- ESO secret retrieval mechanics for bootstrap secrets — **component A17 / ESO** (referenced only).
- The `k8-platform` baseline that establishes IRSA trust at cluster provisioning — **companion repo**.
- Azure specifics beyond the architecture-supported pattern — **deferred until Azure is a real
  target** (ADR 0033 scopes v1.0 to AWS + GitHub + kind).

## 3. Context & Dependencies

Realizing components and what each contributes to the view:
- **B1** (SSO/auth proxy layer) — configures oauth2-proxy + the UI redirect-to-Keycloak flow
  (Flow 1) and consumes the Platform JWT to scope UI sessions.
- **A15** (Reloader + oauth2-proxy) — installs the oauth2-proxy that B1 configures.
- **B4 / A21** (TenantOnboarding) — write the cluster-OIDC-side claim mappers + default
  ServiceAccounts wired to the cluster OIDC issuer (Flow 2 enrichment source).
- **ESO** (A17) — fetches Keycloak admin creds, JWT signing keys, bootstrap secrets via an
  IRSA-bound ServiceAccount (trust bootstrap).

ADR decisions honored:
- **ADR 0028** — the identity federation chain / topology (three flows, Keycloak as platform adapter,
  cluster OIDC as workload root).
- **ADR 0029** — the required Keycloak JWT claim schema (the §4.4 table is its realization).
- **ADR 0033** — initial v1.0 implementation targets AWS (EKS) + GitHub + kind; Azure parity is a
  configuration concern, not architectural.
- **ADR 0018** — the claims feed the RBAC-as-floor / OPA-as-restrictor model.
- **ADR 0031** — federation/security events flow under `platform.security.*` (e.g. repeated authn
  failures).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
This view introduces no CRD of its own. It interacts with:
- `TenantOnboarding` (XRD, owner B4) — `clusterOIDCClaimMapping`, `defaultServiceAccounts[]`: the
  Flow-2 enrichment artifacts. Detailed in V6-09.

### 4.2 APIs / SDK surfaces
- **OIDC token exchange** — the ServiceAccount projected token (signed by the cluster OIDC issuer) is
  exchanged for (a) cloud credentials via IRSA / Workload Identity, and (b) a Keycloak Platform JWT
  via standard OIDC token exchange.
- **Keycloak federation** — Keycloak federates the cluster OIDC issuer as an Identity Provider and
  applies cluster-OIDC-side mappers to enrich the JWT.
- **`kubectl` OIDC login** — API server configured with `--oidc-issuer-url=<keycloak-realm>`,
  `--oidc-client-id=<kubectl-client>`, username/groups claim flags; developer uses a kubelogin /
  `kubectl oidc-login` plugin.
- **Cloud-side annotations** (exact, reproduce verbatim): AWS IRSA
  `eks.amazonaws.com/role-arn` (→ STS `AssumeRoleWithWebIdentity` → scoped AWS creds); Azure
  Workload Identity `azure.workload.identity/client-id` (→ Azure AD federated token).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: `platform.security.*` — repeated authn failures, policy-bypass attempts surfaced by the
  identity layer. (Audit of identity events under `platform.audit.*`.) Per-event schemas → B12.
- Consumed: none architecturally owned by this view.

### 4.4 Data schemas / connection-secret contracts — **the Platform JWT claim schema (authoritative)**

Keycloak MUST emit a Platform JWT carrying exactly the following claims (Canon; ADR 0029 / §6.9). All
consumers (LiteLLM, OPA, Headlamp, LibreChat) bind to these names verbatim.

| Claim | Type | Required | Purpose |
|---|---|---|---|
| `sub` | string | yes | Subject identity (user or service-account principal). |
| `iss` | string | yes | Issuer; must be the platform's Keycloak realm. |
| `aud` | string or array | yes | Audience; one of the platform service identifiers (gateway, headlamp, …). |
| `exp`, `iat`, `nbf` | int | yes | Standard JWT lifetime claims. |
| `email` | string | for users | User email; absent for service-account principals. |
| `preferred_username` | string | yes | Display name used in UIs and audit. |
| `platform_tenants` | array&lt;string&gt; | yes | Tenants the principal belongs to. Empty array = "no tenant context" (cross-tenant admins scoped via roles, not absence). |
| `platform_namespaces` | array&lt;string&gt; | yes | Kubernetes namespaces the principal may operate in. **Authoritative for OPA decisions on resource scope.** |
| `platform_roles` | array&lt;string&gt; | yes | Platform-level roles (`platform-admin`, `operator`, `developer`, `viewer`, `auditor`, …). Catalog is design-time; claim presence/shape is architecture-level. |
| `tenant_roles` | object&lt;tenant, array&lt;role&gt;&gt; | for tenant-scoped principals | Per-tenant role mappings (e.g. `{tenant-a: [admin], tenant-b: [developer]}`). Empty for purely platform-level principals. |
| `capability_set_refs` | array&lt;string&gt; | for service-account principals (Platform Agents) | CapabilitySets the principal may bind to. Used by OPA to gate dynamic registration + virtual-key issuance. |

No bootstrap connection-secret schema is owned here; bootstrap secrets (Keycloak admin creds, JWT
signing keys, initial DB passwords) are fetched from AWS Secrets Manager via ESO over an IRSA-bound
ServiceAccount (kind: local secret backend or directly provided).

## 5. OSS-vs-Custom Decision
N/A — VIEW. Realizing pieces: Keycloak (upstream, configured per ADR 0028/0029), oauth2-proxy
(A15 install / B1 config), ESO (A17), and the kind-cluster bootstrap utility (custom, ships with the
platform) carry their own decisions.

## 6. Functional Requirements

- **REQ-V6-11-01** (invariant): Every privileged platform identity flow MUST resolve to a
  Keycloak-issued Platform JWT carrying the §4.4 claim schema. No platform component MUST trust
  upstream identity directly.
- **REQ-V6-11-02** (constraint): Keycloak MUST emit exactly the §4.4 claim set with the stated types
  and required-ness; consumers (LiteLLM, OPA, Headlamp, LibreChat) MUST bind to these claim names
  verbatim.
- **REQ-V6-11-03** (constraint): `platform_namespaces` MUST be authoritative for OPA resource-scope
  decisions; an empty `platform_tenants` array MUST mean "no tenant context", with cross-tenant
  admins scoped via `platform_roles`, not via absence of the claim.
- **REQ-V6-11-04** (constraint): **Flow 1 (human)** — a UI session MUST be established by redirecting
  to Keycloak, which brokers the upstream IdP and applies mappers to produce the Platform JWT every
  UI and service consumes.
- **REQ-V6-11-05** (constraint): **Flow 2 (workload)** — a pod ServiceAccount token signed by the
  cluster OIDC issuer MUST be exchangeable for (a) cloud credentials via IRSA
  (`eks.amazonaws.com/role-arn`) or Azure Workload Identity (`azure.workload.identity/client-id`) and
  (b) a Keycloak Platform JWT via OIDC token exchange, both anchored on the cluster OIDC issuer.
- **REQ-V6-11-06** (constraint): **Flow 3 (developer)** — the API server MUST be configured with
  `--oidc-issuer-url=<keycloak-realm>` + `--oidc-client-id` + username/groups claim flags so a
  kubelogin-style plugin yields a Keycloak JWT with the §4.4 schema, after which RBAC + OPA admission
  decide authorization.
- **REQ-V6-11-07** (constraint): **Mapper boundary** — upstream-IdP mappers (Flow 1) MUST be out of
  scope (install owner); cluster-OIDC-side mappers (Flow 2) MUST be in scope, shipped and tested by
  the platform.
- **REQ-V6-11-08** (constraint): The cluster OIDC issuer is the workload-identity federation root;
  IRSA / Workload Identity is the cloud-resource adapter; Keycloak is the platform identity adapter.
  These three roles MUST remain distinct.
- **REQ-V6-11-09** (constraint): **Per-environment OIDC availability** — EKS provides a managed
  issuer (nothing extra); AKS requires `--enable-oidc-issuer --enable-workload-identity` (the
  platform commits to enabling them); kind has none, so the platform MUST ship a kind-cluster
  bootstrap utility (kubeadm issuer patch + static JWKS hosting + Keycloak federation) to reach
  feature parity for Flows 2 and 3.
- **REQ-V6-11-10** (constraint): Trust bootstrap MUST use ESO over an IRSA-bound ServiceAccount to
  fetch Keycloak admin creds, JWT signing keys, and initial DB passwords from AWS Secrets Manager on
  AWS (local/direct on kind); IRSA trust is established once by the `k8-platform` baseline.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** the claims *are* the tenancy carrier; `platform_namespaces` and
  `tenant_roles` drive every scoped decision. See V6-09.
- **Security:** no component trusts upstream identity directly; the JWT is the only trust token;
  signing keys come from a secret store via ESO; secret-store access is itself identity-federated.
  `platform.security.*` surfaces authn-failure / bypass signals.
- **Observability (§6.5):** `preferred_username` / `sub` feed UI + audit attribution; trace
  correlation is unaffected by identity flow.
- **Versioning (ADR 0030):** the claim schema is an interface contract; additive claims are
  backward-compatible, a breaking change to a required claim is a coordinated change (ADR 0029).

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. Realized by components; oauth2-proxy config, Keycloak realm/mappers, the kind bootstrap
utility, OPA claim-binding integration, and 3-flow tests belong to B1/A15/A17/B4/A21, verified
end-to-end in PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-11-01** (→REQ-01/02): Every privileged call presents a Keycloak-issued Platform JWT
  carrying exactly the §4.4 claims; a request bearing only an upstream-IdP token (no Keycloak JWT) is
  rejected by LiteLLM / OPA / Headlamp / LibreChat.
- **AC-V6-11-02** (→REQ-03): An OPA resource-scope decision keys off `platform_namespaces`; a
  principal with empty `platform_tenants` but a `platform-admin` role is still authorized via roles.
- **AC-V6-11-03** (→REQ-04): A browser hitting a platform UI is redirected to Keycloak and returns
  with a Platform JWT carrying the schema.
- **AC-V6-11-04** (→REQ-05): A pod ServiceAccount token is exchanged for both scoped AWS creds (via
  `eks.amazonaws.com/role-arn` → STS) and a Keycloak Platform JWT, both anchored on the cluster OIDC
  issuer.
- **AC-V6-11-05** (→REQ-06): A developer using `kubectl oidc-login` against a configured API server
  authenticates as themselves via Keycloak, then RBAC/OPA decide actions.
- **AC-V6-11-06** (→REQ-07): The platform ships and tests cluster-OIDC-side mappers; no
  upstream-IdP mapper is included in platform deliverables.
- **AC-V6-11-07** (→REQ-09): On kind, with the bootstrap utility applied, Flow 2 and Flow 3 complete
  end-to-end; without it, kind cannot exercise Flow 2.

## 10. Risks & Open Questions
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review; it coins no new names. The §4.4
  claim table is reproduced verbatim from §6.9 / ADR 0029.
- **(high)** Self-red-team: the cluster OIDC issuer is the single workload-trust root — compromise of
  its signing material federates everywhere. Mitigated by sourcing signing keys from a secret store
  via ESO (REQ-10) and by keeping the three federation roles distinct (REQ-08); residual risk
  flagged for B22 threat model and F4 security review.
- **(med)** Self-red-team: kind's **statically-hosted JWKS** is a parity hack — acceptable for
  dev/integration only; it MUST NOT be used as a production trust root. Flagged; production targets
  EKS/AKS managed issuers (REQ-09).
- **(med)** Azure (Workload Identity + Key Vault) is architecture-supported but **not exercised in
  v1.0** (ADR 0033); specifics TBD. This view fixes the pattern, not the Azure wiring.
- **(low)** The concrete `platform_roles` catalog is design-time; this view fixes the claim's
  presence/shape only.

## 11. References
- architecture-overview.md §6.11 (identity federation, ~L837–941): Flow 1 (~L843–852), Flow 2
  (~L854–874), Flow 3 (~L876–885), diagram (~L889–927), trust bootstrap + commitments (~L929–940);
  §6.9 claim table (~L728–740).
- ADR 0028 (federation topology), ADR 0029 (claim schema), ADR 0033 (AWS+GitHub+kind targets),
  ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0031 (`platform.security.*`).
- Realizing components: B1, A15, A17 (ESO), B4/A21 (cluster-OIDC mappers). Related views: V6-09
  (multi-tenancy — claim semantics), V6-06 (security/policy), V6-12 (CRD inventory).

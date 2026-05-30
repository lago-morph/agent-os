# SPEC B1 — SSO/auth proxy layer

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A1, A2, A8, A9] · downstream: [] · adrs: [0007, 0028, 0029] · views: [6.11, 6.9, 6.1]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

Several platform UIs ship without enterprise SSO in their OSS tiers (LiteLLM admin SSO/SAML is enterprise-only; Langfuse advanced RBAC/SCIM is paid; Argo/Headlamp need OIDC wiring) — none of which can sit directly in front of Keycloak fronting an upstream AD/Okta without a fill-in (§9). B1 is the **SSO/auth proxy layer**: the configuration of **oauth2-proxy** (installed by A15) in front of the platform UIs that lack adequate native SSO — LiteLLM admin, Langfuse, Headlamp, Argo Workflows/ArgoCD — and the OIDC-native wiring where a UI does support Keycloak directly. Its job is to ensure every browser-facing platform UI resolves the user to a Keycloak-issued platform JWT carrying the §6.9 claim schema (ADR 0029) before the UI is reached, per the Flow-1 (human user identity) federation chain in §6.11 (ADR 0028).

B1 is a **configuration + glue** component, not a new daemon: oauth2-proxy the binary is installed by A15; Keycloak is an external (install-owned upstream-IdP-mapped) dependency; B1 owns the per-UI proxy configuration, the Keycloak client/redirect registration shape, the claim-to-header pass-through, and the group-claim-based admin-class gating for LiteLLM. It deliberately does **not** own workload (Flow-2) or kubectl (Flow-3) identity — those resolve elsewhere — only the human-browser-to-UI front door.

## 2. Scope

### 2.1 In scope
- oauth2-proxy configuration in front of each UI lacking adequate native SSO: **LiteLLM admin**, **Langfuse**, **Headlamp**, **Argo Workflows / ArgoCD** (§9 fill-in table; §6.11 Flow 1).
- Keycloak OIDC client registration shape per fronted UI (redirect URIs, scopes, audience) — consuming the platform realm; **upstream-IdP mappers remain out of scope** (§6.9, ADR 0029).
- Pass-through of the §6.9 platform JWT / claims to the upstream UI (header injection / cookie session), so downstream UIs scope their views from `platform_roles`, `platform_tenants`, `platform_namespaces` (ADR 0029).
- LiteLLM admin-class gating: two or three admin classes via **Keycloak group claims** (§9 — "we don't need fine-grained RBAC").
- Per-UI session, logout, and token-refresh behavior at the proxy boundary.

### 2.2 Out of scope (and where it lives instead)
- oauth2-proxy / Reloader **install** — Component A15.
- Keycloak install, realm, and **upstream-IdP-side mappers** — install-owned (§6.9, ADR 0029); not a platform deliverable.
- **Cluster-OIDC-side (workload, Flow 2) mappers** and the kind OIDC bootstrap utility — Component A21 / the identity-federation Flow-2 owners (ADR 0028); B1 is Flow-1 only.
- **kubectl (Flow 3)** API-server OIDC config — `k8-platform` baseline / cluster provisioning (§6.11).
- LibreChat SSO wiring and `librechat.yaml` lockdown — Component **A8** (ADR 0007); LibreChat is SSO'd directly through Keycloak, not via this proxy.
- Mattermost GitLab-OAuth-spoof SSO — Component **A19** (ADR 0036).
- OpenSearch SSO — intentionally excluded; its OSS Security plugin has native OIDC/SAML (§9).
- LiteLLM custom callbacks (audit/PII/OPA) — Component **B2**.

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM)** — the admin UI fronted by oauth2-proxy; admin classes from group claims.
- **A2 (Langfuse)** — UI fronted / OIDC-wired for SSO.
- **A8 (LibreChat)** — consumed only as a sibling: B1 must not duplicate A8's own Keycloak SSO; B1 confirms the boundary. (LibreChat endpoint visibility is driven by the same §6.9 claims — ADR 0007.)
- **A9 (Headlamp install + framework)** — Headlamp's OIDC + auth-handoff hooks; plugin gating logic is DIY in plugin code (§9), but the front-door SSO is B1's.

**ADRs honored:**
- **ADR 0007** — LibreChat is SSO'd directly through Keycloak (not via this proxy); B1 honors the boundary and the §6.9-claim-driven endpoint visibility.
- **ADR 0028** — Flow-1 human identity resolves to a Keycloak JWT; no platform component (including these UIs) trusts upstream identity directly. Upstream-IdP mappers out of scope; B1 consumes the platform JWT only.
- **ADR 0029** — every fronted UI receives / consumes only the §6.9 claim schema (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`).

**Downstream consumers:** none in `piece-index.csv` (B1 downstream empty). Operationally, every browser user of the fronted UIs depends on B1.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — B1 introduces no CRD/XRD. It is proxy + Keycloak-client configuration delivered as Helm values / manifests. Dashboards (if any) follow the `GrafanaDashboard` XR model (ADR 0021) but none are owned here. Versioning of any config API follows ADR 0030 (per-component).

### 4.2 APIs / SDK surfaces
- **Consumes** the Keycloak OIDC discovery + token endpoints (platform realm) and the §6.9 platform JWT. `[PROPOSED — not in source]` the exact oauth2-proxy header names used to pass claims to upstream UIs are config-time, not Canon-named.
- **Exposes** no new platform HTTP API. The proxy is transparent; the fronted UI's own API is unchanged.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Authn-failure / policy-bypass-attempt signals at the proxy boundary, if emitted, MUST fall under `platform.security.*` (repeated authn failures are named in that namespace). `[PROPOSED — not in source]` — no specific event name for proxy-level authn failure exists in Canon; concrete schema lives in B12's registry.
- Audit-relevant access events (who reached which UI), if emitted, fall under `platform.audit.*` via the audit adapter (ADR 0034). Emission is via the audit adapter library, never direct writes.

### 4.4 Data schemas / connection-secret contracts
- Keycloak client secrets / oauth2-proxy cookie secrets are fetched via **ESO** from the secret backend (AWS Secrets Manager on AWS; local on kind) per §6.11 trust bootstrap. No connection-secret XRD shape (ADR 0041) is owned here.
- Consumed claim schema: the §6.9 table verbatim (ADR 0029).

## 5. OSS-vs-Custom Decision

**Wrap / configure**, no new build. Upstream project: **oauth2-proxy** (installed by A15) configured against **Keycloak**; for UIs with adequate native OIDC (Headlamp, Argo) the decision is **configure native OIDC** rather than front with a proxy. Rationale (§9): many OSS tools lack SSO or only support OAuth-with-built-in-user-store, neither of which works against Keycloak fronting AD/Okta — the recurring fill-in is "auth proxy in front." No fork. ADR linkage: ADR 0028/0029 (identity), ADR 0007 (LibreChat handled separately, not by this proxy).

## 6. Functional Requirements

- **REQ-B1-01:** Each in-scope UI (LiteLLM admin, Langfuse, Headlamp, Argo Workflows/ArgoCD) MUST be reachable by a browser user only after authentication resolving to a Keycloak-issued platform JWT carrying the §6.9 claim schema (ADR 0028/0029).
- **REQ-B1-02:** B1 MUST NOT front LibreChat (A8), Mattermost (A19), or OpenSearch — these are excluded per §9 and ADR 0007/0036; B1 documents the boundary.
- **REQ-B1-03:** For UIs fronted by oauth2-proxy, the proxy MUST pass the authenticated principal's §6.9 claims (at minimum `preferred_username`, `platform_roles`, `platform_tenants`, `platform_namespaces`) to the upstream UI so the UI scopes its views.
- **REQ-B1-04:** LiteLLM admin access MUST be gated into two or three admin classes derived from **Keycloak group claims** (§9); no fine-grained per-resource RBAC is implemented at this layer.
- **REQ-B1-05:** B1 MUST consume only the §6.9 claims; it MUST NOT trust upstream-IdP identity directly nor author upstream-IdP mappers (ADR 0028/0029).
- **REQ-B1-06:** Each fronted UI MUST have a registered Keycloak OIDC client with correct redirect URIs, scopes, and `aud` matching the §6.9 `aud` service-identifier expectation.
- **REQ-B1-07:** Proxy secrets (client secret, cookie secret) MUST be sourced via ESO from the environment secret backend, not embedded in manifests (§6.11).
- **REQ-B1-08:** Logout at a fronted UI MUST terminate the proxy session and initiate Keycloak single-logout where the UI supports it.
- **REQ-B1-09:** Token expiry MUST be handled by the proxy (refresh or re-challenge) without leaking an expired-JWT-bearing session to the upstream UI.
- **REQ-B1-10:** Config changes to the proxy/UI client MUST roll via Reloader (A15) consistent with ADR 0035 for services without in-process reload.

## 7. Non-Functional Requirements

- **Security:** no platform component trusts upstream identity directly (ADR 0028); secrets via ESO; cookie secrets rotated. Authn-failure signals available for `platform.security.*`.
- **Multi-tenancy (§6.9):** claims passed through unmodified; tenant/namespace scoping is the consuming UI's job (Headlamp UI scoping, Langfuse project scoping) driven by the §6.9 claims — B1 must not strip or rewrite them.
- **Observability (§6.5):** proxy access/error metrics exported via the platform OTel path; audit-relevant access via the adapter (ADR 0034).
- **Scale:** stateless proxy; horizontally scalable; no per-tenant state at the proxy.
- **Versioning (ADR 0030):** config schema versioned per-component; Keycloak claim-schema dependency pinned to the ADR 0029 schema version.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests — **applicable** (oauth2-proxy values + per-UI Keycloak client manifests, in `envs/<stage>/`).
- Per-product docs (10.5) — **applicable** (SSO wiring guide per fronted UI).
- Runbook (10.7) — **applicable** (SSO outage / Keycloak-down / cookie-secret-rotation runbook).
- Alerts — **applicable** (proxy 5xx, Keycloak-unreachable, authn-failure spike).
- Grafana dashboard (Crossplane XR) — **applicable** (login success/fail per UI) — delivered as a `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — N/A — B1 fronts Headlamp; cross-cutting Headlamp plugins are B5. B1 ships no plugin.
- OPA/Rego integration — **applicable** (whether a principal may reach a UI can be OPA-restricted; minimal in v1.0). `[PROPOSED — not in source]` no named proxy-level OPA policy in Canon.
- Audit emission (ADR 0034) — **applicable** (UI-access events via adapter).
- Knative trigger flow — **applicable** (authn-failure → `platform.security.*`; no v1.0 mandated flow).
- HolmesGPT toolset — **applicable** (SSO-health query helper). `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (Playwright for login flows; PyTest for config/claim-mapping logic; Chainsaw minimal — no CRD).
- Tutorials & how-tos — **applicable** ("front a new UI with platform SSO" how-to).

## 9. Acceptance Criteria

- **AC-B1-01** (REQ-B1-01): Unauthenticated browser request to each in-scope UI redirects to Keycloak and, after login, the UI loads with a valid platform JWT session. (Playwright)
- **AC-B1-02** (REQ-B1-02): LibreChat, Mattermost, and OpenSearch are demonstrably **not** routed through the B1 proxy. (config assertion / Chainsaw)
- **AC-B1-03** (REQ-B1-03): For a fronted UI, the upstream receives headers carrying `preferred_username`, `platform_roles`, `platform_tenants`, `platform_namespaces`. (Playwright/PyTest)
- **AC-B1-04** (REQ-B1-04): A user in LiteLLM admin group A sees admin-class-A capabilities; a user in group B sees class-B; a user in neither is denied. (Playwright)
- **AC-B1-05** (REQ-B1-05): A token from a non-Keycloak issuer or a raw upstream-IdP token is rejected at the proxy. (Playwright)
- **AC-B1-06** (REQ-B1-06): Each fronted UI has a Keycloak client whose redirect URIs and `aud` validate against the §6.9 expectation. (PyTest)
- **AC-B1-07** (REQ-B1-07): No manifest contains a literal client/cookie secret; all resolve via ESO. (Chainsaw/scan)
- **AC-B1-08** (REQ-B1-08): Logout terminates the session and a subsequent request re-challenges. (Playwright)
- **AC-B1-09** (REQ-B1-09): An expired JWT does not reach the upstream UI; the proxy refreshes or re-challenges. (PyTest/Playwright)
- **AC-B1-10** (REQ-B1-10): A proxy/client config change triggers a Reloader-driven rollout. (Chainsaw)

## 10. Risks & Open Questions

- **OQ-B1-1** (med): Exact header/cookie mechanism for claim pass-through per UI is config-time; some UIs (LiteLLM admin) may need claim→native-role translation beyond group mapping. `[PROPOSED — not in source]`.
- **OQ-B1-2** (low): Whether Headlamp and Argo are best served by native OIDC vs. a fronting proxy — per-UI decision; reconciliation: prefer native OIDC where adequate (§9 says Headlamp OIDC is supported), proxy only where SSO is missing.
- **R-B1-1** (med): Keycloak as a single front-door dependency is a shared point of failure for all fronted UIs; mitigated by alerts + runbook; blast radius platform-wide for browser access.
- **R-B1-2** (low): Boundary drift with A8 (LibreChat) / A19 (Mattermost) if either is accidentally proxied; mitigated by AC-B1-02.
- **OQ-B1-3** (low): Whether proxy-level UI-access OPA restriction ships in v1.0 or defers; no Canon-named policy exists. `[PROPOSED — not in source]`.

## 11. References

- architecture-overview.md §6.11 Identity federation (~837), Flow 1 (~843–852), trust bootstrap (~929–933), commitments (~935–940).
- architecture-overview.md §6.9 Required Keycloak JWT claims (~726–742).
- architecture-overview.md §9 OSS limitations / fill-in table (LiteLLM, Langfuse, Headlamp, Argo rows; ~1392–1423).
- architecture-overview.md §6.1 Gateway (virtual-key auth + Keycloak claims, ~179).
- ADR 0028 (identity federation chain), ADR 0029 (Keycloak JWT claim schema), ADR 0007 (LibreChat SSO handled separately).
- Glossary: oauth2-proxy, Keycloak, Platform JWT. Interface-contract §1 (no CRD owned), §2 (`platform.security.*`, `platform.audit.*`).
- Related pieces: A15 (install), A8, A19 (sibling SSO surfaces), B5 (Headlamp plugins).

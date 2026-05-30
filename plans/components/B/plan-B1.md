# PLAN B1 — SSO/auth proxy layer

> spec: SPEC-B1 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: M
> upstream-pieces: [A1, A2, A8, A9] · downstream-pieces: []

## 1. Implementation Strategy

Configure oauth2-proxy (installed by A15) in front of each platform UI that lacks adequate native Keycloak SSO (LiteLLM admin, Langfuse), and wire native OIDC where the UI supports it adequately (Headlamp, Argo/ArgoCD), so every browser-facing UI resolves the user to a Keycloak platform JWT (§6.9, ADR 0028/0029) before reaching the UI. Build per-UI Keycloak OIDC client manifests, proxy claim-pass-through config, and LiteLLM admin-class group gating. Deliver as Helm values + manifests under `envs/<stage>/`, with secrets via ESO and rollout via Reloader. Explicitly preserve the boundary with A8 (LibreChat) and A19 (Mattermost), which own their own SSO.

## 2. Ordered Task List

- **TASK-01:** Inventory fronted-vs-native-OIDC decision per UI — produces: per-UI SSO matrix — depends-on: [].
- **TASK-02:** Define Keycloak OIDC client shape (redirect URIs, scopes, `aud`) per UI — produces: client manifests — depends-on: [TASK-01].
- **TASK-03:** oauth2-proxy config for LiteLLM admin + Langfuse (claim pass-through headers) — produces: proxy values — depends-on: [TASK-02].
- **TASK-04:** Native OIDC wiring for Headlamp + Argo/ArgoCD — produces: OIDC config — depends-on: [TASK-02].
- **TASK-05:** LiteLLM admin-class gating from Keycloak group claims — produces: group→class mapping config — depends-on: [TASK-03].
- **TASK-06:** ESO wiring for client/cookie secrets — produces: ExternalSecret manifests — depends-on: [TASK-02].
- **TASK-07:** Reloader annotations for config rollout — produces: rollout config — depends-on: [TASK-03, TASK-04].
- **TASK-08:** Audit/security event emission at proxy boundary via adapter — produces: emission wiring — depends-on: [TASK-03].
- **TASK-09:** Tests (Playwright login flows; PyTest claim mapping; Chainsaw config/secret assertions) — produces: 3-layer suite — depends-on: [TASK-03..07].
- **TASK-10:** Grafana XR dashboard, alerts, runbook, how-to — produces: cross-cutting deliverables — depends-on: [TASK-09].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- A1 (LiteLLM admin UI to front), A2 (Langfuse UI), A9 (Headlamp OIDC/auth-handoff hooks). A8 consumed as boundary reference (must NOT re-front).
- A15 (oauth2-proxy binary install) — implicit prerequisite (W0).
- Keycloak platform realm — install-owned upstream.

### 3.2 Downstream blocked on this
- None per piece-index.csv. Operationally all browser UI users.

### 3.3 Continuous (non-blocking) inputs
- B14 test framework, B22 security threat model, A18 audit adapter library.

## 4. Parallelizable Subtasks
- TASK-03 (proxy UIs) and TASK-04 (native OIDC UIs) run concurrently after TASK-02.
- TASK-06 (ESO) parallel with TASK-03/04.
- Dashboard/alerts/runbook (TASK-10) parallel with test authoring once config stabilizes.

## 5. Test Strategy
- **Playwright:** AC-B1-01, -03, -04, -05, -08, -09 (login/redirect/claim-header/admin-class/logout/expiry flows).
- **PyTest:** AC-B1-03, -06, -09 (claim-mapping, client-config validation logic).
- **Chainsaw:** AC-B1-02, -07, -10 (boundary, no-literal-secret, Reloader rollout).
- **Fixtures:** fake Keycloak issuer + JWKS for laptop runs (mirrors kind OIDC bootstrap); stub UIs for header-assertion when A1/A2 not yet up.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A1/A2/A8/A9 specs).
### 6.2 PR — `piece/B1-sso-auth-proxy` → base `wave/1`; carries spec-B1 + plan-B1.
### 6.3 Merge order — independent of W1 siblings (B2, B8, B13); wave/1 rolls up to main.

## 7. Effort Estimate
- TASK-01 S, 02 S, 03 M, 04 M, 05 S, 06 S, 07 S, 08 S, 09 M, 10 M. Rollup **M**.
- Critical path: TASK-01 → 02 → 03 → 05 → 09.

## 8. Rollback / Reversibility
Revert the proxy/client manifests; UIs fall back to their prior auth state (most are unusable without SSO, so revert = take fronted UIs offline). No data migration. Reverting does not break downstream pieces (none); operationally blocks browser UI access until re-applied.

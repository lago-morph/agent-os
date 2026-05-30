# SPEC A15 — Reloader + oauth2-proxy

> kind: COMPONENT · workstream: A · tier: T2
> upstream: [] · downstream: [] · adrs: [0007] · views: [6.6]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

A15 is the small-but-real install/configure package for two cross-cutting infrastructure utilities (§14.1, A15 row): **Reloader** (config/secret rolling-restart) and **oauth2-proxy** (the SSO proxy that sits in front of platform UIs). Reloader watches ConfigMaps/Secrets and triggers rolling restarts of dependent workloads when they change, so components that cannot hot-reload pick up new config/secrets without manual intervention — the install half of the dynamic-config story (cross-ref the staged-restart posture in ADR 0035). oauth2-proxy is the SSO proxy install; its detailed configuration against Keycloak in front of LiteLLM, Langfuse, Headlamp, and Argo is owned by **B1** (SSO/auth proxy layer). A15 owns getting both products installed and operable; B1 owns the oauth2-proxy auth configuration.

## 2. Scope

### 2.1 In scope
- Reloader install (Helm values/manifests), pinned version, ArgoCD-synced; annotation-driven watch of ConfigMaps/Secrets for rolling restarts.
- oauth2-proxy **install** (Helm values/manifests), pinned version — the deployable in front of platform UIs.
- Standard Workstream A operate deliverables scaled to a T2 utility: docs, runbook, component-failure alerts, dashboard, audit emission where applicable, tests.

### 2.2 Out of scope (and where it lives instead)
- **oauth2-proxy configuration** against Keycloak (clients, claims, upstream wiring for LiteLLM/Langfuse/Headlamp/Argo) — **B1** (SSO/auth proxy layer; §14.2 B1 row).
- The `LogLevel` reconciler / dynamic toggle mechanics — **ADR 0035** / per-component; A15 only provides the rolling-restart actuator Reloader represents.
- Keycloak itself — baseline / ADR 0028/0029.
- LibreChat config — **A8** (ADR 0007).

## 3. Context & Dependencies

**Upstream consumed (HARD):** none (CSV upstream empty; W0 foundation).

**Downstream consumers:** none in CSV. Effective consumer: **B1** configures the A15-installed oauth2-proxy; ADR 0035's staged-restart path relies on Reloader as the actuator for non-reloadable services. `[PROPOSED — not in source]` these are functional relationships, not CSV edges.

**ADR decisions honored:**
- **ADR 0007** — oauth2-proxy is the SSO proxy in front of platform UIs (glossary; ADR 0007 context — Keycloak SSO for the UIs). A15 installs it; B1 configures it.
- **ADR 0035** — non-reloadable services use a staged restart; Reloader is the rolling-restart mechanism A15 provides.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — A15 owns no CRD/XRD. Reloader is annotation-driven on existing workloads; oauth2-proxy is a Deployment.

### 4.2 APIs / SDK surfaces
- oauth2-proxy presents the standard SSO reverse-proxy surface (auth callback, upstream proxying); its auth config is B1's. `[PROPOSED — not in source]` no platform-specific API is added by A15.
- Reloader exposes no API beyond its annotation contract.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- N/A for v1.0 emission by A15 itself. Restart actions Reloader triggers are observable via standard Kubernetes events; if A15 emits audit, it is under **`platform.audit.*`** via the adapter. `[PROPOSED — not in source]` no dedicated event type is defined here (deferred to B12 if needed).

### 4.4 Data schemas / connection-secret contracts
N/A — neither utility owns platform data or produces a substrate connection secret.

## 5. OSS-vs-Custom Decision

- **Upstream projects:** Reloader and oauth2-proxy (glossary, exact casing). **Mode: config**, install unmodified at pinned versions; no fork, no wrap beyond Helm values. oauth2-proxy's substantive configuration is B1.
- **Rationale:** both are standard OSS utilities; the work is real but small (§14.1 A15 row).
- `[PROPOSED — not in source]` exact versions/chart coordinates not in Canon; pinned at install.

## 6. Functional Requirements

- REQ-A15-01: A15 installs Reloader via Helm at a pinned version, ArgoCD-synced; annotated workloads roll-restart when their watched ConfigMap/Secret changes.
- REQ-A15-02: A15 installs oauth2-proxy via Helm at a pinned version as the SSO proxy deployable in front of platform UIs; auth configuration is left to B1.
- REQ-A15-03: A15 delivers docs, an operator runbook, component-failure alert rules, and a dashboard for both utilities.
- REQ-A15-04: Reloader is available as the rolling-restart actuator for the staged-restart path of non-reloadable services (ADR 0035).

## 7. Non-Functional Requirements

- **Security:** oauth2-proxy fronts UIs; A15's install must not expose an unauthenticated upstream (auth correctness is B1's, but A15 must not ship an open default). Reloader's restart permissions are least-privilege (restart only annotated workloads).
- **Observability (§6.5):** both emit standard metrics; alerts on Reloader/oauth2-proxy unavailability.
- **Versioning (ADR 0030):** per-component pin.

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm values / manifests | Applicable |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable |
| Backup / restore | N/A — stateless utilities |
| Alert rules | Applicable — both utilities down |
| Grafana dashboard (Crossplane XR) | Applicable |
| Headlamp plugin | N/A — no useful per-component plugin for these utilities |
| OPA / Rego integration | N/A — A15 owns no CRD requiring admission policy (oauth2-proxy auth policy is B1) |
| Audit emission (ADR 0034) | Applicable (light) — restart/proxy actions auditable via adapter where relevant |
| Knative trigger flow | N/A — no CloudEvents authored for these utilities in v1.0 |
| HolmesGPT toolset | Applicable (light) — restart-status / proxy-health queries |
| 3-layer tests | Applicable — Chainsaw (Reloader restart-on-change), PyTest (install/health); Playwright N/A (no own UI) |
| Tutorials & how-tos | Applicable (light) |

## 9. Acceptance Criteria

- AC-A15-01 (REQ-A15-01): Changing a watched Secret triggers a rolling restart of the annotated workload; an unannotated workload is untouched.
- AC-A15-02 (REQ-A15-02): oauth2-proxy installs and serves at a pinned version; no upstream is proxied unauthenticated by default before B1 configuration.
- AC-A15-03 (REQ-A15-03): Docs, runbook, alert rules, and a dashboard exist for both utilities.
- AC-A15-04 (REQ-A15-04): A non-reloadable service annotated for Reloader restarts as part of a staged-restart exercise (ADR 0035 actuator).

## 10. Risks & Open Questions

- R-A15-1 (low): split ownership with B1 (install vs config) risks a gap where oauth2-proxy is installed but mis-fronts a UI. Mitigation: A15 ships a deny-by-default posture; B1 owns the working config. Reconciliation note: glossary assigns "A15 install, B1 config" explicitly.
- R-A15-2 (low): Reloader restart blast radius if over-broad annotations are applied. Mitigation: least-privilege RBAC + annotation discipline documented in the runbook.
- Open question (low): does A15 emit any dedicated CloudEvent, or rely solely on Kubernetes events + adapter audit? `[PROPOSED — not in source]`; default to no dedicated event type for v1.0.

## 11. References

- architecture-overview.md §14.1 (A15 row 1681), §6.6 (security/SSO context), §6.5 (observability).
- ADR 0007 (LibreChat/SSO context — oauth2-proxy as the SSO proxy); ADR 0035 (staged restart for non-reloadable services — Reloader actuator); ADR 0030 (versioning).
- interface-contract §2 (CloudEvent taxonomy), §5 (audit adapter). glossary (Reloader, oauth2-proxy — "A15 install, B1 config"). Related: B1, A8, Keycloak baseline.

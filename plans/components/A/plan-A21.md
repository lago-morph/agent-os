# PLAN A21 — Tenant onboarding reconciler

> spec: SPEC-A21 · kind: COMPONENT · tier: T1
> wave: W3 · estimate: M
> upstream-pieces: [A9, A22] · downstream-pieces: []

## 1. Implementation Strategy

Build the `TenantOnboarding` XRD + Composition as declarative Crossplane artifacts on the B4 composition runtime, then build the Headlamp editor on the A22 framework. Develop the composition first (the reconcile behaviour — namespace + default SAs + cluster-OIDC mapper, plus symmetric offboarding teardown) against a Chainsaw-driven kind cluster, since that is the load-bearing tenancy primitive; the editor is a thin claim-driven layer on top. Keep identity and capability decoupled by construction (no CapabilitySet reference anywhere in the XRD or composition). Mock the Keycloak/cluster-OIDC integration for the claim-mapper record until B1/install-admin surfaces land. Coordinate the A21/A22 and A21/B4 ownership boundaries (OQ-A21-1/-2) before cutting code so the composition manifest and editor are each built once.

## 2. Ordered Task List

- **TASK-01:** Define the `TenantOnboarding` XRD (`tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`), namespaced, `v1alpha1` — produces: XRD manifest — depends-on: [] (B4 runtime available, W1).
- **TASK-02:** Composition — namespace creation with platform tenancy-metadata labels — produces: namespace-provisioning composition — depends-on: [TASK-01].
- **TASK-03:** Composition — templated default ServiceAccounts wired to cluster OIDC issuer — produces: SA-provisioning composition — depends-on: [TASK-01].
- **TASK-04:** Composition — cluster-OIDC-side claim mapper record (no upstream-IdP mapper) — produces: claim-mapper composition — depends-on: [TASK-01]. (Mock Keycloak/cluster-OIDC integration.)
- **TASK-05:** Offboarding — deletion reconciles namespace teardown + SA cleanup + audit-reference archive + operator notification (Headlamp suggestion-card / Mattermost); CapabilitySets + audit data untouched — produces: offboarding behaviour — depends-on: [TASK-02, TASK-03, TASK-04].
- **TASK-06:** `TenantOnboarding` Headlamp editor on the A22 framework — schema validation, claim-mapping proposal, multi-namespace handling, diff-against-Git, PR submission, CapabilitySet-seeding recommendation (UX only) — produces: editor plugin — depends-on: [A22, TASK-01].
- **TASK-07:** Audit emission (onboarding/offboarding via adapter) + `platform.tenant.*` lifecycle events — produces: audit + event wiring — depends-on: [TASK-05].
- **TASK-08:** OPA Rego — admission gating who may create/delete `TenantOnboarding` + namespace-label conformance — produces: Rego contribution (B3/B16) — depends-on: [TASK-02].
- **TASK-09:** Cross-cutting — alerts, `GrafanaDashboard` XR, runbook, HolmesGPT tenant-list tool, per-product docs — depends-on: [TASK-05, TASK-06, TASK-07].
- **TASK-10:** 3-layer tests (Chainsaw reconcile + offboarding, Playwright editor, PyTest claim-mapping logic) mapping all ACs — depends-on: [TASK-05, TASK-06, TASK-07].
- **TASK-11:** Tutorials & how-tos ("Onboard a tenant") — depends-on: [TASK-09].
- **TASK-12:** XRD versioning machinery — `v1alpha1`→`v1` path + conversion webhook (ADR 0030/0041) — produces: version path — depends-on: [TASK-01].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A9** — Headlamp base install + auth handoff (editor surface).
- **A22** — editor framework / shared widgets the `TenantOnboarding` editor is built on.
- **B4** (Crossplane Compositions runtime) — composition engine + `TenantOnboarding` composition surface (W1 foundation; consumed for reconcile). Coordinate manifest ownership (OQ-A21-1).

### 3.2 Downstream blocked on this
- None at the piece level (CSV downstream empty). At runtime, every tenant-scoped Platform Agent / Sandbox / audit emission consumes the provisioned namespaces + SAs.

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework (Chainsaw/Playwright/PyTest). **B22** security threat model (tenant boundary is a primary control). **B1 / install-admin** Keycloak + cluster-OIDC integration (mock until landed). **F1** audit retention (offboarding archives references; expiry deferred).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02, TASK-03, TASK-04, TASK-12 are independent composition/versioning fan-out, and TASK-06 (editor) parallelizes once the XRD schema exists.
- TASK-08 (OPA), TASK-09 (cross-cutting), TASK-11 (docs) parallelize once their inputs exist.

## 5. Test Strategy

- **Chainsaw (operator/CRD):** AC-A21-01 (schema + namespaced-only), AC-A21-02 (multi-namespace + labels), AC-A21-03 (SAs + OIDC wiring), AC-A21-04 (claim mapper, no upstream mapper), AC-A21-05 (zero CapabilitySets), AC-A21-09/-10 (offboarding teardown + decoupling/retention), AC-A21-12 (version conversion).
- **Playwright (UI/e2e):** AC-A21-06 (PR-only, no cluster write), AC-A21-07 (schema block + diff), AC-A21-08 (CapabilitySet recommendation declinable).
- **PyTest (logic):** AC-A21-11 (audit + `platform.tenant.*` emission), claim-mapping computation logic.
- **Fakes/fixtures:** mock Keycloak/cluster-OIDC for the claim mapper (OQ-A21-3); Git/cluster divergence fixture for the editor diff (reuse A22 fixture); mock GitHub PR API; mock Mattermost/suggestion-card sink for the offboarding notification. Replace with real surfaces when landed and re-run.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/3` (contains A22 from wave/2 + B4 W1; A21 bases on merged A22).
### 6.2 PR — `piece/A21-tenant-onboarding-reconciler` → base `wave/3`; carries SPEC-A21 + PLAN-A21.
### 6.3 Merge order — independent of W3 siblings; wave/3 rolls up to main after A22/B4 are in the base.

## 7. Effort Estimate

- XRD + composition (TASK-01..05): M (offboarding teardown + claim mapper are the trickiest).
- Editor (TASK-06): S–M (thin layer on A22).
- Cross-cutting + tests + docs (TASK-07..12): M.
- **Rollup: M** (matches CSV). **Critical path:** TASK-01 → 02/03/04 → 05 (offboarding) → 10 (tests).

## 8. Rollback / Reversibility

Back out by removing the `TenantOnboarding` XRD + Composition and unregistering the editor; existing tenant namespaces/ServiceAccounts created by prior reconciles persist (they are real cluster objects) and revert to manual/runbook management. Reverting the editor leaves Git-authored tenant definitions intact (plain manifests). **Downstream break:** none at the piece level; operationally, onboarding/offboarding returns to the manual `kubectl` + Keycloak-admin runbook the component replaced. Archived audit references are unaffected by rollback (retained per F1).

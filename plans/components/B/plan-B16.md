# PLAN B16 — Initial OPA policy library content

> spec: SPEC-B16 · kind: COMPONENT · tier: T0
> wave: W2 · estimate: L
> upstream-pieces: [B3, A1, A6, A5, B13] · downstream-pieces: []

## 1. Implementation Strategy
Fill the B3 framework with v1.0 Rego content, decision-point by decision-point, each conforming to
B3's bundle layout, decision-document contract, dry-run hook, and test convention. Build the
**admission** policies first (all v1.0 CRDs/XRDs + the substrate-mismatch rule) since they need only
B3 + the CRD shapes; then the **runtime** policies (LiteLLM callbacks, egress) that bind to A1/A6 and
to B13-reconciled OPA `data`; then the **gating/elevation** policies (Headlamp action gates,
`approval.elevation`); then multi-tenancy narrowing. Every policy is authored as an OPA-as-restrictor
(never grants past the B3 floor predicate) and ships with Rego unit tests and a dry-run path. A
coverage map + a B22 extension point are maintained throughout so threat-model output (ADR 0027)
lands additively.

## 2. Ordered Task List
- **TASK-SIGN:** Implement cryptographic signing of policy bundles + load-time verification; engine refuses unsigned/tampered bundles (#25). The decision-document format is owned by B3, not B16 (D-03).
- **TASK-ROLLOUT:** Implement staged rollout — new/changed bundles run in audit mode first, then flip to enforce after review, gated by the Kargo promotion pipeline (#26). Global/cross-tenant bundle changes go through PR + security-reviewer approval (#24).
- **TASK-01:** Bind to B3's frozen decision-document/input contract + bundle layout; stand up the B16
  package skeleton and coverage-map scaffold — produces: B16 bundle skeleton + coverage map — depends-on: []
- **TASK-02:** Admission policies for all ARK/sandbox/kopf CRDs (`Agent`, `AgentRun`, `Sandbox`,
  `SandboxTemplate`, `Memory`, `MemoryStore`, `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`,
  `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`, `Approval`, `LogLevel`) — produces: admission
  Rego + tests — depends-on: [TASK-01]
- **TASK-03:** Admission policies for the Crossplane XRs (`AgentEnvironment`, `SyntheticMCPServer`,
  `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `AgentDatabase`, substrate XRDs) — produces:
  XR admission Rego + tests — depends-on: [TASK-01]
- **TASK-04:** **Substrate-mismatch** admission rule (ADR 0044) using B4's supplied input shape —
  produces: substrate-mismatch Rego + test — depends-on: [TASK-03]
- **TASK-05:** LiteLLM-callback runtime policies: tool/model authorization, rate limiting, content
  checks, **budget enforcement** (reads B13-reconciled `BudgetPolicy` data) — produces: gateway Rego +
  tests — depends-on: [TASK-01]
- **TASK-06:** Dynamic-registration policy (A2A/MCP interface registration in-namespace) — produces:
  registration Rego + test — depends-on: [TASK-05]
- **TASK-07:** Egress policies (allowed destinations per agent class, over `EgressTarget` FQDN
  allowlist) — produces: egress Rego + tests — depends-on: [TASK-01]
- **TASK-08:** Headlamp action-gating policies (card approval, virtual-key issuance, budget edits,
  privileged ops) + virtual-key-issuance/capability-access narrowing — produces: gating Rego + tests —
  depends-on: [TASK-01]
- **TASK-09:** `approval.elevation` policy (elevate-only, never lower) consumed by B19-core — produces:
  elevation Rego + test — depends-on: [TASK-01]
- **TASK-10:** Multi-tenancy narrowing + cross-namespace capability-publication check (JWT claims) —
  produces: tenancy Rego + tests — depends-on: [TASK-01]
- **TASK-11:** RBAC-floor lint/test gate — proves no allow-rule grants past the B3 floor predicate (a
  planted granting rule must fail) — produces: floor-conformance CI gate — depends-on: [TASK-02..TASK-10]
- **TASK-12:** Dry-run/`simulated` coverage across every entrypoint (ADR 0038) + `platform.policy.*`
  audit-content verification — produces: dry-run tests — depends-on: [TASK-02..TASK-10]
- **TASK-13:** Coverage map completion (every §6.6 decision point → package) + **B22 extension point** +
  operability `GrafanaDashboard` XR + alerts + docs + how-tos — produces: coverage map, dashboard,
  docs — depends-on: [TASK-11, TASK-12]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **B3** — framework: bundle layout, shared library (floor predicate, claim extraction, decision
  constructor, deny-aggregation, dry-run wrapper), decision contract, test scaffold.
- **A1 (LiteLLM)** — runtime callback decision point. **A6 (Envoy/sandbox)** — egress decision point.
- **A5 (ARK)** — the CRDs admission gates. **B13 (kopf)** — reconciles `BudgetPolicy`/capability/
  `VirtualKey` into OPA `data` B16 reads.
### 3.2 Downstream pieces blocked on this
- None as a build piece (no CSV downstream). Enforced by A7/Gatekeeper, B2, A6, A9/A22, B19-core, A20.
### 3.3 Continuous (non-blocking) inputs
- **B22 (threat model)** — sets B16's OPA policy *targets* (ADR 0027); absorbed via the extension point
  (REQ-B16-13) as B22 publishes. **B14 (test framework)** — Rego/PyTest harness.

## 4. Parallelizable Subtasks
- After TASK-01, the policy families fan out independently: **TASK-02/03 (admission)**,
  **TASK-05/06 (gateway)**, **TASK-07 (egress)**, **TASK-08 (gating)**, **TASK-09 (elevation)**,
  **TASK-10 (tenancy)** — one author per surface.
- TASK-11 (floor gate) and TASK-12 (dry-run) are cross-cutting and join after the families land.

## 5. Test Strategy
- **Rego unit tests** (per policy, B3 convention): every REQ-B16-01..10 family.
- **Chainsaw (operator/CRD):** AC-B16-01 (admit/deny each CRD against real objects), AC-B16-02
  (substrate-mismatch deny).
- **PyTest (logic):** AC-B16-03 (budget data fixtures), AC-B16-10 (floor lint gate — planted granting
  rule fails), AC-B16-11 (dry-run no-side-effect), AC-B16-13 (coverage map completeness).
- **Playwright:** N/A — no UI (gating *decisions* are tested as Rego/PyTest; the UI is A22).
- **Fixtures/fakes:** B13-reconciled OPA `data` faked until B13 lands; Platform JWT claim fixtures for
  tenancy/gating; an A20 simulator stub for the dry-run path; B19 elevation consumer faked.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains B16's W2 siblings + the W1 specs it binds to).
### 6.2 PR — `piece/B16-opa-policy-content` → base `wave/2`; carries SPEC-B16 + PLAN-B16 + Rego content.
### 6.3 Merge order — independent of W2 siblings; depends on B3 (W1) merged first. Policy-family sub-PRs
can stack under the piece branch; rolls up `wave/2` → main.

## 7. Effort Estimate
- TASK-01 S, TASK-02 L, TASK-03 M, TASK-04 S, TASK-05 L, TASK-06 S, TASK-07 M, TASK-08 M, TASK-09 S,
  TASK-10 M, TASK-11 M, TASK-12 M, TASK-13 M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 → (TASK-02/05) → TASK-11/12 → TASK-13.

## 8. Rollback / Reversibility
Rollback by reverting the B16 bundle revision via B3's SHA-pinned ArgoCD path — OPA falls back to the
prior bundle (fail-safe: with no B16 content, admission/runtime decisions default to the framework's
deny-or-RBAC-floor behaviour, never to grant). **Downstream impact if reverted:** enforcement surfaces
lose their v1.0 restriction rules (budget, egress, elevation, tenancy) — the platform falls back to
RBAC-floor only, which is *safe but coarser* (no widening). Individual policy packages are
independently reversible because they are partitioned per decision point.

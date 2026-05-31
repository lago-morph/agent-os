# PLAN A8 — LibreChat (locked-down config; OpenAI ↔ A2A adapter if needed)

> spec: SPEC-A8 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: L
> upstream-pieces: [] · downstream-pieces: [B1]

## 1. Implementation Strategy
Install LibreChat unmodified via Helm and drive the whole locked-down posture through
`librechat.yaml` (native agents/plugins/MCP off; endpoint picker + history + upload on). Layer the
file-upload pipeline on top: route to the `Memory` CRD (ADR 0025), gate through OPA's
allowed/forbidden/scan-then-allow model, and invoke the approval step (ADR 0017) when policy
demands it. Carry **both** code paths for the OpenAI ↔ A2A translation — none (LiteLLM handles it)
and the small A8-owned Python adapter — selecting at install per ADR 0007. Build against mocks for
LiteLLM translation, the `Memory` CRD, the scan callback, and B19 approval per the §10
documentation-before-implementation / mock-out process. Deliver the §14.1 set.

## 2. Ordered Task List
- **TASK-01:** Helm values + ArgoCD application for unmodified LibreChat — produces: install manifests — depends-on: [].
- **TASK-02:** Author locked-down `librechat.yaml` (agents/plugins/MCP off; upload on) — produces: config profile — depends-on: [TASK-01].
- **TASK-03:** Keycloak SSO + JWT-claim endpoint scoping — produces: SSO + scoping wiring — depends-on: [TASK-01].
- **TASK-04:** Upload → `Memory` CRD routing — produces: upload-to-memory wiring — depends-on: [TASK-02].
- **TASK-05:** OPA three-outcome upload gate (allowed/forbidden/scan-then-allow) + abuse limits — produces: upload-gate integration — depends-on: [TASK-04].
- **TASK-06:** Approval-step invocation on scan-then-allow (ADR 0017) — produces: approval wiring — depends-on: [TASK-05].
- **TASK-07:** OpenAI ↔ A2A adapter (conditional Python service) + install-time selection logic — produces: adapter + selector — depends-on: [TASK-01].
- **TASK-08:** Streaming preservation validation across OpenAI-compat ↔ A2A — produces: streaming test — depends-on: [TASK-07].
- **TASK-09:** LibreChat state on `Postgres` + backup/PITR inheritance — produces: state wiring — depends-on: [TASK-01].
- **TASK-10:** Audit emission of conversation events; dashboard XR + alerts — produces: audit + dashboard/alerts — depends-on: [TASK-02].
- **TASK-11:** Three-layer tests + docs/runbook/tutorial — produces: tests + docs — depends-on: [TASK-03, TASK-05, TASK-06, TASK-08, TASK-09, TASK-10].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None declared (W0). Functional bindings (mock until landed): A1 LiteLLM (translation),
  A5 `Memory` CRD, B4 `Postgres`, Keycloak baseline, B19 approval, B3/B16 upload OPA bundle.
### 3.2 Downstream pieces blocked on this
- B1 (SSO/auth proxy in front of LibreChat).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat-model standards; A16 (default endpoint — runtime, not build-blocking).

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-07, TASK-09 run concurrently.
- Upload chain TASK-04 → TASK-05 → TASK-06 is serial.
- TASK-08 depends only on TASK-07; TASK-10 depends only on TASK-02 — both parallel to the upload chain.

## 5. Test Strategy
- **Chainsaw:** AC-A8-01 (install), AC-A8-06 (approval via stub `Approval`), AC-A8-08 (adapter present/absent install paths).
- **Playwright:** AC-A8-02 (locked-down surface), AC-A8-03 (claim-scoped endpoints), AC-A8-07 (streaming render).
- **PyTest:** AC-A8-04 (upload→Memory), AC-A8-05 (OPA outcomes/abuse limits), AC-A8-09 (state persistence), AC-A8-10 (audit), and the logic half of AC-A8-08.
- **Fixtures/fakes:** mock LiteLLM (translation on/off), fake `Memory` CRD + backing store, stub
  OPA decision returning each of the three outcomes, stub scan callback, stub `Approval` (B19), mock Keycloak.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A8-librechat` → base `wave/0`; carries spec-A8.md + plan-A8.md.
### 6.3 Merge order — independent of W0 siblings; wave/0 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 M · TASK-06 M · TASK-07 L · TASK-08 M · TASK-09 S · TASK-10 M · TASK-11 L.
- Rollup: **L**. Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-05 → TASK-06 → TASK-11
  (the upload-gate-then-approval chain), with the conditional adapter (TASK-07) as a parallel pole.

## 8. Rollback / Reversibility
Remove the ArgoCD application (and the adapter Deployment if present). LibreChat state in Postgres
is retained/dropped by choice. Downstream impact: end-user chat goes dark; B1 has nothing to front.
Operator chat (Mattermost, A19) and the agent runtime are unaffected — LibreChat is a frontend, not
on any agent execution path.

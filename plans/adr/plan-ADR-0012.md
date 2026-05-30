# PLAN ADR-0012 — HolmesGPT as a first-class Platform Agent [PROPOSED]

> spec: SPEC-ADR-0012 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A14] · downstream-pieces: [A16, B10, A1, A7, A18, B19]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0012 is enforced primarily by **A14** (HolmesGPT installed/configured as a Platform Agent: `Agent` CRD, A2A `exposes`, sandbox run, observability toolset, AlertManager trigger flow, toolset deliverable), with the governance layers enforced by **A1** (LiteLLM routing), **A7** (one central OPA decision point for the three-state model), **B19** (upon-approval routing via `Approval`), and **A18** (audit emission). The "every component contributes a HolmesGPT toolset" invariant is enforced by the cross-cutting checklist in every component SPEC §8. Conformance is tested by proving the bootstrap read-only guardrails, the rewired steady-state path (LiteLLM+OPA+audit+sandbox), the three-state classification at a single OPA point, A2A handoff, KB-via-CapabilitySet, and OPA-gated toggle requests.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify A14 SPEC carries Agent-CRD + A2A + sandbox + observability-toolset + AlertManager-flow obligations — produces: A14 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify the single central OPA decision point + three-state model in A7/B16 content — produces: OPA conformance note — depends-on: [TASK-01]
- TASK-04: Verify upon-approval routes solely through B19 `Approval` and denied emits audit — produces: approval/audit trace — depends-on: [TASK-01]
- TASK-05: Define conformance tests for bootstrap read-only guardrails vs rewired steady-state path — produces: Chainsaw+PyTest set — depends-on: [TASK-02, TASK-03]
- TASK-06: Verify "every component contributes a HolmesGPT toolset" §8 obligation platform-wide — produces: cross-component audit — depends-on: [TASK-01]
- TASK-07: Define AlertManager → HolmesGPT trigger-flow conformance test — produces: Chainsaw test — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A14 — HolmesGPT install/config as the Platform Agent enforcing every obligation.
### 3.2 Downstream pieces blocked on this
- A16 (handoff endpoint), B10 (Coach observation).
### 3.3 Continuous (non-blocking) inputs
- A1 (LiteLLM), A7 (OPA), A18 (audit), B19 (Approval) — enforcement deps; B22 threat model (ADR 0027).

## 4. Parallelizable Subtasks
TASK-03, TASK-04, TASK-06 fan out independently once TASK-01 lands. TASK-05 depends on TASK-02+TASK-03; TASK-07 parallel to TASK-05.

## 5. Test Strategy
- AC-01/AC-07 → Chainsaw (Agent CRD reconciled, sandbox-owned) + PyTest.
- AC-02 → Playwright/PyTest A2A handoff through LiteLLM.
- AC-03 → Chainsaw: remove KB RAGStore from CapabilitySet, assert access lost.
- AC-04 → PyTest: single OPA point classifies a sampled action set, no second allowlist.
- AC-05 → Chainsaw: upon-approval creates `Approval`, blocks; denied writes audit.
- AC-06 → PyTest: bootstrap config write attempts fail, IAM has no secret-store reach; post-rewire path produces LiteLLM+OPA+audit records.
- AC-08 → Chainsaw: AlertManager alert → filtered CloudEvent → HolmesGPT finding.
- AC-09 → PyTest: toggle request denied/granted by OPA per identity.
- AC-10 → PyTest doc-lint over component specs §8.
- Fixtures/fakes: stub LiteLLM, OPA decision endpoint, audit endpoint, AlertManager source for not-yet-landed A1/A7/A18/A4.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0012-holmesgpt-first-class-agent` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 S, 04 S, 05 M, 06 S, 07 S. Critical path: TASK-01 → 02 → 05.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, A14 loses its conformance contract for the three-state model and bootstrap→rewire trajectory; no runtime artifact is deleted. Write-capability expansion remains reversible per-capability via the central OPA bundle.

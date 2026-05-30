# PLAN C3 — Diataxis how-to guides

> spec: SPEC-C3 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; each guide authored before/alongside its component) · estimate: L
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy

Author the §10.2 how-to set as docs-as-code into C1's how-to navigation slot, one guide per §10.2
item, each problem-oriented with goal/prereqs/steps/verification. Treat the **GitHub Actions
reference pipeline** guide as the flagship: document the security-first posture (ADR 0010) as native
syntax around the `agent-platform` CLI subcommand contract, validated against the real pipeline once
B15/B9 land. Follow the docs-before-implementation process (§10) — embed concrete test expectations,
add mock-out instructions for un-landed dependencies, re-run against the real dependency when it
arrives. Guides fan out independently once C1's skeleton and conventions exist.

## 2. Ordered Task List

- TASK-01: Adopt C1 conventions + create how-to section scaffold + shared how-to template (goal/prereqs/steps/verification) — produces: how-to template — depends-on: [C1]
- TASK-02: GitHub Actions reference-pipeline guide (security-first posture around CLI subcommands) + tests + mock-out — produces: flagship how-to — depends-on: [TASK-01]
- TASK-03: Issue-a-`VirtualKey`-with-`BudgetPolicy`-and-model-allowlist guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-04: Write-an-OPA-policy-for-tool-access guide (RBAC-floor/OPA-restrictor) + tests — produces: how-to — depends-on: [TASK-01]
- TASK-05: Debug-an-agent-with-Langfuse+Tempo-traces guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-06: Write-a-Headlamp-plugin guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-07: Canary-rollout-with-Langfuse-A/B-labels guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-08: BYO-SDK-into-the-BYO-harness guide (raw LangGraph / third-party harness, unsupported note) + tests — produces: how-to — depends-on: [TASK-01]
- TASK-09: Expose-an-agent-over-A2A guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-10: Choose-gVisor-vs-Kata-for-a-`SandboxTemplate` guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-11: Write-a-Crossplane-Composition-for-a-new-`AgentEnvironment` guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-12: Register-a-new-Keycloak-client guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-13: Red-team-an-agent-before-promotion guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-14: Set-per-agent-egress-rules (`EgressTarget` via Envoy) guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-15: Add-a-toolset-to-HolmesGPT-for-a-new-component guide + tests — produces: how-to — depends-on: [TASK-01]
- TASK-16: Canon-name lint + cross-guide consistency review + C8 corpus-shape conformance — produces: clean re-indexable corpus — depends-on: [TASK-02..15]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- C1 — portal infrastructure, how-to nav slot, authoring conventions, contribution-workflow check.

### 3.2 Downstream pieces blocked on this
- C8 — indexes the how-to corpus into `platform-knowledge-base`.

### 3.3 Continuous (non-blocking) inputs
- B15 (CI/CD reference pipeline) + B9 (`agent-platform` CLI) — the flagship CI guide validates
  against these; mock CLI contract until landed.
- A1, A7, A2, A13, A9, A6, B4, Keycloak, A14, B7 — components each guide exercises; consumed
  continuously, authored ahead against mocks per docs-before-implementation.
- B14 test framework — executes embedded test expectations (ADR 0011); fakes until landed.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 through TASK-15 (the fourteen guides) are independent and fan out concurrently.
- TASK-16 converges the fan-out (lint + consistency + C8 conformance).

## 5. Test Strategy

Each guide's embedded test expectations map to the three layers (ADR 0011):
- Chainsaw: CRD-driven guides (`VirtualKey`/`BudgetPolicy`, `EgressTarget`, `SandboxTemplate`,
  `AgentEnvironment`) — reconcile + verification step. Fixtures: fake reconcilers for un-landed A1/A6/B4.
- Playwright: AC-C3-01 (each guide renders/navigable in-portal), AC-C3-04 (structure present),
  AC-C3-08 (docs-check pass on a how-to PR).
- PyTest: AC-C3-02 (CI guide has all security-first behaviors + CLI-subcommand mapping),
  AC-C3-03 (no GitLab/Jenkins guide present), AC-C3-06 (BYO-SDK unsupported note), AC-C3-07
  (Canon-name lint), AC-C3-09 (egress=`EgressTarget`+Envoy, OPA=RBAC-floor). The CI guide is
  additionally validated end-to-end against the real B15 pipeline once it lands.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains spec-C1 skeleton + conventions)
### 6.2 PR — `piece/C3-diataxis-how-to-guides` → base `wave/consumer`; carries spec-C3 + plan-C3
### 6.3 Merge order — after C1; independent of C2/C4/C5 siblings; rolls up to main

## 7. Effort Estimate

- TASK-01 S · TASK-02 L (flagship CI guide) · TASK-03..15 each S/M (thirteen guides) · TASK-16 M
- Rollup: L (fourteen guides + a heavy security-first CI flagship).
- Critical path: TASK-01 → TASK-02 (CI flagship, gated on B15/B9) → TASK-16.

## 8. Rollback / Reversibility

Revert the how-to Markdown via GitOps; fully reversible — authored content, no runtime state, no
CRD. If reverted, the platform loses its problem-oriented recipe layer (including the only CI
reference) and C8 loses how-to content from `platform-knowledge-base`.

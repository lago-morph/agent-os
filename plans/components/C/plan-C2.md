# PLAN C2 — Diataxis tutorials

> spec: SPEC-C2 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; each tutorial authored before its component lands) · estimate: L
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy

Author the §10.1 tutorial set as docs-as-code into C1's tutorials navigation slot, one tutorial
per §10.1 item, all using Langchain Deep Agents (ADR 0019). Follow the docs-before-implementation
process (§10): write each tutorial — including its embedded concrete test expectations and, where
the dependency is not yet landed, its mock-out instructions — ahead of the documented component, get
the test expectations signed off, then re-run the same tests against the real dependency when it
arrives. Tutorials fan out independently once C1's skeleton and conventions exist; each converges
with its component's acceptance gate (ADR 0011, via the `agent-platform` CLI / B9 / B14).

## 2. Ordered Task List

- TASK-01: Adopt C1 conventions + create the tutorials section scaffold + shared tutorial template (prereqs/steps/expected-output/end-state) — produces: tutorial template — depends-on: [C1]
- TASK-02: Build-your-first-Platform-Agent tutorial (`Agent` CRD → ArgoCD → LibreChat/A2A) + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-03: First-agent-with-Platform-SDK (BYO container) tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-04: Add-an-MCP-server (`MCPServer` CR → registry) tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-05: Compose-synthetic-MCP-from-OpenAPI (`SyntheticMCPServer`) tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-06: Create-attach-test-a-`Skill` tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-07: Run-an-`Evaluation`-suite tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-08: Trigger-on-a-schedule tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-09: Trigger-from-an-event (S3 drop / webhook) tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-10: Long-running-agent-with-checkpointing tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-11: Use-the-Knowledge-Base-RAG-from-a-custom-agent tutorial + tests + mock-out — produces: tutorial — depends-on: [TASK-01]
- TASK-12: Canon-name lint pass + cross-tutorial consistency review + C8 corpus-shape conformance — produces: clean re-indexable corpus — depends-on: [TASK-02..11]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- C1 — portal infrastructure, tutorials nav slot, authoring conventions, contribution-workflow check.

### 3.2 Downstream pieces blocked on this
- C8 — indexes the tutorial corpus into `platform-knowledge-base` (consumes C2's Markdown output).

### 3.3 Continuous (non-blocking) inputs
- A1, A5, A6, A8, A12, A17, B6, B7, B9, B13 and the other components each tutorial exercises —
  consumed continuously as they land; tutorials authored ahead against mocks per the docs-before-
  implementation commitment, then re-run against the real dependency.
- B14 test framework / B9 CLI — execute the embedded test expectations (ADR 0011); fakes until landed.
- ADR 0019 Deep Agents SDK surface (B7) — exact method signatures `[PROPOSED]` until published.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 through TASK-11 (the ten tutorials) are independent and fan out concurrently.
- TASK-12 converges the fan-out (lint + consistency + C8 conformance).

## 5. Test Strategy

Each tutorial's embedded test expectations map to the three layers (ADR 0011) and double as the
documented component's acceptance tests:
- Chainsaw: CRD-driven tutorials (Agent/MCPServer/SyntheticMCPServer/Skill/Evaluation) — reconcile +
  expected end state. Fixtures: fake reconciler/registry for not-yet-landed A5/A12/A17/B13.
- Playwright: AC-C2-01 (each tutorial renders and is navigable in-portal), AC-C2-03 (structure
  present per tutorial), AC-C2-07 (docs-check pass on a tutorial PR).
- PyTest: AC-C2-02 (no raw-LangGraph usage; `deep-agents` only), AC-C2-06 (Canon-name lint),
  AC-C2-08 (C8 corpus-shape conformance — fake indexer until C8 lands). AC-C2-05 mock-out
  round-trip executed per tutorial.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains spec-C1 skeleton + conventions)
### 6.2 PR — `piece/C2-diataxis-tutorials` → base `wave/consumer`; carries spec-C2 + plan-C2
### 6.3 Merge order — after C1; independent of C3/C4/C5 siblings; rolls up to main

## 7. Effort Estimate

- TASK-01 S · TASK-02..11 each M (ten tutorials) · TASK-12 M
- Rollup: L (ten end-to-end tutorials, each with embedded tests + mock-out, authored ahead of impl).
- Critical path: TASK-01 → (longest tutorial, e.g. TASK-02 first-agent end-to-end) → TASK-12.

## 8. Rollback / Reversibility

Revert the tutorial Markdown via GitOps; fully reversible — authored content, no runtime state, no
CRD. If reverted, newcomers lose the learning on-ramp and C8 loses tutorial content from the
`platform-knowledge-base`; documented components lose their acceptance test source until re-authored.

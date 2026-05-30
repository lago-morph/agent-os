# PLAN B21 — Development environment for agents

> spec: SPEC-B21 · kind: COMPONENT · tier: T2
> wave: W4 · estimate: M
> upstream-pieces: [B7, B6, B9] · downstream-pieces: []

## 1. Implementation Strategy
B21 assembles a coherent, supported on-ramp for building Platform Agents over pieces that already
exist: the agent SDK (B7), the Platform SDK (B6), and the `agent-platform` CLI (B9). The approach
is documentation-and-thin-tooling: a getting-started + conventions doc set, scaffold/local-up
tooling that drives the B9 CLI, and a decision guide for the two supported paths (local kind vs
shared dev cluster). Everything converges on the same SDKs, the same CLI, and the same Git → CI →
ArgoCD iteration loop, and keeps developer agents inside the LiteLLM/OPA/Envoy perimeter so local
behavior matches the cluster. Cross-Stage promotion (Kargo) and the cluster install are explicitly
not built here.

## 2. Ordered Task List
- TASK-01: Align scaffold layout + workflow command names with B9's shipped subcommand surface
  (mark any new ones `[PROPOSED]`) — produces: convention spec — depends-on: [].
- TASK-02: Author the local-kind bootstrap/run/iterate tooling + getting-started doc (deep-agents
  default, Platform SDK calls) — produces: tooling + docs — depends-on: [TASK-01].
- TASK-03: Author the shared-dev-cluster path + the local-vs-shared decision guide — produces: docs —
  depends-on: [TASK-01].
- TASK-04: Document Keycloak-OIDC `kubectl` access (§6.11 Flow 3, kind-supported) and the Git → CI →
  ArgoCD loop (§7.4) — produces: docs — depends-on: [TASK-02].
- TASK-05: Provide perimeter-conformance conventions + an example off-perimeter call that fails;
  document running the three test layers via the CLI — produces: example + test docs — depends-on: [TASK-02].
- TASK-06: Seed the two tutorials/how-tos (first agent on kind; deploy to shared cluster), starting
  from B17/B18 by reference — produces: tutorial seeds — depends-on: [TASK-02, TASK-03].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B7 — agent SDK (LangGraph / Deep Agents) developers build against.
- B6 — Platform SDK surfaces (`memory.*`/`rag.*`/OTel/A2A) the dev agent calls.
- B9 — `agent-platform` CLI the tooling drives.

### 3.2 Downstream pieces blocked on this
- None hard. (Feeds developer training E2 and how-to docs C2/C3 as content, non-blocking.)

### 3.3 Continuous (non-blocking) inputs
- B17/B18 (profiles + compositions used as starting points), B14 (test framework) + B15 (CI pipeline)
  for the iteration loop, A21 (dev-namespace tenant onboarding), k8-platform / A15 (cluster + OIDC
  baseline), B22 (security conventions for dev agents).

## 4. Parallelizable Subtasks
- TASK-02 (kind path) and TASK-03 (shared-cluster path + decision guide) run concurrently after TASK-01.
- TASK-04, TASK-05, TASK-06 (docs/examples) parallelize once the two paths exist.

## 5. Test Strategy
- AC-B21-01/02/05 → manual/scripted walkthrough: bootstrap on kind, the shared-cluster path, and a
  successful Keycloak-OIDC `kubectl` login on kind.
- AC-B21-03/04/09 → PyTest on the scaffold/bootstrap tooling: asserts deep-agents default, a Platform
  SDK call present, CLI invocation (no parallel toolchain), and B17/B18 referenced not copied.
- AC-B21-07 → Chainsaw/integration: a dev agent's off-perimeter call (non-allowlisted egress) fails;
  LLM/MCP/A2A traffic goes via LiteLLM. Needs Envoy (A6) + LiteLLM (A1) fixtures if not landed.
- AC-B21-08 → CLI-orchestrated three-layer run (ADR 0011) against the sample agent.
- AC-B21-06 → doc-review gate: the iteration loop is documented and defers promotion to Kargo.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/4` (contains B7, B6, B9).
### 6.2 PR — `piece/B21-agent-dev-environment` → base `wave/4`; carries spec-B21 + plan-B21.
### 6.3 Merge order — independent of W4 siblings; rebase on B9's shipped CLI surface before merge.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 S · TASK-05 M · TASK-06 S. Rollup: **M**.
- Critical path: TASK-01 → TASK-02 → TASK-05.

## 8. Rollback / Reversibility
Fully reversible: B21 is docs + thin tooling; removing it removes the on-ramp but breaks no runtime
service. No downstream piece hard-depends on it, so blast radius is loss of the supported developer
experience (developers fall back to raw SDK/CLI usage). No schema, data, or cluster-state change to
unwind — content/tooling-only revert.

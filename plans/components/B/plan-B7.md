# PLAN B7 — Initial agent SDK support (LangGraph + Langchain Deep Agents)

> spec: SPEC-B7 · kind: COMPONENT · tier: T0
> wave: W3 · estimate: L
> upstream-pieces: [B6, A6, A5] · downstream-pieces: [B18, B21]

## 1. Implementation Strategy
B7 builds the agent base container image as a multi-SDK harness that wraps the platform SDK (B6) and
bundles LangGraph + Langchain Deep Agents. The strategy leans on ADR 0019's key fact — Deep Agents is
a LangGraph application — so a single binding layer exposes B6's four surfaces through both idioms
behind B6's one contract, and the harness shape is kept generic so future SDKs enroll by adding an
allowed `Agent.sdk` value plus a matrix column, never a schema break. The harness wires only the
documented surfaces (CRDs, the LiteLLM call path via B6, the Envoy egress proxy) so the platform's
external enforcement perimeter applies unconditionally. ARK (A5) reconciles `Agent` CRDs into
sandboxed pods running the image with no operator change for the SDK choice. A build-pipeline variant
dimension builds+scans the v1.0 variant and is structured for additive variant families. The
BYO-harness contract is documented (unsupported) because enforcement is external.

## 2. Ordered Task List
- **TASK-01:** Build the **agent base container image** skeleton bundling LangGraph + Deep Agents +
  B6 — produces: base image + build file — depends-on: [].
- **TASK-02:** Implement the **`Agent.sdk` runtime selector** (`langgraph` / `deep-agents`) — produces:
  harness entrypoint — depends-on: [TASK-01].
- **TASK-03:** Implement the **B6 binding layer** exposing `memory.*`/`rag.*`/OTel/A2A through both
  LangGraph primitives and Deep Agents idioms behind B6's contract — produces: SDK binding layer —
  depends-on: [TASK-01].
- **TASK-04:** Wire the **interaction surfaces**: CRD consumption (`Agent`/`CapabilitySet`/`Memory`/
  `MCPServer`/`EgressTarget`/`Skill`), LiteLLM call path via B6, Envoy egress routing (ADR 0003) —
  produces: harness wiring — depends-on: [TASK-03].
- **TASK-05:** Surface **capability-change notifications** (ADR 0013) through B6's callback +
  `refresh_capabilities()` in both SDKs — produces: notification handling — depends-on: [TASK-03].
- **TASK-06:** Preserve the **multi-SDK harness shape** + additive-enrollment seam (allowed-value
  extension, no schema break) — produces: harness extension contract — depends-on: [TASK-02].
- **TASK-07:** **Build pipeline** variant-matrix dimension: build + scan the LangGraph+Deep Agents
  variant — produces: image build/scan pipeline — depends-on: [TASK-01].
- **TASK-08:** **Docs**: LangGraph + Deep Agents reference, BYO-harness contract (unsupported),
  per-SDK compatibility matrix (two columns), version-mismatch runbook; the Deep-Agents "build your
  first agent" tutorial — produces: §10.5 docs + runbook + tutorial — depends-on: [TASK-04, TASK-05,
  TASK-07].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **B6 (Platform SDK)** — the harness wraps it and binds its four surfaces.
- **A6 (agent-sandbox + Envoy egress proxy)** — pods run in `Sandbox`es; outbound HTTP routes via Envoy.
- **A5 (ARK)** — reconciles `Agent` (incl. `sdk`) into pods running the base image.

### 3.2 Downstream pieces blocked on this
- **B18** (recommended compositions), **B21** (agent dev environment). Indirectly the tutorial chain
  and B17 usage via B18.

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework; **B22** threat model (harness security standards).

## 4. Parallelizable Subtasks
After TASK-01: TASK-02, TASK-03, and TASK-07 run concurrently. TASK-04 and TASK-05 follow TASK-03
and can run in parallel with each other. TASK-06 follows TASK-02. TASK-08 is the serial tail.

## 5. Test Strategy
- **Chainsaw:** `Agent` with `sdk: langgraph` and `sdk: deep-agents` reconciles via ARK/sandbox into a
  pod from the base image (AC-B7-02, AC-B7-06); egress allow/deny via Envoy (AC-B7-04).
- **PyTest:** binding-layer parity — the same program under both SDKs (AC-B7-03); capability-change
  surfacing (AC-B7-05); additive-enrollment-without-schema-break test (AC-B7-07); image-contents
  assertion (AC-B7-01).
- **Playwright:** N/A — no UI.
- **Pipeline checks:** build+scan variant + matrix dimension (AC-B7-08); compatibility-matrix columns
  (AC-B7-10); BYO-contract doc completeness (AC-B7-09).
- **Fixtures/fakes:** a fake/pinned B6 build (B6's frozen contract), a stub ARK/sandbox if not landed,
  a non-allowlisted FQDN target for the egress test, a mock third `sdk` value for AC-B7-07.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/3` (contains B6/A5/A6 specs)
### 6.2 PR — `piece/B7-agent-sdk-langgraph-deepagents` → base `wave/3`; carries spec-B7 + plan-B7
### 6.3 Merge order — independent of W3 siblings (B10/B14/B19-core); wave/3 rolls up to main; B18/B21
(W4) branch after B7 merges.

## 7. Effort Estimate
TASK-01 M · TASK-02 S · TASK-03 L · TASK-04 M · TASK-05 M · TASK-06 S · TASK-07 M · TASK-08 M.
Rollup: **L**. Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-08.

## 8. Rollback / Reversibility
Back out by reverting the base image tag and pinning agents to the prior image via the compatibility
matrix; ARK keeps running prior-image pods. Downstream B18/B21 pin the older image. No CRD/operator
state to unwind (B7 owns no CRD). Reverting does not affect the enforcement perimeter (external).

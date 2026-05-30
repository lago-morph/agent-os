# PLAN B6 — Platform SDK (Python + TypeScript)

> spec: SPEC-B6 · kind: COMPONENT · tier: T0
> wave: W2 · estimate: XL
> upstream-pieces: [A1, A2] · downstream-pieces: [B7, B10, B21]

## 1. Implementation Strategy
B6 is built contract-first: because B7/B10/B21 bind to its method surface, the highest-leverage move
is to freeze the four surface groups' signatures (`memory.*`, `rag.*`, OTel emission, A2A
registration) plus the ADR-0013 capability-change client as the versioned API before deep
implementation. The SDK is delivered as **two packages (Python, TypeScript) over one shared contract**,
each wrapping OSS clients (OTel SDK, Letta client, LiteLLM HTTP/A2A). A shared cross-language contract
test suite guards parity. Memory and RAG terminate at the platform call path (Letta + LiteLLM) so all
enforcement (OPA, audit, Langfuse) fires upstream; the SDK only surfaces denials. Audit goes through
the A18 adapter, never direct writes. Semantic versioning + a published compatibility matrix
(gateway/ARK/Letta) and deprecation warnings complete the §6.13 contract. LangGraph + Deep Agents
idiom bindings ship behind the single contract so B7 can wrap them.

## 2. Ordered Task List
- **TASK-01:** Freeze the SDK contract — signatures for `memory.*`, `rag.*`, OTel, A2A helpers, and
  the capability-change client (`on_capability_changed` + `refresh_capabilities()`) — produces:
  language-neutral API contract + contract test spec — depends-on: [].
- **TASK-02:** Implement the **OTel emission** surface with `trace_id` correlation (ADR 0015) —
  produces: OTel surface (Py+TS) — depends-on: [TASK-01].
- **TASK-03:** Implement the **`rag.*`** surface routed through LiteLLM (incl.
  `platform-knowledge-base`) — produces: rag surface (Py+TS) — depends-on: [TASK-01].
- **TASK-04:** Implement the **`memory.*`** surface via the platform memory path, honoring
  `MemoryStore` access modes (ADR 0025) — produces: memory surface (Py+TS) — depends-on: [TASK-01].
- **TASK-05:** Implement **A2A registration helpers** (register/expose/discover/call) with `myAgent.vN`
  interface versioning — produces: a2a surface (Py+TS) — depends-on: [TASK-01].
- **TASK-06:** Implement the **capability-change client**: subscribe to `platform.capability.changed`
  + `refresh_capabilities()` poll fallback (ADR 0013) — produces: notification client (Py+TS) —
  depends-on: [TASK-01]; needs B12 schema fixture.
- **TASK-07:** Link the **audit adapter** (A18) so SDK-mediated actions audit under `platform.audit.*`
  (ADR 0034) — produces: audit integration — depends-on: [TASK-02..05].
- **TASK-08:** Add **LangGraph + Deep Agents idiom bindings** behind the one contract (ADR 0019) —
  produces: SDK binding layer — depends-on: [TASK-02..06].
- **TASK-09:** Versioning + packaging: semver enforcement, compatibility matrix (gateway/ARK/Letta),
  deprecation-warning machinery, PyPI/npm publish — produces: release tooling + matrix — depends-on:
  [TASK-08].
- **TASK-10:** Docs: dual-language API reference, compatibility/migration guides, version-mismatch
  runbook, "build your first agent" tutorial entry — produces: §10.5 docs + runbook — depends-on:
  [TASK-09].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A1 (LiteLLM)** — call path for `rag.*`, model, A2A; OpenAI↔A2A translation.
- **A2 (Langfuse)** — trace sink for OTel correlation.
- (Interface bindings, not wave edges) **A10 (Letta)** for `memory.*`; **A18** audit adapter; **B12**
  for the `platform.capability.changed` schema.

### 3.2 Downstream pieces blocked on this
- **B7** (agent SDK harness), **B10** (Coach), **B21** (agent dev environment).

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework (cross-language contract tests); **B22** threat model.

## 4. Parallelizable Subtasks
After TASK-01, the four surface implementations (TASK-02..05) plus TASK-06 fan out concurrently
(independent surfaces). TASK-07 joins after the four surfaces. TASK-08 joins all surfaces +
notification client. TASK-09→TASK-10 are serial tail.

## 5. Test Strategy
- **PyTest + TS unit/contract tests:** the shared contract suite run in both languages enforces
  parity (AC-B6-07, AC-B6-12); surface behavior (AC-B6-01..06); semver packaging check (AC-B6-08);
  deprecation-warning emission (AC-B6-10); Canon-namespace static check (AC-B6-13); audit-via-adapter
  + no-direct-write assertion (AC-B6-11).
- **Playwright:** N/A — no UI.
- **Chainsaw:** N/A — no CRD/operator.
- **Fixtures/fakes:** fake LiteLLM endpoint, fake Letta, a **B12-published `platform.capability.changed`
  fixture** (interim `[PROPOSED]` schema until B12 freezes), Langfuse/Tempo trace-id assertion harness,
  a matrix fixture for AC-B6-09.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains A1/A2 specs from earlier waves)
### 6.2 PR — `piece/B6-platform-sdk` → base `wave/2`; carries spec-B6 + plan-B6
### 6.3 Merge order — independent of W2 siblings (B9/B12/B16/B17); wave/2 rolls up to main; B7 (W3)
branches after B6 merges.

## 7. Effort Estimate
TASK-01 M · TASK-02 M · TASK-03 M · TASK-04 L · TASK-05 L · TASK-06 M · TASK-07 S · TASK-08 L ·
TASK-09 M · TASK-10 M. Rollup: **XL**. Critical path: TASK-01 → TASK-04/05 → TASK-08 → TASK-09 →
TASK-10.

## 8. Rollback / Reversibility
SDK is a published library; back out by yanking the affected version and pinning consumers to the
prior release via the compatibility matrix. Reverting a major version forces B7/B10/B21 to pin the
older SDK — manageable because semver + matrix isolate the break. No cluster state to unwind.

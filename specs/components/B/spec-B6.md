# SPEC B6 — Platform SDK (Python + TypeScript)

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [A1, A2] · downstream: [B7, B10, B21] · adrs: [0013, 0019, 0030, 0031, 0034, 0015, 0025] · views: [6.2, 6.3, 6.5, 6.13]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B6 is the **Platform SDK** — the in-process library (Python and TypeScript) Platform Agents use to
call **into** the platform. It is the harness-to-platform contract surface named in interface-contract
§3.1 and architecture-overview §6.2: the four source-named surface groups `memory.*`, `rag.*`, OTel
emission, and A2A registration helpers. It is the layer through which agent code reaches Letta-backed
memory, the Knowledge Base and other `RAGStore`s, emits traces correlated by `trace_id` (ADR 0015),
and registers/discovers A2A interfaces — all terminating at LiteLLM (A1) where authentication, OPA,
audit, and Langfuse (A2) callbacks fire.

Without a single SDK, every agent and every supported agent SDK (B7) would re-implement memory/RAG
access, trace emission, and A2A registration against LiteLLM's raw surfaces, fragmenting the
enforcement-relevant call path and the version-compatibility story. B6 is **distinct from the agent
SDK (B7)**: B6 is *what agents call into the platform with*; B7 is *the agent-authoring harness that
binds B6 into LangGraph and Deep Agents idioms*. B6 is T0 foundation because B7, the Coach Component
(B10), and the agent dev environment (B21) all build on it, and because it owns the
`platform.capability.changed` subscription + `refresh_capabilities()` contract mandated by ADR 0013.

## 2. Scope

### 2.1 In scope
- The **`memory.*` surface**: agent-facing API to read/write memory through Letta (A10) via the SDK,
  honoring `MemoryStore` access modes (private / namespace-shared / RBAC-OPA per ADR 0025). Method
  signatures `[PROPOSED — not in source]`.
- The **`rag.*` surface**: query/ingest against approved `RAGStore`s including the
  `platform-knowledge-base`, routed through LiteLLM. Method signatures `[PROPOSED — not in source]`.
- **OTel emission surface**: trace/span/metric emission correlated by `trace_id` so SDK-emitted
  traces line up with LiteLLM + Langfuse (A2) and Tempo (ADR 0015).
- **A2A registration helpers**: register/expose and discover/call A2A interfaces brokered by LiteLLM,
  including the per-agent declared interface version (`myAgent.v1`, §6.13).
- **Capability-change notification client** (ADR 0013): subscribe to `platform.capability.changed`
  and the `refresh_capabilities()` poll fallback, surfaced as a callback/interrupt to agent code.
- **Both Python and TypeScript** packages with a shared API contract; **semantic versioning** and a
  **version-pinned compatibility matrix** against gateway / ARK / Letta versions (ADR 0030, §6.13).
- SDK-level integration with the **audit adapter** (A18) so SDK-mediated actions audit under
  `platform.audit.*` without agents writing audit directly (ADR 0034).

### 2.2 Out of scope (and where it lives instead)
- **Agent-authoring harness / LangGraph + Deep Agents bindings** → **B7** (consumes B6).
- **LiteLLM gateway itself, OPA enforcement, Langfuse/Tempo backends** → A1 / A7 / A2 / A13.
- **Letta deployment + memory backend adapter** → A10 / B11; B6 calls through, does not host.
- **CloudEvent per-event-type schemas** (incl. `platform.capability.changed` schema) → **B12**.
- **CapabilitySet overlay resolution semantics** → ADR 0032 / B13; B6 only consumes the change event.
- **`agent-platform` CLI** → **B9** (separate versioned surface, §6.13).
- **Audit endpoint + adapter library internals** → **A18**; B6 links the adapter, does not own it.
- **Memory access-mode enforcement** is on `MemoryStore` + OPA, not the SDK; B6 honors but does not
  decide.

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM)** — every `rag.*`, model, and A2A call terminates here; B6 targets LiteLLM's
  call path. OpenAI-compatible HTTP ↔ A2A translation is LiteLLM's (§7.1).
- **A2 (Langfuse)** — trace destination; B6's OTel emission correlates by `trace_id` (ADR 0015).

**Also consumed (not in the CSV upstream edge but required by contract):**
- **A10 (Letta)** for `memory.*`; **A18** audit adapter library; **B12** for the
  `platform.capability.changed` schema the notification client deserializes. These are honored as
  interface bindings, not wave-ordering edges. `[PROPOSED — not in source]` that B6 links A18 at
  build time (interface-contract §5 states every component links the adapter).

**Downstream consumers:**
- **B7** (agent SDK harness) binds B6 into LangGraph + Deep Agents idioms.
- **B10** (Coach Component) uses B6 to read traces/observe.
- **B21** (agent dev environment) ships B6 for local development.

**ADRs honored:**
- **ADR 0013** — B6 owns the agent-facing capability-change callback + `refresh_capabilities()`
  poll fallback; best-effort, not a correctness guarantee (enforcement stays at LiteLLM/OPA/Envoy).
- **ADR 0019** — `memory.*`, `rag.*`, OTel, A2A surfaces are exposed through both LangGraph
  primitives and Deep Agents idioms; bindings stay behind one SDK API contract so agent code is
  portable across enrolled SDKs.
- **ADR 0030 / §6.13** — semantic versioning; version-pinned compatibility matrix vs
  gateway/ARK/Letta; deprecation warnings on deprecated SDK APIs.
- **ADR 0031** — the notification client consumes `platform.capability.changed` under the closed
  taxonomy; SDK never mints a new top-level namespace.
- **ADR 0034** — SDK-mediated actions audit via the adapter; no direct writes to Postgres/S3/OpenSearch.
- **ADR 0015** — trace correlation by `trace_id` across SDK / LiteLLM / Langfuse / Tempo.
- **ADR 0025** — `memory.*` honors per-store access modes; SDK never overrides them.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B6 defines no CRD. It **consumes** Canon CRDs as read context: `Agent` (`sdk`,
`capabilitySetRefs[]`, `memoryRefs[]`, `modelRef`, `exposes`), `Memory` (`memoryStoreRef`),
`MemoryStore` XR (`accessMode`, `backendType`), `RAGStore` (`backend`, `indexes[]`), `CapabilitySet`.
SDK reads these (or their LiteLLM-applied projection) to know what an agent may reach; it does not
mutate them.

### 4.2 APIs / SDK surfaces
B6 **is** an SDK surface. interface-contract §3.1 names only the four surface groups and states
**"specific method signatures are not specified in source."** All method names below are therefore
`[PROPOSED — not in source]`; the *surface groups* and *language/versioning facts* are Canon.

- **`memory.*`** (Canon group) — `[PROPOSED]` `memory.read(ref, key)`, `memory.write(ref, key,
  value)`, `memory.list(ref)`, `memory.append(ref, ...)`. Resolves `memoryRefs[]` → `MemoryStore`;
  access mode (ADR 0025) enforced upstream, surfaced as deny errors.
- **`rag.*`** (Canon group) — `[PROPOSED]` `rag.query(store, query, k)`, `rag.ingest(store, doc)`,
  `rag.stores()`. `store` defaults include `platform-knowledge-base`. Routed via LiteLLM.
- **OTel emission** (Canon group) — `[PROPOSED]` `tracing.span(name)`, `tracing.event(...)`,
  `metrics.counter(...)`. Propagates/correlates `trace_id` (ADR 0015); exports to the platform OTel
  collector → Tempo/Langfuse.
- **A2A registration helpers** (Canon group) — `[PROPOSED]` `a2a.register(interface, version)`,
  `a2a.expose(handler)`, `a2a.discover(peerRef)`, `a2a.call(peerRef, method, args)`. Interface
  version string form `myAgent.v1` is Canon (§6.13); helper names are `[PROPOSED]`.
- **Capability-change client** (ADR 0013, Canon behavior) — `[PROPOSED]` names
  `on_capability_changed(callback)` (subscribe to `platform.capability.changed`) and
  `refresh_capabilities()` (poll fallback). The **event name `platform.capability.changed`** and the
  **method name `refresh_capabilities()`** are Canon (glossary / ADR 0013); the subscribe-helper name
  is `[PROPOSED]`.
- **Versioning surface** — semantic versioning; both Python and TS expose the same contract; ships a
  compatibility matrix (Canon, §6.13). Matrix file format `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed:** `platform.capability.changed` (Canon, ADR 0013) — the notification client subscribes;
  concrete schema lives in **B12**.
- **Emitted (mediated):** SDK-mediated actions surface audit under `platform.audit.*` via the audit
  adapter (A18, ADR 0034) — B6 does not emit CloudEvents directly to the broker; it links the adapter.
  Any additional SDK-originated emission MUST fall under exactly one Canon namespace; **no new
  per-event-type names are coined here** (deferred to B12). `[PROPOSED — not in source]` that B6
  emits any event beyond the audit-adapter path.

### 4.4 Data schemas / connection-secret contracts
N/A — B6 holds no datastore and writes no connection secret. It reaches Letta/RAG **through
LiteLLM and the SDK**, not via direct DB connection; substrate connection secrets (ADR 0041) are
consumed by the backing services (A10, RAGStore backends), not by the SDK.

## 5. OSS-vs-Custom Decision
**Build-new** Python + TypeScript libraries that **wrap** OSS clients: the OpenTelemetry SDKs
(emission), a Letta client for `memory.*`, and LiteLLM's HTTP/A2A surfaces for `rag.*` and A2A.
No suitable single OSS package spans these four platform-specific surfaces with the platform's
versioning/compatibility contract, so B6 is custom glue (Python is the platform default language,
ADR 0006 rationale). Upstream OSS versions (OTel SDK, Letta client, LiteLLM) **pinned in the
compatibility matrix** — exact versions `[PROPOSED — not in source]`. ADR linkage: 0019 (two-SDK
binding surface), 0030 (semver + matrix), 0015 (OTel correlation).

## 6. Functional Requirements
- **REQ-B6-01:** The SDK MUST expose a **`memory.*`** surface that reads/writes agent memory via
  the platform memory path (Letta), resolving `Agent.memoryRefs[]` → `MemoryStore`.
- **REQ-B6-02:** The `memory.*` surface MUST honor `MemoryStore.accessMode` (private /
  namespace-shared / RBAC-OPA, ADR 0025) and surface upstream denials as errors, never bypassing them.
- **REQ-B6-03:** The SDK MUST expose a **`rag.*`** surface that queries/ingests approved `RAGStore`s
  (including `platform-knowledge-base`) routed through LiteLLM.
- **REQ-B6-04:** The SDK MUST expose an **OTel emission** surface that correlates spans/metrics by
  `trace_id` so SDK traces align with LiteLLM, Langfuse (A2), and Tempo (ADR 0015).
- **REQ-B6-05:** The SDK MUST expose **A2A registration helpers** to register/expose, discover, and
  call A2A interfaces brokered by LiteLLM, carrying a declared interface version of form `myAgent.v1`.
- **REQ-B6-06:** The SDK MUST provide a **capability-change callback** subscribing to
  `platform.capability.changed` AND a **`refresh_capabilities()`** poll fallback (ADR 0013);
  behavior is best-effort, never a substitute for LiteLLM/OPA/Envoy enforcement.
- **REQ-B6-07:** The SDK MUST ship as **both Python and TypeScript** packages exposing the same API
  contract.
- **REQ-B6-08:** The SDK MUST follow **semantic versioning** (major=break, minor=additive,
  patch=fix) per ADR 0030 / §6.13.
- **REQ-B6-09:** The SDK MUST ship a **version-pinned compatibility matrix** against gateway / ARK /
  Letta versions (§6.13).
- **REQ-B6-10:** The SDK MUST **emit deprecation warnings** (logs/metrics) when callers use
  deprecated SDK APIs (§6.13).
- **REQ-B6-11:** SDK-mediated actions MUST audit under `platform.audit.*` **via the audit adapter**
  (A18), never by writing Postgres/S3/OpenSearch directly (ADR 0034).
- **REQ-B6-12:** The `memory.*`, `rag.*`, OTel, and A2A surfaces MUST be exposed through **both
  LangGraph primitives and Deep Agents idioms** behind one SDK API contract (ADR 0019), so agent code
  stays portable across the two supported SDKs.
- **REQ-B6-13:** The SDK MUST NOT mint a new top-level CloudEvent namespace; any emission falls under
  exactly one Canon namespace (ADR 0031).

## 7. Non-Functional Requirements
- **Security:** the SDK is not an enforcement point — it MUST surface (never suppress) upstream
  OPA/LiteLLM/Envoy denials; it carries no static credentials beyond the agent's virtual key context.
- **Multi-tenancy (§6.9):** `memory.*` / `rag.*` operate within the agent's namespace/tenant and
  access-mode scope; the SDK MUST NOT enable cross-tenant reach absent OPA-checked publication.
- **Observability (§6.5):** OTel emission is first-class; `trace_id` correlation (ADR 0015) is
  mandatory so agent spans join the platform trace.
- **Scale:** SDK calls are on the agent hot path; clients MUST be connection-pooled and
  non-blocking-capable; the capability-change subscription MUST degrade to poll without crashing.
- **Versioning (ADR 0030):** Python and TS bump in lockstep on the shared contract; compatibility
  matrix updated each release; breaking changes are major bumps with migration guidance (§10.5).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A** — B6 is a published library (PyPI/npm), not a deployed workload.
- Per-product docs (10.5) — **applicable** (API reference for both languages, compatibility matrix,
  migration guides).
- Runbook (10.7) — **applicable** (version-mismatch / matrix-incompatibility diagnosis).
- Alerts — **N/A** — library emits no service-level alerts of its own; consuming agents/A2 alert.
- Grafana dashboard (Crossplane XR) — **N/A** — no runtime service; agent/gateway dashboards cover
  SDK-emitted traces.
- Headlamp plugin — **N/A** — no operator UI surface.
- OPA/Rego integration — **applicable (consumer)** — surfaces OPA denials; honors capability change;
  does not author policy.
- Audit emission (ADR 0034) — **applicable** — SDK-mediated actions audit via the adapter.
- Knative trigger flow — **applicable (consumer)** — consumes `platform.capability.changed`; emits
  none of its own to the broker.
- HolmesGPT toolset — **N/A** — B6 is agent-facing, not a self-management surface.
- 3-layer tests — **applicable** (PyTest + TS unit/contract tests; Playwright N/A — no UI;
  Chainsaw N/A — no CRD/operator).
- Tutorials & how-tos — **applicable** — the "build your first agent" chain (B6→B7→…) targets this
  SDK (ADR 0019, §10).

## 9. Acceptance Criteria
- **AC-B6-01:** A `memory.*` read/write round-trips through the platform memory path against a
  referenced `MemoryStore`. (→ REQ-B6-01)
- **AC-B6-02:** A `memory.*` call against a store whose access mode forbids it returns an upstream
  denial error and performs no write. (→ REQ-B6-02)
- **AC-B6-03:** A `rag.*` query against `platform-knowledge-base` returns results routed via LiteLLM.
  (→ REQ-B6-03)
- **AC-B6-04:** An SDK-emitted span carries the same `trace_id` as the corresponding LiteLLM/Langfuse
  trace. (→ REQ-B6-04)
- **AC-B6-05:** An A2A interface registered via the helper is discoverable and callable through
  LiteLLM with its declared `vN` interface version. (→ REQ-B6-05)
- **AC-B6-06:** On a `platform.capability.changed` event the registered callback fires; with the
  subscription disabled, `refresh_capabilities()` returns updated capabilities within one turn.
  (→ REQ-B6-06)
- **AC-B6-07:** The same agent program compiles/runs against both the Python and TypeScript packages
  using the same API. (→ REQ-B6-07)
- **AC-B6-08:** A breaking API change is released only under a major version bump; a packaging test
  enforces semver. (→ REQ-B6-08)
- **AC-B6-09:** The release ships a compatibility matrix pinning supported gateway/ARK/Letta versions.
  (→ REQ-B6-09)
- **AC-B6-10:** Calling a deprecated SDK API emits a deprecation warning to logs/metrics. (→ REQ-B6-10)
- **AC-B6-11:** An SDK-mediated action produces a `platform.audit.*` record via the adapter and writes
  no row directly to Postgres/S3/OpenSearch. (→ REQ-B6-11)
- **AC-B6-12:** The four surfaces are reachable from both a LangGraph agent and a Deep Agents agent via
  one API contract. (→ REQ-B6-12)
- **AC-B6-13:** A static check confirms every SDK-emitted event type is under a Canon top-level
  namespace. (→ REQ-B6-13)

## 10. Risks & Open Questions
- **R1 (high):** All method signatures are `[PROPOSED — not in source]`. B7, B10, B21 bind to them; a
  late signature change is a cross-component break. Mitigation: B6 freezes signatures first and treats
  them as the versioned contract (REQ-B6-08). Blast radius: high.
- **R2 (med):** Python/TS contract parity drift — a surface added in one language but not the other
  breaks portability (REQ-B6-07). Mitigation: shared contract test suite run in both languages.
- **R3 (med):** `platform.capability.changed` schema is owned by **B12** but B6 must deserialize it;
  if B12 lands later, B6 needs a fixture. Mitigation: contract test against a B12-published fixture;
  `[PROPOSED]` interim schema flagged until B12 freezes.
- **R4 (med):** Whether B6 emits any CloudEvent beyond the audit-adapter path is `[PROPOSED — not in
  source]`; over-reaching here could coin event types that belong to other components. Mitigation:
  default to adapter-only emission; defer any new event to B12.
- **R5 (low):** Compatibility-matrix maintenance burden grows with each enrolled SDK/backend version;
  mitigated by the matrix being additive (ADR 0019, §9.5).
- **OQ1:** Does `memory.*` reach Letta directly via a client or strictly through LiteLLM? §6.3 shows
  the SDK calling both Letta and the gateway. `[PROPOSED]` `memory.*`→Letta client, `rag.*`+model→
  LiteLLM, pending A10/A1 surface confirmation.
- **OQ2:** Exact compatibility-matrix file format and where it ships (in-package vs release notes,
  §9.5). `[PROPOSED]` in-package machine-readable + human matrix in §10.5 docs.

## 11. References
- architecture-overview.md §6.2 (lines 286–292; platform-SDK interaction surfaces, harness contract,
  `sdk` field), §6.3 (lines 300–323; SDK `memory.*`+`rag.*` to Letta and gateway), §6.7
  (`platform.capability.*`), §6.13 (lines 989, 996–999; SDK semver, compatibility matrix, deprecation
  warnings).
- interface-contract.md §3.1 (Platform SDK surfaces; signatures not specified in source), §2
  (`platform.capability.changed`), §5 (audit adapter linkage).
- ADR 0013 (capability-change callback + `refresh_capabilities()`), ADR 0019 (two-SDK bindings),
  ADR 0030 (versioning), ADR 0031 (CloudEvent taxonomy), ADR 0034 (audit adapter), ADR 0015 (trace
  correlation), ADR 0025 (memory access modes).
- Related pieces: A1 (LiteLLM), A2 (Langfuse), A10 (Letta), A18 (audit adapter), B7 (agent SDK),
  B10 (Coach), B11 (memory adapter), B12 (event schemas), B21 (dev env).

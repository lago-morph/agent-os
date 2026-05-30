# SPEC V6-02 — Agent runtime architecture `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0001, 0003, 0019] · views: [6.2]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the agent runtime slice of the Agentic Execution Platform: the slice in which a Platform Agent declared as an `Agent` CRD is reconciled by **ARK** into a pod running inside a **`Sandbox`** with kernel isolation (gVisor / Kata), reaching the outside world only through the **LiteLLM** gateway and the **Envoy egress proxy**. It is not a buildable component; it is realized by the ARK operator (A5), the agent-sandbox + Envoy egress install (A6), the Platform SDK (B6), and the initial agent SDK (B7).

The problem it solves: agent execution must be governed entirely at an **external perimeter** so the harness internals do not have to be trusted. By fixing the declarative-state contract (`Agent`/`Sandbox` CRDs), the call path (everything via LiteLLM), the egress path (everything else via Envoy with FQDN allowlist), and the SDK surface, the platform can support multiple agent SDKs — and even unsupported third-party harnesses — without weakening enforcement. This view fixes the invariants every participating component must honor so the perimeter holds regardless of what runs inside the sandbox.

## 2. Scope
### 2.1 In scope
- The reconcile chain invariant: `Agent` CRD → ARK (A5) → agent-sandbox controller (A6) → agent pod in a `Sandbox`.
- Sandbox isolation invariant: agent pods run under gVisor or Kata; warm-pool / hibernation lifecycle owned by agent-sandbox, not ARK.
- The two-path egress invariant: LLM/MCP/A2A traffic via LiteLLM; all other outbound HTTP via the Envoy egress proxy restricted to `EgressTarget`-declared FQDNs.
- The Platform-SDK interaction surface (`memory.*`, `rag.*`, model invocation, A2A registration helpers) as the harness↔platform contract (B6).
- The agent SDK set (B7): `Agent.sdk` accepts `langgraph` and `deep-agents` in v1.0; additive enrollment without schema break (ADR 0019).
- Skills served via LiteLLM's skill gateway from Git; admin-configured, not user-controllable.
- The "build-your-own-harness" path: documented but not officially supported because enforcement is external.

### 2.2 Out of scope (and where it lives instead)
- LiteLLM gateway internals (callbacks, registry, translation) — view V6-01 / components A1, B2, B13.
- Capability-registry CRD model and CapabilitySet layering — view V6-08 / ADR 0013 / ADR 0032.
- Memory store backends and access modes — view V6-03 / ADR 0005 / ADR 0025.
- Knowledge Base / RAG primitive — view V6-04.
- Security/policy enforcement model in full (OPA decision points, defense-in-depth) — view V6-06.
- Persistent-volume access for agents — component B20 (consumed, not defined here).
- Eventing that creates `AgentRun`s — view V6-07 / components A4, B8.

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A5 ARK agent operator** — reconciles `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`; owns the agent lifecycle (ADR 0001).
- **A6 agent-sandbox + Envoy egress proxy** — reconciles `Sandbox`, `SandboxTemplate`; provides gVisor/Kata isolation, warm pool, hibernation; provides the FQDN-allowlisted Envoy egress proxy (ADR 0003).
- **B6 Platform SDK (Python + TypeScript)** — the in-agent surface (`memory.*`, `rag.*`, OTel emission, A2A registration helpers) that terminates at LiteLLM.
- **B7 Initial agent SDK — LangGraph + Deep Agents** — the agent-authoring harness; sets the `Agent.sdk` accepted values (ADR 0019).

ADR decisions honored:
- **ADR 0001** — ARK is the agent operator; agent state is declarative via CRDs reconciled from Git.
- **ADR 0003** — Envoy egress proxy is the CNI-agnostic egress control; only allowlisted FQDNs are reachable.
- **ADR 0019** — LangGraph is the supported v1.0 SDK; Deep Agents is the opinionated default; `Agent.sdk` accepts `langgraph` and `deep-agents` and grows by enrollment without a schema break.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Owned by ARK (A5):
- `Agent` — `capabilitySetRefs[]`, `overrides`, `sdk` (`langgraph`|`deep-agents` in v1.0), `image`, `sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes` (A2A/MCP).
- `AgentRun` — `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state`.
- `Team` — `members[]`, `coordinationStrategy`.
- `Tool` — defers to ARK (not further specified in source).
- `Memory` — `memoryStoreRef` (access mode lives on `MemoryStore`, ADR 0025).
- `Evaluation` — `agentRef`, `datasetRef`, `evaluators[]`.
- `Query` — defers to ARK (not further specified in source).

Owned by agent-sandbox (A6):
- `Sandbox` — `templateRef`, `runtime` (gVisor/Kata), `state`.
- `SandboxTemplate` — `runtime`, `warmPoolSize`, `hibernationEnabled`, `resourceLimits`.

Consumed (owned by B13) for the egress/skill perimeter:
- `EgressTarget` — `fqdn`, `port`, `scheme`, `allowedMethods`.
- `Skill` — `gitRef`, `versionPin`, `schemaRef` (served via LiteLLM's skill gateway).

All namespaced; versioning per ADR 0030; ARK owns `Agent`/`AgentRun`/`Team`/`Tool`/`Memory`/`Evaluation`/`Query` versioning; agent-sandbox owns `Sandbox`/`SandboxTemplate`.

### 4.2 APIs / SDK surfaces
- **Platform SDK (B6)** — `memory.*`, `rag.*`, OTel emission, A2A registration helpers; Python + TypeScript; method signatures beyond these named groups are not specified in source.
- **Agent SDK (B7)** — LangGraph and Deep Agents harnesses; `Agent.sdk` values `langgraph` / `deep-agents`; multi-SDK harness shape preserved for additive enrollment; third-party harness images NOT officially supported.
- **LiteLLM call path** — model invocation, MCP, A2A terminate at LiteLLM (defined in V6-01; consumed here).
- **Envoy egress proxy** — all non-LiteLLM outbound HTTP; only `EgressTarget`-declared destinations reachable.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the runtime slice (per-event-type names deferred to B12 registry):
- `platform.lifecycle.*` — `Agent`, `AgentRun`, `Sandbox` lifecycle (created/started/paused/resumed/completed/failed/deleted).
- `platform.audit.*` — ARK lifecycle events; agent-sandbox creation/destruction/hibernation; Envoy egress connections (via the audit adapter).
- `platform.security.*` — sandbox-escape signal (distinct from audit).
Consumed:
- `platform.lifecycle.*` — `AgentRun` creation requests arriving via Knative adapters (view V6-07).

### 4.4 Data schemas / connection-secret contracts
- Skill artifacts resolved from Git via `Skill.gitRef`/`versionPin` into a skill init container / skill volume; mounted read-only into the agent pod.
- No connection-secret of its own in this slice; memory/data backends are provisioned and secret-wired in view V6-03.

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom decisions for ARK, agent-sandbox/Envoy, the Platform SDK, and the agent SDK live in component SPECs A5, A6, B6, B7).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-02-01:** Every Platform Agent MUST exist as an `Agent` CRD reconciled by ARK (A5) into a pod; no agent may run outside the ARK reconcile chain. (§6.2, line ~249)
- **REQ-V6-02-02:** Agent pods MUST run inside a `Sandbox` under gVisor or Kata kernel isolation; sandbox creation/destruction/hibernation lifecycle MUST be owned by the agent-sandbox controller (A6), not ARK. (§6.2, lines ~249–280; §6.6 line ~522)
- **REQ-V6-02-03:** LLM, MCP, and A2A traffic MUST go through LiteLLM; all other outbound HTTP MUST go through the Envoy egress proxy and is reachable only for `EgressTarget`-declared FQDNs (ADR 0003). (§6.2, lines ~279, ~290)
- **REQ-V6-02-04:** The harness↔platform contract MUST be exactly the declared CRDs plus the Platform SDK surface (`memory.*`, `rag.*`, model invocation, A2A registration helpers) terminating at LiteLLM. (§6.2, lines ~286–290)
- **REQ-V6-02-05:** The `Agent.sdk` field MUST accept `langgraph` and `deep-agents` in v1.0 and MUST grow by additive enrollment without a schema break (ADR 0019). (§6.2, line ~284)
- **REQ-V6-02-06:** Skills MUST be served from Git via LiteLLM's skill gateway, be admin-configured as part of the CapabilitySet, and MUST NOT be user-controllable or exposed at the user-facing UI. (§6.2, line ~282)
- **REQ-V6-02-07:** A third-party / self-built harness MUST be governable without trusting harness internals — enforcement (LiteLLM for LLM/MCP/A2A, OPA for policy, Envoy for egress, Kubernetes for declarative state) MUST hold regardless of harness; such images are documented but NOT officially supported. (§6.2, line ~292)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** kernel isolation via gVisor/Kata; egress confined to FQDN allowlist; agents run only in approved namespaces; capability scope enforced externally at the gateway and egress proxy.
- **Observability (§6.5):** agent pods emit OTel spans carrying `trace_id`; ARK and the sandbox controller emit audit via the adapter; lifecycle emits `platform.lifecycle.*`.
- **Scale:** warm pod pool and hibernation (via `SandboxTemplate.warmPoolSize` / `hibernationEnabled`) for fast cold-start and idle cost control.
- **Versioning (ADR 0030):** `Agent.sdk` enrollment is additive (no schema break); CRD versioning owned per-component by A5 / A6.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A5, A6, B6, B7).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-02-01:** Applying an `Agent` CRD yields a running pod reconciled through ARK; no path exists to run an agent pod that bypasses ARK. (→ REQ-01)
- **AC-V6-02-02:** A running agent pod is confirmed under gVisor/Kata; killing/hibernating it is driven by the agent-sandbox controller, and ARK does not own that transition. (→ REQ-02)
- **AC-V6-02-03:** An agent's outbound HTTP to a non-`EgressTarget` FQDN is blocked at Envoy; LLM/MCP/A2A traffic transits LiteLLM. (→ REQ-03)
- **AC-V6-02-04:** An agent reaching memory/RAG/model/A2A does so only via the Platform SDK surface terminating at LiteLLM; no alternate in-pod path reaches those backends. (→ REQ-04)
- **AC-V6-02-05:** An `Agent` with `sdk: langgraph` and one with `sdk: deep-agents` both admit; an enrollment of a new SDK value is additive with no conversion-webhook break. (→ REQ-05)
- **AC-V6-02-06:** A skill is served from Git via the skill gateway and is absent from any user-facing UI surface. (→ REQ-06)
- **AC-V6-02-07:** A non-supported harness image complying with the interface is still fully governed (LiteLLM/OPA/Envoy/K8s enforce) and is flagged unsupported. (→ REQ-07)

## 10. Risks & Open Questions
- **OQ-1 (med):** Platform SDK method signatures beyond the four named surface groups are not specified in source; resolved in B6 component SPEC (`[PROPOSED — not in source]` for any added method).
- **OQ-2 (low):** `Tool` and `Query` CRD field sets defer to ARK; resolved in A5 SPEC.
- **R-1 (med):** A permissive `EgressTarget` set could widen the egress perimeter; mitigated by OPA-at-egress (view V6-06) and admission validation.
- **R-2 (low):** Unsupported third-party harnesses may emit non-conformant OTel/audit; mitigated because enforcement is external and harness misbehavior cannot bypass the perimeter.

## 11. References
- architecture-overview.md §6.2 Agent runtime architecture (lines ~247–292); §6.6 (line ~522, sandbox lifecycle owned by agent-sandbox); §6.5 (OTel/trace_id).
- ADRs: 0001 (ARK as agent operator), 0003 (Envoy egress proxy), 0019 (LangGraph + Deep Agents SDK).
- Realizing components: A5 (ARK), A6 (agent-sandbox + Envoy), B6 (Platform SDK), B7 (agent SDK).
- Related views: V6-01, V6-03, V6-04, V6-06, V6-07, V6-08.

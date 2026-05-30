# SPEC ADR-0019 — LangGraph supported v1.0 agent SDK; Langchain Deep Agents the opinionated default [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [B7, B6] · downstream: [A5, B17, B18, B21, C2, C3, C4] · adrs: [0019] · views: [6.2]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0019 is a settled T0 decision: v1.0 ships **LangGraph** as the supported agent SDK with **Langchain Deep Agents** (built on LangGraph) as the opinionated, batteries-included default. The `Agent.sdk` field accepts exactly `langgraph` and `deep-agents` in v1.0; the multi-SDK harness shape (B7) is preserved so other SDKs enroll later as additive changes; third-party harness images are documented-but-not-officially-supported because all enforcement (LiteLLM, OPA, Envoy, CRDs) sits outside the harness. This SPEC states the constraints, interfaces, and conformance the decision imposes on B7/B6 and on documentation/profile/evaluation surfaces. It does not re-argue the SDK choice.

The problem the decision solves: Platform Agents need a low-level graph/state SDK for advanced control flow plus a batteries-included default, without multiplying the binding/build/test/CVE surface across unrelated SDKs.

## 2. Scope
### 2.1 In scope
- The obligation that v1.0 supports exactly LangGraph (lower-level) + Deep Agents (default), both tested/supported.
- The `Agent.sdk` field accepting `langgraph` and `deep-agents`; future SDKs enrolled by adding allowed values, not a schema migration.
- The default-path obligation: tutorials, how-tos, "build your first agent" chain, agent profile library (B17), recommended compositions (B18) target Deep Agents end-to-end; reference docs cover both.
- Platform SDK (B6) `memory.*` / `rag.*` / OTel / A2A-registration surfaces exposed through both LangGraph primitives and Deep Agents idioms behind one SDK API contract.
- The documented (unsupported) third-party harness contract — CRD shapes, LiteLLM call path, Envoy egress, platform SDK surface.
- Capability-change notification (ADR 0013) + overlay (ADR 0032) surfaced/composed identically across both SDKs.

### 2.2 Out of scope (and where it lives instead)
- Agent base image / harness implementation — component **B7** SPEC.
- Platform SDK implementation — component **B6** SPEC.
- ARK operator (reconciles `Agent` regardless of SDK) — ADR 0001 / **A5**.
- Enrolling additional SDKs (OpenAI Agents SDK, Strands, Anthropic Agent SDK, Mastra, ARK ADK) — deferred (architecture-backlog §3.15).
- Detailed SDK method signatures — not specified in source.

## 3. Context & Dependencies
Upstream consumed: **B7** (agent SDK / harness) ships LangGraph + Deep Agents; **B6** (Platform SDK) binds both behind one contract. Downstream consumers: **A5** ARK (reconciles `Agent.sdk`), **B17** profile library, **B18** recommended compositions, **B21** agent dev environment, **C2/C3/C4** docs.

ADR decisions honored:
- **ADR 0019** (this) — LangGraph supported + Deep Agents default; harness shape preserved; third-party unsupported.
- **ADR 0001** — ARK reconciles `Agent` CRDs into pods regardless of which SDK runs inside; SDK is a harness-level decision.
- **ADR 0013** — capability-change notifications surfaced through the platform SDK callback in both SDKs.
- **ADR 0032** — CapabilitySet overlays compose identically across LangGraph and Deep Agents.
- **ADR 0030** — `Agent.sdk` enrollment is additive (new allowed value, no schema break); Platform SDK semantic versioning + compatibility matrix.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `Agent` (namespaced, ARK-owned A5) — `sdk` field MUST accept `langgraph` and `deep-agents` in v1.0; additional values enrolled additively. Other fields per interface-contract (`capabilitySetRefs[]`, `image`, `sandboxTemplateRef`, etc.).
- Versioning: enrolling a new `sdk` value is an additive change, not a `vN` migration (ADR 0030).

### 4.2 APIs / SDK surfaces
- Agent SDK (B7): **LangGraph** (`Agent.sdk=langgraph`) — lower-level graph runtime, tested/supported; **Langchain Deep Agents** (`Agent.sdk=deep-agents`) — opinionated default on top of LangGraph, tested/supported. Multi-SDK harness shape preserved.
- Platform SDK (B6): `memory.*`, `rag.*`, OTel emission, A2A registration helpers exposed through both LangGraph primitives and Deep Agents idioms behind one SDK API contract so agent code stays portable. Python + TypeScript; semantic versioning; per-SDK compatibility matrix starts with two columns. Method signatures beyond the named surface groups: not specified in source — `[PROPOSED — not in source]` if detailed.
- Third-party harness contract (documented, NOT officially supported): CRD shapes (`Agent`, `CapabilitySet`, `Memory`, `MCPServer`), LiteLLM call path for LLM/MCP/A2A traffic, Envoy egress proxy for outbound HTTP, platform SDK API surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Agent/`AgentRun` lifecycle under `platform.lifecycle.*`; capability-change callback driven by `platform.capability.changed` (ADR 0013) surfaced in both SDKs. No new top-level namespace.

### 4.4 Data schemas / connection-secret contracts
- N/A — the SDK introduces no substrate primitive; `memory.*`/`rag.*` reach Letta/RAGStores through the Platform SDK, not new secrets.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream **LangGraph** + **Langchain Deep Agents** wrapped by the custom multi-SDK harness (B7) and platform SDK bindings (B6) — config/wrap, no fork. Rejected alternatives — Deep Agents as sole SDK; multiple unrelated SDKs from day one; officially supporting third-party harness images — are not built in v1.0.)

## 6. Functional Requirements
- REQ-ADR-0019-01: v1.0 MUST support exactly two agent SDKs — LangGraph (lower-level) and Langchain Deep Agents (default) — both tested and supported; no third unrelated SDK is officially supported.
- REQ-ADR-0019-02: The `Agent.sdk` field MUST accept `langgraph` and `deep-agents` in v1.0; enrolling a future SDK MUST be an additive allowed-value change, not a schema migration.
- REQ-ADR-0019-03: The multi-SDK harness shape (B7) MUST be preserved so OpenAI Agents SDK / Strands / Anthropic Agent SDK / Mastra / ARK ADK can be enrolled later as additive changes.
- REQ-ADR-0019-04: All tutorials, how-to guides, the "build your first agent" chain, the agent profile library (B17), and recommended compositions (B18) MUST target Deep Agents end-to-end; reference documentation MUST cover both Deep Agents and LangGraph.
- REQ-ADR-0019-05: The platform SDK (B6) `memory.*`, `rag.*`, OTel emission, and A2A registration helpers MUST be exposed through both LangGraph primitives and Deep Agents idioms behind one SDK API contract so Platform Agent code stays portable.
- REQ-ADR-0019-06: The evaluation harness, red-team tooling, and agent base image build pipeline MUST exercise both SDKs; the per-SDK compatibility matrix MUST start with two columns and grow by adding columns, not restructuring.
- REQ-ADR-0019-07: The third-party harness contract MUST be documented (CRD shapes, LiteLLM call path, Envoy egress, platform SDK surface), while third-party harness images remain NOT officially supported.
- REQ-ADR-0019-08: Enforcement MUST remain external (LiteLLM, OPA, Envoy, CRDs) regardless of harness internals; a compliant third-party image MUST run in the ecosystem subject to that perimeter.
- REQ-ADR-0019-09: Capability-change notifications (ADR 0013) and CapabilitySet overlays (ADR 0032) MUST be surfaced/composed identically across LangGraph and Deep Agents.

## 7. Non-Functional Requirements
- Security: the enforcement perimeter (LiteLLM, OPA, Envoy, CRDs) is SDK-independent; a third-party harness does not expand the trusted base.
- Observability (§6.5): both SDKs emit correlated `trace_id` spans (ADR 0015) via the platform SDK OTel surface.
- Versioning (ADR 0030): platform SDK semantic versioning with a two-column-and-growing compatibility matrix; `Agent.sdk` enrollment additive.
- Portability: agent code MUST stay portable across enrolled SDKs behind the same SDK API contract.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (B7, B6, B17, B18, C2/C3/C4).

## 9. Acceptance Criteria
- AC-ADR-0019-01: Honored when both a LangGraph and a Deep Agents agent run under ARK and pass their tests, and no third SDK is enrolled. (REQ-01)
- AC-ADR-0019-02: Honored when `Agent.sdk` admits `langgraph`/`deep-agents` and rejects an unenrolled value, and adding a value requires no `vN` migration. (REQ-02)
- AC-ADR-0019-03: Honored when the harness can host an additional SDK by configuration without redesign. (REQ-03)
- AC-ADR-0019-04: Honored when sampled tutorials/how-tos/profile-library/compositions use Deep Agents, and reference docs cover both. (REQ-04)
- AC-ADR-0019-05: Honored when the same agent code using `memory.*`/`rag.*`/OTel/A2A runs unchanged on both SDKs. (REQ-05)
- AC-ADR-0019-06: Honored when eval/red-team/build pipeline run both SDK variants and the compatibility matrix has two columns. (REQ-06)
- AC-ADR-0019-07: Honored when the third-party harness contract doc exists and states third-party images are unsupported. (REQ-07)
- AC-ADR-0019-08: Honored when a compliant third-party image is still subject to LiteLLM/OPA/Envoy enforcement and a non-compliant one is bounded by the perimeter. (REQ-08)
- AC-ADR-0019-09: Honored when a capability change and a CapabilitySet overlay behave identically on a LangGraph and a Deep Agents agent. (REQ-09)

## 10. Risks & Open Questions
- R-1 (med): Preserving harness shape without exercising it risks bit-rot of the enrollment path; mitigated by the matrix dimension existing in pipeline config (REQ-06).
- R-2 (low): Documenting an unsupported third-party harness can be read as implied support; mitigated by explicit "not officially supported" framing (REQ-07).
- OQ-1 (low): Detailed SDK method signatures are not specified in source; downstream B6/B7 specs define them — `[PROPOSED]` here.

## 11. References
- ADR 0019 (`adr/0019-langchain-deep-agents-single-sdk.md`) — the decision enforced here.
- architecture-overview.md §6.2 (agent runtime), §5.3, §8, §9.5, §10, §11; architecture-backlog.md §2.16, §3.15.
- Enforcing components: B7 (agent SDK/harness, owner), B6 (platform SDK), A5 (ARK), B17 (profiles), B18 (compositions), C2/C3/C4 (docs).
- Related ADRs: 0001, 0013, 0015, 0030, 0032.

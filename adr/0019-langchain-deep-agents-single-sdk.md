# ADR 0019: LangGraph as the supported v1.0 agent SDK with Langchain Deep Agents as the opinionated default

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs an agent SDK story for Platform Agents that run inside Sandboxes under ARK (architecture-overview.md § 6.2, ADR 0001). The agent base container image (B7) is described as a multi-SDK harness wrapping the platform SDK — LangGraph, Langchain Deep Agents, OpenAI Agents SDK, Strands, Anthropic Agent SDK, Mastra, and ARK ADK were all considered as candidates the harness should be able to host (overview § 5.3 inventory).

Two related concerns shape v1.0. First, Platform Agents need a low-level SDK that exposes the full graph/state model so advanced users can build custom control flow, sub-agent delegation, and tool-use patterns that do not fit a higher-level abstraction. Second, the typical adopter — building long-running agents with sub-agent delegation, structured planning, MCP/A2A tool use, and `memory.*` / `rag.*` integration — benefits from a batteries-included default that does not require assembling those primitives from scratch.

LangGraph is the lower-level graph runtime; Langchain Deep Agents is built on top of LangGraph and packages the Platform Agent patterns v1.0 actually needs. Supporting both costs little because Deep Agents is already a LangGraph application: the harness, platform SDK bindings, and integration surfaces are shared.

Separately, the platform's enforcement perimeter (LiteLLM for LLM/MCP/A2A traffic, OPA for policy, Envoy egress proxy for outbound HTTP, Kubernetes CRDs — Agent, CapabilitySet, Memory, MCPServer — for declarative configuration) sits **outside** the harness. Whatever the harness does internally, the perimeter applies. That makes it tractable to document the harness contract for adventurous users who want to build their own container images, even though the platform does not officially support third-party harness images.

## Decision

v1.0 ships **LangGraph as the supported agent SDK**, with **Langchain Deep Agents as the opinionated, batteries-included default** layered on top of it.

- **Langchain Deep Agents** is the SDK used in all tutorials, how-to guides, the "build your first agent" chain (B6 → B7 → CI/CD reference pipeline → deployment, per overview § 10), the agent profile library (B17), and the recommended compositions (B18). It is tested and supported as the default path.
- **LangGraph** is exposed as the lower-level SDK for advanced users who need more control than Deep Agents provides. It is tested and supported; reference documentation covers both SDKs.
- The multi-SDK harness shape (B7) is preserved so additional SDKs — OpenAI Agents SDK, Strands, Anthropic Agent SDK, Mastra, ARK ADK — can be enrolled later as additive changes. The `Agent` CRD's `sdk` field continues to exist so additional values can be enrolled without a schema break (architecture-backlog.md § 3.15 trigger applies).
- The platform documents how its harness interacts with the integration surfaces — Kubernetes CRDs (Agent, CapabilitySet, Memory, MCPServer, etc.), the LiteLLM call path for LLM/MCP/A2A traffic, and the Envoy egress proxy for outbound HTTP — so adventurous users can build their own container images that comply with this interface and run in the ecosystem. **Third-party harness images are not officially supported**, but the contract is documented because all enforcement happens externally (LiteLLM, OPA, Envoy) regardless of what the harness does internally.

## Alternatives considered

- **Langchain Deep Agents as the sole supported SDK** (the prior decision in this ADR) — Rejected because it forces advanced users who need direct access to the graph/state model to either accept Deep Agents' abstractions or wait for an enrollment of a lower-level SDK. Since Deep Agents is built on LangGraph, exposing LangGraph directly costs almost nothing extra: the harness, platform SDK bindings, and integration surfaces are shared.
- **Multiple SDKs from day one (OpenAI Agents SDK + Strands + Anthropic Agent SDK + Mastra + ARK ADK alongside LangGraph and Deep Agents)** — Each additional unrelated SDK multiplies the platform SDK (B6) binding surface, the agent base image build matrix, the BYO-container documentation, the evaluation and red-team tooling, the agent profile library (B17), and the per-SDK compatibility matrix. Without a v1.0 use case that fits a non-LangGraph SDK better, the cost lands on every component while delivering no functional differentiation. Deferred per architecture-backlog.md § 3.15.
- **Officially supporting third-party harness images** — Rejected for v1.0. Documenting the harness contract is cheap because all enforcement happens at LiteLLM, OPA, and Envoy regardless of harness internals, but supporting third-party images would expand the test, evaluation, and CVE-response surface beyond what v1.0 can absorb.

## Consequences

### SDK surface and harness

- The agent base container image and platform SDK bindings ship LangGraph and Langchain Deep Agents in v1.0; the multi-SDK harness shape that would host additional unrelated SDKs is preserved but not exercised.
- The `Agent` CRD's `sdk` field (architecture-overview.md § 11, B7) accepts `langgraph` and `deep-agents` in v1.0; future SDKs are enrolled by adding allowed values rather than performing a schema migration.

### Documentation and onboarding

- All tutorials, how-to guides, the "build your first agent" chain, the agent profile library (B17), and recommended compositions (B18) target Deep Agents end-to-end so the v1.0 onboarding story stays coherent. Reference documentation covers both Deep Agents and LangGraph for advanced users.
- The evaluation harness and red-team tooling exercise both SDKs; the per-SDK compatibility matrix in the platform SDK release notes (overview § 9.5) starts with two columns. Subsequent SDK enrollments add columns rather than restructure the matrix.
- The agent base image build pipeline (overview § 8) builds and scans the LangGraph + Deep Agents variant in v1.0; the matrix dimension exists in pipeline configuration so that adding a second variant family is a configuration change, not a pipeline redesign.

### Platform SDK bindings

- The platform SDK (B6) `memory.*`, `rag.*`, OTel emission, and A2A registration helpers are exposed through both LangGraph primitives and Deep Agents idioms; when an unrelated SDK is enrolled, equivalent bindings are added behind the same SDK API contract so Platform Agent code stays portable across enrolled SDKs.

### Third-party harness path

- The harness contract — Kubernetes CRD shapes (Agent, CapabilitySet, Memory, MCPServer), the LiteLLM call path for LLM/MCP/A2A traffic, the Envoy egress proxy for outbound HTTP, and the platform SDK API surface — is documented so adventurous users can build compliant container images. Third-party harness images are not officially supported, but they run in the ecosystem because enforcement is external (LiteLLM, OPA, Envoy) and applies regardless of harness internals.
- Capability change notifications (ADR 0013) are surfaced through the platform SDK callback in both LangGraph and Deep Agents; CapabilitySet overlays (ADR 0032) compose the same way regardless of which of the two SDKs the agent uses.
- Adopters with an in-house agent on an unenrolled SDK must port to LangGraph or Deep Agents, build a third-party harness image against the documented contract (unsupported), or wait for enrollment of their SDK.
- Revisit trigger (architecture-backlog.md § 3.15): a concrete use case that fits an unrelated SDK better than LangGraph or Deep Agents. At that point the work is enrolling that SDK into the existing harness and platform SDK bindings, not re-architecting the agent runtime.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.2](../architecture-overview.md#62-agent-runtime-architecture), § 5.3, [§ 8](../architecture-overview.md#8-cicd-integration-requirements), § 9.5, [§ 10](../architecture-overview.md#10-documentation-plan), [§ 11](../architecture-overview.md#11-grafana-dashboards)
- [architecture-backlog.md](../architecture-backlog.md) [§ 2.16](../architecture-backlog.md#216-initial-agent-sdk-choice), [§ 3.15](../architecture-backlog.md#315-adding-sdks-beyond-langchain-deep-agents), § 7 item 19
- [ADR 0001](./0001-ark-as-agent-operator.md) (ARK as agent operator) — ARK reconciles `Agent` CRDs into pods regardless of which SDK runs inside; SDK choice is a harness-level decision and does not constrain the operator layer
- [ADR 0013](./0013-capability-crd-and-capabilityset-layering.md) (Capability CRD and CapabilitySet layering) — capability change notifications are surfaced through the platform SDK callback in both supported SDKs
- [ADR 0032](./0032-capabilityset-overlay-semantics.md) (CapabilitySet overlay) — overlay composition is SDK-independent and applies identically to LangGraph and Deep Agents agents

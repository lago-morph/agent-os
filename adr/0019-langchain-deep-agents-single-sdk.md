# ADR 0019: Langchain Deep Agents as the single v1.0 agent SDK

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs an initial agent SDK choice for Platform Agents that run inside Sandboxes under ARK (architecture-overview.md § 6.2). The agent base container image (B7) is described as a multi-SDK harness wrapping the platform SDK — LangGraph, OpenAI Agents SDK, Strands, Anthropic Agent SDK, Mastra, and ARK ADK were all considered as candidates the harness should be able to host (overview § 5.3 inventory).

Supporting all of them on day one would multiply the surface area for the platform SDK (B6) language bindings, the BYO-container how-tos, the evaluation harness, the red-team tooling, the agent profile library (B17), and the per-SDK compatibility matrix the platform commits to. There is no v1.0 use case identified that fits a non-Deep-Agents SDK better, so the spread would land cost on every component without unblocking a concrete adopter.

Langchain Deep Agents covers the Platform Agent patterns v1.0 actually needs — long-running agents with sub-agent delegation, structured planning, tool use through MCP and A2A, and integration with the platform SDK's `memory.*` and `rag.*` APIs. A single SDK choice keeps v1.0 scope tight while leaving the door open to add others when a concrete need appears.

## Decision

v1.0 ships **Langchain Deep Agents as the only agent SDK** Platform Agents may use. The multi-SDK harness shape is preserved in the agent base container image and the platform SDK bindings, so adding LangGraph, OpenAI Agents SDK, Strands, Anthropic Agent SDK, Mastra, ARK ADK, or another SDK later is an additive change rather than a redesign. The `Agent` CRD's `sdk` field continues to exist precisely so additional values can be enrolled later without a schema break.

## Alternatives considered

- **Multiple SDKs from day one (LangGraph + OpenAI Agents SDK + Strands + Anthropic Agent SDK + Mastra + ARK ADK alongside Langchain Deep Agents)** — Each additional SDK multiplies the platform SDK (B6) binding surface, the agent base image build matrix, the BYO-container documentation, the evaluation and red-team tooling, the agent profile library (B17), and the per-SDK compatibility matrix. Without a v1.0 use case that fits a non-Deep-Agents SDK better, the cost lands on every component while delivering no functional differentiation. Deferred per architecture-backlog.md § 3.15.

## Consequences

- The agent base container image and platform SDK bindings ship configured for Langchain Deep Agents only in v1.0; the multi-SDK harness shape that would host additional SDKs is preserved but not exercised.
- The `Agent` CRD's `sdk` field is retained (architecture-overview.md § 11, B7) so future SDKs are enrolled by adding allowed values rather than performing a schema migration.
- v1.0 documentation, the agent profile library (B17), the recommended compositions (B18), the evaluation harness, and the red-team tooling all target a single SDK, keeping the v1.0 surface area narrow and the BYO-container how-to focused.
- Per-SDK compatibility matrices in the platform SDK release notes (overview § 9.5) start with a single column; subsequent SDK enrollments add columns rather than restructure the matrix.
- The agent base image build pipeline (overview § 8) builds and scans a single SDK variant in v1.0; the matrix dimension exists in pipeline configuration so that adding a second SDK variant is a configuration change, not a pipeline redesign.
- The "build your first agent" tutorial chain (B6 → B7 → CI/CD reference pipeline → deployment path, per overview § 10) targets Langchain Deep Agents end-to-end so the v1.0 onboarding story stays coherent rather than splitting across SDK choices.
- Adopters with an in-house agent that uses a different SDK must either port to Langchain Deep Agents or wait for that SDK to be enrolled into the harness; the platform does not support unenrolled SDKs in v1.0.
- Revisit trigger (architecture-backlog.md § 3.15): a concrete use case that fits another SDK better than Langchain Deep Agents. At that point the work is enrolling that SDK into the existing harness and platform SDK bindings, not re-architecting the agent runtime.
- Aligns with ADR 0001 (ARK as agent operator) — ARK reconciles `Agent` CRDs into pods regardless of which SDK runs inside, so the SDK choice is a harness-level decision and does not constrain the operator layer.
- The platform SDK (B6) `memory.*`, `rag.*`, OTel emission, and A2A registration helpers are exposed through Langchain Deep Agents idioms in v1.0; when a second SDK is enrolled, equivalent bindings are added behind the same SDK API contract so Platform Agent code stays portable across enrolled SDKs.

## References

- architecture-overview.md § 6.2
- architecture-backlog.md § 2.16, § 3.15, § 7 item 19
- ADR 0001 (ARK as agent operator)
- ADR 0013 (Capability CRD and CapabilitySet layering) — agent capability composition is SDK-independent

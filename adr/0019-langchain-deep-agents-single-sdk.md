# ADR 0019: Langchain Deep Agents as the single v1.0 SDK

## Status

Accepted

## Context

The agent base container image is designed as a multi-SDK harness: a single
container shape that can wrap any of LangGraph, OpenAI Agents SDK, Strands,
Anthropic Agent SDK, Mastra, or ARK ADK around the Platform SDK
(`memory.*`, `rag.*`, OTel emission, A2A registration helpers). The harness
is the integration surface; the SDK inside it provides the agent loop,
tool-calling shape, and developer ergonomics for the agent author.

Two questions sit underneath that shape:

1. Which SDKs do we actually ship and support in v1.0?
2. How do we keep the door open for more later without paying for them now?

Shipping all six SDKs day one would multiply the surface area we test,
document, instrument, and support — base image variants, Platform SDK
bindings (Python and TypeScript), tutorial set, callback wiring, and
trace shape conformance — well beyond what v1.0 can carry. Shipping only
a custom-built loop would duplicate work that the upstream SDK ecosystem
already does well, and would orphan agent authors from external community
patterns and examples.

Langchain Deep Agents covers the agent shapes v1.0 needs (planner +
sub-agents, file-shaped working memory, long-horizon tool use), is
actively maintained upstream, and integrates cleanly with the Platform
SDK call surfaces we already require.

## Decision

Ship **Langchain Deep Agents as the single agent SDK in v1.0.** The
agent base container image keeps its multi-SDK harness shape — directory
layout, Platform SDK wrapping, OTel and A2A wiring, BYO-container
extension points — so additional SDKs can be plugged in later without
re-architecting the harness. Only the Langchain Deep Agents wrapping is
built, tested, documented, and supported for v1.0.

`CapabilitySet` declares the SDK choice per agent (see ADR 0013), and
ARK Agents host the resulting SDK runtime as a pod (see ADR 0001). For
v1.0, the only valid declared value is Langchain Deep Agents.

## Consequences

- One SDK to test, document, and support: tutorials, reference docs,
  base image variants, and stress probes all target a single runtime.
- Multi-SDK harness shape is preserved as a non-functional invariant —
  reviews on the base image and Platform SDK bindings reject changes
  that bake in Langchain-Deep-Agents-only assumptions where the harness
  contract should be SDK-agnostic.
- Agent authors with a use case that fits another SDK better than
  Langchain Deep Agents are blocked in v1.0; the escape hatch is BYO
  container against the same Platform SDK contract.
- Adding a second SDK later is a scoped piece of work (new harness
  wrapping + Platform SDK binding coverage + docs), not a re-platforming.

## Alternatives considered

- **Ship multiple SDKs day one** (LangGraph + OpenAI Agents SDK +
  Anthropic Agent SDK, etc.). Rejected: surface area and support burden
  across base image variants, Platform SDK bindings, tutorials, and
  trace-shape conformance is not justified by v1.0 demand.
- **Custom-only SDK built in-house.** Rejected: duplicates upstream
  agent-loop work, orphans authors from community patterns and examples,
  and adds a maintenance line we don't need.
- **No SDK opinion — only BYO container.** Rejected: leaves every agent
  author to wire their own loop and Platform SDK integration, which
  defeats the point of shipping a base image.

## Related

- ADR 0001 — ARK Agents host the SDK runtime as the agent pod.
- ADR 0013 — `CapabilitySet` declares the SDK choice per agent.
- Architecture overview §5 — Agent base container image and Platform SDK rows.
- Architecture backlog §2.16 — initial agent SDK choice.
- Architecture backlog §3.15 — trigger to add another SDK is a use case
  that fits it better than Langchain Deep Agents.

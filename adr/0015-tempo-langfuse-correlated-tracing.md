# ADR 0015: Tempo + Langfuse correlated by trace_id

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform produces two flavors of trace data: general OpenTelemetry spans
from the agent SDK, LiteLLM gateway, Envoy egress, ARK operator, and UIs; and
LLM-grade traces (prompts, completions, costs, evaluator scores, dataset runs)
that need first-class tooling. A single backend cannot serve both well —
generic OTel backends lack prompt/cost/eval semantics, and LLM-observability
backends are not a general distributed-tracing store. Developers debugging an
agent need to move fluidly between "what did the system do" and "what did the
model see and say." HolmesGPT also needs both surfaces to perform diagnostics
across the full stack.

## Decision

The platform runs **Tempo for general OTel traces and Langfuse for LLM-grade
traces**, correlated by a shared `trace_id`. The agent SDK and LiteLLM gateway
emit the same `trace_id` into spans sent to both backends, so a developer
viewing a trace in Grafana/Tempo can deep-link into Langfuse for prompt-level
detail and back. HolmesGPT consumes both backends through its observability
toolset.

## Alternatives considered

- **Langfuse-only** — would simplify operations to one backend but loses
  general-purpose distributed-trace semantics for non-LLM components (Envoy,
  ARK, kopf operators, eventing). Langfuse is purpose-built for LLM workflows
  and is not appropriate as the system-wide tracing store; we would either
  under-instrument the rest of the platform or shoehorn non-LLM spans into a
  schema that does not fit them.

## Consequences

- The platform operates **two trace backends** (Tempo and Langfuse) plus their
  storage, retention, and dashboard configuration — a real ops cost accepted
  in exchange for fit-for-purpose tooling on each side.
- The **agent SDK and LiteLLM gateway are responsible for emitting the same
  `trace_id` to both backends**; this is a load-bearing SDK invariant, not an
  optional convention, and must be covered by tests.
- Grafana dashboards include **deep links from Tempo spans into Langfuse**
  (and the reverse where applicable), making the correlation usable without
  manual ID copy-paste.
- **HolmesGPT consumes both backends via its observability toolset**
  (Tempo + Mimir + Loki + Langfuse), so platform self-diagnostics span both
  generic and LLM-flavored traces from day one.
- promptfoo red-team results and LiteLLM prompt/cost/eval data flow into
  Langfuse; non-LLM spans never need to enter Langfuse, keeping its dataset
  and evaluator surfaces uncluttered.
- Future migration to a single backend (if one ever covers both use cases
  well) is bounded: the SDK already centralizes span emission, so the change
  is a configuration swap rather than an instrumentation rewrite.
- Any component that originates a request without a `trace_id` breaks the
  correlation; SDK defaults must generate one at request entry and propagate
  it through A2A, MCP, and gateway hops.
- **Trace granularity in both Tempo and Langfuse is gated by the dynamic
  log/trace toggle from ADR 0035**, so HolmesGPT (and platform admins via
  Headlamp) can raise verbosity on a specific component for diagnostics and
  lower it back afterwards. v1.0 explicitly does **not** run always-on tracing
  with sample-and-drop — span creation is conditional on the live config, so
  off-mode has no performance cost and Tempo/Langfuse cost scales with
  verbosity rather than with traffic.
- **Non-unit test runs publish run metrics and logs via OpenTelemetry** along
  the Tempo/Mimir/Loki path (per ADR 0011), and any traces those runs emit
  flow through the same Tempo/Langfuse correlation described above — test
  diagnostics share the platform's observability surfaces rather than running
  on a parallel pipeline.

## References

- architecture-overview.md § 6.5
- architecture-backlog.md § 2.4
- ADR 0011 (three-layer testing — non-unit runs publish via OTel)
- ADR 0012 (HolmesGPT)
- ADR 0035 (dynamic log/trace toggle — gates trace granularity)

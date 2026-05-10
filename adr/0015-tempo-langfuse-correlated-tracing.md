# ADR 0015: Tempo + Langfuse correlated by trace_id

## Status

Accepted

## Context

The platform needs two kinds of trace visibility. **General distributed
tracing** spans every component an agent touches: ARK, agent pods, Envoy
egress, LiteLLM, the kopf operator, Headlamp, LibreChat, Knative event
adapters. This is standard OpenTelemetry territory. **LLM-grade
observability** captures prompt-level detail — message-by-message rendering,
prompt versions in use, costs per call, evaluator scores, datasets and
experiments, red-team results from promptfoo. This needs a backend that
understands LLM concepts as first-class objects, not a generic span store.

A single backend would not serve both audiences well. Tempo is excellent
for OTel spans and integrates with Grafana, but it does not model prompts,
costs, or evaluations. Langfuse is purpose-built for LLM-grade traces,
prompt management with A/B labels, datasets, experiments, and evaluators —
but feeding all platform spans into it would dilute its value and stretch
it past its sweet spot.

A developer debugging a slow or failing agent run wants to start in one
view (often Grafana, since that is where dashboards and alerts live) and
drill into prompt-level detail without losing context.

## Decision

Run **both Tempo and Langfuse**, correlated by **`trace_id`**.

- **OTel collector** is the single ingestion point for general traces from
  every component (agent pods, LiteLLM, Envoy egress, ARK, LibreChat,
  Headlamp, kopf operator). The collector exports to **Tempo**, which
  Grafana reads as a data source.
- **Langfuse** receives **LLM-grade traces** directly from the agent SDK
  and the LiteLLM gateway: prompts, completions, token costs, model and
  provider used, evaluator scores, prompt versions. It also serves as the
  home for prompt management (A/B labels), datasets, experiments, and
  evaluator results. promptfoo red-team output from CI lands here too.
- **Same `trace_id` on both sides.** Every span emitted by the platform
  SDK and the gateway carries the same trace_id whether it is
  LLM-flavored or not. Langfuse traces are tagged with the OTel trace_id
  the agent run started under.
- **Grafana deep-links to Langfuse by trace_id.** Trace panels in Grafana
  expose a link that opens the corresponding Langfuse trace. Developers
  start in Grafana (latency, errors, dashboards), click into Langfuse for
  prompt-level detail, and back. Headlamp does the same handoff for admin
  surfaces.

Langfuse is **always-on for every agent run.** It is not an opt-in
debugging mode; the LLM-grade trace is part of the production telemetry
contract.

## Consequences

- Two trace stores to operate, but each plays the role it is good at.
  Tempo retention is tuned for span volume; Langfuse retention is tuned
  for prompt and evaluation history.
- The platform SDK and the LiteLLM Python callbacks both must propagate
  the OTel trace_id into Langfuse's trace metadata. This is a fixed
  contract for every agent run.
- Developer training covers both UIs: Grafana for system-level views,
  Langfuse for prompt-level views, and the trace_id link between them.
- Coach and HolmesGPT can query both backends to correlate behavior:
  "this prompt version regressed latency for these agents" needs both
  Langfuse (prompt version) and Tempo (latency).
- If Langfuse later grows OTel-native ingestion that subsumes Tempo's
  role for our scope, we revisit — but the platform's span volume from
  non-LLM components (egress, admission, eventing) makes that unlikely.

## Alternatives considered

- **Langfuse only.** Rejected. Langfuse is purpose-built for LLM-grade
  traces; using it as a general OTel backend stretches it past its sweet
  spot and weakens the prompt-management experience. It also leaves the
  non-agent components (Envoy egress, ARK operator, Knative adapters,
  Headlamp) without a natural home for spans.
- **Tempo only.** Rejected. Tempo has no concept of prompts, prompt
  versions, costs per call, datasets, experiments, or evaluators.
  Building those on top of generic spans would reinvent Langfuse poorly.
- **A custom correlation layer (single UI joining both stores).**
  Rejected for v1.0. Grafana's trace_id deep-link to Langfuse covers the
  common workflow without custom UI work; Headlamp does the same for
  admin paths. Revisit only if the deep-link handoff proves insufficient.

## Related

- ADR 0009 — OpenSearch audit index. Separate concern (audit, not tracing)
  but shares the same trace_id, so an audit event can be correlated to
  the corresponding Tempo span and Langfuse trace.
- ADR 0014 — Storage roles (Postgres primary, OpenSearch retrieval).
  Tempo and Langfuse are observability stores, not primary storage; the
  same reproducibility-from-primary discipline applies in spirit.
- Overview §5 (component table — Tempo, Mimir, Langfuse, promptfoo),
  §6.5 (observability architecture diagram and the trace_id correlation
  statement), §14 (workstream A and D ownership).
- Backlog §2.4 (decision rationale: chose both correlated by trace_id
  over Langfuse-only), §7 (ADR candidate #15).

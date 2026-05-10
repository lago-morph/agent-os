# ADR 0035: Dynamic log-level and trace-granularity toggle with staged restart for non-reloadable services

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The observability architecture (architecture-overview.md § 6.5) collects logs, traces, and metrics through the OTel collector into Tempo / Loki / Mimir, with Langfuse correlated by `trace_id`. HolmesGPT (§ 6.10) is the platform's diagnostic agent and, together with platform admins working through Headlamp, must be able to **raise verbosity on a specific component while a problem is being investigated and lower it again afterward**. Defaulting every component to maximum verbosity is not acceptable because v1.0 commits to "no performance impact when observability is off" — we do not run always-on tracing with sample-and-drop. Loggers and tracers are conditionally instantiated, which means a runtime change of level only takes effect after the affected component re-reads its configuration.

Different components have different reload capabilities. Components we author (the SDK, the kopf operator per ADR 0006, the audit endpoint per ADR 0034, custom adapters) can implement an in-process reload. LiteLLM, by contrast, is FastAPI/Uvicorn-only and has no built-in hot-reload of logging or tracing config. We need one decision that covers both cases without forcing a uniform implementation.

## Decision

Every Platform Component declares a **toggle pattern** as part of its component definition. The component template in architecture-overview.md gains a "Toggle pattern" deliverable alongside the existing per-component deliverables, and HolmesGPT's diagnostic runbooks treat that declaration as the authoritative description of how a given component is made noisier on demand. Two patterns are supported:

- **In-process toggle** (preferred wherever the runtime supports it).
  - A `LogLevel` CRD (or a per-component admin endpoint) instructs the component to re-read its logging and tracing configuration in place, SIGHUP-style.
  - Used for the audit endpoint (ADR 0034), the agent SDK, the kopf operator (ADR 0006), and custom adapters we own.
  - Re-reading the config is what causes loggers and tracers to be re-instantiated at the new level; when the level returns to the default, the conditional instantiation drops them again, preserving the "no impact when off" invariant.
  - No pod restart, no in-flight request impact.

- **Rolling-restart toggle** (for components that cannot reload in process).
  - The controller updates a ConfigMap or env var and triggers a rolling restart of the Deployment.
  - Required guardrails: `replicas >= 2`, readiness probes that drop the pod from Service endpoints before SIGTERM is sent, `preStop: sleep 10–15s` to cover endpoint propagation across kube-proxy / Envoy, and `terminationGracePeriodSeconds` raised above the worst-case streaming response (60–300s, sized for SSE / long LLM streams).
  - LiteLLM uses this pattern. LiteLLM is FastAPI/Uvicorn-only and has no built-in hot reload of logging or tracing config; the user has confirmed the trade-off is acceptable given the diagnostic value.

The platform prioritizes **diagnostic agility over peak performance under light workloads** for v1.0. A rolling restart to raise verbosity is acceptable when the alternative is flying blind during an incident.

The Test CLI (ADR 0011) gains a debug/tracing flag that activates the toggle for the duration of a single test run and disables it on exit, so failed tests can be re-run with full instrumentation without leaving the cluster permanently noisy. The CLI uses the same `LogLevel` CRD / admin endpoint surface that HolmesGPT uses; there is one toggle mechanism, two callers.

## Consequences

- Components authored by the platform team carry an in-process reload path as a build-time requirement. The SDK provides a shared implementation so this is not re-invented per component.
- LiteLLM deployments must be sized with `replicas >= 2` even in small clusters, and the Deployment manifest carries the readiness / preStop / terminationGracePeriod fields above. A single-replica LiteLLM is not a supported configuration once this ADR lands.
- Streaming responses in flight at the moment of a verbosity change can take up to `terminationGracePeriodSeconds` to drain. This is visible to users as a slightly delayed shutdown of an old pod, not as a dropped response, because readiness is dropped before SIGTERM.
- HolmesGPT's runbooks for diagnosis can include "raise log level on component X" as a concrete action; the action is policy-controlled like any other write per § 6.10.
- The Test CLI's debug flag gives developers the same lever HolmesGPT has, scoped to one test execution. Forgetting to disable it after a test is not possible because the CLI re-asserts the default on teardown.
- The "Toggle pattern" deliverable in the component template becomes a review checkpoint: a component PR that does not declare its pattern is not complete.
- Trace granularity follows the same model as log level — components that emit OTel spans gate span creation behind the same conditional instantiation, so raising granularity activates spans that were not being created at all rather than enabling sampling on always-on spans. This preserves the § 6.5 invariant that Tempo / Langfuse cost scales with verbosity, not with traffic.
- Headlamp's component view surfaces the current effective level per component and provides the same write path admins would otherwise hit through `kubectl`; this is the human-in-the-loop counterpart to HolmesGPT's automated use.
- Audit emission (ADR 0034) treats every level change as an auditable event regardless of pattern, so the trail of "who raised verbosity, when, on what, for how long" is preserved alongside the diagnostic data the change produced.
- **Revisit triggers**: a future component whose runtime cannot meet either pattern (e.g. cannot tolerate a rolling restart and cannot reload in process) reopens this decision; a shift to always-on sampled tracing as the default would also reopen it.

## References

- architecture-overview.md § 6.5 (observability architecture), § 6.10 (HolmesGPT)
- ADR 0006 (Python kopf packaging — in-process toggle target)
- ADR 0011 (three-layer testing CLI — debug/tracing flag)
- ADR 0012 (HolmesGPT as first-class agent — primary consumer of the toggle)
- ADR 0015 (Tempo + Langfuse correlated tracing — what trace granularity feeds)
- ADR 0034 (audit endpoint — in-process toggle target)

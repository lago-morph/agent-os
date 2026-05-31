# ADR 0051: Observability alerts via the event bus

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

The platform alarms off two observability data sources: **Mimir** (metrics) and
**Loki** (logs). AlertManager evaluates rules against both and fires alerts when a
condition trips. The open question was how those alerts reach the components that act
on them — most immediately HolmesGPT (which investigates and explains incidents), but
potentially many other consumers (dashboards, the self-healing agent, future
analysers).

The corpus as written specifies a **direct `AlertManager → HolmesGPT` trigger** flow
(see the glossary and §6.7): AlertManager wired point-to-point to HolmesGPT. That
wiring predates the event-bus discipline this platform has since settled on. A
point-to-point wire couples the alert producer to one named consumer: every new
consumer means a new wire and a change to AlertManager's configuration, and the
producer must know who is listening. That is exactly the coupling the event bus exists
to eliminate — its purpose is **emit-once / consumers-choose**: a producer publishes a
fact once, and any number of consumers subscribe without the producer knowing or caring
who they are.

There is also a second, distinct relationship that must not be confused with alerting:
HolmesGPT pulls **metrics from Mimir directly** to do its investigative work. That is a
continuous data *stream* it queries on demand — not discrete events, and not something
that belongs on the bus.

The CloudEvent taxonomy (ADR-0031) already reserves the `platform.observability`
namespace, and the single-owner principle (ADR-0045) assigns ownership of that
namespace's schemas to the metrics & tracing stack — Tempo + Mimir (A13). This gives
alerts a natural home on the bus with a clear schema owner.

## Decision

**Observability alerts are events on the bus; metrics are a direct stream. Keep them
separate.**

- **Alerts (events, on the bus).** AlertManager — alarming off both Mimir metrics and
  Loki logs — publishes its alerts onto the **event bus** under the
  `platform.observability` namespace. The schema for those events is **owned by the
  metrics & tracing stack (Tempo + Mimir, A13)**, consistent with the single-owner rule
  of ADR-0045 and the namespace reserved by ADR-0031.
- **Consumers subscribe via the bus.** HolmesGPT — and any other consumer (dashboards,
  the self-healing agent, future analysers) — receives alerts by **subscribing to the
  bus**, not by a dedicated wire. AlertManager emits once; consumers choose what they
  consume.
- **No direct AlertManager → HolmesGPT wire.** Point-to-point coupling between the alert
  producer and a single named consumer is prohibited; it defeats the purpose of the bus.
- **Metrics (a direct stream, NOT events).** HolmesGPT pulls metrics **directly from
  Mimir** as a query stream it uses to investigate. Metrics are not events and must
  **not** be published onto the bus. This direct read path is unaffected by the alerting
  decision above and remains as-is.

This **corrects** the corpus. The existing direct `AlertManager → HolmesGPT` trigger
described in the glossary and §6.7 is superseded by bus-mediated alert consumption. The
corpus change is a fix to apply in the implementation pass.

## Consequences

- New alert consumers are added by subscribing to `platform.observability` on the bus;
  no change to AlertManager and no new wires. The producer stays ignorant of its
  consumers, as the bus intends.
- Alerts and metrics are cleanly distinguished: alerts flow through the bus as discrete
  events with an A13-owned schema; metrics flow as a direct Mimir query stream. The bus
  carries the events and never the metric stream.
- The glossary and §6.7 must be updated in the implementation pass to remove the direct
  `AlertManager → HolmesGPT` trigger and describe bus-mediated alert consumption
  instead. Until that fix lands, the corpus is internally inconsistent with this ADR,
  and this ADR is authoritative.
- HolmesGPT's direct read from Mimir is preserved unchanged; this decision does not
  route metrics through the bus or otherwise alter that path.

## References

- [ADR 0045](./0045-single-owner-per-cloudevent-namespace.md) (single owner per CloudEvent namespace; A13 owns `platform.observability`)
- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (CloudEvent top-level taxonomy; the `platform.observability` namespace)
- [ADR 0015](./0015-tempo-langfuse-correlated-tracing.md) (Tempo + Langfuse correlated tracing)
- `_meta/reviews/DECISIONS-LOG.md` ("Observability note (corrected)" — the authoritative ruling recorded here)

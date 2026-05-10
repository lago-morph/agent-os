# ADR 0030: CRD and API versioning policy with per-component ownership

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform exposes several distinct API surfaces — CRDs, CloudEvent schemas, the Platform SDK (Python and TypeScript), the `agent-platform` CLI, A2A/MCP interfaces declared by Platform Agents, and HTTP APIs exposed by custom services (kopf operator admin, Knative adapters, audit pipeline, etc.). Each surface needs a versioning model that is explicit to callers, predictable across breaking changes, and discoverable in per-product documentation. The architecture-level invariant (architecture-backlog.md § 6) is that every API surface carries an explicit version, owned by the component that exposes it; without an ADR pinning the model per surface, individual components would drift toward incompatible conventions and synchronized "platform release" coordination would creep in by default.

## Decision

CRDs use Kubernetes API versioning (`v1alpha1`, `v1beta1`, `v1`); breaking changes go through a new `vN` API group with conversion webhooks, and `vN-1` is deprecated for at least one minor platform release before removal. CloudEvent schemas in the B12 registry carry the CloudEvents-native `specversion` plus a per-event-type `schemaVersion` field, with backward-compatible additions bumping minor and breaking changes minting a new event type rather than breaking subscribers. The Platform SDK and `agent-platform` CLI use semantic versioning, and HTTP APIs use URL-path versioning (`/v1/...`) with deprecated versions reachable for at least one platform release after replacement. Each surface is owned end-to-end by the component that exposes it — declaring the current version, documenting the compatibility matrix, emitting deprecation warnings, and providing migration guidance — with no centrally-coordinated lockstep platform version.

## Consequences

- Each component (kopf operator B13 for capability/key/budget CRDs; ARK install A5 for ARK CRDs; agent-sandbox install A6 for sandbox CRDs; Crossplane compositions B4 for XRs; B19 for Approval; B6 for SDK; B9 for CLI; B12 for the event registry; B17/B18 agents for their A2A/MCP interfaces) owns its versioning lifecycle and its per-product compatibility documentation (per architecture-overview.md § 10.5).
- Minimum deprecation window is one minor platform release, but the calendar definition of "platform release" is design-time and deferred per architecture-backlog.md § 1.18.
- CRD breaking changes require a new API group plus a conversion webhook; reconciler authors must plan webhook delivery, certificate rotation, and stored-version migration as part of the change.
- CloudEvent breaking changes never break subscribers — they ship as new event types under the same top-level taxonomy (ADR 0031), and old types continue to flow until subscribers migrate.
- The Platform SDK ships a version-pinned compatibility matrix against gateway, ARK, and Letta versions; matrix maintenance and CI verification mechanics are deferred per architecture-backlog.md § 1.18.
- Components evolve independently within the compatibility matrix; there is no synchronized platform version that bumps every surface together, and CI is not expected to enforce such a thing.
- Conversion webhook patterns, exact deprecation calendars, and compatibility-matrix maintenance tooling are explicitly deferred to per-component design (architecture-backlog.md § 1.18).

## References

- architecture-overview.md § 6.13
- architecture-backlog.md § 1.18, 6
- ADR 0031 (CloudEvent taxonomy), ADR 0013 (CapabilitySet schema), ADR 0029 (JWT claim schema)

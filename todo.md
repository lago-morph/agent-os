# TODO

Wrap-up list from the architecture-definition session. Items here represent decisions made but not yet applied to the architecture document, plus open issues that need attention in future sessions.

## Pending edits to apply

These are decisions made in conversation but not yet reflected in the architecture overview or backlog. Apply at the start of the next session.

- **B5 scope finalized to "cross-cutting Headlamp plugins."** Update the B5 row in section 14.2 from its current "Cross-cutting Headlamp framework" framing to specify that B5 owns cross-cutting plugins specifically (capability inspector, approval queue UI, virtual key admin, anything that spans multiple CRDs and doesn't naturally belong to one component). The Headlamp framework / base / branding work stays in A9. Per-component plugins remain owned by their components.

- **Workstream F status.** Confirmed as its own workstream (already added to the document in section 14.6, no edit needed — this is here only as confirmation).

## Open issues requiring decisions

These were flagged at the end of the last response and not yet resolved.

- **Coach component (B10) scope clarification.** Architecture treats Coach and HolmesGPT as two distinct Platform Agents (A14 and B10). One earlier conversation phrasing — "Coach and HolmesGPT are two different agents, probably implemented at different times" — was read as confirming they're separate. If "two different agents at different times" was meant differently (e.g., Coach has multiple phases), confirm.

- **OpenAI ↔ A2A translation source.** Architecture treats this as conditional: handled by LiteLLM if available in OSS, otherwise a small adapter delivered by Component A8 (LibreChat). Needs verification when LiteLLM is actually being installed. If the adapter is required, its design lands in A8.

## Open questions deferred to future sessions

Carried forward from the backlog (kept here for visibility):

- AI agent topology for the implementation effort itself — what kinds of agents implement the platform, how they coordinate, what human-in-the-loop patterns apply. Held for a separate dedicated conversation.
- Specific agent profiles to ship in B17.
- Specific recommended agent compositions for B18.
- Specific OPA policy library content for B16.
- Whether Coach has access to OPA and other components, decided per the design of B10.
- Headlamp deep-link strategy (URL schemas, auth handoff, context preservation).
- Glossary maintenance process — keeping terminology consistent as the platform evolves.

## Cleanup tasks

- Sweep the architecture overview for any remaining "agent" usages that should be "Platform Agent" but were missed in the bulk replace. Cases that are correctly generic (agent SDK, agent-sandbox, agent specs as a generic phrase) should stay.
- Review section 14.7 (dependency graph) for new dependencies introduced by the new components (B16-B21, A17). Some edges may be missing.
- Review the audit and OPA hook-points list in section 6.6 to ensure newly added components (Approval system, dynamic registration, A17 MCP services) are reflected.

## Things that did NOT make it into the architecture document but should be considered

- Capability set pseudocode example uses `+` syntax for additive merge. The actual syntax is design-time but the choice should be made early since it affects how implementers think about layering.
- The two initial Knative trigger flows (AlertManager → HolmesGPT, budget-exceeded → email) are described but not given separate use case diagrams. If we want them as use cases in section 7, that's an addition.
- Production readiness (Workstream F) is a workstream in the document but doesn't have a dependency-graph entry showing what it depends on (everything in A and B finishing).

## Out of session

- README.md is final for now.
- architecture-backlog.md is current with this session's resolutions.
- architecture-overview.md needs the B5 edit listed above; otherwise current.

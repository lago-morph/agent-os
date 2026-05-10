# TODO

Open questions held for future sessions. Items previously listed here as "pending edits" or "cleanup tasks" have been applied to the architecture overview, the backlog, or the new `future-enhancements.md`.

## Open questions deferred to future sessions

These don't need answers now but should be revisited at the right moment.

- **AI agent topology for the implementation effort itself** — what kinds of agents implement the platform, how they coordinate, what human-in-the-loop patterns apply. Held for a separate dedicated conversation, after the architecture is solid enough to plan the implementation against.
- **Specific agent profiles to ship in B17.** Tracked in `architecture-backlog.md` section 4.
- **Specific recommended agent compositions for B18.** Tracked in `architecture-backlog.md` section 4.
- **Specific OPA policy library content for B16.** Tracked in `architecture-backlog.md` section 4.
- **Whether Coach has access to OPA and other components**, decided per the design of B10.
- **Headlamp deep-link strategy** (URL schemas, auth handoff, context preservation). Tracked in `architecture-backlog.md` section 1.15. ADR 0039 (graphical editors for platform CRDs) informs but does not resolve this.
- **Glossary maintenance process** — keeping terminology consistent as the platform evolves. Tracked in `architecture-backlog.md` section 4.
- **Audit retention durations and redaction rules** — ADR 0034 fixes the audit pipeline topology but explicitly defers retention durations, redaction rules, and lifecycle specifics to Workstream F (architecture-backlog § 1.13).
- **Tenant offboarding / decommissioning specifics** — ADR 0037 covers onboarding via XRD; offboarding (tenant deletion ordering, retention-bound cleanup, tenant-scoped quotas) remains deferred.

## Items needing verification during implementation, not now

- **OpenAI ↔ A2A translation source** — handled by LiteLLM if available in OSS at install time, otherwise a small adapter delivered by Component A8 (LibreChat). Verify when LiteLLM is being installed; if the adapter is required, its design lands in A8. Tracked in `architecture-backlog.md` section 1.16.

## Possible additions worth considering (not committed)

- The two initial Knative trigger flows (AlertManager → HolmesGPT, budget-exceeded → email) are described inline in section 6.7 but not given separate use case diagrams in section 7. If we want them as standalone use cases, that's an addition.

# ADR 0022: Knowledge Base as a separate primitive

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Multiple consumers need access to the same body of platform documentation, runbooks, and pinned vendor docs: HolmesGPT for diagnostics, the Interactive Access Agent that fronts LibreChat for developer chat, Coach, and any other Platform Agent that elects to include it in its CapabilitySet (architecture-overview.md § 6.4). Folding this corpus into HolmesGPT (or any single agent) would couple its lifecycle to that agent, prevent reuse, and force every other consumer to route queries through an unrelated agent. The corpus also has its own ingestion cadence (major/minor release flagging for our docs, separate companion project for vendor docs) and lives in an existing retrieval store (OpenSearch, ADR 0009). Treating it as a shared, named primitive matches both the consumer fan-out and the independent ingestion lifecycle.

## Decision

The Knowledge Base is a first-class `RAGStore` named `platform-knowledge-base`, independent of HolmesGPT and any other agent. Any Platform Agent that includes it in its CapabilitySet reaches it through the SDK's `rag.*` API on the standard LiteLLM-mediated call path. It is not built into HolmesGPT, the Interactive Access Agent, Coach, or any other agent — those agents are simply consumers.

## Consequences

- HolmesGPT (ADR 0012) stays cleanly scoped to diagnostics; the Knowledge Base ships and evolves on its own cadence and is reusable by every other Platform Agent that opts in via CapabilitySet.
- All consumer access goes through the SDK `rag.*` API on the LiteLLM-mediated call path, so retrieval is observable, policy-gated, and traced uniformly with the rest of agent traffic.
- The Interactive Access Agent is the default LibreChat endpoint precisely because it includes `platform-knowledge-base` in its CapabilitySet; LibreChat itself does not bind to the Knowledge Base directly.
- Storage and indexing reuse OpenSearch (ADR 0009) — no separate retrieval substrate is introduced.
- Sub-decision deferred (architecture-backlog.md § 1.7): the agent-pod access pattern between SDK API calls and a read-only filesystem mount of indexed content is decided per agent at design time (semantic queries via SDK, structured grep-style reference via mount); cache invalidation and per-content-type behavior are pinned down when that decision is made.
- Revisit trigger (architecture-backlog.md § 3.10): a richer search-and-browse UI beyond LibreChat is added when platform builders express a felt need for it; until then, conversational access via the Interactive Access Agent is the sole user-facing surface.
- Vendor documentation acquisition and re-indexing is handled by a separate companion project referenced by the architecture but not part of v1.0 scope (ADR 0024); this ADR commits only to the Knowledge Base being the consumption-side primitive that ingests whatever that project publishes.

## References

- architecture-overview.md § 6.4
- architecture-backlog.md § 1.7, § 3.10, § 7 item 22
- ADR 0009 (OpenSearch as search and vector store)
- ADR 0012 (HolmesGPT as a first-class Platform Agent)
- ADR 0024 (Vendor doc acquisition as a separate companion project)

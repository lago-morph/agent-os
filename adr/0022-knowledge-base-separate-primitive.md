# ADR 0022: Knowledge Base as a separate primitive

## Status
Accepted

## Context
The platform needs a shared, queryable corpus of authoritative content — our own portal, per-product docs and runbooks, and pinned vendor documentation — that agents and humans can rely on. Several consumers want it: HolmesGPT for diagnostics, the Interactive Access Agent for chat-driven Q&A in LibreChat, Coach for skill authoring guidance, and any future Platform Agent that benefits from grounded retrieval.

The shape question is whether the Knowledge Base should be embedded inside the agent that most visibly uses it (HolmesGPT was the obvious candidate), maintained as per-agent private corpora, or established as its own platform primitive that any agent can include in its CapabilitySet. The platform already commits to `RAGStore` as the retrieval primitive, OpenSearch as the storage layer (ADR 0009), and `RAGStore` references being attached to agents through CapabilitySets (ADR 0013). Those primitives already support the third option directly.

Re-indexing cadence is the second question. Authored content (the MkDocs portal and per-product docs) churns constantly with patch-level edits that don't justify an indexing pass. Vendor documentation arrives on the vendor's schedule, not ours, and acquiring and normalising it is a project of its own.

## Decision
The Knowledge Base is a first-class `RAGStore` named `platform-knowledge-base`, **independent of any single agent**. It is a shared platform primitive that any Platform Agent may reference through its CapabilitySet.

- **Content sources.** The MkDocs portal (ADR 0008), per-product architecture-specific docs and runbooks, and pinned versions of vendor documentation.
- **Storage and retrieval.** OpenSearch indexes (ADR 0009) — the same retrieval primitive used elsewhere; nothing KB-specific in the storage layer.
- **Access.**
  - Developers reach the KB through LibreChat (ADR 0007) by talking to the **Interactive Access Agent**, a general-purpose Platform Agent that includes `platform-knowledge-base` in its CapabilitySet and is exposed as the default LibreChat endpoint. It is implemented early so LibreChat has a working LLM-reaching path from day one using our own infrastructure.
  - Agents (HolmesGPT per ADR 0012, Coach, and any other Platform Agent that includes the KB in its CapabilitySet) reach it through the SDK's `rag.*` API, going through LiteLLM as part of the standard call path.
- **Re-indexing rules.**
  - Authored content: the contributor flags a commit as a major or minor release change; major and minor changes trigger re-indexing, patch-level changes do not. Human-judgment-driven, intentionally — small edits don't earn the indexing churn.
  - Vendor content: acquisition, normalisation, and re-indexing are owned by a **separate companion project** (ADR 0024) that scans for vendor version differences and triggers re-indexing on major or minor releases.
- **Per-content-type access pattern (SDK API vs filesystem mount) is deferred** to design (architecture backlog §1.7). Both options remain on the table; the choice is per-agent and per-content-type, made when an agent is designed.

## Consequences
**Positive.**
- One corpus, one indexing pipeline, one set of governance and observability hooks. New agents that need grounded retrieval reference the existing `RAGStore` rather than standing one up.
- Decoupling the KB from HolmesGPT means HolmesGPT, the Interactive Access Agent, Coach, and future agents query exactly the same content, with the same freshness — no fork, no drift.
- Re-indexing tied to author-flagged major/minor releases keeps the index stable and the cost predictable while still capturing meaningful changes.
- The vendor-content companion project can evolve on its own cadence without entangling the platform's release schedule.

**Negative / accepted trade-offs.**
- A shared resource is everyone's resource and no one's: ownership of indexing freshness, retrieval quality, and cost belongs to the platform team, not to any single agent's team. This is a feature for governance and a friction point for ad-hoc tuning.
- Author judgment on "major/minor vs patch" is fallible; some material changes will land tagged as patch and miss re-indexing until a later release bumps the cadence. Acceptable: the alternative is mechanical re-indexing on every commit, which costs more than it's worth at v1.0.
- Vendor doc dependency on a companion project means KB freshness for vendor content is bounded by that project's delivery, not ours.

## Alternatives considered
- **Bundle the KB inside HolmesGPT.** Rejected — couples a platform-shared resource to a single consumer; other agents would have to call HolmesGPT to reach content that has nothing to do with diagnostics. Inverts the dependency.
- **Per-agent private KBs.** Rejected — each agent re-acquiring and re-indexing the same docs produces content sprawl, divergent freshness, duplicated cost, and no single place to govern access or audit retrieval quality.
- **Mechanical re-indexing on every commit.** Rejected — patch-level edits dominate authoring traffic; indexing churn would dwarf signal.
- **Defer vendor docs entirely.** Rejected — vendor documentation is high-value retrieval content; a separate companion project is the right ownership boundary, not abandonment.

## Related
- ADR 0007 — LibreChat as locked-down frontend (developers reach the KB via the Interactive Access Agent through LibreChat).
- ADR 0008 — Material for MkDocs as documentation portal (the portal is a KB content source).
- ADR 0009 — OpenSearch as search/vector store (indexes the KB).
- ADR 0012 — HolmesGPT as Platform Agent (consumes the KB via CapabilitySet, does not embed it).
- ADR 0013 — Capability CRD and CapabilitySet layering (the mechanism by which agents reference the KB).
- ADR 0024 — Vendor documentation acquisition companion project (owns vendor-content acquisition and re-indexing).
- Architecture overview §6.4 (Knowledge Base as separate primitive), §10 (docs portal), §14 (Workstream C8 indexing pipeline).
- Architecture backlog §1.7 (per-content-type access pattern deferred), §3.10 (richer KB UI trigger), §7 (ADR candidate).

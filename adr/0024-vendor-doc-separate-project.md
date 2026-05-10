# ADR 0024: Vendor documentation acquisition is a separate companion project

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The Knowledge Base (`platform-knowledge-base` RAGStore, ADR 0022) is a shared
primitive consumed by HolmesGPT, the Interactive Access Agent, and any other
Platform Agent that includes it in its CapabilitySet. Its content sources include
our own MkDocs portal, per-product runbooks, and pinned versions of vendor
documentation. Acquiring vendor docs is qualitatively different from authoring our
own: it requires crawling or syncing third-party sources, tracking upstream
versions, navigating licensing and redistribution constraints, and triggering
re-indexing on major and minor vendor releases. That pipeline has its own lifecycle,
its own ownership, and its own risk profile, and bundling it into v1.0 architecture
scope would conflate "Knowledge Base as a primitive" with "data plane that fills
the Knowledge Base."

## Decision

Vendor documentation acquisition and re-indexing live in a **separate companion
project**, referenced from this architecture but explicitly out of v1.0
architecture scope. The architecture defines the consumption contract — the
`platform-knowledge-base` RAGStore and the `rag.*` SDK API — and treats the
companion project as an upstream producer that writes pinned vendor doc versions
into that store. Architecture-side ownership is limited to the integration point
(Workstream F, item F3) and to keeping the cross-reference current; project
ownership and links are filled in later.

## Consequences

- The Knowledge Base primitive (ADR 0022) consumes whatever the companion project
  produces; the platform does not gate on a vendor-doc pipeline existing.
- v1.0 ships with the Knowledge Base populated from our own authored docs and
  runbooks (Workstream C, item C8); vendor doc coverage grows as the companion
  project matures, with no architectural change required.
- Clean separation of concerns: licensing, crawl politeness, upstream version
  tracking, and freshness SLAs are the companion project's problem, not ours.
- The contract between the two projects is narrow: the companion project writes
  into the named RAGStore using the same indexing conventions as our authored
  docs, and re-indexes on upstream major/minor releases.
- Risk: until the companion project lands, RAG queries that depend on vendor doc
  coverage will be answered only from our authored material. This is acceptable
  for v1.0 and is called out in the documentation plan.
- Risk: the companion project's choices (which vendors, which versions, how
  pinned) shape Knowledge Base quality. Coordination is owned by Workstream F
  (item F3) so the integration point doesn't drift.
- Future ADR scope: if the companion project ever requires architectural changes
  to the Knowledge Base contract (new metadata fields, new access patterns), that
  is a separate ADR against ADR 0022, not a re-litigation of this scoping
  decision.

## References

- architecture-overview.md § 6.4 (The Knowledge Base as a separate primitive)
- architecture-overview.md § 14.6 Workstream C item C8, Workstream F item F3
- architecture-overview.md § Glossary entry "Knowledge Base"
- README.md "What's NOT in this repository"
- ADR 0022 (Knowledge Base as a separate primitive)
- ADR 0008 (Material for MkDocs documentation portal)
- ADR 0009 (OpenSearch as RAG / vector store backend)

# ADR 0008: Material for MkDocs as the documentation portal

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform treats documentation as a first-class deliverable: Diataxis-structured content (tutorials, how-tos, reference, explanation), per-product docs, runbooks, maintainer/extender docs, and a docs-on-docs section. The same corpus is also indexed as the `platform-knowledge-base` `RAGStore`, so Platform Agents (LibreChat, HolmesGPT) can query it. We need a portal that supports docs-as-code in the same repos as the components being documented, plays well with GitOps and PR review, has good versioning and search, and stays low-maintenance for a small team.

## Decision

We use **Material for MkDocs** as the documentation portal. Markdown sources live in the same repositories as the code they document, ship through normal PR review, and are built and deployed as a static site. The portal is reasonable-looking out of the box, supports per-version builds, and emits Markdown that the C8 indexing pipeline can re-index into the `platform-knowledge-base` RAGStore on contributor-flagged major/minor commits.

## Alternatives considered

- **Backstage** — Heavy to operate and only earns its weight if we also adopt its software catalog and plugin ecosystem; we do not, and our CRD-driven catalog story already lives in Headlamp and Crossplane. The TechDocs portion alone does not justify the footprint.
- **Confluence** — Breaks docs-as-code and GitOps: edits happen in a separate WYSIWYG tool, version pinning to code releases is awkward, PR-based review and automated indexing into the RAGStore become custom integrations, and authoring lives outside the engineer's normal workflow.

## Consequences

- Documentation is committed as Markdown alongside the code it describes; PR review covers code and docs together (docs-as-code commitment).
- Workstream C owns the portal infrastructure (component C1: MkDocs setup, search, versioning, contribution workflow), Diataxis content (C2-C5), cross-cutting runbooks (C6), maintainer/extender docs (C7), the RAG indexing pipeline (C8), and docs-on-docs (C9). Per-product docs and per-product runbooks remain with the Workstream A component that owns each product.
- Re-indexing into the `platform-knowledge-base` RAGStore is gated on contributor-flagged major/minor commits; patch-level edits do not trigger churn.
- We do not get a software catalog, scorecards, or a plugin marketplace — capabilities Backstage would have provided. Catalog-style views remain the responsibility of Headlamp + Crossplane; if a true service catalog need emerges later, it is a separate decision and can sit alongside the portal.
- Per-version doc builds are required so that pinned vendor docs and our own platform docs can be served at the same versions referenced by deployed components (see ADR 0024 on vendor doc separation).
- Authoring requires Markdown fluency and a local MkDocs preview; non-engineering contributors edit through PRs rather than a WYSIWYG surface, which is an accepted trade-off.
- Section 1.15 of the backlog (documentation portal vs Headlamp linking strategy) remains open and is resolved later; this ADR does not preempt it.

## References

- architecture-overview.md § 10, § 14.3
- architecture-backlog.md § 2.6

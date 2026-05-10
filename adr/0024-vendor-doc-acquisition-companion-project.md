# ADR 0024: Vendor doc acquisition is a separate companion project

## Status
Accepted

## Context
The Knowledge Base (architecture overview §6.4, ADR 0022) is a single first-class `RAGStore` named `platform-knowledge-base` that any Platform Agent may include in its CapabilitySet. Its content has three sources: our own MkDocs portal (ADR 0008), our per-product runbooks, and **pinned vendor documentation** for the products we install and operate (LiteLLM, Langfuse, Argo Workflows, OpenSearch, Tempo, Mimir, etc. — see §14).

For our own content, the contributor flags a commit as major/minor (re-index) or patch (skip) — human-judgment-driven, deliberately lightweight. Vendor documentation is a different problem entirely. It lives outside our Git repos, ships on its own cadence, has its own copyright and licensing constraints, and needs version-diff scanning to know when a pinned version's docs have changed enough to warrant RAGStore re-indexing on major/minor release. Building all of that — fetchers, normalizers, version-diff scanners, license bookkeeping, refresh schedulers — inside the v1.0 architecture would meaningfully expand scope and entangle a moving-target external pipeline with the platform's own release cadence.

Architecture backlog §7 item 24 calls this out as an explicit decision rather than letting it drift into the platform by default.

## Decision
Vendor documentation acquisition, version-diff scanning, and the trigger that drives RAGStore re-indexing on major/minor vendor releases are owned by a **separate companion project**. The v1.0 architecture references this companion project as the source of pinned vendor documentation feeding `platform-knowledge-base`, but does not own its design, implementation, or operation within v1.0 scope.

The contract between the platform and the companion project is narrow:
- The companion project produces pinned vendor documentation in a form the Workstream C indexing pipeline (component C8) can consume.
- Re-indexing is triggered on major or minor vendor releases; patch-level vendor changes do not trigger re-indexing, mirroring the rule for our own docs.
- Project ownership, repository location, and integration links are filled in later; the architecture only commits to the seam.

## Consequences
Positive:
- v1.0 scope stays disciplined. The platform owns its own MkDocs portal and per-product runbooks; vendor docs come in through a clearly named external seam rather than as a hidden in-tree subsystem.
- Vendor copyright, licensing, and refresh-cadence concerns live in a project whose lifecycle can match those concerns — independent of platform releases.
- The Knowledge Base contract (ADR 0022) is unchanged: consumers see one `RAGStore`, regardless of which source produced any given chunk.
- The companion project can evolve its acquisition strategy (scrape, vendor-provided archives, git mirrors, API pulls) without architectural churn on our side.

Negative:
- v1.0 ships without authoritative vendor docs in the Knowledge Base unless the companion project is ready in time. Until it lands, agents that need vendor specifics fall back to our own runbooks plus whatever the LLM already knows — which is acceptable for v1.0 but is a real gap.
- A cross-project seam now exists that must be kept honest: indexing-pipeline expectations on one side, companion-project outputs on the other. Drift between them would silently degrade Knowledge Base quality.
- Operational ownership (who runs the companion project, who is paged when version-diff scanning breaks) must be settled before the seam is load-bearing.

## Alternatives considered
- **Acquire vendor docs directly inside the platform (in-tree under Workstream C).** Rejected on scope grounds. Building fetchers, version-diff scanners, license tracking, and refresh schedulers for an open-ended set of vendor sites is a non-trivial subsystem with its own cadence; folding it into v1.0 would meaningfully delay the platform without making it materially better at what it is supposed to do.
- **Skip pinning entirely and have agents query vendor-hosted docs at runtime.** Rejected. We lose version pinning (vendor sites generally show "latest"), we lose offline access in air-gapped or restricted-egress installs, we add an external dependency to every retrieval, and we cannot index the content into `platform-knowledge-base` for semantic retrieval the way ADR 0022 requires.
- **Defer vendor docs from the Knowledge Base altogether.** Rejected as the steady-state answer. Pinned vendor docs are valuable enough to plan for; the question is only who builds the pipeline. Naming a companion project keeps the door open without forcing the work into v1.0.

## Related
- ADR 0022 — Knowledge Base as a separate primitive (the consumer of vendor doc output).
- ADR 0008 — Material for MkDocs as the documentation portal (a different, in-tree content source for the same `RAGStore`).
- Architecture overview §6.4 — Knowledge Base sources and re-indexing rules.
- Architecture overview §14 — Workstream C and the indexing pipeline (C8) on the platform side of the seam.
- Architecture backlog §7 item 24 — decision register entry this ADR records.

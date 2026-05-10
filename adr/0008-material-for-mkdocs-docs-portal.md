# ADR 0008: Material for MkDocs as the documentation portal

## Status
Accepted

## Context
The platform needs a documentation portal that hosts tutorials, how-to guides, reference, explanation, per-product docs, and runbooks (architecture overview §10), and that is also indexed as the `platform-knowledge-base` `RAGStore` available to any Platform Agent that includes it in its CapabilitySet (architecture overview §5 — Documentation portal row, §6.4). Two properties are non-negotiable. First, docs-as-code: documentation lives in the same Git repos as the code it documents, reviewed via PRs, deployed via the same GitOps pipeline as everything else. Second, the rendered output must be cheap to re-index into a `RAGStore` so the Knowledge Base stays in sync with what humans read.

Three options were on the table (architecture backlog §2.6, §7 decision register entry 8): Backstage, Material for MkDocs, and Confluence. Backstage is a developer portal with a service catalog; its docs surface (TechDocs) is built on MkDocs anyway, and the rest of Backstage is heavy infrastructure that is only worth running if we also adopt its catalog — which we do not. Confluence is a wiki-style SaaS product whose source of truth lives in its own database, breaking docs-as-code and GitOps and making clean re-indexing into a `RAGStore` substantially harder.

## Decision
Use **Material for MkDocs** as the documentation portal. Markdown lives next to the code it documents, the portal builds to a static site, the build runs in CI, the rendered site deploys via the same GitOps path as the rest of the platform, and the same content is indexed into the `platform-knowledge-base` `RAGStore` by the Workstream C indexing pipeline (component C8). Component C1 owns the portal infrastructure — theme, search, versioning, contribution workflow — and components C2–C7 own the Diataxis content, with per-product docs and runbooks delivered by Workstream A components alongside the components they describe.

## Consequences
Positive:
- Docs-as-code preserved end-to-end: every change is a PR against Git, reviewed, versioned, and deployed by the same GitOps pipeline as code.
- The same Markdown that renders the portal feeds the `platform-knowledge-base` `RAGStore`; humans and agents read the same source of truth.
- Material for MkDocs is well-themed out of the box, has good built-in search, supports versioned docs, and has a low operational footprint — a static site behind the standard ingress.
- Tutorials and how-tos can ship in parallel with the components they document (architecture overview §10), because the toolchain has no per-product setup cost.
- HolmesGPT and the Interactive Access Agent (architecture overview §6.4) reach the same indexed content through the SDK's `rag.*` API, going through LiteLLM under the standard auth and audit path.

Negative:
- No service catalog, no scaffolding/templating UI, no plugin ecosystem comparable to Backstage's — if those needs grow, we will revisit (likely as a separate component, not as a docs-portal swap).
- Markdown is plain text; richer in-page interactivity (embedded React widgets, live API playgrounds) is not free and would have to be built.
- Authoring is in Markdown in Git, not a WYSIWYG editor. Non-engineer contributors need a basic Git workflow; the docs-on-docs section (architecture overview §10.8) covers the contribution flow.

## Alternatives considered
- **Backstage.** Rejected: heavy to operate and only worth running if we also adopt its service catalog, which we are not. Its TechDocs layer is MkDocs underneath, so we would inherit the cost without a proportional benefit.
- **Confluence.** Rejected: the source of truth lives in Confluence's database, not Git. That breaks docs-as-code and GitOps, makes re-indexing into a `RAGStore` substantially harder, and reintroduces a separate user-and-permissions surface outside Keycloak SSO.
- **Plain MkDocs (no Material theme).** Considered and rejected as a non-improvement: same engine, worse defaults for theme, search, navigation, and versioning. Material adds polish without changing the underlying contract.
- **Docusaurus / Hugo / other static-site generators.** Considered. None offered a meaningful advantage over Material for MkDocs for our content mix, and switching would not change the docs-as-code or RAG-indexing story.

## Related
- Architecture overview §5 (Documentation portal row), §6.4 (Knowledge Base as a separate primitive), §10 (documentation content plan), §14.3 (Workstream C — Documentation portal and content).
- Architecture backlog §2.6 (this decision), §7 (decision register entry 8).
- ADR 0022 (Knowledge Base as a separate primitive — content sources include the MkDocs portal).
- ADR 0024 (Vendor doc acquisition is a separate companion project that feeds the same `RAGStore`).

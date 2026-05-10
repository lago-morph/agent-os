# ADR 0020: Initial MCP services set

## Status

Accepted

## Context

The platform exposes external tools to Platform Agents through LiteLLM's MCP gateway (architecture §6.1). Each MCP server is declared as an `MCPServer` CRD; the kopf operator (ADR 0006) reconciles it into LiteLLM's registry, secrets are injected via External Secrets Operator (ESO), and per-agent access is gated by CapabilitySet inclusion plus OPA runtime decisions (ADR 0002).

For v1.0 we need a concrete starting set of third-party MCP services that covers the most common platform-builder needs without committing the team to a long tail of integrations on day one. Architecture §14 (component A17) names that set; this ADR records the choice and the credential model. Per-service implementation specifics — OAuth scopes, revocation flows, exact secret shapes, budget enforcement details — are deferred to design-time per architecture-backlog §1.11.

## Decision

The initial MCP services set for v1.0 is:

- **GitHub** — registered as an `MCPServer` CRD, available in two modes: a **system-credential mode** (a platform-owned GitHub App / token, ESO-injected, used by agents that operate on platform-owned repos) and a **user-credential mode** for interactive scenarios where the human user authenticates GitHub themselves and **LiteLLM brokers the OAuth flow** on their behalf. Per-agent access is restricted via CapabilitySet membership and OPA.
- **Google Drive** — same dual-mode shape as GitHub: system-credential mode for service-account access to platform-owned drives, user-credential mode for user-owned content with LiteLLM brokering OAuth.
- **Firecrawl** — system-credential only (one platform account serves many agents). Secret stored via ESO and referenced by the `MCPServer` CRD. Because a single credential fans out across agents, **per-agent budget enforcement** is required — handled through the same BudgetPolicy / callbacks machinery the gateway already runs.
- **Context7** — simplest of the four: install the MCP server, register an `MCPServer` CRD, no per-user credential flow.

All four follow the same pattern: declared as `MCPServer` CRDs, reconciled by the kopf operator into LiteLLM (ADR 0006), secrets via ESO, access gated by CapabilitySet + OPA. User-credential modes use LiteLLM as the OAuth broker — LibreChat does not surface user-mode MCP itself; the gateway does (consistent with ADR 0007's locked-down frontend posture).

Per-service implementation details — exact OAuth scopes, revocation flows, budget thresholds, MCP server image selection, and the precise ESO `SecretStore` shape — are deferred to design (architecture-backlog §1.11) and tracked under component A17.

## Consequences

- Platform builders get useful tools (code, documents, web scraping, library docs) on day one without each team having to negotiate their own MCP integration.
- Two credential modes (system vs user) must be supported by the gateway and the kopf operator from v1.0; this complexity is intrinsic to GitHub and Drive and would surface eventually regardless.
- Firecrawl's single-credential / many-agent fan-out makes budget enforcement a launch-blocker for that service specifically.
- Adding more MCP services after v1.0 is a CRD-and-OPA-policy exercise, not a new architectural pattern.
- The four chosen services define the test surface for the MCP gateway path through ESO, OPA, and LiteLLM brokering.

## Alternatives considered

- **Defer all third-party MCP integrations to post-v1.0.** Rejected — the platform's value is composing agents with real tools; shipping with zero external MCP services makes early users build their own integrations and exercises none of the gateway's MCP path in production.
- **Ship a much larger initial set (Slack, Jira, Notion, Linear, Confluence, GitLab, AWS, etc.).** Rejected — each integration carries its own credential model, scope review, and operational surface; the blast radius and review load would push v1.0. The four chosen cover the dominant platform-builder use cases (source, documents, web, library docs) and validate every credential mode the gateway needs to support.

## Related

- ADR 0002 — OPA Gatekeeper as policy engine; OPA gates per-agent MCP access.
- ADR 0006 — Python kopf operator reconciles `MCPServer` CRDs into LiteLLM.
- ADR 0007 — LibreChat is locked-down; user-mode MCP OAuth is brokered by the gateway, not surfaced by the frontend.
- Architecture overview §6.1 (Gateway), §14 component A17.
- architecture-backlog §1.11 (per-service implementation details deferred).

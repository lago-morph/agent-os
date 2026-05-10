# ADR 0020: Initial MCP services set — GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, Web search + scrape

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Platform Agents reach external tools through MCP, brokered by LiteLLM and
governed by `MCPServer` CRDs (architecture-overview.md §6.1, §6.8). For v1.0
the platform commits to a fixed initial set of MCP services rather than letting
adoption be ad-hoc, so that ESO secret plumbing, OPA admission rules,
CapabilitySet wiring, audit tagging, and Headlamp inspection have concrete,
exercised integrations from day one. Picking the set up front also lets early
Platform Agents (Interactive Access Agent, HolmesGPT, Coach) compose against a
known capability surface rather than wait on per-team MCP onboarding.

## Decision

The v1.0 initial MCP services integration (component **A17**) is the following
set, each registered as a namespaced `MCPServer` CRD reconciled by the kopf
operator (B13) into the LiteLLM registry, with secrets sourced via ESO and
access gated by CapabilitySet inclusion plus OPA:

- **GitHub** — system-credential and user-credential modes.
- **Google Drive** — system-credential and user-credential modes.
- **Context7** — install + register; documentation/library lookup.
- **OpenSearch** — *system-mediated mode* against the platform's own
  OpenSearch (deployed in-cluster on kind, AWS-managed via Crossplane on AWS
  per ADR 0009/0033), plus an *agent/tenant/user-credentialed mode* for
  connecting to externally operated OpenSearch instances.
- **Postgres** — agent/tenant/user-scoped database access. Per-agent /
  per-tenant / per-user database provisioning is wrapped by a **new Crossplane
  XRD** (working name `AgentDatabase`) that creates the database, role, and
  grants. RBAC + OPA decide which agent gets which database.
- **MongoDB** — same pattern as Postgres (its own XRD-fronted provisioning,
  same RBAC/OPA gating).
- **Web search + web scrape** — generic in-cluster MCP servers, either
  implemented by us or adopted from an existing OSS implementation. The
  endpoints themselves do **not** carry the platform's full RBAC/OPA stack
  internally; access control is enforced at the **MCP-server-access** layer,
  where OPA can block specific search or scrape requests by policy.

**Removed** from the initial set: Firecrawl. Its role is taken by the generic
web-search + web-scrape services. A commercial fallback (Firecrawl, Google
Search API, etc.) for IP-block / rate-limit scenarios is acknowledged but
deferred to future-enhancements.

Per-service implementation details — exact auth flows, scopes, secret shapes,
revocation, budget enforcement, per-XRD field shape, OPA bundle contents —
remain out of scope for this ADR and deferred per architecture-backlog.md
§1.11.

## Consequences

- **LiteLLM is the single broker for MCP credentials and OAuth flows.** All
  services route through the gateway's MCP path and share the same callback
  chain for audit, OPA, and Langfuse instrumentation
  (architecture-overview.md §6.1).
- **Two credential modes are first-class.** GitHub and Google Drive ship with
  both system-credential mode (a platform-managed identity shared across
  agents) and user-credential mode (LiteLLM brokers the user's own OAuth flow
  for interactive scenarios). This shape is load-bearing because it sets the
  pattern any later MCP service with delegated user auth must follow
  (architecture-overview.md §14.1 A17).
- **System-mediated mode is exercised by OpenSearch.** Connecting to the
  platform's own OpenSearch with platform-mediated identity is a third
  distinct credential shape alongside system- and user-credential, and it
  validates the in-cluster vs AWS-managed dual-hosting story (ADR 0009,
  ADR 0033).
- **Per-agent/tenant/user database provisioning becomes a Crossplane XR.**
  Postgres and MongoDB each get an XRD (e.g., `AgentDatabase`,
  `AgentMongoDatabase`) that follows the platform's composition pattern from
  ADR 0021. This is the first place the XR pattern is used to mint *runtime*
  per-agent state rather than only infra; the precedent matters for later
  per-agent resources.
- **RBAC + OPA decide database assignment.** Which agent / tenant / user
  resolves to which provisioned database is gated at admission and at request
  time, not hard-coded into the MCPServer spec (ADR 0014).
- **MCP-server-access is the policy enforcement point for web search/scrape.**
  The generic search/scrape endpoints stay simple internally; OPA at the
  access layer blocks disallowed queries, domains, or scrape targets. This
  bundle is a primary target for the policy simulator (ADR 0038) before
  rollout, since false-positives degrade agent capability and false-negatives
  leak egress.
- **IP-block risk is acknowledged.** Self-hosted scraping can be blocked by
  upstream sites; the commercial-fallback path (Firecrawl, Google Search API)
  is deferred to future-enhancements rather than landed in v1.0.
- **Per-service details deferred.** Exact OAuth broker mechanics, requested
  scopes, revocation flows, ESO secret shapes, per-XRD field schemas, OPA
  bundle contents for search/scrape, and budget enforcement specifics are all
  design-time per architecture-backlog.md §1.11.
- **Health-check signals are deferred.** A17 inherits the architecture's
  no-synthetic-health-layer stance: LiteLLM uses only what the MCP protocol
  itself exposes, with surfacing details deferred per
  architecture-backlog.md §1.9 (architecture-overview.md §6.1).
- **No bypass path.** Unapproved MCP servers are not reachable from a
  Platform Agent; adding another service in v1.0 means landing a new
  `MCPServer` CRD plus its ESO/OPA wiring, not a gateway-side toggle
  (architecture-overview.md §6.8 glossary "Approved capability").
- **Initial-targets alignment.** The set is sized for the v1.0 initial
  implementation targets (ADR 0033) and is what the early Platform Agents
  compose against via CapabilitySet (ADR 0013).

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.1](../architecture-overview.md#61-gateway-architecture), [§ 14.1](../architecture-overview.md#141-workstream-a--platform-installation-and-operations) (component A17)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.9](../architecture-backlog.md#19-mcp-server-health-check-details), [§ 1.11](../architecture-backlog.md#111-initial-mcp-services-details-a17)
- [ADR 0009](./0009-opensearch-search-vector-store.md) (OpenSearch hosting)
- [ADR 0013](./0013-capability-crd-and-capabilityset-layering.md) (Capability CRD and CapabilitySet layering)
- [ADR 0014](./0014-postgres-primary-opensearch-retrieval.md) (Postgres / managed databases)
- [ADR 0021](./0021-grafanadashboard-xrs.md) (Crossplane XR composition pattern)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (initial implementation targets / dual-mode hosting)
- [ADR 0038](./0038-policy-simulators.md) (policy simulator — used for search/scrape OPA bundle)

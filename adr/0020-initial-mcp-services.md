# ADR 0020: Initial MCP services set — GitHub, Google Drive, Firecrawl, Context7

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

The v1.0 initial MCP services integration (component **A17**) is exactly four
servers: **GitHub** (in both system-credential and user-credential modes),
**Google Drive**, **Firecrawl**, and **Context7**. Each is registered as a
namespaced `MCPServer` CRD reconciled by the kopf operator (B13) into the
LiteLLM registry, with secrets sourced via ESO and access gated by
CapabilitySet inclusion plus OPA. Per-service implementation details — exact
auth flows, scopes, secret shapes, revocation, budget enforcement — are
out of scope for this ADR and deferred per architecture-backlog.md §1.11.

## Consequences

- **LiteLLM is the single broker for MCP credentials and OAuth flows.** All
  four services route through the gateway's MCP path and share the same
  callback chain for audit, OPA, and Langfuse instrumentation
  (architecture-overview.md §6.1).
- **Two credential modes are first-class.** GitHub and Google Drive ship with
  both system-credential mode (a platform-managed identity shared across
  agents) and user-credential mode (LiteLLM brokers the user's own OAuth flow
  for interactive scenarios). This shape is load-bearing because it sets the
  pattern any later MCP service with delegated user auth must follow
  (architecture-overview.md §14.1 A17).
- **Firecrawl exercises platform-managed secrets and shared-account budget
  enforcement.** Because one Firecrawl account fans out to many agents, the
  per-agent budget surface (OPA + LiteLLM) gets tested on a system-credential
  service from the start.
- **Per-service details deferred.** Exact OAuth broker mechanics for GitHub
  user-credential mode, requested scopes, revocation flows, Firecrawl secret
  storage shape and per-agent restriction model, Google Drive specifics, and
  Context7 install particulars are all design-time per
  architecture-backlog.md §1.11.
- **Health-check signals are deferred.** A17 inherits the architecture's
  no-synthetic-health-layer stance: LiteLLM uses only what the MCP protocol
  itself exposes, with surfacing details deferred per
  architecture-backlog.md §1.9 (architecture-overview.md §6.1).
- **No bypass path.** Unapproved MCP servers are not reachable from a
  Platform Agent; adding a fifth service in v1.0 means landing a new
  `MCPServer` CRD plus its ESO/OPA wiring, not a gateway-side toggle
  (architecture-overview.md §6.8 glossary "Approved capability").
- **Initial-targets alignment.** The set is sized for the v1.0 initial
  implementation targets (ADR 0033) and is what the early Platform Agents
  compose against via CapabilitySet (ADR 0013).

## References

- architecture-overview.md § 6.1, § 14.1 (component A17)
- architecture-backlog.md § 1.9, § 1.11
- ADR 0013 (Capability CRD and CapabilitySet layering)
- ADR 0033 (initial implementation targets)

# ADR 0007: LibreChat locked down as frontend-only

## Status
Accepted

## Context
The platform needs an end-user chat UI that connects to Platform Agents through the standard gateway path, not a parallel agent runtime. The UI must integrate with Keycloak SSO, route every model and tool call through LiteLLM (so OPA, audit, virtual keys, and Langfuse all apply uniformly), and stay narrow enough that it does not become a second place where agents, tools, or RAG stores can be defined outside the CRD model.

LibreChat ships with a native agent feature, plugins, MCP support, and file upload. Each of these would, if left enabled, create an alternate path for users to define capabilities that bypass `Agent`, `Tool`, `MCPServer`, `CapabilitySet`, and `RAGStore` CRDs — undermining the governance model the rest of the architecture rests on. The lockdown surface in `librechat.yaml` is real and sufficient to disable these features declaratively.

Three options were evaluated: LibreChat (locked down), Open WebUI, and a custom UI.

## Decision
Adopt **LibreChat as an end-user chat UI, locked down to frontend-only** via `librechat.yaml`. The native agent feature, plugins, MCP support, and file upload are all disabled. Users see only the endpoint picker (which lists Platform Agents exposed as models via LiteLLM) and chat history. Authentication is Keycloak SSO; the LibreChat database lives in the platform's managed Postgres.

LibreChat reaches Platform Agents through LiteLLM's **OpenAI-compat ↔ A2A translation** so each agent appears as a model in the endpoint picker. The default endpoint is the **Interactive Access Agent** (architecture overview §6.4), the general-purpose Platform Agent that includes the Knowledge Base `RAGStore` in its CapabilitySet and gives developers conversational access to platform documentation and runbooks from day one.

If the OSS LiteLLM build does not ship the OpenAI ↔ A2A translation in the version the platform deploys, a small adapter ships alongside the LibreChat install (Workstream A8) rather than blocking on upstream. This is the only piece of custom code the LibreChat component owns.

## Consequences
Positive:
- Lowest-friction path to a working chat UI: install + configure, no UI to build or maintain.
- All model and tool traffic flows through LiteLLM by construction — OPA, audit, virtual keys, Langfuse, and budget enforcement apply uniformly to anything the UI can do.
- The Knowledge Base (ADR 0022) gets a usable interactive surface from day one through the Interactive Access Agent, with no UI-specific work.
- Keycloak SSO and managed Postgres reuse platform baselines.

Negative:
- Power users cannot define personal lightweight agents, plugins, or MCP servers in the UI. This is the **trigger to revisit (architecture backlog §3.12)**: when power users develop a real felt need for personal lightweight agents, the lockdown is reopened — likely by exposing a curated subset of LibreChat features behind RBAC/OPA, not by re-enabling the native surfaces wholesale.
- LibreChat's release cadence and config schema must be tracked; lockdown settings can drift across versions and need test coverage.
- If LiteLLM OSS lacks OpenAI ↔ A2A translation in a deployed version, the platform owns and maintains the small adapter shipped with A8.

## Alternatives considered
- **Open WebUI.** Rejected: comparable feature set, but no clear advantage over LibreChat in the locked-down configuration we want, and the lockdown surface is less well-trodden.
- **Custom chat UI.** Rejected: building and maintaining a chat UI is non-trivial sustained work for a surface where the locked-down LibreChat is already sufficient. Revisit only if lockdown becomes structurally inadequate.

## Related
- Architecture overview: §4 (high-level — LibreChat in Interfaces), §5 (LibreChat row), §6.1 (LiteLLM OpenAI-compat ↔ A2A translation), §6.4 (Knowledge Base via Interactive Access Agent in LibreChat), §14.1 Workstream A (components A8 LibreChat, A16 Interactive Access Agent).
- Architecture backlog: §1.16 (OpenAI ↔ A2A translation source), §2.10 (LibreChat vs Open WebUI vs custom), §3.12 (power-user personal agents — trigger to revisit), §7 ADR list.
- Other ADRs:
  - ADR 0006 — Custom Python kopf operator for LiteLLM, which reconciles the `A2APeer` and `VirtualKey` resources LibreChat depends on.
  - ADR 0018 — RBAC-as-floor / OPA-as-restrictor enforcement, applied to LibreChat traffic via LiteLLM callbacks.
  - ADR 0022 — Knowledge Base as a separate primitive, surfaced in LibreChat through the Interactive Access Agent.

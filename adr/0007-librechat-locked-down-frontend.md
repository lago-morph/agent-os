# ADR 0007: LibreChat (locked down) as the chat frontend

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

End users need a chat UI to talk to Platform Agents, with Platform Agents
surfaced as endpoints (architecture-overview.md §6.1, §7.1). The platform's
posture is that skills, tools, MCP, and RAG are configured server-side on the
agent and are not user-controllable from the UI; the chat frontend must
therefore expose only the endpoint picker and chat history. Three options were
evaluated for the chat UI (architecture-backlog.md § 2.10): LibreChat, Open
WebUI, and a custom UI. LibreChat is OSS, Keycloak-SSO friendly, and —
critically — supports a real lockdown profile via `librechat.yaml`, which is the
mechanism this decision relies on.

## Decision

The platform adopts **LibreChat (locked down)** as the chat frontend (component
**A8**, architecture-overview.md §5, §7.1, §14.1). Lockdown is configured via
`librechat.yaml`: the native agent feature, plugins, MCP, and file upload are
all disabled, leaving only the endpoint picker (Platform Agents via LiteLLM)
and chat history visible to users. LibreChat is installed unmodified via Helm
and SSO'd through Keycloak; visible endpoints are scoped from platform JWT
claims (architecture-overview.md §6.9).

## Alternatives considered

- **Open WebUI** — Rejected: weaker lockdown story for the "frontend-only, no
  per-conversation agent/tool/MCP exposure" posture; less clear path to disable
  the same feature surface that LibreChat exposes via `librechat.yaml`
  (architecture-backlog.md § 2.10).
- **Custom UI** — Rejected: highest build cost and ongoing maintenance for a
  frontend that is intentionally minimal; LibreChat already covers the required
  surface once locked down, so a custom build buys nothing the lockdown profile
  does not already deliver (architecture-backlog.md § 2.10).

## Consequences

- Component **A8** scope is the LibreChat Helm install plus the locked-down
  `librechat.yaml` configuration and Keycloak SSO wiring
  (architecture-overview.md §14.1, table row A8).
- A8 also owns delivery of the **OpenAI ↔ A2A adapter** if LiteLLM does not
  provide that translation in OSS by install time: a small Python adapter sits
  between LibreChat and LiteLLM so LibreChat itself stays unmodified
  (architecture-overview.md §7.1; architecture-backlog.md § 1.16). The decision
  on whether the adapter is needed is confirmed during implementation.
- Streaming is preserved end-to-end through LibreChat ↔ LiteLLM ↔ Platform
  Agent, with stream chunks preserved across the OpenAI-compat ↔ A2A boundary
  (architecture-overview.md §6.1, §7.1).
- Endpoint visibility in LibreChat is driven by platform JWT claims consumed
  from Keycloak; users only see the Platform Agents their `capability_set_refs`
  and roles permit (architecture-overview.md §6.9, §6.11).
- The **Interactive Access Agent** (A16) is the default endpoint LibreChat
  connects to from day one, so the Knowledge Base and conversational
  diagnostics are reachable through LibreChat without extra agents
  (architecture-overview.md §6.10, §14.1 table row A16).
- The lockdown removes user-side power-user affordances by design; this is
  revisited per **architecture-backlog.md § 3.12** when power users have a real
  felt need for personal lightweight agents.
- LibreChat-resident state (conversations, accounts) lives in the shared
  Postgres deployment alongside other platform state, and inherits its backup,
  PITR, and DR posture (architecture-overview.md §7.2).

## References

- architecture-overview.md §5, §6.1, §6.9, §6.10, §6.11, §7.1, §7.2, §14.1
- architecture-backlog.md § 2.10, 1.16, 3.12

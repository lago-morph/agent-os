# ADR 0007: LibreChat (configured) as the chat frontend

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
critically — supports a real configuration profile via `librechat.yaml`, which
is the mechanism this decision relies on.

The original framing of this ADR was "locked down": no plugins, no
user-controllable agents, no MCP. That posture still holds for those surfaces.
**File upload, however, is permitted** — it is genuinely useful (HolmesGPT in
particular benefits from log and screenshot uploads during diagnostics), and
abuse is contained by policy rather than by RBAC-gating the feature off.

## Decision

The platform adopts **LibreChat (configured)** as the chat frontend (component
**A8**, architecture-overview.md §5, §7.1, §14.1). The native agent feature,
plugins, and MCP are disabled in `librechat.yaml`, leaving the endpoint picker
(Platform Agents via LiteLLM), chat history, and **file upload** visible to
users. LibreChat is installed unmodified via Helm and SSO'd through Keycloak;
visible endpoints are scoped from platform JWT claims (architecture-overview.md
§6.9).

Uploads are routed to the **Memory CRD** per ADR 0025 access modes and exposed
to the agent the user is connected to. **OPA** gates upload behavior with
three outcomes per file type / MIME:

1. **allowed** — upload proceeds and is attached to the conversation's memory.
2. **forbidden** — upload is rejected with a policy reason.
3. **allowed-after-scanning** — upload is held, scanned (AV / DLP / size /
   content checks), and only then attached. This path may invoke an approval
   step (ADR 0017) when the policy demands a human-in-the-loop confirmation.

The default policy is **permissive for everyone**, with OPA enforcing abuse
limits (rate, size, MIME); upload is not RBAC-gated to specific users.

## Alternatives considered

- **Open WebUI** — Rejected: weaker configuration story for the
  "frontend-only, no per-conversation agent/tool/MCP exposure" posture; less
  clear path to disable the same feature surface that LibreChat exposes via
  `librechat.yaml` (architecture-backlog.md § 2.10).
- **Custom UI** — Rejected: highest build cost and ongoing maintenance for a
  frontend that is intentionally minimal; LibreChat already covers the
  required surface once configured, so a custom build buys nothing the
  configuration profile does not already deliver (architecture-backlog.md §
  2.10).
- **RBAC-gated upload** — Rejected: gating upload to a named user set creates
  ticket-driven friction for a feature whose abuse vectors (size, type,
  malware) are better addressed by policy and scanning than by allowlist
  membership.

## Consequences

- Component **A8** scope is the LibreChat Helm install plus the configured
  `librechat.yaml`, the upload → Memory CRD wiring, and Keycloak SSO
  (architecture-overview.md §14.1, table row A8).
- A8 also owns delivery of the **OpenAI ↔ A2A adapter** if LiteLLM does not
  provide that translation in OSS by install time: a small Python adapter sits
  between LibreChat and LiteLLM so LibreChat itself stays unmodified
  (architecture-overview.md §7.1; architecture-backlog.md § 1.16). The
  decision on whether the adapter is needed is confirmed during
  implementation.
- Streaming is preserved end-to-end through LibreChat ↔ LiteLLM ↔ Platform
  Agent, with stream chunks preserved across the OpenAI-compat ↔ A2A boundary
  (architecture-overview.md §6.1, §7.1).
- Endpoint visibility in LibreChat is driven by platform JWT claims consumed
  from Keycloak; users only see the Platform Agents their
  `capability_set_refs` and roles permit (architecture-overview.md §6.9,
  §6.11).
- The **Interactive Access Agent** (A16) is the default endpoint LibreChat
  connects to from day one, so the Knowledge Base and conversational
  diagnostics are reachable through LibreChat without extra agents
  (architecture-overview.md §6.10, §14.1 table row A16).
- Power-user affordances (personal agents, plugin marketplace, user-level
  MCP) remain off by design; this is revisited per **architecture-backlog.md
  § 3.12** when power users have a real felt need.
- The upload-policy OPA bundle is editable via the **Headlamp policy editor**
  (ADR 0039) and testable against canned upload payloads via the **policy
  simulator** (ADR 0038) before promotion.
- LibreChat-resident state (conversations, accounts) lives in the shared
  Postgres deployment alongside other platform state, and inherits its
  backup, PITR, and DR posture (architecture-overview.md §7.2).

## References

- [architecture-overview.md](../architecture-overview.md) [§5](../architecture-overview.md#5-software-added-to-baseline), [§6.1](../architecture-overview.md#61-gateway-architecture), [§6.9](../architecture-overview.md#69-multi-tenancy-and-namespacing), [§6.10](../architecture-overview.md#610-platform-self-management-with-holmesgpt), [§6.11](../architecture-overview.md#611-identity-federation), [§7.1](../architecture-overview.md#71-interactive-chat), [§7.2](../architecture-overview.md#72-triggered-and-long-running), [§14.1](../architecture-overview.md#141-workstream-a--platform-installation-and-operations)
- [architecture-backlog.md](../architecture-backlog.md) [§ 2.10](../architecture-backlog.md#210-chat-ui-librechat-vs-open-webui-vs-custom), [1.16](../architecture-backlog.md#116-openai--a2a-translation-source), [3.12](../architecture-backlog.md#312-power-user-personal-agents-in-librechat)
- [ADR 0017](./0017-generalized-approval-system.md) (approval flow — invoked by the scan-then-allow upload path)
- [ADR 0025](./0025-memory-access-modes-per-store.md) (Memory CRD access modes — destination for uploaded artifacts)
- [ADR 0038](./0038-policy-simulators.md) (policy simulator — testing the upload OPA policy)
- [ADR 0039](./0039-headlamp-graphical-editors-for-platform-crds.md) (Headlamp policy editor — surfaces the upload-policy OPA bundle)

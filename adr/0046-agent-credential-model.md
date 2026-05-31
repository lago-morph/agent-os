# ADR 0046: Agent credential model

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

The v1 design (ADR 0020, `_meta/interface-contract.md` §1.4) gave `MCPServer.authMode`
two first-class values, `system` and `user-cred`, with `system-mediated` described as a
third distinct credential shape. That enumeration assumed an interactive end-user behind
the agent — `user-cred` brokered "the user's own OAuth flow." It does not hold for
agent-os: agents are headless autonomous services. There are no end-users supplying
credentials; humans interact only through the approval workflow and operator RBAC/policy.
An authMode whose meaning is "the agent holds and manages the user's credential" therefore
names something that does not exist in this platform.

This ADR records the decision made during the review session (`_meta/reviews/DECISIONS-LOG.md`,
"Authentication & secrets", D-01) that fixes the credential model and corrects the authMode
enumeration accordingly.

## Decision

**Agents never hold or manage credentials. The LiteLLM gateway does.** The gateway is the
single broker that holds credentials and presents them to upstream services on the agent's
behalf; an agent only ever presents its own key to the gateway.

There are exactly **two credential modes**, and `user-cred` is **retired**:

- **`system`** — credentials needed by *system / platform processes* (for example,
  HolmesGPT reaching inference). These are **not tenant-specific**: one platform-managed
  identity, used by the platform's own processes.
- **`system-mediated`** — credentials the platform (LiteLLM) **holds on behalf of a
  specific tenant's agents**. LiteLLM enforces **per-tenant isolation**: a tenant's key
  unlocks only that tenant's credentials and never another tenant's. The agent still does
  not hold the credential — it presents its tenant key, and LiteLLM mediates.

A system agent that needs *direct* access to a service (for example, HolmesGPT) uses a
**platform-granted workload identity — a Kubernetes service account**. That workload
identity is **separate from `authMode`**: it is the agent's own platform identity, not a
brokered credential, and it is not one of the two credential modes above.

*How* LiteLLM enforces per-tenant isolation (the specific keying and access-control
mechanism) is **detailed design** and is not fixed here.

## Consequences

- The `MCPServer.authMode` enumeration changes from `{system, user-cred}` (plus
  `system-mediated` as a separate shape) to **`{system, system-mediated}`**. The
  source `_meta/interface-contract.md` §1.4 entry and the ADR 0020 discussion of
  credential modes are corrected by this ADR; `user-cred` is removed.
- Because no agent holds a credential, there is no per-agent OAuth-lifecycle machinery to
  build. OAuth credentials, where needed, are operator-entered secrets that LiteLLM holds —
  treated as just another secret, not an end-user flow.
- Per-tenant isolation becomes a property the gateway enforces rather than something each
  agent must implement, which keeps the isolation boundary in one auditable place.
- System agents needing direct access are modelled by workload identity (a service
  account), keeping that concern cleanly separate from the brokered-credential modes so the
  two are not conflated in specs or admission rules.

## References

- [ADR 0020](./0020-initial-mcp-services.md) (initial MCP services; defines `MCPServer.authMode`)
- [`_meta/interface-contract.md`](../_meta/interface-contract.md) §1.4 (the `authMode` enum this ADR corrects)
- [ADR 0047](./0047-secrets-via-external-secrets-operator.md) (where the credentials live — secrets via the External Secrets Operator)
- [`_meta/reviews/DECISIONS-LOG.md`](../_meta/reviews/DECISIONS-LOG.md) — "Authentication & secrets", D-01 (the authoritative decision recorded here)

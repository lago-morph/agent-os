# ADR 0047: Secrets via the External Secrets Operator

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

agent-os components need credentials and other secret material at runtime — model
provider keys, OAuth credentials, backend connection strings, and similar. Two
questions have to be answered: where the authoritative secret material lives, and how
a change to that material reaches the running pods that consume it.

agent-os does not want to own a secrets *store*. Encryption at rest, the choice of KMS,
and rotation cadence are properties of whatever external store the operator already
runs (a cloud secret manager, Vault, or equivalent). Reinventing any of that inside
agent-os would duplicate platform responsibility and couple the platform to one
provider. At the same time, components must be able to consume secrets through a single,
provider-neutral mechanism, and a secret change must actually take effect without manual
pod surgery.

A complication is that not every secret originates in the external store. Operators
enter some credentials — including OAuth credentials — through the LiteLLM GUI (see
ADR 0046, which establishes that LiteLLM, not the agents, holds credentials). Those
credentials originate inside the cluster and must flow *outward* to the external store,
the reverse of the normal pull direction.

## Decision

**Secrets are abstracted by the External Secrets Operator (ESO), delivered as part of
the baseline agent-os runs on.** Components consume secrets as ordinary Kubernetes
Secrets that ESO materializes from the external store; they do not talk to the external
store directly.

- **The external store is out of scope (baseline).** What ESO is configured to read
  from, how that store encrypts secrets at rest, which KMS backs it, and on what
  cadence material is rotated are all properties of the baseline / external store and
  are **not specified by agent-os**. agent-os depends on ESO being present and
  configured; it does not own the store behind it.

- **Propagation uses a reloader-style controller.** When ESO updates a Kubernetes
  Secret (or a ConfigMap changes), a reloader-style controller restarts the pods that
  consume it so the new value takes effect. This reloader is **expected to be present
  in the baseline**. If it is absent, agent-os startup **gates** on its presence —
  agent-os does not come up without a working propagation path. The full chain is:

  > external store → ESO updates the Kubernetes Secret → reloader restarts the consuming pods.

- **Push path for operator-entered credentials.** Operators may enter credentials
  through the **LiteLLM GUI**; OAuth credentials are treated as just another secret,
  with no separate OAuth-lifecycle machinery. LiteLLM writes what it is given into a
  Kubernetes Secret in its own namespace, and ESO **PushSecret** propagates that Secret
  *outward* to the external store. This is the reverse direction of the normal pull
  chain.

- **Single authority per secret.** For every secret, it must be declared whether the
  **external store is authoritative (pull)** or **LiteLLM/Kubernetes is authoritative
  (push)** — **never both on one secret.** Allowing two writers for the same secret
  would let the pull and push paths overwrite each other; the per-secret declaration
  removes that ambiguity by naming exactly one source of truth.

## Consequences

- agent-os carries no secrets-store implementation. The hard, provider-specific
  problems — encryption at rest, KMS selection, rotation cadence — stay with the
  baseline external store, where they belong, and the platform stays portable across
  stores that ESO can target.

- Every consumer reads secrets the same way: as a Kubernetes Secret. Components need no
  knowledge of the external store or of ESO beyond the Secret it produces.

- Secret changes propagate without manual intervention, but correctness depends on the
  reloader. Because agent-os **gates startup on the reloader's presence**, a baseline
  that ships without one fails fast and visibly rather than silently leaving stale
  credentials in running pods.

- The push path means a credential can legitimately originate inside the cluster
  (LiteLLM GUI) and be promoted to the external store via PushSecret. The
  one-authority-per-secret rule is what keeps this safe: pull-authoritative and
  push-authoritative secrets are disjoint sets, so the two flows never contend for the
  same key.

## References

- [ADR 0046](./0046-agent-credential-model.md) (agents never hold credentials; LiteLLM holds them and is the operator's entry point — the source of the push path described here)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline; secret-backed connections to durable stores are materialized through the mechanism described here)
- `_meta/reviews/DECISIONS-LOG.md` — "Authentication & secrets" (the authoritative record of this decision)

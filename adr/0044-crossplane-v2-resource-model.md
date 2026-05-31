# ADR 0044: Crossplane v2 resource model

- **Status**: Accepted
- **Date**: 2026-05-31
- **Supersedes**: [ADR 0041 — Substrate abstraction via Crossplane Compositions](0041-substrate-abstraction-via-crossplane-compositions.md)

## Context

ADR 0041 committed the platform to the Crossplane v1 two-layer model: a cluster-scoped composite
resource (XR, named with an `X` prefix) paired with a namespaced claim (XRC, dropping the `X`
prefix) — e.g. `XPostgres` (XR) ↔ `Postgres` (claim). Users created claims; Crossplane bound each
claim to a composite.

The platform has adopted **Crossplane v2**, which removes the claim (XRC) layer entirely.
Composite resources (XRs) are now **namespace-scoped directly**: users create the XR in their
namespace, not a claim. There is no second resource type, no binding step, and no `spec.claimNames`
in the XRD — only `spec.names`.

The `X`-prefix naming convention existed solely to distinguish the cluster-scoped XR from the
namespaced claim. With v2, no such distinction exists; the XR is the user-facing resource. The
`X`-prefix is therefore dropped: `XPostgres` becomes `Postgres`, `XAgentDatabase` becomes
`AgentDatabase`, and so on.

Everything else from ADR 0041 carries forward unchanged: one XRD + one Composition per supported
substrate (kind + AWS for v1.0); both Compositions write the same connection-secret shape; XR
status is substrate-agnostic; the OPA Gatekeeper admission guardrail rejects any XR with no
matching Composition for `platform.io/environment`; XRD versioning per ADR 0030 applies; the
`CapabilitySet` overlay model (ADR 0032) is unaffected.

## Decision

The platform adopts the **Crossplane v2 resource model**:

1. **No claims (XRCs).** Composite resource claims are removed. Users create XRs directly in
   their namespace. XRDs have only `spec.names`; `spec.claimNames` is absent.

2. **Namespace-scoped XRs.** All substrate XRDs produce namespace-scoped composite resources.
   Tenant isolation is enforced via the namespace boundary (consistent with the rest of the
   platform CRD model).

3. **Drop the `X`-prefix naming convention.** The v1.0 XR kinds are renamed:

   | v1 name (dropped) | v2 name (canonical) | Purpose |
   |---|---|---|
   | `XPostgres` | `Postgres` | Substrate-abstracted Postgres |
   | `XSearchIndex` | `SearchIndex` | Substrate-abstracted search/index |
   | `XObjectStore` | `ObjectStore` | Substrate-abstracted object storage |
   | `XMongoDocStore` | `MongoDocStore` | Substrate-abstracted Mongo-compatible store |
   | `XAgentDatabase` | `AgentDatabase` | Per-agent/tenant/user database (composes above) |
   | `XAuditLog` | `AuditLog` | Audit pipeline composite (composes Postgres + ObjectStore) |
   | `XGrafanaDashboard` | `GrafanaDashboard` | Grafana dashboard composite |

   Existing non-`X`-prefixed XRs (`MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer`,
   `TenantOnboarding`) were already v2-consistent and are unchanged.

4. **Admission guardrail updated.** The OPA Gatekeeper policy (A7/B16) that previously rejected
   claims with no matching Composition now rejects **XRs** with no matching Composition for the
   cluster's `platform.io/environment` label. The guardrail logic is identical; only the resource
   type changes.

5. **Terminology.** "Claim" in the Crossplane sense is retired from all platform documentation.
   Use "composite resource" or "XR" instead. "Claim shape" → "XR schema" or "XR spec".
   "Claim admission" → "XR admission" or "composite resource admission".

6. **JWT / OIDC "claims" are unaffected.** The word "claim" in the context of Keycloak JWTs,
   OIDC tokens, and the platform claim schema (`platform_tenants`, `platform_roles`, etc.) is a
   different concept and is NOT changed by this ADR.

All other decisions from ADR 0041 carry forward: uniform connection-secret shape, substrate-agnostic
status, environment-label admission guardrail, capability-parity caveat, versioning policy (ADR 0030),
dual-mode support (kind + AWS, ADR 0033), Kargo cross-substrate promotion (ADR 0040).

## Alternatives considered

- **Keep claims (stay on Crossplane v1):** Rejected. The platform already runs Crossplane v2; the
  two-layer claim model is not available. This is an adoption of reality, not a new design choice.

- **Keep `X`-prefix names in v2:** Possible (Crossplane v2 does not mandate naming conventions),
  but confusing — the `X` prefix has no semantic meaning without a claim to distinguish from.
  Dropping it simplifies the model and aligns with idiomatic v2 usage.

## Consequences

- All spec/plan documents are updated from v1 to v2 terminology (Crossplane v2 corpus sweep).
- ADR 0041 is superseded; its spec/plan moved to `specs/adr/superseded/` and `plans/adr/superseded/`.
- Canon Glossary and Interface Contract are updated with v2 XR names (no `X`-prefix substrate XRDs).
- Component B4 owns all Crossplane v2 Compositions under the new XR names.
- Consumers (A18, A21, A23, B11, B19) bind to the new XR kinds, not claim kinds.

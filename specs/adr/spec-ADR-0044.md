> kind: ADR · workstream: — · tier: T0
> upstream: [A7; A11] · downstream: [B4; A18; A21; A23; B19; B11] · adrs: [0044] · views: [6.3]
> canon-glossary: FROZEN · canon-interface: FROZEN

# ADR-0044 — Uniform Substrate XRD Contract (Crossplane v2)

## 1. Purpose

ADR 0044 supersedes ADR 0041. The platform runs on Crossplane v2, where composite resources
(XRs) are namespace-scoped directly: users create an XR in their own namespace and Crossplane
reconciles it through the selected Composition. There is no claim layer and no `X`-prefix
convention.

ADR 0044 fixes the contract every substrate abstraction must satisfy so that platform consumers
can provision infrastructure (Postgres, search, object storage, document stores, and
higher-level compositions) without knowing whether the backing implementation is an in-cluster
operator (kind) or a cloud service (AWS). The platform exposes each substrate as a single
Crossplane composite resource definition (XRD) with one Composition per backing implementation,
a uniform connection-secret shape, and a substrate-agnostic status surface.

Everything else from ADR 0041 carries forward: one XRD + one Composition per substrate (kind +
AWS), the uniform connection-secret shape, the substrate-agnostic status surface, the OPA
Gatekeeper admission guardrail (now rejecting XRs with no matching Composition), and XRD
versioning per ADR 0030. Only the v1 claim/`X`-prefix model is retired.

Every consumer codes against the XRD, never against the backing implementation. Swapping kind
for AWS (or vice versa) is a Composition selection concern, never a consumer-visible change.

## 2. Scope

**In scope**

- The v2 XR naming: `Postgres`, `SearchIndex`, `ObjectStore`, `MongoDocStore`, `AgentDatabase`,
  `AuditLog`, `GrafanaDashboard` (no `X` prefix; the `X` prefix is retired with claims).
- The namespace-scoped XR model: users create XRs directly in their namespace; no claim layer.
- One XRD per substrate; one Composition per backing implementation (kind + AWS) selected by a
  label selector.
- Uniform connection-secret shape written to the XR's namespace.
- Substrate-agnostic status conditions (`Ready`, `Synced`) surfaced on the XR.
- The OPA Gatekeeper admission guardrail rejecting any XR with no matching Composition.
- XRD versioning per ADR 0030 (conversion webhooks + deprecation windows).
- **D-07** — `SearchIndex`, `MongoDocStore`, and `ObjectStore` are added to the glossary as v2
  composite types (previously listed only as `X`-prefixed v1 XRDs).

**Out of scope**

- Knative-based substrates (see ADR 0038 exceptions).
- Bootstrap ordering of the Crossplane provider installation (see ADR 0012).
- Per-substrate field schemas (each substrate spec defines its own).

## 3. Context and Dependencies

The platform's substrate layer rests on Crossplane. ADR 0041 was written against Crossplane v1,
where each substrate was modeled as a cluster-scoped composite resource (XR) plus a namespaced
claim (XRC); users created claims and Crossplane bound each claim to an XR.

Crossplane v2 removes the claim layer entirely. XRs are namespace-scoped directly: a user creates
the XR (e.g. `Postgres`) in their namespace, and Crossplane reconciles it through the selected
Composition. Because there is no claim to distinguish the XR from, the `X`-prefix convention has
no purpose and is dropped. ADR 0044 restates the ADR 0041 contract in the v2 model.

**Upstream**

- **A7** — Crossplane provider install + family configuration.
- **A11** — Substrate catalog (the list of substrates the platform offers).

**Downstream**

- **B4** — OPA Gatekeeper admission policies.
- **A18** — Postgres substrate.
- **A21** — Search substrate.
- **A23** — Object store substrate.
- **B19** — Higher-level agent-database composition.
- **B11** — Audit-log composition.

## 4. Interfaces and Contracts

Each substrate XRD MUST expose:

- **`spec.names`** — the namespace-scoped XR names (no `X` prefix, e.g. `Postgres`). In v2 the XR
  is itself namespaced; there is no separate claim type.
- **`spec.versions[]`** — at least one served, referenceable version per ADR 0030.
- **`spec.connectionSecretKeys[]`** — the uniform connection-secret key set.

`spec.claimNames` does not exist in Crossplane v2 and MUST NOT appear on any substrate XRD.

The connection secret MUST be written to the XR's namespace and MUST carry the same key set
across every Composition for that substrate (capability parity).

The status surface MUST expose `Ready` and `Synced` conditions on the XR, independent of the
backing implementation.

## 5. OSS vs Custom

- **OSS:** Crossplane v2 (XRD/Composition engine), OPA Gatekeeper (admission).
- **Custom:** the per-substrate XRDs and Compositions, the admission constraint template.

## 6. Functional Requirements

- **REQ-01** — Each substrate MUST be exposed as exactly one XRD.
- **REQ-02** — Each XRD MUST define `spec.names` for a namespace-scoped XR and MUST NOT define
  `spec.claimNames` (which does not exist in Crossplane v2).
- **REQ-03** — Each substrate MUST ship one Composition per backing implementation (kind + AWS).
- **REQ-04** — Composition selection MUST be driven by a label selector on the XR.
- **REQ-05** — The connection secret MUST carry an identical key set across all Compositions.
- **REQ-06** — The XR MUST surface `Ready` and `Synced` status conditions.
- **REQ-07** — An XR with no matching Composition MUST be rejected at admission.
- **REQ-08** — XRDs MUST be versioned per ADR 0030 (conversion webhooks + deprecation windows).
- **REQ-09** — Connection-secret keys MUST be documented in the substrate's spec.

## 7. Non-Functional Requirements

- **NFR-01 (Portability)** — No consumer manifest may reference a backing implementation directly.
- **NFR-02 (Capability parity)** — Every Composition for a substrate MUST satisfy the same XR schema.
- **NFR-03 (Auditability)** — Composition selection MUST be observable on the XR status.

## 8. Deliverable Checklist

- [ ] One XRD per substrate with `spec.names` (namespace-scoped XR) and no `spec.claimNames`.
- [ ] One Composition per backing implementation (kind + AWS).
- [ ] Uniform connection-secret shape documented and enforced.
- [ ] `Ready`/`Synced` status surfaced on every XR.
- [ ] OPA Gatekeeper constraint rejecting XRs with no matching Composition.
- [ ] XRD versioning wired per ADR 0030.

## 9. Acceptance Criteria

- **AC-01** — A consumer can provision every substrate without referencing kind or AWS.
- **AC-02** — Swapping a Composition does not change any consumer manifest.
- **AC-03** — The connection secret key set is identical across Compositions for a substrate.
- **AC-04** — `Ready`/`Synced` appear on every XR regardless of backing implementation.
- **AC-05** — An XR with no matching Composition is rejected at admission.
- **AC-06** — Honored when `Postgres` is a namespaced XR that users create directly in their namespace.
- **AC-07** — XRD version changes follow ADR 0030 conversion + deprecation rules.

## 10. Risks and Open Questions

- **Risk:** Composition drift between kind and AWS breaks capability parity.
  *Mitigation:* contract tests assert identical connection-secret keys + status shape.
- **Risk:** Residual v1 (`X`-prefix / claim) references leak into consumer manifests or specs.
  *Mitigation:* naming convention enforced by the XRD contract (namespace-scoped XR, no prefix);
  glossary and interface contract updated to v2.
- **Open question:** none outstanding.

## 11. References

- Supersedes ADR 0041 — Uniform Substrate XRD Contract (Crossplane v1).
- ADR 0030 — CRD/API versioning (applies to XRDs).
- ADR 0012 — Crossplane bootstrap ordering.
- ADR 0038 — Knative substrate exceptions.
- Enforcing components: B4, A7, A18, A21, A23, B11.

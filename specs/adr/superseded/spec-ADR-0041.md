# SPEC ADR-0041 — Substrate abstraction via Crossplane Compositions `[SUPERSEDED]`

> ⚠️ **STATUS: SUPERSEDED by ADR-0044.** This document describes the Crossplane v1 resource model
> (claims + cluster-scoped composites). The platform uses **Crossplane v2** (namespace-scoped
> composites, no claims). Do not use this document to implement anything. See `spec-ADR-0044.md`.

> kind: ADR · workstream: — · tier: T0
> upstream: [A7; A11] · downstream: [B4; A18; A21; A23; B19; B11] · adrs: [0041] · views: [6.3]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0041 is a settled decision: each substrate-asymmetric primitive is wrapped in **one XRD with one
Composition per supported substrate** (kind + AWS for v1.0), all writing the **same connection-secret
shape** and exposing the **same substrate-agnostic status fields**. An **OPA Gatekeeper admission
policy** rejects any claim with no matching Composition for the cluster's `platform.io/environment`
label. The connection-secret-shape and status-field discipline are **tested architectural invariants,
not conventions**. This SPEC states what honoring the decision requires of every substrate XRD and its
consumers — it does not re-argue the Crossplane-Composition pattern.

The problem the decision solves: without a uniform abstraction, every workload consuming a
substrate-asymmetric primitive would need to know its substrate, parallel manifest sets would
proliferate, and Kargo cross-substrate promotion (ADR 0040) would break — a manifest that works on kind
would silently fail on AWS. One XRD + per-substrate Compositions makes the *claim shape* uniform while
the *Composition* absorbs the asymmetry.

## 2. Scope

### 2.1 In scope
- The wrap rule: each substrate-asymmetric primitive = one XRD + one Composition per substrate (kind + AWS for v1.0).
- The v1.0 substrate XRDs (`XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`) and the higher-level XRDs that compose them (`XAuditLog`, `XAgentDatabase`, `XGrafanaDashboard`).
- The connection-secret contract (`host`, `port`, `user`, `password`, `dbname` or equivalent) as a tested invariant both Compositions write.
- Substrate-agnostic XR status (`ready`, `endpoint`, `version`); substrate-specific fields deliberately absent from user-visible status.
- The `platform.io/environment=kind|aws` label + the OPA Gatekeeper admission guardrail (reject claim with no matching Composition).
- XR↔claim naming (drop the `X` prefix for the namespaced claim); XRD versioning (ADR 0030) applies to both Compositions.
- Capability-parity caveat (claim shape consistent; runtime behaviour may differ, e.g. kind `XObjectStore` "no archive").

### 2.2 Out of scope (and where it lives instead)
- The intentionally-NOT-wrapped exceptions: Knative event sources (ADR 0023), cluster bootstrap, cloud-provider stack selection, resource sizing/replicas/region (Kustomize).
- Audit retention/redaction lifecycle on `XObjectStore` — deferred to Workstream F (F1).
- The Composition builds themselves (kind + AWS per XRD) — owned by component B4.
- The OPA admission policy content — owned by B16 over B3 / Gatekeeper (A7).
- The Gatekeeper engine — owned by ADR 0002 / component A7.

## 3. Context & Dependencies

Upstream consumed: A7 (OPA/Gatekeeper — the admission policy rejecting claims with no matching
Composition); A11 (OpenSearch — the `XSearchIndex` kind-side backend). Downstream consumers: B4 (owns
all Compositions), A18 (`XAuditLog` composes `XPostgres`+`XObjectStore`), A21 (`TenantOnboarding`
provisioning), A23 (Kargo promotes claim shapes), B19 (approval-related provisioning), B11 (memory
backend adapter over `MemoryStore`).

ADR decisions honored:
- **ADR 0041** (this): one XRD + one Composition per substrate; uniform connection-secret + status; environment-label admission guardrail; tested invariants.
- **ADR 0002**: an OPA Gatekeeper admission policy rejects any claim with no matching Composition for the cluster's label.
- **ADR 0030**: XRD claim-shape changes are versioning events affecting both Compositions; conversion webhooks + deprecation windows apply identically to XRDs.
- **ADR 0033**: dual-mode hosting (kind + AWS) is realized via this pattern, not parallel manifest sets.
- **ADR 0040**: Kargo promotes uniform claim shapes; cross-substrate promotion is invisible to Kargo because Compositions absorb asymmetry.
- **ADR 0009 / ADR 0014 / ADR 0020 / ADR 0021 / ADR 0034**: OpenSearch, Postgres, agent databases, GrafanaDashboard, and the audit pipeline are natural instances of this pattern, not separate designs.
- **ADR 0023**: Knative event sources are a documented exception — intentionally NOT wrapped.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
All namespaced (claim form drops the `X` prefix; both names refer to the same XRD). Source-stated fields:
- **`XPostgres`** — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: CloudNativePG `Cluster`; AWS: Crossplane `RDSInstance`. (ADR 0014)
- **`XSearchIndex`** — `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`. kind: in-cluster OpenSearch operator; AWS: AWS-managed OpenSearch via Crossplane MR. (ADR 0009)
- **`XObjectStore`** — `bucketName`, `lifecycle`, `connectionSecretRef`, `substrateClass`. kind: MinIO or no-op; AWS: Crossplane S3 bucket. Capability-parity caveat: kind may have no archive lifecycle. (ADR 0034)
- **`XMongoDocStore`** — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: Bitnami MongoDB; AWS: DocumentDB or self-managed. (ADR 0020)
- **`XAuditLog`** — composes `XPostgres` + `XObjectStore` + an indexer (ADR 0034). Claim form `AuditLog`.
- **`XAgentDatabase`** — `engine` (postgres/mongodb), `scope` (agent/tenant/user), `ownerRef`, `credentialsSecretRef`. Composes `XPostgres` or `XMongoDocStore` by declared engine (ADR 0020). Claim form `AgentDatabase`.
- **`XGrafanaDashboard`** — `dashboardJson`, `folder`, `visibility`. kind: plain Grafana folders; AWS: same Grafana via Compositions (ADR 0021). Claim form `GrafanaDashboard`.

XRD versioning follows ADR 0030: changing an XRD's claim shape is a versioning event affecting both
Compositions; conversion webhooks + deprecation windows apply identically.

### 4.2 APIs / SDK surfaces
N/A — no SDK surface of its own. Consumers contract against the claim (connection-secret + status), not
the Composition internals. Crossplane v2 Compositions are the realization mechanism (component B4).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — the substrate abstraction emits no CloudEvent of its own. Composed XRD lifecycle (e.g. MemoryStore
provisioning) falls under `platform.lifecycle.*` per the owning component; an admission rejection by the
environment-label guardrail surfaces as a `platform.policy.*` event per OPA/Gatekeeper.

### 4.4 Data schemas / connection-secret contracts
- **Uniform connection-secret contract (tested architectural invariant):** both Compositions of every
  substrate-asymmetric primitive write the SAME secret shape — `host`, `port`, `user`, `password`,
  `dbname` (or the equivalent fields per primitive; the five are the canonical shape). Secret-shape
  conformance is tested, not left to convention. The per-primitive field list beyond the five is **not
  enumerated in source** `[PROPOSED — not in source]`.
- **Substrate-agnostic XR status:** `ready`, `endpoint`, `version`. Substrate-specific fields (RDS ARN,
  in-cluster Service paths) are deliberately absent from user-visible status; where absolutely needed
  they live in a substrate-specific subfield marked "implementation detail, do not depend on."
- **Environment label + admission guardrail:** every cluster carries `platform.io/environment=kind|aws`;
  an OPA Gatekeeper admission policy rejects any claim with no matching Composition for that label,
  preventing silent stuck reconciliation from typos/missing Compositions.
- **Day-2 drift** is standard Crossplane behaviour: substrate-side changes (e.g. someone resizing RDS in
  the console) reconcile back to declared state.

## 5. OSS-vs-Custom Decision
N/A — ADR (architectural pattern). The realizing component B4 builds Compositions over OSS Crossplane v2,
wrapping OSS backends per substrate (CloudNativePG / OpenSearch operator / MinIO / Bitnami MongoDB on
kind; RDS / managed OpenSearch / S3 / DocumentDB on AWS). Those OSS-vs-custom calls live in the B4 SPEC.

## 6. Functional Requirements

- REQ-ADR-0041-01: Each substrate-asymmetric primitive MUST be wrapped in exactly one XRD with one Composition per supported substrate (kind + AWS for v1.0); parallel manifest sets per substrate MUST NOT be used.
- REQ-ADR-0041-02: Both Compositions of every wrapped primitive MUST write the SAME connection-secret shape (`host`, `port`, `user`, `password`, `dbname` or the equivalent fields per primitive); secret-shape conformance MUST be tested as an architectural invariant, not left to convention.
- REQ-ADR-0041-03: XR status fields MUST be substrate-agnostic (`ready`, `endpoint`, `version`); substrate-specific fields (RDS ARN, in-cluster Service paths) MUST be absent from user-visible status (or confined to a clearly-marked "do not depend on" subfield).
- REQ-ADR-0041-04: Every cluster MUST carry the `platform.io/environment=kind|aws` label; an OPA Gatekeeper admission policy MUST reject any claim with no matching Composition for the cluster's label.
- REQ-ADR-0041-05: The v1.0 substrate XRDs MUST be `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`; `XAuditLog`, `XAgentDatabase`, and `XGrafanaDashboard` MUST be higher-level XRDs composing those primitives.
- REQ-ADR-0041-06: The user-facing namespaced claim MUST drop the `X` prefix (XR `XPostgres` ↔ claim `Postgres`); both names MUST refer to the same XRD.
- REQ-ADR-0041-07: XRD claim-shape changes MUST be versioning events (ADR 0030) affecting both Compositions, with conversion webhooks + deprecation windows applied identically.
- REQ-ADR-0041-08: Capability parity MUST NOT be promised across substrates: the claim *shape* MUST be consistent while runtime *behaviour* MAY differ (e.g. kind `XObjectStore` "no archive"); each XRD MUST document its substrate-specific behavioural differences.
- REQ-ADR-0041-09: The documented exceptions (Knative event sources, cluster bootstrap, cloud-provider stack selection, resource sizing/replicas/region) MUST remain intentionally NOT wrapped by this pattern.

## 7. Non-Functional Requirements
- Portability: cross-substrate promotion (ADR 0040) is mechanically uniform because the claim shape is identical across substrates; consumers never know their substrate.
- Security/admission: the environment-label guardrail (ADR 0002) prevents silent stuck reconciliation from missing/typo'd Compositions.
- Testability: connection-secret-shape and status-field discipline are tested invariants (a real test budget, not convention).
- Multi-tenancy (§6.9): all substrate XRDs are namespaced; there are no cluster-scoped platform CRDs in v1.0.
- Versioning (ADR 0030): both Compositions track the XRD's served version; conversion webhooks keep older stored versions readable.
- Day-2: standard Crossplane drift reconciliation back to declared state.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing component B4 carries the §14.1 deliverables (the kind + AWS Composition per
XRD, the connection-secret/status conformance tests, the Gatekeeper admission policy integration,
3-layer tests); conformance to this pattern is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0041-01: Honored when each wrapped primitive has exactly one XRD + one Composition per supported substrate, and no parallel per-substrate manifest set exists for it. (→ REQ-01)
- AC-ADR-0041-02: Honored when a conformance test asserts both Compositions of a primitive write the identical connection-secret shape on kind and on AWS. (→ REQ-02)
- AC-ADR-0041-03: Honored when the XR's user-visible status exposes only `ready`/`endpoint`/`version` and no substrate-specific field (RDS ARN, Service path) leaks into it. (→ REQ-03)
- AC-ADR-0041-04: Honored when a claim with no matching Composition for the cluster's `platform.io/environment` label is rejected at admission by the Gatekeeper policy. (→ REQ-04)
- AC-ADR-0041-05: Honored when `XPostgres`/`XSearchIndex`/`XObjectStore`/`XMongoDocStore` exist as substrate primitives and `XAuditLog`/`XAgentDatabase`/`XGrafanaDashboard` are shown to compose them. (→ REQ-05)
- AC-ADR-0041-06: Honored when the namespaced claim for `XPostgres` is `Postgres` and both resolve to the same XRD. (→ REQ-06)
- AC-ADR-0041-07: Honored when an XRD claim-shape change is shown to trigger conversion-webhook + deprecation-window handling on both Compositions. (→ REQ-07)
- AC-ADR-0041-08: Honored when kind `XObjectStore` produces "no archive" while the claim shape stays identical to the AWS one, and the difference is documented on the XRD. (→ REQ-08)
- AC-ADR-0041-09: Honored when a documented exception (e.g. a Knative SQS source) is shown NOT wrapped by an XRD/Composition. (→ REQ-09)

## 10. Risks & Open Questions
- (high) Capability parity is explicitly not promised — a consumer assuming AWS-only behaviour (e.g. object-store archive) silently degrades on kind; REQ-08 requires each XRD to document the difference, and dual-mode CI (ADR 0033) is the catch.
- (med) `[PROPOSED]` — the per-primitive connection-secret field list beyond the canonical five is not enumerated in source; treat the five as the canonical shape and flag per-primitive extensions for B4 design.
- (med) A missing Composition for a target's environment label would strand claims; the Gatekeeper admission guardrail (REQ-04) converts that from silent stuck reconciliation into a fast admission rejection.
- (low) Day-2 drift reconciliation may fight a deliberate out-of-band substrate change; standard Crossplane behaviour, documented as the day-2 model.

## 11. References
- ADR 0041 (this decision). Enforcing/realizing components: B4 (Crossplane Compositions, kind + AWS per XRD; conformance tests), A7 (Gatekeeper admission guardrail), A18 (`XAuditLog` consumer), A21 (`TenantOnboarding`), A23 (Kargo claim promotion), B11 (memory backend adapter), A11 (`XSearchIndex` kind backend).
- architecture-overview.md §6.3 (memory/data architecture), §6.12 (CRD inventory). architecture-backlog.md §1.13 (retention deferred). ADR 0002, 0009, 0014, 0020, 0021, 0023, 0026, 0030, 0033, 0034, 0040.

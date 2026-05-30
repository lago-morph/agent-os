# SPEC ADR-0030 — CRD and API versioning policy with per-component ownership [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [] · downstream: [B13, A5, A6, B4, B19, B6, B9, B12, B17, B18] · adrs: [0030] · views: [6.13]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0030 is a settled, T0 decision governing **every API surface the platform exposes**: CRDs/XRDs, CloudEvent schemas, the Platform SDK (Python + TypeScript), the `agent-platform` CLI, A2A/MCP interfaces declared by Platform Agents, and HTTP APIs of custom services. It fixes a distinct versioning model per surface, mandates that each surface is owned end-to-end by the component that exposes it, and forbids a centrally-coordinated lockstep platform version. This SPEC states what honoring that policy obliges of every owning component: which versioning scheme each surface uses, the conversion-webhook and deprecation-window obligations for CRD/XRD breaking changes, the never-break-subscribers rule for CloudEvents, the URL-path versioning + ≥1-release reachability for HTTP APIs, semantic versioning for SDK/CLI, and the per-component compatibility documentation duty. It does not re-argue the choice of schemes.

The architecture-level invariant (backlog §6) is that every API surface carries an explicit version, owned by the component that exposes it. Without this ADR pinning the model per surface, components would drift toward incompatible conventions and a synchronized "platform release" coordination would creep in by default — exactly what the per-component-ownership decision exists to prevent.

## 2. Scope

### 2.1 In scope
- The per-surface versioning obligations imposed on every owning component (CRDs/XRDs, CloudEvents, SDK, CLI, A2A/MCP, HTTP APIs).
- The CRD/XRD breaking-change obligation: new `vN` API group + conversion webhook + `vN-1` deprecated ≥1 minor platform release before removal, with webhook delivery, certificate rotation, and stored-version migration planned by the reconciler author.
- The CloudEvent obligation: `specversion` + per-event-type `schemaVersion`; backward-compatible additions bump minor; breaking changes mint a NEW event type under the same taxonomy (never break subscribers).
- The HTTP-API obligation: URL-path versioning (`/v1/...`) with deprecated versions reachable ≥1 platform release after replacement.
- The SDK/CLI obligation: semantic versioning; the SDK ships a version-pinned compatibility matrix against gateway/ARK/Letta.
- The ownership obligation: each surface owned end-to-end (current version, compatibility matrix, deprecation warnings, migration guidance) per §10.5; no central lockstep platform version.

### 2.2 Out of scope (and where it lives instead)
- The concrete schema contents of each surface — the owning component SPECs (B13, A5, A6, B4, B19, B6, B9, B12, B17, B18).
- The CloudEvent taxonomy itself — **ADR 0031** (this ADR governs how schemas *within* it version).
- The JWT claim schema contents — **ADR 0029** (this ADR governs how a change to it versions, dragging the cluster-OIDC mapper bundles).
- The calendar definition of "platform release" — design-time, deferred (backlog §1.18).
- Conversion-webhook patterns, exact deprecation calendars, and compatibility-matrix maintenance tooling — deferred to per-component design (backlog §1.18).

## 3. Context & Dependencies

Upstream consumed: none — this is the cross-cutting versioning policy every surface obeys.
Downstream consumers (the owning components, per the ADR's own enumeration): **B13** (capability/key/budget CRDs), **A5** (ARK CRDs), **A6** (sandbox CRDs), **B4** (Crossplane XRs/XRDs), **B19** (`Approval`), **B6** (SDK), **B9** (CLI), **B12** (CloudEvent registry), **B17/B18** (agent A2A/MCP interfaces). Also binds **ADR 0029**'s cluster-OIDC mapper bundles (versioned in lockstep with the claim schema) and **ADR 0041**'s XRD/Composition changes (conversion webhooks + deprecation windows on both Compositions when a claim shape changes).

ADR decisions honored:
- **ADR 0030** (this) — per-surface versioning model; per-component ownership; no central lockstep.
- **ADR 0031** — CloudEvent breaking changes ship as new event types under the same top-level taxonomy; old types flow until subscribers migrate.
- **ADR 0029** — a JWT claim-schema change is breaking and versions through this discipline; the cluster-OIDC mapper bundles version in lockstep, pinned to the schema version.
- **ADR 0041** — XRDs/Compositions version identically to CRDs: conversion webhooks + deprecation windows on both substrate Compositions when a claim shape changes.
- **ADR 0013** — the `CapabilitySet`/capability CRD shapes (owned by B13) version under this policy.
- **ADR 0019** — `Agent.sdk` accepted values evolve under ARK's (A5) CRD versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (versioning obligations per surface)
All v1.0 platform CRDs/XRDs are namespaced (glossary invariant). Each uses Kubernetes API versioning (`v1alpha1`, `v1beta1`, `v1`). **Owners (interface-contract §1):**
- ARK-reconciled (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`) — owner **A5**.
- agent-sandbox-reconciled (`Sandbox`, `SandboxTemplate`) — owner **A6**.
- kopf-operator-reconciled (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) — owner **B13**.
- `Approval` — owner **B19** (Argo Workflow); `LogLevel` — per-component.
- Crossplane XRs/XRDs (`MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer`, `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `XAgentDatabase`, `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`) — owner **B4**.

Breaking-change obligation per CRD/XRD: introduce a new `vN` API group; provide a conversion webhook; deprecate `vN-1` for ≥1 minor platform release before removal; the reconciler author plans webhook delivery, certificate rotation, and stored-version migration as part of the change. XRDs follow this identically (ADR 0041), including both substrate Compositions when a claim shape changes.

### 4.2 APIs / SDK surfaces
- **Platform SDK** (owner B6, Python + TypeScript) — semantic versioning; ships a version-pinned compatibility matrix against gateway/ARK/Letta versions. Named surfaces: `memory.*`, `rag.*`, OTel emission, A2A registration helpers (interface-contract §3.1).
- **`agent-platform` CLI** (owner B9) — semantic versioning; the subcommand surface is the versioned API.
- **A2A/MCP interfaces** declared by Platform Agents (owners B17/B18) — each agent declares a version (e.g. `myAgent.v1`); peers pin a major.
- **HTTP APIs of custom services** (kopf admin, Knative adapters, audit pipeline) — URL-path versioning (`/v1/...`); deprecated versions reachable ≥1 platform release after replacement.

### 4.3 CloudEvents emitted / consumed
- Every CloudEvent carries CloudEvents-native `specversion` plus a per-event-type `schemaVersion` (interface-contract §2). Backward-compatible additions bump minor; breaking changes **mint a new event type** under the same closed top-level taxonomy (ADR 0031) rather than break subscribers; old types continue to flow until subscribers migrate. The B12 registry owns per-event-type schemas.

### 4.4 Data schemas / connection-secret contracts
- The uniform connection-secret shape (`host`, `port`, `user`, `password`, `dbname`, ADR 0041) is a contract surface that versions with the owning XRDs (B4); a breaking change to a connection-secret shape follows the conversion-webhook + deprecation-window discipline on both substrate Compositions.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: the policy rides native versioning mechanisms — Kubernetes API-group versioning + conversion webhooks, CloudEvents `specversion`, semantic versioning, URL-path versioning. No new tooling is mandated; compatibility-matrix maintenance tooling is deferred per backlog §1.18.)

## 6. Functional Requirements
- REQ-ADR-0030-01: Every platform API surface MUST carry an explicit version owned end-to-end by the component that exposes it; there MUST be no centrally-coordinated lockstep platform version that bumps every surface together.
- REQ-ADR-0030-02: CRDs/XRDs MUST use Kubernetes API versioning (`v1alpha1`/`v1beta1`/`v1`); a breaking change MUST introduce a new `vN` API group WITH a conversion webhook.
- REQ-ADR-0030-03: For any CRD/XRD breaking change, `vN-1` MUST remain deprecated for at least one minor platform release before removal.
- REQ-ADR-0030-04: Reconciler authors MUST plan conversion-webhook delivery, certificate rotation, and stored-version migration as part of any CRD/XRD breaking change.
- REQ-ADR-0030-05: XRDs MUST version identically to CRDs (ADR 0041) — conversion webhooks and deprecation windows on BOTH substrate Compositions when a claim shape changes.
- REQ-ADR-0030-06: Every CloudEvent MUST carry `specversion` + a per-event-type `schemaVersion`; backward-compatible additions MUST bump minor; breaking changes MUST mint a NEW event type under the same taxonomy and MUST NOT break existing subscribers (old types keep flowing until subscribers migrate).
- REQ-ADR-0030-07: The Platform SDK and `agent-platform` CLI MUST use semantic versioning; the SDK MUST ship a version-pinned compatibility matrix against gateway/ARK/Letta versions.
- REQ-ADR-0030-08: HTTP APIs of custom services MUST use URL-path versioning (`/v1/...`); a deprecated version MUST remain reachable for at least one platform release after its replacement ships.
- REQ-ADR-0030-09: A2A/MCP interfaces declared by Platform Agents MUST each declare a version (e.g. `myAgent.v1`); peers MUST pin a major; the shipping component (B17/B18) MUST own that version.
- REQ-ADR-0030-10: Each surface owner MUST declare the current version, document the per-product compatibility matrix (§10.5), emit deprecation warnings, and provide migration guidance.
- REQ-ADR-0030-11: A JWT claim-schema change (ADR 0029) MUST go through this discipline, and the platform-shipped cluster-OIDC mapper bundles MUST be versioned in lockstep, pinned to the schema version they target.

## 7. Non-Functional Requirements
- Independence: components MUST evolve independently within the compatibility matrix; CI MUST NOT enforce a synchronized platform-wide version bump.
- Discoverability: every surface's current version and deprecation status MUST be discoverable in per-product documentation (§10.5).
- Compatibility (SDK): the version-pinned matrix is the contract; matrix-maintenance and CI-verification mechanics are deferred (backlog §1.18) but the matrix itself is a shipped artifact.
- Security/blast-radius: the JWT-schema/mapper lockstep (REQ-11) bounds the blast radius of an identity-contract change to a coordinated, versioned rollout rather than an uncoordinated break.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set — per-product docs (10.5), 3-layer tests — applies to every owning component, with this ADR supplying the versioning obligations each is tested against.

## 9. Acceptance Criteria
- AC-ADR-0030-01: Honored when every shipped surface declares an explicit version and no CI/release artifact enforces a lockstep platform-wide version. (REQ-01)
- AC-ADR-0030-02: Honored when a CRD/XRD breaking change is realized as a new `vN` API group with a working conversion webhook converting `vN-1` resources. (REQ-02/04)
- AC-ADR-0030-03: Honored when a removed CRD/XRD version was deprecated for ≥1 minor platform release before removal (deprecation recorded, not abrupt). (REQ-03)
- AC-ADR-0030-04: Honored when an XRD claim-shape change ships conversion webhooks + deprecation windows on both the kind and AWS Compositions. (REQ-05)
- AC-ADR-0030-05: Honored when a breaking CloudEvent change appears as a new event type while the old type still flows to unmigrated subscribers, each carrying `specversion` + `schemaVersion`. (REQ-06)
- AC-ADR-0030-06: Honored when the SDK and CLI follow semver and the SDK ships a compatibility matrix pinning gateway/ARK/Letta versions. (REQ-07)
- AC-ADR-0030-07: Honored when a replaced HTTP API keeps `/vN-1/...` reachable for ≥1 platform release after `/vN/...` ships. (REQ-08)
- AC-ADR-0030-08: Honored when a Platform Agent's A2A/MCP interface declares a version and a peer pins a major against it. (REQ-09)
- AC-ADR-0030-09: Honored when each surface's docs show current version, deprecation warnings, and migration guidance. (REQ-10)
- AC-ADR-0030-10: Honored when a simulated JWT-schema change versions through this discipline AND the kind/EKS/AKS cluster-OIDC mapper bundles bump in lockstep, pinned to the new schema version. (REQ-11)

## 10. Risks & Open Questions
- OQ-1 (high): The calendar definition of "platform release" (the unit the ≥1-release deprecation window is measured in) is design-time and deferred (backlog §1.18); deprecation-window ACs are measured in an as-yet-undefined unit until it lands. `[PROPOSED]`
- OQ-2 (med): Conversion-webhook patterns and compatibility-matrix maintenance tooling are deferred to per-component design (backlog §1.18); per-component plans must fill these without re-litigating the policy. `[PROPOSED]`
- R-1 (med): Independent per-component evolution risks matrix incompatibility (e.g. SDK vs gateway skew); mitigated by the mandatory version-pinned compatibility matrix, though its CI enforcement is deferred.
- R-2 (med): The JWT-schema/mapper lockstep (REQ-11) couples an identity change across kind/EKS/AKS mapper bundles; a missed bundle bump silently malforms principals — mitigated by pinning bundle versions to the schema version and ADR 0029's platform-CI mapper tests.

## 11. References
- ADR 0030 (`adr/0030-crd-and-api-versioning-policy.md`) — the decision enforced here.
- architecture-overview.md §6.13 (versioning policy), §10.5 (per-product documentation).
- architecture-backlog.md §1.18 (versioning specifics per surface; platform-release calendar), §6 (invariant).
- interface-contract.md §1.1 (versioning policy applies to every CRD/XRD), §2 (event versioning), §3 (SDK/CLI/A2A/MCP/HTTP surfaces), §4 (connection-secret).
- Owning components: B13, A5, A6, B4, B19, B6, B9, B12, B17, B18.
- ADR 0031 (CloudEvent taxonomy), 0029 (JWT claim schema), 0041 (substrate XRDs), 0013 (CapabilitySet), 0019 (Agent.sdk values).

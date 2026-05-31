# SPEC V6-13 — Versioning policy [PROPOSED]

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0030, 0031, 0044] · views: [6.13]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's versioning policy: the architectural
commitment that **every distinct API surface is explicitly versioned with a per-surface model, owned
by the component that exposes it, with no centrally-coordinated lockstep platform version** (§6.13,
ADR 0030). Six surface classes are governed — CRDs/XRDs, CloudEvent schemas, the Platform SDK, the
`agent-platform` CLI, A2A/MCP interfaces exposed by Platform Agents, and HTTP APIs of custom
services — each with its own versioning approach and compatibility-documentation duty.

This SPEC builds nothing. As a T0 view it fixes the per-surface versioning rules, the
**per-component ownership** invariant, the compatibility-matrix and deprecation-warning duties, and
the "no synchronized platform release version" constraint that every CRD owner and SDK owner must
honor. The detailed bump/deprecation *timing* is per-component design; this view owns the *policy
each component binds to*.

## 2. Scope

### 2.1 In scope
- The six versioned-surface classes and the versioning approach committed for each (§4 / §6).
- The **per-component ownership** invariant: each component owns the version of every API it exposes
  and the duty to document compatibility, emit deprecation warnings, and provide migration guidance.
- The **no-lockstep** constraint: there is no synchronized platform release version; components
  evolve independently within a compatibility matrix.
- The cross-surface consistency rules: CRD Kubernetes-API versioning + conversion webhooks + ≥1
  minor-release deprecation window; CloudEvent `specversion` + per-type `schemaVersion`; SDK/CLI
  semver; agent-interface major-pinning; HTTP URL-path versioning.

### 2.2 Out of scope (and where it lives instead)
- The CRD **inventory** the policy applies to — **V6-12**.
- The CloudEvent **taxonomy** the schema-versioning applies to — **V6-07 / ADR 0031**; the schema
  registry — **component B12**.
- **Per-component bump/deprecation timing** and the concrete compatibility-matrix contents — each
  owning component's spec + per-product docs (§10.5).
- Substrate Composition versioning specifics — **ADR 0044 / V6-03** (this view asserts XRDs are
  governed identically to CRDs).
- The Platform SDK / agent SDK / CLI **method/subcommand surfaces** — components B6 / B7 / B9.

## 3. Context & Dependencies

Realizing components — **all CRD/XRD/SDK/CLI/interface/HTTP-API owners** bind to this policy:
- **A5** (ARK CRDs), **A6** (sandbox CRDs), **B13** (capability/key/budget CRDs), **B19**
  (`Approval`), **B4** (XRs/XRDs), per-component (`LogLevel`) — CRD versioning owners.
- **B12** — owns the CloudEvent schema registry; each component owns its event types.
- **B6** — Platform SDK semver + compatibility matrix.
- **B9** — `agent-platform` CLI semver.
- **B17 / B18** — A2A/MCP interfaces exposed by shipped Platform Agents.
- Owners of custom HTTP services (kopf admin, Knative adapters, audit pipeline) — URL-path versioning.

ADR decisions honored:
- **ADR 0030** — the CRD/API versioning policy with per-component ownership (this view's spine).
- **ADR 0031** — CloudEvent versioning (`specversion` + `schemaVersion`; breaking → mint new type).
- **ADR 0044** — Crossplane v2 XRDs are versioned identically to CRDs (conversion webhooks + deprecation windows on
  both Compositions when an XR schema changes).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
This view introduces no CRD. It governs the versioning of the entire V6-12 inventory:
- **CRDs/XRDs:** Kubernetes API versioning (`v1alpha1`, `v1beta1`, `v1`); breaking changes go through
  a **new `vN` API group with conversion webhooks**; `vN-1` is **deprecated for ≥1 minor platform
  release** before removal. Owner = the reconciler-owning component (V6-12 §3). ADR 0044 applies the
  same rule to Crossplane v2 XRDs (both Compositions get conversion webhooks + deprecation windows on an XR-schema
  change).

### 4.2 APIs / SDK surfaces
- **Platform SDK (B6):** semantic versioning — major = SDK API break, minor = additive, patch =
  bugfix; ships a **version-pinned compatibility matrix** against gateway / ARK / Letta versions.
- **`agent-platform` CLI (B9):** semantic versioning; the subcommand surface is the versioned API;
  flag-level changes follow deprecation-warning conventions.
- **A2A/MCP interfaces exposed by Platform Agents:** each agent declares a version on its exposed
  interface (e.g. `myAgent.v1`); peers pin a major; breaking changes ship as a new major and run
  side-by-side until peers migrate. Owner = the shipping component (B17 / B18).
- **HTTP APIs of custom services** (kopf admin, Knative adapters, audit pipeline): URL-path
  versioning (`/v1/...`); deprecated versions remain reachable ≥1 platform release after replacement.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **CloudEvent schemas:** every event carries CloudEvents-native `specversion` plus a per-event-type
  `schemaVersion`. Backward-compatible additions bump minor; **breaking changes mint a new event
  type** rather than break subscribers. Introducing a new top-level `platform.*` namespace is a
  breaking change requiring a new ADR. Owner = B12 registry; each component owns its event types.

### 4.4 Data schemas / connection-secret contracts
The connection-secret shape (ADR 0044) is itself a versioned interface; a breaking change to it is
governed as an XR-schema change (conversion + deprecation window). Detailed in V6-03.

## 5. OSS-vs-Custom Decision
N/A — VIEW. The policy is realized by component-owned versioning machinery (Kubernetes conversion
webhooks, semver tooling, B12 registry) — each owner's decision.

## 6. Functional Requirements

- **REQ-V6-13-01** (invariant): Every distinct platform API surface MUST be explicitly versioned
  using its committed model: CRDs/XRDs → Kubernetes API versioning; CloudEvents → `specversion` +
  `schemaVersion`; Platform SDK + CLI → semver; agent A2A/MCP interfaces → declared version + major
  pin; custom HTTP APIs → URL-path versioning.
- **REQ-V6-13-02** (invariant): Versioning MUST be **per-component, not centrally coordinated**; each
  component owns the version of every API it exposes. There MUST be **no synchronized platform
  release version** that bumps everything in lockstep.
- **REQ-V6-13-03** (constraint): A **breaking CRD/XRD change** MUST go through a new `vN` API group
  with a conversion webhook, and `vN-1` MUST remain deprecated for **at least one minor platform
  release** before removal.
- **REQ-V6-13-04** (constraint): A **breaking CloudEvent change** MUST mint a new event type rather
  than break subscribers; a backward-compatible addition bumps the per-type `schemaVersion` minor; a
  new top-level `platform.*` namespace requires a new ADR.
- **REQ-V6-13-05** (constraint): The Platform SDK MUST ship a **version-pinned compatibility matrix**
  against gateway / ARK / Letta versions; major bumps signal SDK API breaks.
- **REQ-V6-13-06** (constraint): A Platform Agent's exposed A2A/MCP interface MUST declare a version;
  peers MUST pin a major; a breaking change MUST ship as a new major running side-by-side until peers
  migrate.
- **REQ-V6-13-07** (constraint): A custom-service HTTP API MUST use URL-path versioning (`/v1/...`);
  a deprecated version MUST remain reachable for ≥1 platform release after replacement.
- **REQ-V6-13-08** (constraint): Each component MUST, for every API it exposes: declare the current
  version, document adjacent-version compatibility in its per-product docs (§10.5), **emit
  deprecation warnings** (logs, metrics, audit events) when callers use a deprecated version, and
  provide migration guidance.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** version skew MUST NOT cross tenant boundaries in a way that leaks one
  tenant's deprecated surface to another; deprecation warnings are per-caller.
- **Security:** deprecation-warning telemetry (audit events) is a security/operability signal;
  unversioned surfaces are forbidden (REQ-01) so no surface evolves opaquely.
- **Observability (§6.5):** deprecation usage MUST be observable via logs/metrics/audit so operators
  can drive migration before removal.
- **Versioning (ADR 0030):** this view *is* the versioning NFR for the whole platform.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. Conversion webhooks, semver release tooling, compatibility matrices, deprecation-warning
emission, and migration docs belong to the owning components, verified end-to-end in PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-13-01** (→REQ-01): Each of the six surface classes in a sampled component declares a version
  in its committed model; an unversioned exposed surface fails the check.
- **AC-V6-13-02** (→REQ-02): No artifact encodes a single global "platform version" that gates
  multiple components' API versions in lockstep; component versions advance independently.
- **AC-V6-13-03** (→REQ-03): A breaking CRD change is served via a new `vN` group with a working
  conversion webhook, and `vN-1` is still served and marked deprecated for ≥1 minor release.
- **AC-V6-13-04** (→REQ-04): A breaking event change appears as a new event type (old subscribers
  unaffected); an additive change appears as a `schemaVersion` minor bump; a new namespace is blocked
  without an ADR.
- **AC-V6-13-05** (→REQ-05): The Platform SDK release ships a compatibility matrix pinning
  gateway/ARK/Letta versions.
- **AC-V6-13-06** (→REQ-06): A Platform Agent exposes `name.vN`; a peer pinned to `vN` keeps working
  when `vN+1` ships side-by-side.
- **AC-V6-13-07** (→REQ-07): A custom HTTP service serves `/v1/...` and keeps a deprecated prior
  version reachable for ≥1 platform release.
- **AC-V6-13-08** (→REQ-08): Using a deprecated version of any surface emits a deprecation
  warning (log/metric/audit) and the per-product docs carry migration guidance.

## 10. Risks & Open Questions
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review; it coins no new names.
- **(med)** Self-red-team: "no lockstep platform version" + "deprecate ≥1 minor *platform* release"
  is a tension — the deprecation window is measured in **platform releases** yet there is no global
  platform version. Reconciliation: a "platform release" is the cadence marker (a coordinated cut of
  the platform's component set), **not** a single API version; components still version
  independently, but the deprecation *clock* is the platform-release cadence. Flagged for ADR 0030
  owners to confirm the wording; this view adopts the cadence-marker reading.
- **(med)** `LogLevel`'s distributed reconcile means no single owner for its CRD-versioning column
  (carried from V6-12 §10). Open question routed to ADR 0030 owners: nominate a lead owner or
  explicitly bless distributed ownership for the versioning duty.
- **(low)** Whether the agent SDK (B7) versions as a surface distinct from the Platform SDK (B6) —
  interface-contract §3.1/3.2 treat them as distinct; this view covers the Platform SDK explicitly
  and binds B7 under the same semver model. Confirm with B7 owner.
- **(low)** The compatibility-matrix *format* is per-component design; this view fixes only its
  existence and the gateway/ARK/Letta pinning requirement.

## 11. References
- architecture-overview.md §6.13 (versioning policy table + component-responsibility, ~L981–1001);
  interface-contract.md §1.1 (CRD versioning), §2 (event versioning), §3 (SDK surfaces).
- ADR 0030 (CRD + API versioning, per-component ownership), ADR 0031 (CloudEvent versioning),
  ADR 0044 (Crossplane v2 XRD versioning parity).
- Realizing components: all CRD/SDK owners — A5, A6, B13, B19, B4, B12, B6, B9, B17, B18 + custom-HTTP
  owners. Related views: V6-12 (CRD inventory), V6-07 (eventing taxonomy), V6-03 (substrate/secret).

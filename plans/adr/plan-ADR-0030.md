# PLAN ADR-0030 — CRD and API versioning policy with per-component ownership [PROPOSED]

> spec: SPEC-ADR-0030 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [] · downstream-pieces: [B13, A5, A6, B4, B19, B6, B9, B12, B17, B18]

## 1. Implementation Strategy
Enforcement/verification map for a T0 cross-cutting policy. ADR 0030 is enforced by **every surface-owning component** applying its mandated scheme: **A5** (ARK CRDs), **A6** (sandbox CRDs), **B13** (capability/key/budget CRDs), **B4** (XRDs + both substrate Compositions, per ADR 0044), **B19** (`Approval`), **B6** (SDK semver + compatibility matrix), **B9** (CLI semver), **B12** (CloudEvent `specversion`+`schemaVersion`, new-type-on-break), **B17/B18** (agent A2A/MCP versions). The load-bearing obligations to verify are the CRD/XRD breaking-change discipline (new `vN` group + conversion webhook + `vN-1` deprecated ≥1 minor release, with cert rotation + stored-version migration planned), the never-break-subscribers CloudEvent rule, URL-path versioning with ≥1-release reachability for HTTP APIs, and the no-central-lockstep ownership model. The JWT-schema/mapper lockstep (ADR 0029) is verified as a cross-cutting special case. Conformance is largely a per-surface audit plus targeted Chainsaw tests for conversion webhooks and CloudEvent dual-flow.

## 2. Ordered Task List
- TASK-01: Map each REQ to each owning surface/component (the §1 enumeration) — produces: per-surface enforcement matrix — depends-on: []
- TASK-02: Verify every shipped surface declares an explicit version and no artifact enforces a lockstep platform version — produces: ownership/no-lockstep audit — depends-on: [TASK-01]
- TASK-03: Define CRD/XRD breaking-change conformance — new `vN` group + working conversion webhook + `vN-1` deprecated ≥1 minor release; cert rotation + stored-version migration planned — produces: Chainsaw conversion-webhook set — depends-on: [TASK-01]
- TASK-04: Verify XRD schema change ships conversion webhooks + deprecation windows on BOTH kind and AWS Compositions (ADR 0044) — produces: B4 dual-Composition check — depends-on: [TASK-03]
- TASK-05: Define CloudEvent versioning conformance — `specversion`+`schemaVersion`; minor-on-additive; new-type-on-break with old type still flowing — produces: B12 dual-flow test — depends-on: [TASK-01]
- TASK-06: Verify SDK/CLI semver + SDK compatibility matrix (gateway/ARK/Letta); HTTP `/v1/...` with ≥1-release reachability; A2A/MCP per-agent version + peer major-pin — produces: surface-scheme audit — depends-on: [TASK-01]
- TASK-07: Verify a JWT-schema change (ADR 0029) routes through this discipline with kind/EKS/AKS mapper bundles bumped in lockstep, pinned to schema version — produces: identity-lockstep conformance — depends-on: [TASK-03]
- TASK-08: Verify per-surface docs show current version, deprecation warnings, migration guidance (§10.5) — produces: doc-lint — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (cross-cutting policy); it constrains every surface-owning component rather than consuming one.
### 3.2 Downstream pieces blocked on this
- B13, A5, A6, B4, B19, B6, B9, B12, B17, B18 — each owns a surface that must conform.
### 3.3 Continuous (non-blocking) inputs
- ADR 0031 (CloudEvent taxonomy), ADR 0029 (JWT schema lockstep), ADR 0044 (XRD dual-Composition); backlog §1.18 (platform-release calendar + webhook/matrix tooling — deferred); B22 threat model.

## 4. Parallelizable Subtasks
TASK-03→TASK-04 and TASK-03→TASK-07 are serial off the conversion-webhook work; TASK-02→TASK-08, TASK-05, and TASK-06 fan out independently once TASK-01 lands.

## 5. Test Strategy
- AC-01 → manifest/release-lint: explicit version per surface; no lockstep artifact.
- AC-02/AC-03 → Chainsaw: apply a new `vN` group + conversion webhook; convert `vN-1` resources; assert `vN-1` deprecated ≥1 minor release before removal.
- AC-04 → Chainsaw: XRD schema change with conversion webhooks on both kind + AWS Compositions.
- AC-05 → PyTest+B12 harness: breaking CloudEvent change appears as new type while old type still flows; both carry `specversion`+`schemaVersion`.
- AC-06 → PyTest: SDK/CLI semver + SDK compatibility matrix present.
- AC-07 → PyTest: replaced HTTP API keeps `/vN-1/...` reachable ≥1 release.
- AC-08 → PyTest: agent A2A/MCP declares version; peer pins major.
- AC-09 → doc-lint: per-surface version/deprecation/migration docs.
- AC-10 → versioning test: JWT-schema change + lockstep mapper-bundle bump (cross-link ADR-0029).
- Fixtures/fakes: conversion-webhook test harness with self-signed cert rotation; B12 schema-registry stub; SDK compatibility-matrix fixture for not-yet-landed B6/B12. Note: deprecation-window ACs use a placeholder "platform release" unit until backlog §1.18 defines the calendar.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0030-crd-api-versioning-policy` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; coordinate with ADR-0029 (JWT lockstep) and ADR-0031/0044 (event + XRD versioning); rolls up to main.

## 7. Effort Estimate
M overall (T0 breadth, but enforcement is an audit + targeted webhook/event tests, not a build). Per-task: TASK-01 M, 02 S, 03 M, 04 M, 05 M, 06 M, 07 M, 08 S. Critical path: TASK-01 → 03 → 04/07.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, every surface-owning component loses the versioning conformance contract — CRD breaking-change discipline, never-break-subscribers, URL-path reachability, semver, and the JWT-schema/mapper lockstep all become per-component conventions, inviting drift and lockstep creep. Highest blast radius of this batch (T0, every surface); reversal is strongly discouraged. No runtime artifact is deleted.

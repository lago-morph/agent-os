# SPEC OPS6 — Rego (OPA) Human-Factors & Lifecycle

> kind: NFR (cross-cutting operational layer) · workstream: — · tier: T0
> upstream: [A7, B3, B16, B2, B13, A6, A1, B19, A20, A22, B14, B9, A18, B12, B22] · downstream: [—] · adrs: [0002, 0018, 0017, 0030, 0031, 0034, 0038, 0027] · views: [6.6]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05
> source-of-scope: _meta/pending-operational-nfr-layer.md (OPS6)

## 1. Purpose & Problem Statement

The platform's entire governance model rests on OPA/Rego: every privileged decision — Kubernetes admission (Gatekeeper), LiteLLM runtime callbacks, Envoy egress, Headlamp action gating, approval-level elevation, virtual-key issuance, substrate-claim admission — is decided in one Rego corpus (ADR 0002, ADR 0018). Because so much enforcement collapses onto a single language and a single bundle, **the lifecycle of the policy itself is a non-functional requirement of the platform**. A defective, untested, or maliciously-altered Rego change has platform-wide blast radius: it can silently widen access (violating ADR 0018), break the admission webhook write-path (A7 R1), or — the cardinal failure — **lock the platform out of its own policy plane** by denying the very identities that author and roll back policy.

OPS6 is the cross-cutting architecture for **how Rego is authored, reviewed, tested, gated in CI, versioned, staged, rolled out, rolled back, observed, and authorized to change** — and how the platform stays operable even when policy is wrong. It owns no engine, no framework, and no rule content; it imposes lifecycle obligations on the pieces that do (A7 install, B3 framework, B16 content, B2/A6 enforcement, B13 data reconciliation, B19 human gate, A20 simulator, A22 editor, B14 CI, B9 CLI) and surfaces the genuinely-open change-control decisions as PROPOSED-ADRs rather than deciding them.

This is an operational/NFR layer per `_meta/pending-operational-nfr-layer.md`; it is realized by existing component pieces plus the gaps named here, and ships its own stacked PR after W4.

## 2. Scope

### 2.1 In scope

- **Authoring ergonomics & guardrails** — the author-facing workflow and the structural guardrails (B3's RBAC-floor predicate, the how-to-add-a-policy convention, the simulator preview-before-commit loop) that make a policy change reviewable and hard to get wrong.
- **Test/CI gating of Rego** — the required-check obligations: Rego unit tests, SHA-pin verification, RBAC-floor (restrictor-only) conformance lint, decision-contract conformance, coverage-map completeness — every one a merge-blocking gate (ADR 0002, ADR 0018).
- **Versioning & bundle integrity** — bundle revision/SHA-pinning, GitOps (ArgoCD) reconciliation, and the bundle-signing obligation `[PROPOSED — not in source]` that closes the unsigned-bundle supply-chain gap left by ADR 0002.
- **Staged rollout & rollback** — the obligation that a policy-bundle change rolls out through observable stages (audit-mode / dry-run → enforce) and is revertible to the last-known-good SHA-pinned revision via Git, with a bounded rollback time.
- **Change authorization** — who may change which policy surface, expressed against Canon (Platform JWT `platform_roles`/`tenant_roles`, the B19 human gate, Headlamp action gating). The **global/cluster-wide policy bundle** ownership question is surfaced, not decided.
- **Anti-lockout safeguards** — the invariant that no policy change may render the policy plane (authoring, review, rollback, the admission webhook, the break-glass path) un-operable; and the break-glass recovery procedure when it nonetheless happens.
- **Observability of the lifecycle** — policy-change history, who-changed-what, decision/deny-rate shift after a rollout, all under `platform.policy.*` (ADR 0031) via the A18 audit adapter (ADR 0034).

### 2.2 Out of scope (and where it lives instead)

- **OPA + Gatekeeper install / admission runtime / fail-policy** → **A7**.
- **Rego bundle layout, shared library, decision contract, dry-run hook, the CI gate definitions** → **B3** (framework). OPS6 imposes that these gates be *required* and *complete*; B3 defines their mechanics.
- **The actual restriction-rule content** (admission, runtime, egress, elevation, the coverage map, the B22 extension point) → **B16**.
- **The policy simulator service** (Headlamp panel + HolmesGPT skill, the aggregator) → **A20**; **the graphical policy editors** → **A22 / ADR 0039**.
- **The human-approval mechanism** (`Approval` CRD, Argo suspend/resume, queue) → **B19**. OPS6 specifies *that* privileged policy changes route through it; B19 owns the gate.
- **CI/test-framework plumbing** (`agent-platform` test framework) → **B14**; **CLI promotion-gate surface** → **B9**.
- **`BudgetPolicy`/capability/virtual-key reconciliation into OPA `data`** → **B13**.
- **The adversarial threat model** that feeds new policy targets → **B22** (ADR 0027).
- **Image-signature verification for images/skills** → explicitly out of v1.0 (ADR 0002 accepted gap); OPS6's signing obligation concerns *policy bundles*, a distinct artifact.

## 3. Context & Dependencies

**Upstream pieces consumed (and exactly what):**
- **A7 (OPA/Gatekeeper)** — the engine, Gatekeeper audit-mode + dry-run admission (the staged-rollout substrate), SHA-pinned GitOps bundle loading, admission fail-policy.
- **B3 (framework)** — the bundle layout, `.manifest` revision, SHA-pin tooling, RBAC-floor predicate, decision contract, dry-run wrapper, how-to-add-a-policy convention, and the CI gate set (Rego unit tests + pin check) OPS6 requires be enforced.
- **B16 (content)** — the v1.0 rule corpus and the §6.6 coverage map OPS6 requires stay complete; the Headlamp action-gating and `approval.elevation` policies that govern *who may change policy*.
- **B2 / A6 / A1** — the runtime enforcement points whose dry-run paths constitute the pre-enforce staging step.
- **B19-core** — the `Approval` human gate that a privileged policy-bundle change routes through (Argo suspend/resume, `platform.approval.*`, OPA elevation).
- **A20 (simulator)** — preview-before-commit; the dry-run aggregation that proves a change's effect before enforce.
- **A22 / ADR 0039** — the Headlamp editors authors use; the simulator panel integration.
- **B14 (test framework) / B9 (CLI)** — the CI harness running the Rego gates and the promotion-gate dry-run.
- **A18 (audit adapter) / B12 (event schemas)** — `platform.policy.*` emission of the change lifecycle.
- **B22** — the threat model that adds policy targets; OPS6's change-control must absorb additions without lockout.

**ADR decisions honored (constraint each imposes):**
- **ADR 0002** — one engine, one language; Rego-as-code: Git-versioned, ArgoCD-reconciled, SHA-pinned, CI unit-tested; **no image-signature verification in v1.0** (bundle-signing here is a PROPOSED extension, not that gap).
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor; the restrictor-only conformance gate is a hard CI obligation; a policy change can never grant beyond RBAC.
- **ADR 0017** — OPA may only *elevate* an `Approval` level; the policy-change gate inherits this (a change can raise but never lower its own required approval).
- **ADR 0030** — bundle/decision-contract versioning; a decision-contract change is a coordinated breaking bump (every decision point binds to it).
- **ADR 0031** — policy-change lifecycle events emit under `platform.policy.*`; bypass attempts under `platform.security.*`.
- **ADR 0034** — all change-lifecycle audit goes through the A18 adapter; no direct store writes.
- **ADR 0038** — staged rollout uses each layer's structured dry-run (`simulated: true`, no enforcement side effect); A20 composes it for preview.
- **ADR 0027** — B22 threat-model output lands as new policy targets; the lifecycle must accommodate continuous additions.

## 4. Interfaces & Contracts

Canon names only; invented detail tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

OPS6 introduces **no new CRD**. It governs the lifecycle of artifacts and reuses existing Canon kinds:
- **`Approval`** (B19; fields `actionType`, `actionAttributes`, `defaultLevel`, `decision`, `decidedBy`, `decidedAt`) — a privileged **policy-bundle change** is modeled as an `Approval` with `actionType` = a policy-change type. `[PROPOSED — not in source]` the concrete `actionType` value (e.g. `"policy.bundle.promote"`) — reconcile with B19/B16.
- **`CapabilitySet.opaPolicyRefs[]`** — per-CapabilitySet policy snippets are change-controlled the same way bundles are; OPS6 requires the same gates apply to `opaPolicyRefs[]` content.
- Gatekeeper `ConstraintTemplate`/Constraint (upstream kinds, owned by A7+B16) and the B3 `.manifest` `revision` are the versioned units; OPS6 adds no field beyond requiring SHA-pin + (PROPOSED) signature.

### 4.2 APIs / SDK surfaces

- **No new SDK surface.** OPS6 binds to existing surfaces: the **A20 aggregator API** (preview), the **`agent-platform` CLI** promotion-gate + dry-run (B9, URL-path/`/v1` per interface-contract §3.3), the **B3 decision entrypoint** convention, and the **Headlamp** policy editor/simulator panel (A22/A20).
- **Required CI checks (gate set)** are an interface OPS6 names but B3/B14 own: `rego-unit`, `sha-pin-verify`, `rbac-floor-conformance`, `decision-contract-conformance`, `coverage-map-complete`. `[PROPOSED — not in source]` the exact check names — reconcile with B3/B14.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **`platform.policy.*`** — OPS6 requires the policy-change lifecycle (proposed / simulated / promoted-to-audit-mode / promoted-to-enforce / rolled-back) be observable here, alongside the existing OPA-decision and simulator-run events. Per-event-type names → **B12**; `[PROPOSED — not in source]` the concrete lifecycle event names.
- **`platform.approval.*`** — emitted by B19 when a policy-change `Approval` is requested/elevated/decided.
- **`platform.security.*`** — a policy-bypass attempt or an unauthorized policy-change attempt (e.g. a push that skips the gate) surfaces here (ADR 0018/0031).
- Consumed: none for enforcement; the lifecycle is Git/PR + reconcile-driven, not event-driven.

### 4.4 Data schemas / connection-secret contracts

- **N/A — no datastore, no connection secret.** The system of record for policy is **Git** (SHA-pinned, ArgoCD-reconciled per ADR 0002); the audit trail of changes is the A18 system of record (ADR 0034). The bundle-signing material `[PROPOSED — not in source]` would be a signing key managed per OPS3 (key/secret lifecycle) — reconcile with OPS3, not specified here.

## 5. OSS-vs-Custom Decision

N/A — cross-cutting NFR layer; introduces no deployable of its own. (Realization note: enforced over upstream **OPA/Gatekeeper** + **ArgoCD** + **Argo Workflows** (A7/A3) and the custom **B3/B16** Rego corpus, the **B14** CI harness, **B9** CLI, and **B19** gate — all config/wrap + existing custom code, no fork. The one genuinely-new build is the **bundle-signing/verification step** `[PROPOSED — not in source]`, which is a PROPOSED-ADR below, not auto-decided here.)

## 6. Functional Requirements

- **REQ-OPS6-01:** Every Rego policy change MUST flow through the Git-versioned, ArgoCD-reconciled, SHA-pinned delivery path (ADR 0002); no out-of-band edit to a running OPA bundle is permitted, and a tampered/unpinned bundle MUST fail to deliver.
- **REQ-OPS6-02:** Rego **unit tests** MUST be a required, merge-blocking CI check for every policy change (ADR 0002); a failing test MUST block merge.
- **REQ-OPS6-03:** A **restrictor-only (RBAC-floor) conformance gate** MUST run in CI and MUST fail any change whose allow-rule grants beyond the B3 floor predicate (ADR 0018) — i.e. a planted granting rule MUST be caught before merge.
- **REQ-OPS6-04:** A **decision-contract conformance gate** MUST fail any change whose entrypoint does not return B3's canonical decision document (incl. the `simulated` flag) (ADR 0038/0030), so the simulator and every enforcement point stay composable.
- **REQ-OPS6-05:** A **coverage-map gate** MUST fail any change that leaves a §6.6 OPA decision point without a serving policy package, and MUST keep B16's B22 extension point intact (ADR 0027).
- **REQ-OPS6-06:** A policy change MUST be **previewable before enforcement**: the author MUST be able to run the change through the A20 simulator / structured dry-run (`simulated: true`, no enforcement side effect) and see the per-layer decision delta before it is promoted (ADR 0038).
- **REQ-OPS6-07:** A policy-bundle change MUST roll out through **observable stages** — at minimum a non-enforcing stage (Gatekeeper audit-mode / dry-run) that emits the prospective decisions under `platform.policy.*` — before it enforces; promotion between stages MUST be an explicit, audited action.
- **REQ-OPS6-08:** Any enforced policy change MUST be **rollback-able to the last-known-good SHA-pinned revision via Git revert + ArgoCD re-sync**, with a bounded, documented rollback time `[PROPOSED — not in source]` target (recommend ≤ one ArgoCD sync interval); the rollback MUST itself be auditable.
- **REQ-OPS6-09:** **Authorization to change a policy surface** MUST be enforced via OPA/Headlamp action-gating reading Platform JWT `platform_roles`/`tenant_roles` (B16, §6.6); a tenant-scoped author MUST NOT be able to change platform-global or another tenant's policy.
- **REQ-OPS6-10:** A change to a **privileged or platform-global policy surface** MUST route through the B19 human gate (an `Approval`), and OPA MAY only **elevate** that change's required approval level, never lower it (ADR 0017/0018).
- **REQ-OPS6-11 (anti-lockout):** No policy change may be promoted to enforce if it would deny the **policy-plane operability set** — the identities/paths that author, review, simulate, promote, and **roll back** policy, plus the admission webhook's own liveness and the break-glass path. A pre-promotion check MUST simulate the change against this set and block on a self-lockout.
- **REQ-OPS6-12 (break-glass):** A documented **break-glass recovery procedure** MUST exist to restore policy-plane operability when enforcement is nonetheless broken (e.g. revert-by-SHA via GitOps, or a bounded admission fail-policy / Gatekeeper exemption for the recovery namespace); every break-glass use MUST emit `platform.security.*` and be audited (ADR 0034).
- **REQ-OPS6-13:** The **policy-change lifecycle** (proposed → simulated → audit-mode → enforce → rolled-back) MUST be observable under `platform.policy.*` via the A18 adapter, including who authored and who approved each change (ADR 0031/0034).
- **REQ-OPS6-14:** Per-CapabilitySet policy snippets (`CapabilitySet.opaPolicyRefs[]`) MUST be subject to the **same** authoring, test, gate, authorization, and rollback obligations as platform bundles.
- **REQ-OPS6-15 `[PROPOSED — not in source]`:** Policy bundles SHOULD be **cryptographically signed at build and signature-verified at load**, closing the unsigned-bundle supply-chain gap; this is a PROPOSED-ADR (§10) — OPS6 states the obligation and defers the decision, it does not mandate a mechanism.

## 7. Non-Functional Requirements

- **Security / supply-chain:** the policy corpus is a top-tier supply-chain asset; SHA-pinning (ADR 0002) is the v1.0 floor, signing is the PROPOSED hardening (REQ-15). The RBAC-floor gate (REQ-03) is the structural guarantee that no change widens access (ADR 0018). Unauthorized change attempts surface under `platform.security.*`.
- **Multi-tenancy (§6.9):** policy authorship is tenant-scoped from Platform JWT claims; tenant authors change only their tenant's policy surface; the platform-global bundle is a separate, more-tightly-gated surface (its owner is a PROPOSED-ADR). No change may bridge RBAC-separated tenants (ADR 0016/0018).
- **Observability (§6.5):** the change lifecycle, post-rollout deny-rate shift, and bundle revision in force are all observable; a Grafana dashboard (`GrafanaDashboard` XR) for policy-change volume and post-promotion deny-rate delta is an OPS6 obligation on the owning component (A7/B16 already carry policy-decision dashboards).
- **Availability / blast radius:** because the admission webhook is in every CR write-path (A7 R1), the staged-rollout (REQ-07) and anti-lockout (REQ-11) requirements are availability controls, not just security controls — a bad enforce-promotion must be caught in audit-mode, not in production admission.
- **Versioning (ADR 0030):** bundle `.manifest` revision is the change unit; a decision-contract change is a coordinated breaking bump across all binding decision points.

## 8. Cross-Cutting Deliverable Checklist

N/A — cross-cutting NFR layer (per VIEW/ADR convention; the §14.1 standard set is owned by the enforcing/realizing components). OPS6 imposes obligations onto those components' existing deliverables rather than shipping its own; the realization/verification map is in PLAN-OPS6. Specifically:
- OPA/Rego integration · Rego CI gates · audit emission (ADR 0034) · runbook (policy rollback, break-glass, false-deny triage) · alerts (post-promotion deny-rate spike, bundle-load failure, pin/signature mismatch) · Grafana dashboard (policy-change lifecycle) — each **applicable but owned by A7 / B3 / B16 / B14**, with OPS6 naming the obligation each must absorb (cross-linked in PLAN §3).
- Headlamp plugin · HolmesGPT toolset · policy simulator — **owned by A20 / A22** (preview-before-commit is realized there).

## 9. Acceptance Criteria

- **AC-OPS6-01** (REQ-01): An attempt to load an unpinned or tampered bundle into running OPA fails delivery; the only path that mutates a live bundle is a Git change reconciled by ArgoCD. *(Chainsaw + CI)*
- **AC-OPS6-02** (REQ-02): A policy change with a failing Rego unit test is blocked at merge by a required CI check. *(PyTest/CI)*
- **AC-OPS6-03** (REQ-03): A planted allow-rule that grants beyond the B3 RBAC-floor predicate fails the restrictor-conformance gate and cannot merge. *(PyTest/Rego unit)*
- **AC-OPS6-04** (REQ-04): A change whose entrypoint omits the canonical decision document / `simulated` flag fails the decision-contract gate. *(PyTest)*
- **AC-OPS6-05** (REQ-05): A change that removes the serving package for a §6.6 decision point fails the coverage-map gate; the B22 extension point remains exercised. *(PyTest/CI)*
- **AC-OPS6-06** (REQ-06): An author can run a proposed change through the A20 simulator and obtain a per-layer decision delta with `simulated: true` and zero enforcement side effect, before promotion. *(PyTest + Playwright)*
- **AC-OPS6-07** (REQ-07): A new bundle revision lands first in a non-enforcing/audit-mode stage emitting prospective decisions under `platform.policy.*`; promotion to enforce is a distinct audited action. *(Chainsaw)*
- **AC-OPS6-08** (REQ-08): After an enforced change, a `git revert` + ArgoCD re-sync restores the prior SHA-pinned revision within the documented rollback target; the rollback is audited. *(Chainsaw + CI)*
- **AC-OPS6-09** (REQ-09): A tenant-scoped author is denied a change to platform-global (and to another tenant's) policy surface by Headlamp/OPA action-gating reading JWT roles; a within-scope change is permitted. *(PyTest)*
- **AC-OPS6-10** (REQ-10): A change to a privileged/global surface creates an `Approval`; an attempt to lower its required level via policy has no effect (OPA may only elevate). *(PyTest)*
- **AC-OPS6-11** (REQ-11): A change that would deny a member of the policy-plane operability set (author/reviewer/rollback identity, admission webhook liveness, break-glass) is blocked by the pre-promotion self-lockout check; a constructed self-lockout case is caught before enforce. *(PyTest + Chainsaw)*
- **AC-OPS6-12** (REQ-12): Executing the break-glass procedure restores policy-plane operability from a deliberately-broken enforce state; the action emits `platform.security.*` and is audited. *(Chainsaw)*
- **AC-OPS6-13** (REQ-13): The lifecycle of a sample change (proposed→simulated→audit-mode→enforce→rolled-back) appears under `platform.policy.*` via the A18 adapter, with author and approver attributed. *(PyTest)*
- **AC-OPS6-14** (REQ-14): A change to a `CapabilitySet.opaPolicyRefs[]` snippet is subject to the same gates, authorization, and rollback as a platform bundle (a failing-test snippet is blocked; an unauthorized author is denied). *(PyTest)*
- **AC-OPS6-15** (REQ-15, PROPOSED): *If* bundle signing is adopted, a bundle whose signature fails verification is refused at load; until adopted, this AC is marked deferred pending the §10 ADR. *(CI — conditional)*

## 10. Risks & Open Questions

- **R1 (high):** **Policy lockout** — a change that denies the identities/paths needed to roll policy back, or that takes the admission webhook out of the write-path, can make the platform un-recoverable through normal means. *Mitigation:* REQ-11 self-lockout check + REQ-12 break-glass (GitOps revert-by-SHA is the primary escape because Git, not the cluster, is the policy SoR). Blast radius platform-wide.
- **R2 (high):** **Unsigned-bundle supply chain** — ADR 0002 ships SHA-pinning but no signature verification; a SHA pin proves *immutability of what was delivered*, not *authenticity of who built it*. A compromised CI/Git path could deliver a pinned-but-malicious bundle. *Reconciliation:* PROPOSED-ADR on bundle signing (below); B22 threat model must cover. Distinct from the image-signature gap (ADR 0002 R2).
- **R3 (high):** **Global-bundle ownership is undefined** — who may change the platform-wide (cross-tenant) policy bundle, and through what gate, is not settled in source. Concentrating it risks a single point of failure; distributing it risks tenant escalation. *Surfaced as PROPOSED-ADR, not decided.* Blast radius platform-wide.
- **R4 (med):** **Staged-rollout mechanism is under-specified in source** — Gatekeeper audit-mode and per-layer dry-run exist (A7/ADR 0038), but a first-class promotion fabric for the *policy bundle itself* (vs. Kargo for app artifacts) is not named. `[PROPOSED]` reuse the audit-mode→enforce promotion + ArgoCD; whether Kargo Stages also gate policy bundles is open. *Surfaced as PROPOSED-ADR.*
- **R5 (med):** **Self-lockout check depends on a correctly-modeled operability set** — if the set is incomplete (e.g. omits a break-glass identity), REQ-11 gives false assurance. *Mitigation:* the operability set is itself version-controlled and reviewed; A20 simulator validates it pre-promotion.
- **R6 (med):** **`actionType` for a policy-change `Approval`** and the concrete `platform.policy.*` lifecycle event names are `[PROPOSED — not in source]`; reconcile with B19/B16/B12 before freeze.
- **R7 (low):** **Admission fail-policy interaction** — break-glass via a fail-open admission window (A7 R1, `[PROPOSED]` default) trades security for recoverability; must be bounded, namespaced, and audited. Reconcile with A7's fail-policy decision.

### PROPOSED-ADR TITLES (genuinely open — surfaced, not decided)

- **PROPOSED-ADR: Policy change-control model — gate, reviewers, and merge authority for Rego changes.** *Rationale:* source fixes Rego-as-code + CI gates but not the human change-control model (who reviews, how many approvers, whether all changes or only privileged ones route through the B19 gate); the platform's entire enforcement posture rides on it.
- **PROPOSED-ADR: Ownership of the global/cross-tenant OPA policy bundle.** *Rationale:* the platform-wide bundle governs every tenant; who may change it (a platform-policy role, a quorum, a dedicated team) and through what gate is undefined and is both a security and an availability single point of failure.
- **PROPOSED-ADR: Policy-bundle signing and load-time signature verification.** *Rationale:* ADR 0002's SHA-pinning proves immutability but not authenticity; a signing/verification step closes a supply-chain gap, but the mechanism (Sigstore/cosign vs. native OPA bundle signing), key custody (→ OPS3), and enforcement point are open.
- **PROPOSED-ADR: Staged-rollout/promotion mechanism for policy bundles (audit-mode→enforce, and whether Kargo Stages gate policy).** *Rationale:* the platform has Kargo (A23) for app-artifact promotion and Gatekeeper audit-mode for policy preview, but no settled promotion fabric for the policy bundle itself; reusing one vs. building a policy-specific path is an open architecture choice.
- **PROPOSED-ADR: Anti-lockout / break-glass policy-plane recovery contract.** *Rationale:* the self-lockout check and break-glass path (fail-policy window vs. GitOps revert-by-SHA vs. a privileged recovery namespace exemption) are safety-critical and not specified in source; the chosen recovery contract bounds the platform's worst-case un-recoverability.

## 11. References

- `_meta/pending-operational-nfr-layer.md` — OPS6 scope (Rego human-factors & lifecycle; relates A7, B3, B16, A20, ADR 0018).
- architecture-overview.md §6.6 (defense-in-depth, OPA decision points line ~532, policy simulator line ~544, RBAC-floor), §6.13 (versioning), §14.2 (B2/B3/B16 policy split).
- ADR 0002 (OPA/Gatekeeper; Rego-as-code, SHA-pinned, CI-tested; no image-sig), ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0017 (approval elevation), ADR 0030 (versioning), ADR 0031 (`platform.policy.*` taxonomy), ADR 0034 (audit adapter), ADR 0038 (policy simulators / dry-run), ADR 0027 (threat-model / B22 feeds B16).
- Related pieces: A7 (engine/admission/audit-mode), B3 (framework/gates/contract), B16 (content/coverage/elevation), B2/A6/A1 (enforcement points), B19-core (`Approval` gate), A20 (simulator), A22/ADR 0039 (editors), B14 (CI), B9 (CLI gates), B13 (OPA data), A18/B12 (audit/events), B22 (threat model). Sibling OPS pieces: OPS3 (key/secret lifecycle — signing-key custody), OPS5 (day-2 ops — upgrade/promotion).

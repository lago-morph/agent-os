# PLAN OPS3 — Key & Secret Management

> spec: SPEC-OPS3 · kind: COMPONENT (cross-cutting NFR layer) · tier: T1
> wave: post-W4 (consumer band; cross-cutting NFR) · estimate: L
> upstream-pieces: [B4, A17, A18, B13, A1, B1, A6, A21] · downstream-pieces: [F4, F2, OPS2]

## 1. Implementation Strategy

OPS3 is a **realization-and-verification plan for a cross-cutting layer**, not a build of a new service. The strategy: (1) freeze the secret-class taxonomy and a provenance map binding each class to its Canon-named origin and trust root; (2) for each SPEC REQ, name the **enforcement point that already exists or must be added** — a CRD/XRD field, an OPA/Gatekeeper rule (authored by B16 from an input shape OPS3 supplies, per the B4 admission-contract pattern), an ESO `SecretStore`/`ExternalSecret` topology setting, or a CI/conformance gate; (3) make rotation testable end-to-end against each mechanism (`VirtualKey` `ttl`/reissue, ESO resync, Composition re-write + Reloader, signing-key rollover with overlap); (4) record the cross-cutting obligations each existing component spec must absorb (no edits to those specs in this PR — captured as follow-up notes routed like F4 findings); (5) surface the two genuinely-open decisions (KMS, rotation cadence) as PROPOSED-ADRs the user ratifies before the enforcing values are pinned. Because every primitive already exists in Canon, the critical path is the **conformance/CI gate work + the OPA scoping input-shape + the rotation test harness**, not new runtime code.

## 2. Ordered Task List

- **TASK-01:** Build the secret-class taxonomy + provenance map (six classes → origin → trust root). — produces: provenance map doc + machine-readable class registry — depends-on: []
- **TASK-02:** Define the ESO topology requirements (`SecretStore`/`ExternalSecret` refresh-interval + IRSA-bound-SA contract) that A17/B1/V6-11 instances must satisfy. — produces: ESO topology requirement spec + template manifests — depends-on: [TASK-01]
- **TASK-03:** Author the OPA secret-read-scoping **input shape** (namespace/principal/secret-ref → allow/deny) and the Gatekeeper admission input for unmatched-namespace secret-reads; hand to B16 for Rego. — produces: OPA input-shape contract for B16 — depends-on: [TASK-01]
- **TASK-04:** Specify the KMS / envelope-encryption-at-rest requirement + the kind-vs-AWS parity caveat; draft the PROPOSED-ADR "KMS provider" for ratification. — produces: at-rest requirement + PROPOSED-ADR draft — depends-on: [TASK-01]
- **TASK-05:** Define the rotation mechanism per class and the no-outage/overlap invariant; draft the PROPOSED-ADR "rotation cadence policy". — produces: rotation design + cadence PROPOSED-ADR draft — depends-on: [TASK-01]
- **TASK-06:** Define revocation + fail-closed + blast-radius-containment rules (namespace+substrate+`scope` bound; Envoy egress bound). — produces: revocation/containment design — depends-on: [TASK-01, TASK-03]
- **TASK-07:** Build the CI/conformance gates: no-secret-in-Git/image/log/trace scan; provenance gate; connection-secret-key-set-invariance check on rotation. — produces: CI gate definitions wired into B14/B9 harness — depends-on: [TASK-01, TASK-02]
- **TASK-08:** Build the rotation test harness (per-class rotation + no-outage assertion + old-value-rejected assertion). — produces: PyTest/Chainsaw rotation suites — depends-on: [TASK-05, TASK-06]
- **TASK-09:** Define the audit/event emission obligations (rotation/revocation/denial → A18 adapter; `platform.security.*`/`platform.policy.*`) + the rotation-freshness `GrafanaDashboard` XR + alerts. — produces: emission obligation + dashboard/alert specs — depends-on: [TASK-05, TASK-06]
- **TASK-10:** Record the cross-cutting obligations each component spec must absorb (A17, B1, B4, V6-11, F4, OPS2) as routed follow-up notes; cross-link OPS2 for secret-store backup. — produces: absorption/cross-link register — depends-on: [TASK-02..TASK-09]

Critical path: TASK-01 → TASK-05 → TASK-08 → TASK-10 (rotation is the load-bearing, hardest-to-verify obligation).

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed

- **B4** — `connectionSecretRef` / `credentialsSecretRef` fields + the tested connection-secret-shape invariant (REQ-B4-12) OPS3 extends with key-set-invariance-on-rotation.
- **A17** — the `MCPServer.credentialsRef` ESO-materialization pattern + the three credential modes whose secrets OPS3 governs.
- **A18** — the audit adapter library OPS3 requires for rotation/revocation/denial emission (no direct writes).
- **B13** — `VirtualKey` reconcile (`ttl`/reissue is the rotation lever).
- **A1** — holds user-credential OAuth tokens (lifecycle governed here, stored by A1).
- **B1** — proxy secrets via ESO + Reloader rollout, a reference rotation path.
- **A6** — sandbox/Envoy egress boundary that bounds leaked-credential blast radius.
- **A21 / B4 `TenantOnboarding`** — `defaultServiceAccounts[]` + `clusterOIDCClaimMapping` establishing the identity that scopes secret reads.

### 3.2 Downstream pieces blocked on this — ids

- **F4** — consumes OPS3's rotation/scoping/secret-handling **targets** as its verification contract (F4 runs the drills against them).
- **OPS2** — consumes the secret-store backup obligation (REQ-OPS3-14) and restore-ordering for signing material.
- **F2** — DR testing exercises the secret-store backup/restore OPS3 mandates.

### 3.3 Continuous (non-blocking) inputs

- **B16** — authors the OPA Rego from OPS3's input shape (TASK-03); continuous, not wave-blocking.
- **B22** — threat model design; OPS3's blast-radius + trust-root targets bind to B22's attack patterns.
- **B14 / B9** — test framework + CLI the CI gates and rotation harness plug into.
- **A7 / A15** — OPA/Gatekeeper engine + Reloader/ESO install (baseline); OPS3 configures, does not install.

## 4. Parallelizable Subtasks

- After TASK-01: **TASK-02, TASK-03, TASK-04, TASK-05** fan out independently (ESO topology, OPA input-shape, KMS-at-rest, rotation design have no mutual dependency).
- **TASK-07** (CI gates) and **TASK-08** (rotation harness) run concurrently once their inputs land (TASK-02/TASK-05/TASK-06).
- **TASK-09** (emission/dashboard/alerts) parallel with TASK-08.
- Serial spine: TASK-01 first; TASK-10 last (it consolidates everything into the absorption register).
- Independent fan-out groups: **{02,03,04,05}**, then **{07,08,09}**, then **{10}**.

## 5. Test Strategy

Maps SPEC §9 ACs to layers. Fixtures/fakes noted for not-yet-landed upstreams.

| AC | Layer | Enforcement point / how tested | Fixtures / fakes |
|---|---|---|---|
| AC-OPS3-01 (provenance, no-inline) | PyTest + Chainsaw | provenance scan over loaded Secrets resolves each to a Canon source; admission/CI rejects inline-in-CRD/image | fake `MCPServer`/XR with injected inline secret (negative case) |
| AC-OPS3-02 (signing keys via IRSA-bound ESO) | PyTest | scan Git/images; assert ESO SA is IRSA-bound | mock cloud secret store on kind (local backend) |
| AC-OPS3-03 (KMS at rest) | Chainsaw (AWS) / PyTest (kind) | assert AWS Secret encrypted via configured KMS; kind reduced-guarantee documented + asserted | kind: local key fake; AWS path gated on KMS PROPOSED-ADR ratification |
| AC-OPS3-04 (no secret in Git/log/trace/audit) | PyTest | grep/scan harness over Git, container logs, traces, audit payloads | trace/audit fixtures from A18 adapter fake |
| AC-OPS3-05 (OPA scope by `platform_namespaces`) | Chainsaw + PyTest | B16 Rego (from OPS3 input shape) denies unauthorized namespace read; RBAC-floor honored | B16 bundle stub if not landed; JWT fixture with scoped `platform_namespaces` |
| AC-OPS3-06 (no cross-tenant reuse; `scope`) | Chainsaw | `XAgentDatabase` `scope` bounds `credentialsSecretRef` resolution | fake `AgentDatabase` claims at agent/tenant/user scope |
| AC-OPS3-07 (admission fail-closed) | Chainsaw | Gatekeeper rejects unmatched-namespace secret-read (ADR 0002) | B16 admission rule stub |
| AC-OPS3-08 (rotation mechanism, key-set retained) | Chainsaw + PyTest | `VirtualKey` `ttl`/reissue; ESO resync; Composition re-write + Reloader; key-set-invariance assertion | B13 `VirtualKey` fake; ESO resync fake |
| AC-OPS3-09 (no outage; signing-key overlap) | PyTest | in-flight traffic continuity during rotation; dual-key validation window | load-generator fixture (coordinate with F5); signing-key pair fixture |
| AC-OPS3-10 (cadence policy + overdue alert) | PyTest | cadence policy present per class; overdue secret raises alert | cadence intervals from PROPOSED-ADR (placeholder until ratified) |
| AC-OPS3-11 (old value rejected; ESO removal fail-closed) | Chainsaw + PyTest | post-rotation old value rejected; remove `ExternalSecret` → consumer fails closed | mirrors A17 AC-09 harness |
| AC-OPS3-12 (deleted secret → deny, no fallback) | PyTest | consumer denies; assert no embedded/default credential path | negative-path fixture |
| AC-OPS3-13 (blast radius bounded; Envoy egress) | Chainsaw + PyTest | leaked secret shown namespace+substrate+`scope`-bound; exfil blocked by Envoy allowlist (ADR 0003) | A6 egress allowlist fixture |
| AC-OPS3-14 (signing material: scope+cadence+backup) | PyTest | IRSA-bound read-only; shortest cadence; present in OPS2/F2 backup set | OPS2 backup-set fake until OPS2 lands |
| AC-OPS3-15 (audit/security emission) | PyTest | rotation/revocation/denial → A18 `platform.audit.*` + `platform.security.*`; no direct writes | A18 adapter fake |

Playwright: **N/A** — OPS3 ships no UI. Note: ACs depending on the two PROPOSED-ADRs (AC-03 KMS provider, AC-10 cadence intervals) are **partial** until ratified; the tests assert the *mechanism/policy-exists*, not the *specific provider/interval*.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/ops` (the cross-cutting NFR layer base; assumes A/B waves W0–W4 and the consumer-band component pieces B4, A17, A18, B13, A1, B1, A6, A21 are merged to main). OPS3 is a **post-W4 cross-cutting layer**, sequenced like the consumer band (it constrains/verifies, it is not a wave-depth dependency for any A/B component).

### 6.2 PR — `piece/OPS3-key-secret-management` → base `wave/ops`; carries SPEC-OPS3 + PLAN-OPS3 (these two new files only; no edits to existing specs/plans/source).

### 6.3 Merge order — independent of OPS-siblings (OPS1, OPS2, OPS4–6) within the OPS band; each OPS piece is its own stacked PR per the pending-NFR-layer plan ("one per OPS piece"). OPS3 should land **before F4** (F4 verifies OPS3's targets) and is **paired with OPS2** for the secret-store-backup cross-link. The two PROPOSED-ADRs (KMS provider, rotation cadence) are authored as **separate ADR PRs only if the user asks** — the OPS3 PR surfaces titles + rationale, it does not author them. Absorption obligations on A17/B1/B4/V6-11 route out as their own follow-up PRs (the F4-findings pattern), not folded into this PR.

## 7. Effort Estimate

| Task | Size |
|---|---|
| TASK-01 taxonomy/provenance | S |
| TASK-02 ESO topology reqs | M |
| TASK-03 OPA input-shape | M |
| TASK-04 KMS-at-rest + ADR draft | S |
| TASK-05 rotation design + ADR draft | M |
| TASK-06 revocation/containment | M |
| TASK-07 CI/conformance gates | L |
| TASK-08 rotation harness | L |
| TASK-09 emission/dashboard/alerts | M |
| TASK-10 absorption register/cross-links | S |

Rollup: **L** (consistent with the SPEC header). Critical path within the piece: TASK-01 → TASK-05 → TASK-08 → TASK-10 (≈ M+M+L+S). The two L tasks (07 gates, 08 harness) run in parallel, so they do not stack on the critical path.

## 8. Rollback / Reversibility

OPS3 ships **specifications, conformance gates, OPA input-shapes, and test harnesses** — no runtime daemon — so rollback is low-risk: revert the PR and the CI gates + rotation suites are removed; no running secret would change shape or rotation behaviour because OPS3 does not own the secret-writing path (B4/ESO do). What breaks if reverted: the platform loses its (a) provenance/no-inline CI gate, (b) secret-read-scoping admission input for B16, (c) rotation no-outage + key-set-invariance tests, and (d) the rotation-cadence/overdue alerting — i.e., the secret lifecycle reverts to the pre-OPS3 scattered state (rotation as an F4 drill only, no cadence enforcement, no provenance gate). Downstream **F4** loses its verification contract for the rotation/scoping/secret-handling drills, and **OPS2** loses the named secret-store-backup obligation. Because the enforcement lives in gates/policies rather than schema mutations, reverting cannot corrupt any in-flight secret or break a consumer of the connection-secret contract. The two PROPOSED-ADRs, if already ratified and authored separately, are **not** reverted by backing out OPS3 (they stand as independent decisions).

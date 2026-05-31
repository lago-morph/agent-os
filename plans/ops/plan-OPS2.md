# PLAN OPS2 — Disaster Recovery Architecture

> spec: SPEC-OPS2 · kind: COMPONENT (cross-cutting NFR) · tier: T1
> wave: post-W4 cross-cutting (operational/NFR layer; real-build after the F-pieces it constrains)
> estimate: L
> upstream-pieces: [B4, A18, A10, B11, A11, A21, B19, A7, A4, A23] · downstream-pieces: [F2, F6]

## 1. Implementation Strategy

OPS2 is a **cross-cutting NFR layer**, not a component that ships a reconciler. Its realization is
(a) a **data-classification + RPO/RTO contract** that constrains existing state-bearing pieces, (b)
**backup configuration** wired onto the substrate-native mechanisms B4 already exposes
(CloudNativePG/RDS backups, S3 versioning/replication, Mongo snapshots) without inventing a new
CRD, (c) a **documented, ordered restore procedure** expressed against Canon claim shapes and the
uniform connection secret, and (d) the **verification** that proves the RPO/RTO targets — Chainsaw
restore tests, a PyTest RPO/RTO measurement harness, and the end-to-end **DR drills owned by F2**.
The strategy is "constrain + configure + verify, do not re-own": every backup/restore *mechanism*
lives in the component that owns the store (A18, B4, A10/B11, A21, B19); OPS2 supplies the targets,
the ordering, and the proof. Because the RPO/RTO numbers and the failover model are genuinely open,
the plan ratifies the proposed ADRs (SPEC §10) **before** F2 builds drills against them.

## 2. Ordered Task List

- **TASK-01:** Ratify the data-classification taxonomy + per-class RPO/RTO numbers (resolve
  **OPS2-ADR-B**) and the failover posture (**OPS2-ADR-D**). — produces: frozen §4.4 class table +
  failover stance — depends-on: [] (gates everything testable).
- **TASK-02:** Decide backup-policy attachment (**OPS2-ADR-A**: out-of-band operator config vs XRD
  field) and control-plane backup posture (**OPS2-ADR-C**: GitOps-reconcile-only vs etcd snapshots).
  — produces: backup-mechanism design per class — depends-on: [TASK-01].
- **TASK-03:** Wire **backup configuration** onto substrate-native mechanisms via B4's XRDs
  (CloudNativePG/RDS backup + PITR on `XPostgres`; S3 versioning/+replication on `XObjectStore`;
  Mongo snapshots on `XMongoDocStore`). — produces: backup config manifests — depends-on: [TASK-02].
- **TASK-04:** Author the **restore-ordering** procedure (control plane → systems of record →
  advisory rebuilds → consumers) including version-skew handling via conversion webhooks (ADR 0030).
  — produces: restore runbook (handed to F6) — depends-on: [TASK-01, TASK-02].
- **TASK-05:** Specify the **per-component NFR obligations** (the backup/restore responsibility each
  of A18, B4, A10/B11, A21, B19 absorbs) as cross-links for those specs. — produces: obligation
  trace table (REQ-OPS2-13) — depends-on: [TASK-02].
- **TASK-06:** Build the **RPO/RTO measurement harness** (PyTest): destructive-event scenario →
  measured restore → pass/fail vs target. — produces: measurement harness — depends-on: [TASK-01].
- **TASK-07:** Build **Chainsaw restore tests**: restored claim re-emits the uniform connection
  secret and is admitted by Gatekeeper substrate-match; `MemoryStore.accessMode` unchanged; tenant
  restore creates no CapabilitySet. — produces: Chainsaw suites — depends-on: [TASK-03, TASK-04].
- **TASK-08:** Author the **DR observability**: `GrafanaDashboard` XR (backup freshness, RPO/RTO
  headroom, last drill) + `platform.observability.*` RPO-at-risk alerts + restore self-audit via the
  A18 adapter. — produces: dashboard claim + alert rules — depends-on: [TASK-03].
- **TASK-09:** Hand the restore procedure + targets to **F2 (drills)** and **F6 (runbook)**;
  integrate the F2 drill as the end-to-end RPO/RTO proof. — produces: F2/F6 integration —
  depends-on: [TASK-04, TASK-06, TASK-07].
- **TASK-10 (proposed, gated):** If OPS2-ADR-C selects etcd snapshots and/or OPS2-ADR-D selects
  active-passive/active-active, scope the etcd-backup tool and cross-region replication work. —
  produces: follow-on work items — depends-on: [TASK-01, TASK-02]. `[PROPOSED]`.

Critical path: **TASK-01 → TASK-02 → TASK-04 → TASK-09** (the targets + restore procedure gate the
drills that prove the whole piece).

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed
- **B4** — substrate XRDs (`XPostgres`/`XSearchIndex`/`XObjectStore`/`XMongoDocStore`), the
  higher-level `AuditLog`/`XAgentDatabase`/`TenantOnboarding`/`GrafanaDashboard` XRDs, the uniform
  connection-secret shape, substrate-agnostic XR status. Backup/restore is expressed against these.
- **A18** — Postgres+S3 audit system of record, immutable verified S3 archive, OpenSearch-rebuild
  path; audit-class RPO inherits A18's batch cadence and failure-mode contract.
- **A10 / B11** — Letta memory backend + `MemoryStore` access modes; memory-class backup/restore.
- **A11** — OpenSearch, the advisory store that is **rebuilt** (not restored authoritative).
- **A21** — `TenantOnboarding` reconciler; tenant recovery reconstructs onboarding state.
- **B19** — `Approval` CRD + backing store; open-approval data class.
- **A7** — Gatekeeper substrate-match admission a restored claim must pass (ADR 0002).
- **A4** — Knative/NATS broker; in-flight event-backlog recovery consideration.
- **A23** — Kargo/ArgoCD GitOps state as the control-plane recovery source of truth.

> Build-order note: B4/A18 are W1, A21/B19 land by W3–W4. OPS2 is a **post-W4 cross-cutting layer**
> (like the F-pieces it constrains), so all hard upstreams have shipped before OPS2's build.

### 3.2 Downstream pieces blocked on this — ids
- **F2 (DR testing)** — drills assert OPS2's RPO/RTO and restore ordering; F2 cannot define a
  pass/fail without OPS2's frozen targets (TASK-01).
- **F6 (production runbook compilation)** — compiles OPS2's restore procedure (TASK-04) into the
  operator runbook set.

### 3.3 Continuous (non-blocking) inputs
- **B14 (test framework)** — provides the Chainsaw/PyTest harness OPS2's tests plug into.
- **B22 (security threat model)** — backups are sensitive assets (audit integrity, tenant data);
  the threat-model standards apply to backup storage/access.
- **OPS1 (scale/cost)** — restore-throughput sizing references OPS1's capacity model (non-blocking
  cross-reference).
- **OPS3 (key/secret management)** — **hard precondition for correctness**: key-material
  recoverability gates encrypted-backup recoverability; consumed as a contract, not a build edge.

## 4. Parallelizable Subtasks
- After TASK-02: **TASK-03** (backup config), **TASK-04** (restore procedure), and **TASK-05**
  (per-component obligations) are independent fan-out and run concurrently.
- **TASK-06** (PyTest RPO/RTO harness) depends only on TASK-01 and can start as soon as targets are
  frozen — in parallel with TASK-03/04/05.
- **TASK-07** (Chainsaw) and **TASK-08** (observability) join after TASK-03/04 and run concurrently.
- **TASK-09** (F2/F6 integration) is the join point and is **not** parallelizable with the tasks it
  consumes.

## 5. Test Strategy

Map ACs (SPEC §9) to layers. Fakes are needed only if OPS2 is built before a late upstream lands,
which it is not (OPS2 is post-W4) — so tests run against real restored stores.

| AC | Layer | Test |
|---|---|---|
| AC-OPS2-01 | PyTest | assert every class in the table has mechanism + RPO + RTO; fail on any unclassified state |
| AC-OPS2-02 | Chainsaw + F2 drill | destroy cluster objects → reconcile from Git/Kargo → assert RPO=0, RTO≤target, no etcd backup used |
| AC-OPS2-03 | F2 drill + PyTest | Postgres-loss → measure audit in-flight loss ≤5 min; assert OpenSearch not used as recovery SoR |
| AC-OPS2-04 | Chainsaw + F2 drill | assert `XSearchIndex` rebuilt from S3/Postgres (not an OpenSearch backup), within RTO |
| AC-OPS2-05 | F2 drill + PyTest | `XAgentDatabase` PITR/backup restore → measure RPO≤15 min, RTO≤4 h |
| AC-OPS2-06 | Chainsaw | restore `MemoryStore`; assert `accessMode` unchanged (no access widening) |
| AC-OPS2-07 | Chainsaw + F2 drill | full restore follows ordering; **fail** if advisory-before-SoR or consumer-before-secret |
| AC-OPS2-08 | Chainsaw | tenant restore rebuilds `TenantOnboarding`; assert **no CapabilitySet** created by onboarding |
| AC-OPS2-09 | Chainsaw | restored store re-emits uniform connection secret; claim admitted by Gatekeeper substrate-match |
| AC-OPS2-10 | PyTest (doc lint) | assert spec states one unambiguous failover posture; cross-region marked AWS-only |
| AC-OPS2-11 | Chainsaw | `vN-1` backup → `vN` cluster restore via conversion webhook; converted objects reconcile |
| AC-OPS2-12 | PyTest + Chainsaw | restore action in `audit_events`; induced backup failure raises `platform.observability.*` alert |
| AC-OPS2-13 | PyTest (doc lint) | per-component obligation table present; each traces to a REQ |
| AC-OPS2-14 | F2 drill harness | every class RPO/RTO has a drill producing a measured number + verdict |

- **Chainsaw** owns claim-restore / connection-secret / admission / access-mode / ordering assertions.
- **PyTest** owns the RPO/RTO measurement harness and the doc-lint conformance checks.
- **Chaos / DR drill (F2)** owns the end-to-end destructive-event → measured-restore proof — this is
  what ultimately proves RPO/RTO; Chainsaw/PyTest prove the invariants a drill relies on.
- **Playwright — N/A** (no UI).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch
OPS2 is a **post-W4 cross-cutting layer**. Per `_meta/waves.md` the F-pieces it constrains are the
`consumer` band (real-build after A/B). Base branch = **`ops/nfr-layer`** (the operational/NFR
integration branch stacked on top of `wave/4`, carrying the OPS1..OPS6 sibling set), which contains
the merged W0–W4 component specs OPS2 binds to (B4, A18, A10, B11, A11, A21, B19, A7, A4, A23).

### 6.2 PR
`piece/OPS2-disaster-recovery` → base `ops/nfr-layer`; carries **only** this piece's two new files
(`specs/ops/spec-OPS2.md`, `plans/ops/plan-OPS2.md`). It edits no existing spec/plan/source. Where a
component spec must absorb a DR obligation (REQ-OPS2-13), that is recorded as a **cross-link in this
PR's body** for the owning piece to absorb in its own follow-up PR — OPS2 does not edit those specs.

### 6.3 Merge order
- Within the OPS layer, **OPS2 is independent of OPS1/OPS4/OPS5/OPS6** and can merge in any order
  with them. It has a **soft sequencing preference after OPS3** (key/secret recoverability is an
  OPS2 precondition) — but they touch disjoint files, so the dependency is contractual, not a merge
  conflict; OPS2 references OPS3, it does not import it.
- OPS2 must merge **before F2/F6** are built so they bind to the frozen targets (down-stack).
- `ops/nfr-layer` rolls up to `main` once the OPS sibling set is merged; per-piece PRs are the unit
  of review.

## 7. Effort Estimate
- TASK-01 (S) · TASK-02 (S) · TASK-03 (M) · TASK-04 (M) · TASK-05 (S) · TASK-06 (M) · TASK-07 (L) ·
  TASK-08 (S) · TASK-09 (M) · TASK-10 (gated, M).
- **Rollup: L.** The classification + restore-procedure design is light; the weight is in the
  Chainsaw restore suites (TASK-07) and the F2 drill integration (TASK-09).
- **Critical path within the piece:** TASK-01 → TASK-02 → TASK-04 → TASK-09 (targets + restore
  procedure gate the end-to-end drill that proves the piece). TASK-10 is off the critical path and
  only materializes if OPS2-ADR-C/D select snapshot/active-passive postures.

## 8. Rollback / Reversibility
- OPS2 adds **no new CRD/XRD and no schema migration** — it is config (backup policy on existing
  XRDs), documentation (restore runbook), tests, and a dashboard/alert set. Reverting the PR removes
  the targets, the backup configuration, the restore procedure, the DR tests, and the dashboard.
- **What breaks if reverted:** the platform loses its *defined* RPO/RTO and its *verified* restore
  procedure — backups configured by TASK-03 stop being scheduled, F2 loses its pass/fail targets, and
  recovery reverts to ad-hoc. No runtime component fails (OPS2 ships no reconciler), so revert is
  **safe at runtime but removes the DR guarantee**. Re-applying is idempotent (config + docs + tests).
- Because backup configuration is operator-native (CloudNativePG/RDS/S3), reverting the OPS2 PR does
  **not** delete already-taken backups; it only stops OPS2-owned scheduling/verification. Existing
  snapshots remain restorable manually.

# PLAN B20 — Persistent volume access for Platform Agents

> spec: SPEC-B20 · kind: COMPONENT · tier: T1
> wave: W2 · estimate: M
> upstream-pieces: [A6, A7, A5] · downstream-pieces: []

## 1. Implementation Strategy
B20 builds a thin, governed layer that maps pre-defined Kubernetes PVs into Platform Agents'
Sandboxes declaratively. The approach: first resolve the mechanism gap (new `AgentVolumeMapping`
CRD vs. extending A6's `SandboxTemplate` volume surface — both `[PROPOSED — not in source]`) with
the A6 owners; then implement the declarative mapping → Sandbox mount realization; then author the
admission (Gatekeeper) + runtime OPA policy that gates which agent may mount which volume under the
RBAC-as-floor / OPA-as-restrictor model; finally wire audit + `platform.policy.*` emission and the
three-layer tests. B20 provisions no storage and adds no SDK surface — it wraps native PV/PVC, A6
sandboxes, and A7 OPA.

## 2. Ordered Task List
- TASK-01: Resolve the mechanism decision with A6 owners (CRD vs `SandboxTemplate` hook); record
  the `[PROPOSED]` outcome / Canon-revision need — produces: design decision note — depends-on: [].
- TASK-02: Define the mapping surface (CRD schema or field reuse), namespaced + apiVersion-pinned —
  produces: CRD/field spec — depends-on: [TASK-01].
- TASK-03: Implement reconcile of a mapping into the A6 `Sandbox`/`SandboxTemplate` volume surface,
  read-only by default — produces: reconcile logic / manifests — depends-on: [TASK-02].
- TASK-04: Author Gatekeeper admission Rego (namespace/tenancy + approved-volume check) and runtime
  OPA decision (RBAC-floor + OPA-restrict + read-write extra check), contributed to B16/B3 — produces:
  Rego — depends-on: [TASK-02].
- TASK-05: Wire `platform.policy.*` emission + audit-adapter records for grant/deny — produces:
  event + audit integration — depends-on: [TASK-03, TASK-04].
- TASK-06: Author tests (Chainsaw admission/reconcile/mount; PyTest OPA logic), runbook, alerts, and
  the "mount a reference dataset" how-to — produces: tests + docs — depends-on: [TASK-03, TASK-04, TASK-05].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A6 — `Sandbox`/`SandboxTemplate` volume surface the mount is realized through.
- A7 — OPA/Gatekeeper engine for admission + runtime mount authorization.
- A5 — `Agent` CRD the mapping references; ARK reconcile ordering.

### 3.2 Downstream pieces blocked on this
- None hard. (Agents/compositions that need mounted reference data benefit but do not hard-block on B20.)

### 3.3 Continuous (non-blocking) inputs
- B3/B16 (OPA policy framework + content B20 contributes into), A18 (audit adapter library),
  B12 (CloudEvent schema registry for the policy/lifecycle event types), B14 (test framework),
  B22 (threat model — PV-mount attack surface), B4 (provisions the underlying storage B20 maps).

## 4. Parallelizable Subtasks
- TASK-03 (mount realization) and TASK-04 (policy authoring) run concurrently once TASK-02 fixes the surface.
- Within TASK-06, runbook/how-to docs run parallel to the test suites.

## 5. Test Strategy
- AC-B20-01/02/04/05/06 → Chainsaw: apply mappings, assert mount realized via A6 surface, assert
  cross-namespace + non-approved-volume rejection, assert read-only default and read-write extra-OPA gate.
- AC-B20-03 → PyTest (+ policy simulator A20 where available): RBAC-grants/OPA-denies → no mount;
  both allow → mount, exercising RBAC-as-floor / OPA-as-restrictor.
- AC-B20-07 → PyTest/integration: admit + deny each emit `platform.policy.*` and an audit record.
- AC-B20-08/09 → lint/review: apiVersion present + `[PROPOSED]` marker; no PV access via MemoryStore/CapabilitySet.
- Fakes needed if upstreams not landed: an A6 sandbox-volume stub and an OPA decision fake; gate
  full e2e on real A6/A7.
- Playwright N/A (no dedicated UI).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains A6, A7, A5).
### 6.2 PR — `piece/B20-agent-pv-access` → base `wave/2`; carries spec-B20 + plan-B20.
### 6.3 Merge order — independent of W2 siblings; the mechanism decision (TASK-01) may require a
  Canon/ADR-revision PR to land first if a new CRD is adopted.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 M. Rollup: **M**.
- Critical path: TASK-01 → TASK-02 → TASK-03/TASK-04 (parallel) → TASK-05 → TASK-06.

## 8. Rollback / Reversibility
Reversible: removing mapping manifests unmounts the PVs at next reconcile; removing the B20 Rego
reverts to no PV-mount admission path (mappings then fail closed — safe default). If a new
`AgentVolumeMapping` CRD was adopted, backing it out requires deleting CRD instances first, then the
CRD. No data is destroyed (PVs are pre-defined and owned elsewhere). Blast radius: agents relying on
a mounted dataset lose filesystem access and must fall back to the RAG/SDK path; no tenancy or policy
weakening results from revert (fail-closed).

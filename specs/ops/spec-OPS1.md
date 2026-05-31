# Component Spec: OPS1 — Resource Quotas (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS1
> Depends on: TenantOnboarding CRD, AgentEnvironment composition

---

## 1. Purpose & Scope

OPS1 originally covered scale & cost management across four areas: autoscaling,
resource quotas, cost showback/chargeback, and scale-related SLIs. After the
scope cut, **only the resource-quota slice survives**. This spec covers quota
enforcement and nothing else.

**Out of scope / not covered here (for traceability):**

- **Autoscaling** of agent workloads and platform components — out-of-scope
  (decision #10).
- **Cost showback / chargeback** via resource attribution — deferred to a
  future version (decision #11).
- **SLI promotion** for capacity planning — out-of-scope (decision #12).

The surviving quota model derives from decisions **#5** and **#9**.

---

## 2. Quota Enforcement (Survived Scope)

### 2.1 TenantOnboarding quota requirements

`TenantOnboarding` MUST declare resource quotas as a precondition of admission:

- `cpu` and `memory` quota entries are **mandatory** (presence required).
- Each entry is either a **concrete limit** (e.g. `cpu: "16"`,
  `memory: "64Gi"`) or an explicit **`unlimited: true`**. Absence is not
  permitted — a tenant must consciously declare either a bound or "unlimited".

### 2.2 Quota schema

Quotas are expressed as an **extensible, typed `quotas` list** discriminated by
a `quotaType` field. This allows additional quota types to be added later
without breaking the schema:

```yaml
quotas:
  - quotaType: cpu          # mandatory entry
    limit: "16"
  - quotaType: memory       # mandatory entry
    unlimited: true
  # additional quotaType entries may be added (extensible)
```

Rules:

- `quotaType: cpu` and `quotaType: memory` MUST both be present.
- For each entry, exactly one of `limit` or `unlimited: true` is set.

### 2.3 AgentEnvironment composition output

The `AgentEnvironment` composition MUST emit, for the tenant namespace:

- a Kubernetes **`ResourceQuota`** object reflecting the declared quotas, and
- a Kubernetes **`LimitRange`** object providing per-pod/container defaults and
  bounds consistent with the quota.

---

## 3. Acceptance Criteria

- **AC-1:** A `TenantOnboarding` resource that omits a `cpu` or `memory` quota
  entry is rejected at admission.
- **AC-2:** A quota entry that sets neither `limit` nor `unlimited: true`, or
  sets both, is rejected at admission.
- **AC-3:** For a valid `AgentEnvironment`, the composition emits exactly one
  `ResourceQuota` and one `LimitRange` in the tenant namespace, with values
  matching the declared `quotas` list.
- **AC-4:** Adding a new `quotaType` value to the `quotas` list does not break
  validation of existing `cpu`/`memory` entries (schema extensibility).

---

## 4. References

- DECISIONS-LOG decisions #5, #9 (quota slice — survived).
- DECISIONS-LOG decisions #10 (autoscaling, OOS), #11 (cost showback, future),
  #12 (SLI promotion, OOS) — explicitly not covered here.

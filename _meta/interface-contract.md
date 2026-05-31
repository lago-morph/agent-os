# Canon Interface Contract (FROZEN registry)

The cross-piece interface registry. Spec authors **READ** this to keep CRD / XRD / event /
SDK / connection-secret names consistent across ~30 parallel specs. Names and fields here
are extracted verbatim from `architecture-overview.md` (§6.12 CRD inventory, §6.13 versioning,
§6.7 eventing, §6.8 capability registries) and the contract-defining ADRs (0013, 0030, 0031,
0034, 0044, 0020, 0037). **No fields are invented.** Where the source does not specify a
field set, this file says **"fields: not specified in source."** Anything an author needs that
is not here must be tagged `[PROPOSED — not in source]` in the spec, not silently added.

> Casing convention: backtick identifiers are exact API kinds / field names / event
> namespaces and must be reproduced verbatim.

---

## 1. CRDs / XRDs

### 1.1 Versioning policy (applies to every CRD/XRD below) — ADR 0030 / §6.13

- CRDs use Kubernetes API versioning: `v1alpha1`, `v1beta1`, `v1`.
- Breaking changes go through a **new `vN` API group with conversion webhooks**; `vN-1` is
  **deprecated for at least one minor platform release** before removal.
- Versioning is **per-component, not centrally coordinated**. Owner of each CRD's versioning
  lifecycle = the component that owns its reconciler (see "Owner" column).
- ADR 0044 confirms CRD/API versioning applies to XRDs identically (conversion webhooks +
  deprecation windows on both Compositions when an XR schema changes).

### 1.2 ARK-reconciled CRDs (owner: ARK install = component A5)

| CRD | Scope | Key fields (source-stated) |
|---|---|---|
| `Agent` | namespaced | `capabilitySetRefs[]`, `overrides`, `sdk`, `image`, `sandboxTemplateRef`, `memoryRefs[]`, `modelRef`, `triggers`, `exposes` (A2A/MCP). `sdk` accepts `langgraph` and `deep-agents` in v1.0 (ADR 0019). |
| `AgentRun` | namespaced | `agentRef`, `inputs`, `traceId`, `triggeredBy`, `state`. Created by Knative event adapters or workflow steps. |
| `Team` | namespaced | `members[]`, `coordinationStrategy`. |
| `Tool` | namespaced | fields: defers to ARK (not further specified in source). |
| `Memory` | namespaced | `memoryStoreRef`. Access mode lives on `MemoryStore`, not here (ADR 0025). |
| `Evaluation` | namespaced | `agentRef`, `datasetRef`, `evaluators[]`. |
| `Query` | namespaced | fields: defers to ARK (not further specified in source). |

### 1.3 agent-sandbox-reconciled CRDs (owner: agent-sandbox install = component A6)

| CRD | Scope | Key fields (source-stated) |
|---|---|---|
| `Sandbox` | namespaced | `templateRef`, `runtime` (gVisor/Kata), `state`. |
| `SandboxTemplate` | namespaced | `runtime`, `warmPoolSize`, `hibernationEnabled`, `resourceLimits`. |

### 1.4 kopf-operator-reconciled CRDs (owner: custom Python kopf operator = component B13) — ADR 0013

`B13` reconciles these into the LiteLLM admin API. ADR 0013 fixes the existence and shape of
the capability CRDs; detailed overlay resolution is deferred to ADR 0032.

| CRD | Scope | Key fields (source-stated) |
|---|---|---|
| `MCPServer` | namespaced | `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility`. |
| `A2APeer` | namespaced | `endpoint`, `direction` (internal/external), `auth`, `tags`. |
| `RAGStore` | namespaced | `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`. |
| `EgressTarget` | namespaced | `fqdn`, `port`, `scheme`, `allowedMethods`. |
| `Skill` | namespaced | `gitRef`, `versionPin`, `schemaRef`. Managed via LiteLLM's skill gateway. |
| `CapabilitySet` | namespaced | `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`. Layerable; overlay semantics in ADR 0032. |
| `VirtualKey` | namespaced | `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`. |
| `BudgetPolicy` | namespaced | `scope` (key/agent/team/tenant), `period`, `limits`, `thresholdActions[]`. Consumed by OPA + LiteLLM. |

### 1.5 Other reconcilers

| CRD | Owner / Reconciler | Scope | Key fields (source-stated) |
|---|---|---|---|
| `Approval` | Argo Workflow + component B19 | namespaced | `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`. |
| `LogLevel` | per-component (in-process or rolling-restart) | namespaced | `componentSelector`, `level`, `traceGranularity`, `scope` (component/tenant/eventClass), `expiresAt`. ADR 0035. |

### 1.6 Crossplane XRs / XRDs (owner: Crossplane Compositions = component B4) — ADR 0044

ADR 0044 (Crossplane v2): substrate XRDs are **namespace-scoped XRs** that users create directly
in their namespace — there is no claim layer and no `X` prefix. The substrate XRs are `Postgres`,
`SearchIndex`, `ObjectStore`, `MongoDocStore`, `AgentDatabase`, `AuditLog`, and `GrafanaDashboard`.
**All are namespaced; there are no cluster-scoped platform CRDs in v1.0.**

| XR / XRD | Scope | Key fields (source-stated) |
|---|---|---|
| `MemoryStore` (XR) | namespaced | `accessMode` (private / namespace-shared / RBAC-OPA), `backendType`. ADR 0025. |
| `AgentEnvironment` (XR) | namespaced | `region`, `quotas`, `defaultCapabilitySetRef`. |
| `SyntheticMCPServer` (XR) | namespaced | `openApiSpecRef`, `authConfigRef`, `mcpServerRef` (back-link). Produced by A12 OpenAPI->MCP converter. |
| `GrafanaDashboard` (XR) | namespaced | `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled). ADR 0021 / 0044. |
| `AuditLog` (XR) | namespaced | `postgresRef`, `s3BucketRef`, `indexerRef`, `batchScheduleSpec`, `endpointReplicas`. ADR 0034 / 0044. |
| `TenantOnboarding` (XR) | namespaced | `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`. ADR 0028 / 0037. CapabilitySets intentionally not coupled. |
| `AgentDatabase` (XR) | namespaced | `engine` (postgres/mongodb), `scope` (agent/tenant/user), `ownerRef`, `credentialsSecretRef`. ADR 0020 / 0044. Composes `Postgres` or `MongoDocStore`. |
| `Postgres` (XR) | namespaced | `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: CloudNativePG; AWS: RDS. |
| `SearchIndex` (XR) | namespaced | `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`. kind: in-cluster OpenSearch; AWS: managed OpenSearch. |
| `ObjectStore` (XR) | namespaced | `bucketName`, `lifecycle`, `connectionSecretRef`, `substrateClass`. kind: MinIO or no-op; AWS: S3. Capability-parity caveat: kind may have no archive lifecycle. |
| `MongoDocStore` (XR) | namespaced | `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: Bitnami MongoDB; AWS: DocumentDB or self-managed. |

**Substrate XRs committed for v1.0** (§14, §6.3): `Postgres`, `SearchIndex`, `ObjectStore`,
`MongoDocStore`. `AuditLog`, `AgentDatabase`, `GrafanaDashboard` are higher-level XRs that
compose those substrate primitives.

---

## 2. CloudEvent taxonomy — ADR 0031 / §6.7

**Closed set of ten top-level type namespaces** (architectural invariant). Every CloudEvent a
platform component emits MUST fall under **exactly one**. Per-event-type names within each
namespace are **design-time per component** and **deferred** (architecture-backlog §4); the
concrete schemas live in **component B12's schema registry** as they are authored. Introducing
a new top-level namespace is a breaking change requiring a new ADR.

| Namespace | What it covers (source-stated) |
|---|---|
| `platform.lifecycle.*` | Agent, AgentRun, Sandbox, Workflow, MemoryStore lifecycle (created/started/paused/resumed/completed/failed/deleted). |
| `platform.audit.*` | Audit-emission events from any component. Consumed by the audit adapter into the Postgres + S3 system of record (ADR 0034); OpenSearch indexing is advisory fanout. |
| `platform.gateway.*` | LiteLLM events that aren't pure audit: routing decisions, provider failover, MCP health changes, A2A handoffs. |
| `platform.policy.*` | OPA decisions, policy violations, dynamic registration accept/deny. |
| `platform.capability.*` | Capability-registry changes: MCPServer / A2APeer / RAGStore / EgressTarget / Skill / CapabilitySet add/update/delete. (`platform.capability.changed` per ADR 0013.) |
| `platform.evaluation.*` | Evaluation run started/completed, A/B test results, red-team findings. |
| `platform.approval.*` | Approval requested, OPA-elevated, decided (approved/rejected), timed out. (Escalation deferred per ADR 0017.) |
| `platform.observability.*` | Threshold crossings, alert routing (e.g. budget-exceeded notifications, SLA-style alerts). |
| `platform.tenant.*` | Tenant onboarding, namespace association changes, cross-tenant publish events. |
| `platform.security.*` | Security events distinct from audit: sandbox-escape signal, repeated authn failures, policy-bypass attempts. |

**Event versioning (ADR 0030 / 0031):** every CloudEvent carries CloudEvents-native
`specversion` plus a per-event-type `schemaVersion`. Backward-compatible additions bump minor;
breaking changes **mint a new event type** rather than break subscribers.

**Two initial v1.0 trigger flows** (exercise the path; not an exhaustive list): AlertManager →
HolmesGPT; budget-exceeded → email user. Each subsequent component designs its own flows.

---

## 3. SDK surfaces

### 3.1 Platform SDK (owner: component B6) — §14.2, §6.13

What Platform Agents use to call **into** the platform. Source-named surfaces:

- `memory.*`
- `rag.*`
- OTel emission
- A2A registration helpers

Python + TypeScript. Semantic versioning (ADR 0030). Ships a version-pinned compatibility
matrix against gateway / ARK / Letta versions. **Distinct from the agent SDK (B7).** Beyond the
four named surface groups above, **specific method signatures are not specified in source.**

### 3.2 Agent SDK (owner: component B7) — ADR 0019

The agent-authoring SDK. v1.0 supported set:

- **LangGraph** — the supported v1.0 SDK. `Agent.sdk` value `langgraph`.
- **Langchain Deep Agents** — opinionated batteries-included default, built on LangGraph;
  used in tutorials, how-tos, the agent profile library. `Agent.sdk` value `deep-agents`.

Multi-SDK harness shape is preserved so OpenAI Agents SDK / Strands / Anthropic Agent SDK /
Mastra / ARK ADK can be enrolled later as additive changes. Third-party harness images are
**NOT officially supported** in v1.0. Detailed SDK method surface: **not specified in source.**

### 3.3 Other versioned surfaces (ADR 0030)

- `agent-platform` CLI (owner B9) — semantic versioning; subcommand surface is the versioned API.
- A2A / MCP interfaces exposed by Platform Agents — each agent declares a version (e.g.
  `myAgent.v1`); peers pin a major; owned by the shipping component (B17 / B18).
- HTTP APIs of custom services (kopf admin, Knative adapters, audit pipeline) — URL-path
  versioning (`/v1/...`); deprecated versions reachable ≥1 platform release after replacement.

---

## 4. Connection-secret schema / substrate XRDs — ADR 0044

**Uniform connection-secret contract (architectural invariant, tested — not convention):**

- Each substrate-asymmetric primitive is wrapped in **one XRD with one Composition per
  supported substrate** (kind + AWS for v1.0).
- **Both Compositions write the same connection-secret shape**: `host`, `port`, `user`,
  `password`, `dbname` — *"or the equivalent fields per primitive"* (exact per-primitive field
  list beyond these is not enumerated in source; treat the five as the canonical shape).
- **XR status fields are substrate-agnostic**: `ready`, `endpoint`, `version`. Substrate-specific
  fields (RDS ARN, in-cluster Service paths) are deliberately absent from user-visible status.
- **Every cluster carries `platform.io/environment=kind|aws`.** An OPA Gatekeeper admission
  policy (ADR 0002) **rejects any XR with no matching Composition** for the cluster's label.
- **Capability-parity is not promised**: XR *schema* is consistent; runtime *behaviour* may
  differ per substrate (e.g. kind `ObjectStore` may produce "no archive").
- **Documented exceptions — intentionally NOT wrapped:** Knative event sources (ADR 0023),
  cluster bootstrap, cloud-provider stack selection, resource sizing/replicas/region (Kustomize).

---

## 5. Audit adapter interface — ADR 0034 / component A18

- **Audit emission is mediated by a single platform audit adapter — a Python library** every
  component links against. **No component writes audit records directly** to Postgres, S3, or
  OpenSearch.
- The adapter posts to a single deployable **audit endpoint** component (a Deployment). The
  endpoint owns its own `LogLevel` toggle (ADR 0035) so verbosity rises per-tenant /
  per-event-class without redeploying callers.
- **System of record = Postgres + S3.** In-flight rows in Postgres `audit_events` table. On AWS
  a ~5-minute batch CronJob aggregates rows → immutable S3 object (verified: exists + byte count
  + checksum) → only then deletes Postgres rows. On kind, S3 is not provisioned; Postgres alone
  is the system of record and the batch step is disabled.
- **OpenSearch is advisory fanout only** — async indexer; rebuildable from S3 (AWS) or Postgres
  (kind); never the system of record; if down, audit ingestion still succeeds.
- The `AuditLog` XRD (§1.6) provisions the whole pipeline: Postgres backing store, S3 bucket +
  lifecycle (AWS only), OpenSearch indexer, batch CronJob (AWS only), audit endpoint Deployment.
- Audit events flow under the `platform.audit.*` CloudEvent namespace (§2).
- Retention durations / redaction rules / lifecycle specifics are **deferred to Workstream F**
  (component F1) — **not specified at architecture level.**

---

## 6. Gaps explicitly left to component design (do not invent)

- Per-event-type CloudEvent **names and schemas** under each `platform.*` namespace — B12 registry.
- CapabilitySet **overlay resolution semantics** — ADR 0032 + design specs.
- Platform SDK / agent SDK **method signatures** beyond the named surface groups.
- Connection-secret **per-primitive field lists** beyond the canonical host/port/user/password/dbname.
- Audit **retention / redaction policy** — Workstream F (F1).
- `Tool` and `Query` CRD field sets — defer to ARK.

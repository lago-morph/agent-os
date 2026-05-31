# SPEC OPS3 — Key & Secret Management

> kind: COMPONENT (cross-cutting NFR layer) · workstream: — (Operational/NFR) · tier: T1
> upstream: [B4, A17, A18, B13, A1, B1, A6, A21] · downstream: [F4, F2, OPS2] · adrs: [0041, 0020, 0028, 0029, 0034, 0030, 0018, 0002, 0003] · views: [6.11, 6.6]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

OPS3 is the cross-cutting **key & secret management architecture** for the Agentic Execution Platform. The platform's most load-bearing security invariant is the **connection-secret contract** (Canon §4, ADR 0041): every substrate XRD Composition writes the uniform secret shape (`host`, `port`, `user`, `password`, `dbname` or per-primitive equivalent) into the secret named by `connectionSecretRef`, and dozens of consumers (A18 audit pipeline, B11 memory, A17 Postgres/Mongo MCP, B19 approval store) read it without branching on substrate. Around that contract sit several other classes of sensitive material the architecture already names but never treats as one lifecycle: ESO-materialized MCP-service credentials (Canon glossary "ESO"; ADR 0020), `VirtualKey` material bound to identity (Canon §1.4; `ttl`), Keycloak admin credentials + JWT signing keys (V6-11 §4.4 trust bootstrap), and the IRSA / Workload Identity trust roots that gate access to the secret store itself (ADR 0028).

The architecture (per `_meta/pending-operational-nfr-layer.md`) specifies this thinly and scatters the obligations: rotation is a drill in F4, ESO is a one-line dependency in A17/B1, signing-key sourcing is a footnote in V6-11. OPS3 makes the lifecycle a coherent layer — **provenance** (where each secret class originates and which trust root mints it), **storage** (KMS/envelope encryption at rest, ESO topology), **scoping** (which namespace/principal may read which secret), **rotation** (cadence + propagation without outage), and **revocation + blast-radius containment** (what a single leaked secret can reach). It binds those obligations to the CRD fields, OPA policies, and CI gates that already exist or that this layer requires existing component specs to absorb.

OPS3 is **architecture + invariants + acceptance**, not a new daemon. It does not redefine the connection-secret shape (that is B4/ADR 0041), the audit pipeline (A18/ADR 0034), or the identity chain (ADR 0028/0029). It constrains them and surfaces the genuinely open decisions (KMS choice, rotation cadence policy) as PROPOSED-ADRs in §10 rather than deciding them.

## 2. Scope

### 2.1 In scope

- **Secret-class taxonomy** spanning the platform's named secret/key material: (a) connection secrets written by substrate XRD Compositions (Canon §4); (b) ESO-materialized MCP-service credentials referenced by `MCPServer.credentialsRef` (ADR 0020); (c) `VirtualKey` material bound to identity (Canon §1.4); (d) Keycloak admin creds + JWT signing keys + oauth2-proxy cookie/client secrets (V6-11 §4.4, B1); (e) `XAgentDatabase.credentialsSecretRef` per-agent/tenant/user DB credentials (ADR 0020/0041); (f) cloud-resource credentials minted via IRSA / Workload Identity (ADR 0028).
- **Provenance**: each secret class is traceable to exactly one origin (cloud secret store via ESO, a substrate Composition, Keycloak, or an IRSA-federated mint) and one trust root; no secret of unknown origin is loadable by a workload.
- **Storage / encryption at rest**: envelope encryption of Kubernetes Secrets and the backing secret store, KMS integration on AWS, the kind-vs-AWS topology asymmetry, ESO `SecretStore`/`ExternalSecret` topology.
- **Scoping**: namespace-bound secret access; OPA/RBAC restriction of who may read `connectionSecretRef`/`credentialsRef` secrets; no cross-tenant secret reuse without policy.
- **Rotation**: cadence targets per secret class, propagation mechanism (ESO resync, `VirtualKey` `ttl`/reissue, Reloader-driven rollout, signing-key rollover), and the no-outage invariant during propagation.
- **Revocation & blast-radius containment**: invalidating leaked/rotated material, fail-closed behaviour on missing secret, and scoping each secret so one leak cannot reach unrelated tenants/substrates.
- **Verification obligations** this layer imposes on existing pieces (which CRD field / OPA policy / CI gate enforces each REQ) — detailed in PLAN-OPS3.

### 2.2 Out of scope (and where it lives instead — name the owning piece)

- **The connection-secret shape itself + per-primitive field mapping** → **B4** (ADR 0041 invariant; OPS3 governs its lifecycle, not its shape).
- **ESO install** → install-owned baseline (Canon glossary; A15-adjacent §6.6); **per-service `ExternalSecret`/`SecretStore` authoring** → **A17** (MCP services), **B1** (proxy secrets), **V6-11**/ADR 0028 (bootstrap secrets). OPS3 sets the topology requirements they must satisfy.
- **The identity federation chain + trust roots** → **ADR 0028 / V6-11**; OPS3 consumes the trust roots, does not re-decide them.
- **The Keycloak JWT claim schema** → **ADR 0029 / V6-11 §4.4**; `VirtualKey` issuance binds to it but OPS3 does not own claims.
- **Audit emission of secret-lifecycle events** → **A18** adapter (ADR 0034); OPS3 mandates emission, A18 owns the mechanism.
- **OPA engine + Gatekeeper install** → **A7**; **the Rego content** enforcing secret-read scoping → **B16** (OPS3 states the requirement + input shape, B16 authors the rule, per the B4 pattern).
- **Credential rotation *drills* / secret-handling audit / pen-test** → **F4** (OPS3 defines the targets F4 verifies against).
- **DR backup of the secret store** → **OPS2** (disaster recovery) and **F2** (DR testing); OPS3 names the secret-store backup obligation, OPS2 owns the topology.
- **Per-service OAuth scopes, secret shapes, user-credential token persistence, revocation specifics** → **deferred** (architecture-backlog §1.11; ADR 0020 OQ-1); OPS3 tags these `[PROPOSED]`.
- **Audit retention / redaction of secret-bearing payloads** → **Workstream F (F1)**.

## 3. Context & Dependencies

**Upstream pieces consumed (and exactly what is consumed):**

- **B4 (Crossplane Compositions)** — the `connectionSecretRef` field on every substrate XRD and the `XAgentDatabase.credentialsSecretRef`; the tested connection-secret-shape invariant (REQ-B4-12) that OPS3 extends with lifecycle obligations.
- **A17 (initial MCP services)** — the `MCPServer.credentialsRef` ESO-materialization pattern and the three credential modes (system-credential, user-credential, system-mediated; ADR 0020) whose secrets OPS3 governs.
- **A18 (audit endpoint + adapter)** — the adapter library every component uses to emit secret-lifecycle audit (no direct writes; ADR 0034).
- **B13 (kopf operator)** — reconciles `VirtualKey` (rotation via `ttl`/reissue) and the capability CRDs whose `credentialsRef` secrets OPS3 scopes.
- **A1 (LiteLLM)** — the single broker that holds user-credential OAuth tokens (ADR 0020); their lifecycle is governed here but stored by A1.
- **B1 (SSO/auth proxy)** — Keycloak client + cookie secrets via ESO (REQ-B1-07); rotation rides Reloader (A15).
- **A6 (agent-sandbox + Envoy egress proxy)** — the workload boundary into which connection/credential secrets are injected; blast-radius containment is keyed off sandbox isolation.
- **A21 (tenant onboarding) / B4 `TenantOnboarding`** — `defaultServiceAccounts[]` + `clusterOIDCClaimMapping` establish the per-tenant identity that scopes secret access (ADR 0028/0037).

**Downstream consumers:** **F4** (verifies OPS3's rotation/scoping targets via drills), **F2/OPS2** (secret-store backup + restore ordering), every component that reads a `connectionSecretRef`/`credentialsRef` secret (binds to the scoping invariant).

**ADR decisions honored (constraint each imposes):**

- **ADR 0041** — uniform connection-secret shape + `connectionSecretRef`; OPS3 may not vary the shape, only govern lifecycle.
- **ADR 0020** — MCP-service secrets MUST be ESO-sourced via `credentialsRef`; three credential modes; per-service revocation deferred.
- **ADR 0028** — cluster OIDC issuer is the single workload-identity root; Keycloak the single platform-identity root; bootstrap secrets fetched via an IRSA-bound ESO ServiceAccount; no component trusts upstream identity directly.
- **ADR 0029** — `VirtualKey` issuance binds to the §6.9 claim schema (`capability_set_refs` gates issuance).
- **ADR 0034** — secret-lifecycle audit emits via the A18 adapter only; system of record is Postgres + S3.
- **ADR 0030** — secret/key formats and the ESO topology are versioned per-component; a rotation that changes secret *shape* is a breaking change.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor governs who may read a secret; OPA cannot grant read beyond the RBAC floor.
- **ADR 0002** — Gatekeeper admission is the enforcement point for secret-scoping admission rules.
- **ADR 0003** — Envoy egress allowlist bounds where a leaked credential can be exfiltrated.

## 4. Interfaces & Contracts

Uses ONLY Canon names. Anything not in Canon is tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

OPS3 introduces **no new CRD/XRD**. It governs the lifecycle of secrets named by existing fields:

- **`connectionSecretRef`** — on `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore` (Canon §1.6). The secret it names carries the connection-secret shape; OPS3 governs its rotation/scoping.
- **`credentialsSecretRef`** — on `XAgentDatabase` (Canon §1.6); per-agent/tenant/user DB credentials.
- **`MCPServer.credentialsRef`** — (Canon §1.4) ESO-materialized MCP-service credentials.
- **`VirtualKey`** — (Canon §1.4) `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, **`ttl`**. The `ttl` field is the source-stated rotation lever for virtual-key material.
- **`TenantOnboarding`** — (Canon §1.6) `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`; the identity that scopes a tenant's secret access.
- `[PROPOSED — not in source]` — no Canon field enumerates a **rotation cadence**, **last-rotated timestamp**, or **KMS key reference** on any of these resources. OPS3 proposes these live in ESO `SecretStore`/`ExternalSecret` configuration (refresh interval) and in cloud-side KMS policy, **not** as new CRD fields, to avoid mutating the frozen XRD schemas. Surfaced as PROPOSED-ADR §10.
- Versioning (ADR 0030): a rotation that changes a secret's **key set** (not its values) is a breaking connection-secret-shape change and MUST follow the conversion-webhook/deprecation discipline B4 owns.

### 4.2 APIs / SDK surfaces

- OPS3 exposes **no new platform API/SDK surface**. Secret retrieval is the existing ESO `ExternalSecret`→Kubernetes Secret path and the substrate Composition write path. `VirtualKey` issuance/rotation is the existing B13 kopf admin path into LiteLLM.
- IRSA / Workload Identity token exchange (V6-11 §4.2): the projected ServiceAccount token is exchanged for cloud-store credentials; OPS3 governs the trust-root scoping, not the exchange mechanics.
- `[PROPOSED — not in source]` — user-credential OAuth token persistence/refresh/revocation in LiteLLM (A1) is deferred (ADR 0020 OQ; A17 OQ2). OPS3 states the obligation (revocable, scoped, audited) and tags the mechanism PROPOSED.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

OPS3 mints no event types; it mandates that secret-lifecycle events ride existing namespaces (concrete per-event-type names → B12 registry, `[PROPOSED — not in source]`):

- **`platform.security.*`** — secret-leak signal, repeated secret-read-denial, policy-bypass attempts against secret-scoping rules (Canon §2 names this namespace for security events distinct from audit).
- **`platform.policy.*`** — OPA allow/deny on a secret-read decision.
- **`platform.audit.*`** — every rotation, revocation, and secret-access event, via the A18 adapter (ADR 0034). System of record Postgres + S3.
- **`platform.lifecycle.*`** — XR provisioning that creates a `connectionSecretRef` secret (Canon §2 lists XR/MemoryStore lifecycle here).
- `[PROPOSED — not in source]` — whether a rotation emits a dedicated `platform.security.*` vs `platform.audit.*` event is a B12 schema design point.

### 4.4 Data schemas / connection-secret contracts

- **Connection-secret shape (Canon §4, ADR 0041)** — `host`, `port`, `user`, `password`, `dbname` (or documented per-primitive equivalent; B4 owns the mapping table). OPS3 **does not vary** this; it adds: (i) the secret is encrypted at rest (envelope/KMS, §6 REQ-OPS3-05); (ii) the secret is namespace-scoped to the consuming workload; (iii) the secret is rotatable without changing its key set.
- **ESO-materialized credential secrets** (MCP services, bootstrap) — per-service/per-secret shapes are **deferred per architecture-backlog §1.11** → `[PROPOSED — not in source]` for exact keys; OPS3 governs lifecycle regardless of shape.
- **JWT signing keys** (V6-11 §4.4) — fetched from the cloud secret store via an IRSA-bound ESO ServiceAccount; the single workload-trust-root signing material. Compromise federates everywhere (V6-11 R-high), so OPS3 mandates strict scoping + rotation of this class specifically.

## 5. OSS-vs-Custom Decision

**Configure / wrap existing platform machinery — no new build.** Upstream projects already chosen and named in Canon: **ESO** (External Secrets Operator; install-owned baseline) for secret materialization; **Crossplane v2 Compositions** (B4) write connection secrets; **Keycloak** issues JWTs/holds signing keys; **AWS IRSA / Azure AD Workload Identity** (ADR 0028) federate cloud-store access; **Reloader** (A15) rolls secret-dependent workloads. The KMS/provider for envelope encryption at rest is **not chosen in source** — `[PROPOSED — not in source]` (AWS KMS is the natural EKS choice given ADR 0033, but the kind path and any provider abstraction are open) → surfaced as PROPOSED-ADR §10. No fork. OPS3 is an architecture spec; its "build" is the set of conformance gates, OPA scoping rules (authored by B16), and ESO topology requirements imposed on existing pieces. Rationale: every primitive needed already exists in Canon; the gap is a coherent lifecycle + the two open decisions (KMS, rotation cadence), which OPS3 surfaces rather than silently decides.

## 6. Functional Requirements

Numbered, testable. Grouped by lifecycle phase.

**Provenance**

- **REQ-OPS3-01:** Every secret a platform workload loads MUST originate from exactly one Canon-named source — a substrate XRD Composition (`connectionSecretRef`/`credentialsSecretRef`), an ESO `ExternalSecret` (`credentialsRef`/bootstrap), Keycloak (JWT/admin), or an IRSA / Workload Identity mint — and MUST be traceable to that source; no secret of unknown or in-image/in-CRD origin is loadable (extends ADR 0020 REQ-08, B1 REQ-07).
- **REQ-OPS3-02:** JWT signing keys, Keycloak admin credentials, and bootstrap secrets MUST be fetched from the cloud secret store via an **IRSA-bound ESO ServiceAccount** (V6-11 REQ-10); on kind the local secret backend substitutes. No signing material is committed to Git or baked into an image.

**Storage / encryption at rest**

- **REQ-OPS3-03:** Every connection secret (`connectionSecretRef`) and credential secret (`credentialsRef`/`credentialsSecretRef`) MUST be stored as a namespaced Kubernetes Secret encrypted at rest via envelope encryption backed by a KMS, on the AWS substrate; the kind substrate MAY use a local key but MUST document the reduced guarantee (parity caveat consistent with Canon §4 "capability-parity is not promised"). `[PROPOSED — not in source]` KMS provider — see §10.
- **REQ-OPS3-04:** No secret material MAY appear in Git, container images, CRD spec fields, logs, traces, or audit/event payloads (extends F4 REQ-02; A17 REQ-09). Secret-bearing fields are referenced (`*Ref`), never inlined.

**Scoping**

- **REQ-OPS3-05:** Read access to a `connectionSecretRef`/`credentialsRef`/`credentialsSecretRef` secret MUST be namespace-bound and gated by **RBAC-as-floor / OPA-as-restrictor** (ADR 0018); OPA keys the decision off `platform_namespaces` (authoritative for resource scope, V6-11 REQ-03) and MUST NOT grant read beyond the RBAC floor. A principal not authorized for a namespace MUST be denied the secret even if the consuming endpoint is reachable (mirrors A17 REQ-07 for DB assignment).
- **REQ-OPS3-06:** No connection/credential secret MAY be reused across tenants without an explicit OPA policy; `XAgentDatabase` `scope` (agent/tenant/user) MUST bound which principal resolves to which `credentialsSecretRef` (ADR 0020).
- **REQ-OPS3-07:** A claim/secret-read whose target namespace has no authorizing policy MUST be **rejected at admission** by Gatekeeper (ADR 0002) — fail-closed, not stuck — using an input shape OPS3 supplies to B16 (the B4 admission-contract pattern).

**Rotation**

- **REQ-OPS3-08:** Each secret class MUST have a defined rotation mechanism: `VirtualKey` material via the source-stated **`ttl`/reissue** (Canon §1.4); ESO-materialized secrets via ESO refresh/resync; connection secrets via Composition re-write + Reloader-driven rollout (A15); JWT signing keys via key rollover with an overlap window. A rotation MUST NOT change the secret's key set (that would be a breaking shape change, ADR 0030).
- **REQ-OPS3-09:** Rotation of any secret class MUST propagate **without request outage**: in-flight traffic MUST continue while the new material is adopted and the old is invalidated (extends F4 REQ-01 / AC-F4-01). For signing keys this MUST use an overlap window where both old and new keys validate until the old is retired.
- **REQ-OPS3-10:** A **rotation cadence policy** MUST exist per secret class as a testable target (interval per class). The concrete intervals are `[PROPOSED — not in source]` (no Canon cadence) — see §10 PROPOSED-ADR; OPS3 fixes that a policy MUST exist and be enforced, not the numbers.

**Revocation & blast-radius containment**

- **REQ-OPS3-11:** Revoking or rotating a secret MUST invalidate the prior material such that a subsequent use of the old value is **rejected** (extends F4 AC-F4-01); for `VirtualKey` this is `ttl` expiry/reissue, for ESO secrets this is removal of the `ExternalSecret` causing the workload to fail closed (A17 AC-09).
- **REQ-OPS3-12:** Missing/deleted secret material MUST cause the consuming workload to **fail closed** (deny), never fall back to an embedded or default credential (mirrors A17 AC-09).
- **REQ-OPS3-13:** Blast radius of any single secret MUST be bounded: a connection/credential secret MUST be scoped to one namespace + one substrate + (for `XAgentDatabase`) one `scope` tier, so a single leak cannot reach unrelated tenants; egress of a leaked credential MUST be bounded by the Envoy egress allowlist (ADR 0003).
- **REQ-OPS3-14:** The cluster OIDC issuer signing material and the JWT signing keys are **single trust roots** (V6-11 REQ-08); compromise federates platform-wide, so these MUST carry the strictest scoping (IRSA-bound read only), the shortest rotation cadence of any class, and MUST be backed up under OPS2/F2 for restore.
- **REQ-OPS3-15:** Every rotation, revocation, and denied secret-read MUST emit audit via the **A18 adapter** (`platform.audit.*`) and surface security-relevant denials under `platform.security.*`; no OPS3-governed component writes audit directly (ADR 0034).

## 7. Non-Functional Requirements

- **Security:** the connection-secret contract is the platform's central trust surface; OPS3 enforces no-inline-secrets (REQ-04), envelope/KMS at rest (REQ-03), and single-trust-root discipline for signing keys (REQ-14). Secret-store access is itself identity-federated (IRSA-bound ESO SA; V6-11 §7).
- **Multi-tenancy (§6.9):** all governed secrets are namespaced; `platform_namespaces` is authoritative for read scope; no cross-tenant reuse without policy (REQ-05/06). `XAgentDatabase.scope` bounds per-principal credential resolution.
- **Observability (§6.5):** rotation/revocation/denial events flow to A18 (audit) and to `platform.security.*`/`platform.policy.*`; a Grafana dashboard (delivered as a `GrafanaDashboard` XR, ADR 0021) surfaces rotation freshness + secret-read-denial rate. `[PROPOSED — not in source]` dashboard signal names.
- **Scale:** ESO refresh and Composition re-writes MUST not create unbounded secret churn; rotation cadence (REQ-10) bounds frequency. Per-agent/tenant/user secret creation rides `XAgentDatabase` and inherits its (deferred) quota/budget gating (A17 R4).
- **Versioning (ADR 0030):** a rotation that changes secret *shape* is breaking and follows B4's conversion-webhook/deprecation discipline; the ESO topology + KMS policy are versioned per-component.

## 8. Cross-Cutting Deliverable Checklist

This is a cross-cutting NFR piece; the §14.1 set is marked applicable / N/A from the standpoint of the obligations OPS3 imposes (it ships conformance gates + requirements, not a daemon).

- Helm/manifests — **applicable** — ESO `SecretStore`/`ExternalSecret` topology templates + KMS-policy manifests it requires existing pieces to satisfy; OPS3 ships the topology requirement, owning pieces (A17/B1/V6-11) ship instances.
- Per-product docs (10.5) — **applicable** (secret-class taxonomy, provenance map, rotation cadence policy, blast-radius model).
- Runbook (10.7) — **applicable** (secret-leak response, emergency rotation, signing-key rollover, KMS-unavailable) — feeds **F4**/F6.
- Backup/restore — **applicable (delegated)** — secret-store backup obligation defined here, topology owned by **OPS2/F2**.
- Alerts — **applicable** (rotation overdue per cadence, secret-read-denial spike, ESO sync failure, KMS-unavailable). `[PROPOSED — not in source]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable** (rotation freshness + denial-rate dashboard as a `GrafanaDashboard` XR).
- Headlamp plugin — **N/A** — OPS3 ships no interactive CRD surface; secret inspection rides existing A22/A9 surfaces.
- OPA/Rego integration — **applicable (contract only)** — OPS3 states the secret-read scoping + admission requirement + input shape; **B16** authors the Rego (B4 pattern).
- Audit emission (ADR 0034) — **applicable** — rotation/revocation/denial via the A18 adapter; no direct writes.
- Knative trigger flow — **applicable** — secret-leak `platform.security.*` → HolmesGPT detection flow (consistent with §6.7); `[PROPOSED — not in source]` no v1.0 mandated OPS3-specific flow.
- HolmesGPT toolset — **applicable** — a rotation-freshness / secret-provenance read tool, gated by HolmesGPT's CapabilitySet. `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (Chainsaw: secret-scoping admission-deny + connection-secret rotation reconcile; PyTest: rotation-propagation + no-secret-in-logs scans + secret-shape-invariance; Playwright: N/A — no UI).
- Tutorials & how-tos — **applicable** ("rotate a connection secret with zero downtime"; "scope a tenant's database credential").

## 9. Acceptance Criteria

Each maps to a REQ in §6.

- **AC-OPS3-01:** A provenance scan resolves every loaded secret to one Canon-named source; an injected secret of unknown origin (or one inlined in a CRD/image) is flagged and the workload that loads it fails the gate. (→ REQ-OPS3-01, -04)
- **AC-OPS3-02:** JWT signing keys + Keycloak admin creds resolve only via an IRSA-bound ESO ServiceAccount; a scan of Git/images finds none. (→ REQ-OPS3-02)
- **AC-OPS3-03:** On AWS, a `connectionSecretRef`/`credentialsRef` Secret is shown encrypted at rest via the configured KMS; on kind the reduced-guarantee local path is documented and tested as such. (→ REQ-OPS3-03)
- **AC-OPS3-04:** A scan of Git, container logs, traces, and audit/event payloads finds zero secret material along the ESO→workload and Composition→workload paths. (→ REQ-OPS3-04)
- **AC-OPS3-05:** A principal not authorized for namespace N (per `platform_namespaces`) is denied read of N's connection/credential secret by OPA, even with the endpoint reachable; OPA grants nothing beyond the RBAC floor. (→ REQ-OPS3-05)
- **AC-OPS3-06:** A cross-tenant secret-read with no authorizing policy is denied; an `XAgentDatabase` principal resolves only to the `credentialsSecretRef` permitted by its `scope`. (→ REQ-OPS3-06)
- **AC-OPS3-07:** A secret-read/claim targeting a namespace with no authorizing policy is rejected at Gatekeeper admission (fail-closed), not left stuck. (→ REQ-OPS3-07)
- **AC-OPS3-08:** Each secret class rotates by its defined mechanism (`VirtualKey` `ttl`/reissue; ESO resync; Composition re-write + Reloader; signing-key rollover) and the rotated secret retains its key set. (→ REQ-OPS3-08)
- **AC-OPS3-09:** During rotation of each class, in-flight traffic sees no outage; for signing keys, both old and new validate during the overlap window, then the old is retired. (→ REQ-OPS3-09)
- **AC-OPS3-10:** A rotation-cadence policy exists per secret class with an enforced interval target; a secret past its cadence raises the overdue alert. (Intervals `[PROPOSED]`.) (→ REQ-OPS3-10)
- **AC-OPS3-11:** After rotation/revocation, a use of the prior secret value is rejected; removing an `ExternalSecret` makes its `credentialsRef` secret disappear and the consumer fail closed. (→ REQ-OPS3-11)
- **AC-OPS3-12:** Deleting a secret causes the consumer to deny rather than fall back to any embedded/default credential. (→ REQ-OPS3-12)
- **AC-OPS3-13:** A leaked connection/credential secret is shown bounded to one namespace + substrate + `scope`; an exfiltration attempt is blocked by the Envoy egress allowlist. (→ REQ-OPS3-13)
- **AC-OPS3-14:** Signing-key/cluster-OIDC material carries IRSA-bound read-only scope, the shortest cadence of any class, and is present in the OPS2/F2 backup set. (→ REQ-OPS3-14)
- **AC-OPS3-15:** Each rotation/revocation/denied-read produces a `platform.audit.*` record via the A18 adapter and a `platform.security.*` signal where security-relevant; no OPS3-governed component writes audit directly. (→ REQ-OPS3-15)

## 10. Risks & Open Questions

**PROPOSED-ADR TITLES (genuinely open decisions — surfaced for the user, NOT auto-decided):**

- **PROPOSED-ADR: "KMS / envelope-encryption provider for secrets at rest"** — *rationale:* source names ESO + the connection-secret contract but never fixes a KMS for envelope encryption of Secrets at rest; AWS KMS is the natural EKS choice (ADR 0033) but the kind-substrate equivalent and whether a provider-abstraction is needed are open. Blast radius: **high** (touches every secret class).
- **PROPOSED-ADR: "Secret/key rotation cadence policy (per-class intervals + overlap windows)"** — *rationale:* no Canon cadence exists; REQ-OPS3-10 mandates a policy but the per-class intervals (connection secrets, `VirtualKey` `ttl`, signing keys, MCP creds) and the signing-key overlap window are a genuine operational decision. Blast radius: **high** (signing-key cadence is the platform trust root).
- **PROPOSED-ADR: "Where rotation metadata lives (ESO config vs. CRD field vs. external policy)"** — *rationale:* the frozen XRD/CRD schemas carry no last-rotated/cadence/KMS-ref field; OPS3 proposes keeping metadata in ESO config + KMS policy to avoid mutating Canon schemas, but this is an open design choice that should be ratified before any field is added. Blast radius: **med**.
- **PROPOSED-ADR: "User-credential OAuth token lifecycle (persistence, refresh, revocation in LiteLLM)"** — *rationale:* ADR 0020 / A17 OQ2 defer whether user-credential tokens persist across sessions and how they are revoked; this is a secret-lifecycle decision OPS3 governs but cannot decide. Blast radius: **med**.

**Risks:**

- **R1 (high):** The connection-secret contract is central; a rotation that accidentally changes the key set breaks every consumer (A18/B11/A17/B19). Mitigation: REQ-OPS3-08 forbids key-set change; treat as breaking-shape change under B4/ADR 0030. Blast radius: high.
- **R2 (high):** The JWT/cluster-OIDC signing material is a single trust root (V6-11 R-high); its compromise federates everywhere. Mitigation: REQ-OPS3-14 (strictest scope + shortest cadence + backup). Blast radius: platform-wide.
- **R3 (med):** Per-service MCP secret shapes, OAuth scopes, and revocation are deferred (architecture-backlog §1.11; ADR 0020 OQ); OPS3 governs lifecycle but provisional shapes risk churn. `[PROPOSED]`. Blast radius: med.
- **R4 (med):** kind-vs-AWS KMS asymmetry means a developer who tested encryption-at-rest on kind may assume a guarantee that only the AWS KMS path provides (parity caveat). Mitigation: REQ-OPS3-03 documents the reduced kind guarantee. Blast radius: med.
- **R5 (med):** No Canon field carries rotation metadata; if a future piece adds one to a frozen XRD it breaks the schema-freeze discipline. Mitigation: PROPOSED-ADR above keeps metadata out of CRD schemas. Blast radius: med.
- **R6 (low):** Secret-store backup is delegated to OPS2/F2; if those slip, REQ-OPS3-14's restore obligation is unverified. Mitigation: cross-link + dependency in PLAN-OPS3 §3.

**Biggest open question:** the **rotation cadence policy** — specifically the signing-key / cluster-OIDC-root interval and overlap window — because that material is the single platform trust root, yet source fixes no cadence and OPS3 must not silently decide it.

## 11. References

- `_meta/pending-operational-nfr-layer.md` (OPS3 scope: secret/key lifecycle, rotation, KMS/envelope, ESO topology, virtual-key + credential handling, IRSA/Workload Identity trust roots; relates A18, B1, V6-11, ADR 0028/0029, F4).
- `_meta/interface-contract.md` §1.4 (`MCPServer.credentialsRef`, `VirtualKey.ttl`), §1.6 (`connectionSecretRef`, `credentialsSecretRef`, `XAgentDatabase`, `TenantOnboarding`), §4 (connection-secret contract — central invariant), §5 (audit adapter / system of record).
- `_meta/glossary.md` ("connection secret", "ESO", "IRSA / Workload Identity", "Platform JWT", "RBAC-as-floor / OPA-as-restrictor").
- ADR 0041 (connection-secret contract / substrate abstraction), ADR 0020 (MCP-service secrets via ESO; credential modes; deferrals), ADR 0028 (identity federation chain / trust roots / IRSA-bound ESO bootstrap), ADR 0029 (JWT claim schema / `VirtualKey` binding), ADR 0034 (audit emission/system of record), ADR 0030 (versioning), ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0002 (Gatekeeper admission), ADR 0003 (Envoy egress).
- Component pieces: B4 (`connectionSecretRef` owner), A17 (`credentialsRef`/ESO pattern), A18 (audit adapter), B13 (`VirtualKey` reconcile), A1 (OAuth broker), B1 (proxy secrets via ESO), A6 (sandbox/egress), A21 (tenant identity). Views: V6-11 (identity federation / signing-key sourcing), V6-06 (security/policy).
- Sibling NFR pieces: OPS2 (disaster recovery — secret-store backup), and F4 (security review — rotation/secret-handling drills that verify OPS3 targets).

# SPEC A12 — OpenAPI→MCP converter

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [] · adrs: [0020, 0044, 0013, 0030, 0034, 0031] · views: [6.8]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A12 provides the **OpenAPI→MCP converter**: it generates **synthetic MCP servers** from OpenAPI
specs, registered behind LiteLLM (architecture-overview.md §5 component table, §6.8). This lets the
platform turn an existing HTTP API described by an OpenAPI document into an approved, governed MCP
capability without hand-authoring a bespoke MCP server — "compose a synthetic MCP server from an
OpenAPI spec" is a first-class tutorial (§10.1) and use case (§5: "Tools, MCP, A2A — including
synthetic MCP from OpenAPI").

The conversion is materialized as a **`SyntheticMCPServer` Crossplane XR** (§6.12) — `openApiSpecRef`,
`authConfigRef`, `mcpServerRef` (back-link) — reconciled by Crossplane (B4); the resulting synthetic
server is registered as an `MCPServer` CRD (B13 → LiteLLM) so it inherits the platform's standard
approval, OPA gating, audit tagging, and Headlamp inspection like any other approved MCP server. No
production-grade OpenAPI→MCP converter exists out of the box (§9), so A12 is **custom hardening on
top of an OSS converter**.

## 2. Scope

### 2.1 In scope
- The converter engine/service that reads an OpenAPI spec + an auth config and emits a synthetic
  MCP server registered behind LiteLLM.
- The **install + per-server OpenAPI specs and auth configs** packaging (§5: "Install + per-server
  OpenAPI specs and auth configs (configuration-heavy)").
- Honoring the `SyntheticMCPServer` XR contract: consume `openApiSpecRef` and `authConfigRef`,
  produce/maintain the `mcpServerRef` back-link to the registered `MCPServer`.
- Custom hardening on top of an OSS converter (§9): input validation, auth handling, error/edge
  handling that a raw OSS converter lacks for production use.
- `LogLevel` honoring (ADR 0035); audit emission of conversion/registration events (ADR 0034).
- Standard §14.1 Workstream-A deliverable set (see §8).

### 2.2 Out of scope (and where it lives instead)
- The **`SyntheticMCPServer` XR / Composition** definition itself — **Crossplane Compositions
  (B4)**; A12 supplies the converter the composition drives and honors its field contract.
- The **`MCPServer` CRD reconciliation into LiteLLM** — **kopf operator (B13)**; A12 produces the
  synthetic server, B13 registers it.
- MCP brokering, OAuth flows, health signals at the gateway — **LiteLLM (A1)** (§6.1).
- OPA admission/runtime gating + audit-on-MCP-call — **OPA (A7)** / LiteLLM callbacks (§6.6);
  A12 integrates with them but does not own them.
- ESO secret plumbing for the synthetic server's credentials — **ESO (A17 pattern)**; A12 consumes
  the secret reference (`authConfigRef`).
- The specific initial MCP services (GitHub, Drive, Context7, etc.) — **A17 (ADR 0020)**; A12 is the
  generic converter, not the curated service set.

## 3. Context & Dependencies

**Upstream consumed:** none declared in piece-index (`consumer` wave; empty upstream/downstream).
Functionally A12 binds to B4 (the `SyntheticMCPServer` XR), B13 (`MCPServer` registration), A1
(LiteLLM registry), and A7 (OPA) — these are foundation/W1 pieces it composes against, mocked per
the §10 documentation-before-implementation / mock-out process where not yet landed.

**Downstream consumers:** none declared in piece-index. (Synthetic MCP servers it produces are
consumed by Platform Agents via CapabilitySet inclusion — a runtime relationship, not a build edge.)

**ADR decisions honored:**
- **ADR 0020** — synthetic MCP from OpenAPI is part of the v1.0 capability story; per-service
  details (auth flows, scopes, secret shapes) deferred per backlog §1.11. A12 is generic; A17 owns
  the curated set.
- **ADR 0044** — `SyntheticMCPServer` is an instance of the composition pattern; A12 honors the XR
  contract (`openApiSpecRef`, `authConfigRef`, `mcpServerRef`).
- **ADR 0013** — the produced synthetic server is an approved capability via `MCPServer` CRD; it
  rides the standard CapabilitySet + OPA governance, no bypass path.
- **ADR 0034** — conversion/registration events emit via the audit adapter.
- **ADR 0031** — A12 events fall under `platform.capability.*` (a new MCP server entering the
  registry) and `platform.audit.*`; per-event names deferred to B12.
- **ADR 0030** — A12's converter service exposes a URL-path-versioned HTTP surface (`/v1/...`,
  §3.3); the `SyntheticMCPServer` XR follows CRD/XRD versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- A12 **produces/maintains** (but does not define) the `SyntheticMCPServer` (Crossplane XR, owner
  B4) — `openApiSpecRef`, `authConfigRef`, `mcpServerRef` (back-link). A12 fills `mcpServerRef`
  once the synthetic server is registered.
- A12 **produces** an `MCPServer` (kopf/B13-owned) — `endpoint`, `authMode` (system/user-cred),
  `credentialsRef`, `tags`, `scopes`, `visibility` — for the synthetic server (registered by B13;
  A12 supplies the synthesized endpoint/spec). A12 invents no fields beyond this Canon set.
- Consumes `LogLevel` (ADR 0035) and delivers a `GrafanaDashboard` XR (B4-owned XRD).

### 4.2 APIs / SDK surfaces
- **Converter service HTTP API** (A12-owned) — URL-path-versioned (`/v1/...`, ADR 0030 §3.3):
  accept an OpenAPI spec reference + auth config and synthesize/register an MCP server.
  `[PROPOSED — not in source: exact converter API shape, supported OpenAPI versions, and
  tool-mapping rules are design-time per backlog §1.11.]`
- Registers the synthesized server into the LiteLLM registry **via B13** (not directly) — A12 does
  not write the LiteLLM admin API itself (§6.8: kopf operator is the reconciler).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.capability.*` when a synthetic MCP server is added/updated/removed
  (mirrors `platform.capability.changed`, ADR 0013); `platform.audit.*` for conversion actions.
  `[PROPOSED — not in source: specific event names deferred to B12.]`
- **Consumed:** none required for core function.

### 4.4 Data schemas / connection-secret contracts
- The synthetic server's credentials are referenced via `authConfigRef` / `credentialsRef`
  (ESO-managed secret); A12 consumes the reference, not the raw secret.
- No platform-owned data store; A12 is a stateless conversion service. Any persisted spec/config
  lives behind references in Git / object storage `[PROPOSED — not in source: storage of specs is
  design-time]`.

## 5. OSS-vs-Custom Decision
- **Named project:** an OSS OpenAPI→MCP converter (unnamed in source — §9: "None production-grade
  out of the box") `[PROPOSED — not in source: specific OSS converter is design-time]`.
- **Mode:** **wrap + harden** — custom hardening on top of an OSS converter (§9 OSS-gap table),
  plus configuration-heavy install (per-server specs + auth configs).
- **Version:** pin the chosen OSS converter version `[PROPOSED — not in source]`.
- **ADR linkage:** ADR 0020 (synthetic MCP in v1.0), ADR 0044 (`SyntheticMCPServer` composition),
  ADR 0013 (approved capability via `MCPServer`).
- **Rationale:** no production-grade converter exists, so the platform hardens an OSS base rather
  than building from scratch or buying — consistent with the §9 stance.

## 6. Functional Requirements
- **REQ-A12-01:** A12 SHALL synthesize an MCP server from a supplied OpenAPI spec + auth config and
  cause it to be registered behind LiteLLM (via B13), inheriting standard approval/OPA/audit.
- **REQ-A12-02:** A12 SHALL honor the `SyntheticMCPServer` XR contract — consume `openApiSpecRef`
  and `authConfigRef`, and set `mcpServerRef` to back-link the registered `MCPServer`.
- **REQ-A12-03:** A12 SHALL register synthetic servers **through B13's `MCPServer` reconciliation**,
  not by writing the LiteLLM admin API directly (§6.8 no-bypass).
- **REQ-A12-04:** A12 SHALL provide custom hardening over the OSS converter: validate the input
  OpenAPI spec and fail closed (no partial/unsafe synthetic server) on malformed input.
- **REQ-A12-05:** A12 SHALL consume credentials by reference (`authConfigRef`/ESO), never embedding
  raw secrets in the synthetic server definition.
- **REQ-A12-06:** A12 SHALL emit `platform.capability.*` events on synthetic-server add/update/
  remove and audit conversion actions via the adapter (ADR 0034).
- **REQ-A12-07:** A12 SHALL honor a `LogLevel` CR for converter log/trace granularity (ADR 0035).
- **REQ-A12-08:** A12 SHALL expose its converter HTTP surface with URL-path versioning (`/v1/...`).
- **REQ-A12-09:** A12 SHALL deliver a per-component Grafana dashboard (`GrafanaDashboard` XR) and
  alert rules for conversion/registration failure conditions.
- **REQ-A12-10:** A12 SHALL provide the "compose a synthetic MCP server from an OpenAPI spec"
  tutorial path end-to-end (§10.1).

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** synthetic servers are namespaced `MCPServer` CRDs governed
  by CapabilitySet + OPA (no bypass, ADR 0013); credentials via ESO; RBAC floor + OPA restriction
  (ADR 0018). Malformed/hostile specs must not yield an unsafe capability.
- **Observability (§6.5):** converter emits OTel through the standard collector; conversion spans
  carry `trace_id`.
- **Scale:** converter handles realistic OpenAPI spec sizes; concrete limits deferred to F5.
- **Versioning (ADR 0030):** URL-path versioning on the converter API; `SyntheticMCPServer` XR
  follows XRD versioning with conversion webhooks.

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable** (converter service install; per-server specs/auth configs).
- Per-product docs (10.5) — **applicable**.
- Runbook (10.7) + backup/restore — **applicable** (backup/restore N/A — stateless converter;
  source specs/configs live in Git).
- Alerts — **applicable** (conversion/registration failures).
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **N/A as A12-owned editor** — `SyntheticMCPServer` / `MCPServer` are surfaced
  by the capability inspector (B5) and the `MCPServer` editor (A22); A12 ships none of its own
  (not in ADR 0039's initial editor set as an A12 deliverable).
- OPA/Rego integration — **applicable** (admission on `SyntheticMCPServer`/synthetic `MCPServer`;
  runtime gating rides the standard MCP path).
- Audit emission (ADR 0034) — **applicable** (conversion/registration events).
- Knative trigger flow — **applicable** (`platform.capability.*` on synthetic-server changes).
- HolmesGPT toolset — **N/A / minimal** — A12 is a conversion utility; no agent-callable tool.
  `[PROPOSED — minimal/none]`
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable**.
- Tutorials & how-tos — **applicable** ("compose a synthetic MCP server from an OpenAPI spec").

## 9. Acceptance Criteria
- **AC-A12-01 (REQ-A12-01):** Given a valid OpenAPI spec + auth config, a synthetic MCP server is
  registered behind LiteLLM and is reachable from an agent whose CapabilitySet includes it. *(PyTest
  + Chainsaw, B13/LiteLLM mocked until landed)*
- **AC-A12-02 (REQ-A12-02):** A `SyntheticMCPServer` XR with `openApiSpecRef`+`authConfigRef`
  reconciles to a synthetic server and its `mcpServerRef` back-link is populated. *(Chainsaw)*
- **AC-A12-03 (REQ-A12-03):** Registration occurs via a created `MCPServer` CR (B13 path); no direct
  LiteLLM admin-API write is performed by A12. *(PyTest — assert no direct admin call)*
- **AC-A12-04 (REQ-A12-04):** A malformed OpenAPI spec is rejected with an error and yields **no**
  synthetic server (fail-closed). *(PyTest)*
- **AC-A12-05 (REQ-A12-05):** The synthetic server definition references credentials by
  `authConfigRef`; no raw secret material appears in the CR/spec. *(PyTest)*
- **AC-A12-06 (REQ-A12-06):** Adding a synthetic server emits a `platform.capability.*` event and an
  audit record via the adapter. *(PyTest)*
- **AC-A12-07 (REQ-A12-07):** A `LogLevel` CR raises/lowers converter verbosity. *(Chainsaw + PyTest)*
- **AC-A12-08 (REQ-A12-08):** The converter API is reachable under `/v1/...`. *(PyTest)*
- **AC-A12-09 (REQ-A12-09):** The `GrafanaDashboard` XR renders; a forced conversion failure fires
  the alert. *(Chainsaw + PyTest)*
- **AC-A12-10 (REQ-A12-10):** Following the tutorial produces a working synthetic MCP server callable
  by a test agent. *(PyTest end-to-end)*

## 10. Risks & Open Questions
- **R1 (high):** No production-grade OSS converter exists (§9) — the chosen base and the breadth of
  OpenAPI it covers are `[PROPOSED — not in source]`; hardening effort is uncertain. *Open: select
  and evaluate the OSS base early.*
- **R2 (med):** Tool-mapping semantics (how OpenAPI operations map to MCP tools), auth-flow shapes,
  and supported OpenAPI versions are deferred (backlog §1.11) → `[PROPOSED]`.
- **R3 (med):** A12 is `consumer`-wave with empty build edges in the CSV but has real functional
  dependencies on B4 (XR), B13 (registration), A1 (LiteLLM), A7 (OPA). Build sequencing relies on
  mock-out; mocks that drift from real B13/B4 contracts are a rework risk. Blast radius med.
- **R4 (low):** Storage of source specs/auth configs is design-time `[PROPOSED]`.

## 11. References
- architecture-overview.md §5 (OpenAPI→MCP converter row ~line 148; SyntheticMCPServer in B4 row);
  §6.8 (capability registries — approved capability, no bypass, lines ~632–685); §6.12
  (`SyntheticMCPServer` XR, line ~968); §9 (OSS-gap table, converter row ~1418); §10.1 (tutorial,
  line ~1444); §14.1 (A12 row, line ~1678).
- ADR 0020 (initial MCP services / synthetic MCP); ADR 0044 (`SyntheticMCPServer` composition);
  ADR 0013 (approved capability via `MCPServer`); ADR 0034 (audit); ADR 0031 (CloudEvents);
  ADR 0030 (versioning).
- View V6-08 (Capability registries and approved primitives).
- Related pieces: B4 (`SyntheticMCPServer` XR/Composition), B13 (`MCPServer` reconciliation),
  A1 (LiteLLM), A7 (OPA), A17 (curated MCP services), A22 (`MCPServer` editor), B5 (capability
  inspector).

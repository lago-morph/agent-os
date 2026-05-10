# Future Enhancements

Capabilities consciously deferred past v1.0. These are not "alternatives we may revisit" (that's the role of `architecture-backlog.md` section 3) — they are committed-future work, scoped out of v1.0 but acknowledged as needed for a fully production-grade platform.

Each entry includes (a) the v1.0 stance, (b) what the future enhancement adds, and (c) cross-references to related backlog and ADR-candidate items where relevant.

## 1. Redundancy and degraded operations

### v1.0 stance

The architecture deploys each component in its single-instance default, with one carve-out: ADR 0035 raised LiteLLM to `replicas >= 2` (with readiness/preStop/`terminationGracePeriodSeconds` tuned for SSE-safe rolling restarts), so the two-replica restart-safety baseline is in v1.0; broader HA-clustering for LiteLLM remains future. Failure of a chokepoint (OPA, the kopf operator, the NATS broker, Postgres) produces a hard error; there is no graceful degradation path defined, no fail-open vs fail-closed policy committed per chokepoint, and no cross-AZ or cross-region replication beyond what the underlying managed services provide by default. (Note: per ADR 0034, audit ingestion does not depend on OpenSearch — Postgres + S3 are the system of record, OpenSearch is advisory fanout and rebuildable, so OpenSearch availability affects search/dashboard freshness rather than audit durability.) Workstream F (Production readiness) includes a one-time DR drill (F2), but ongoing redundancy is not in scope.

### Future enhancement

- **HA topologies for chokepoints**: LiteLLM gateway clustering beyond the two-replica restart-safety baseline established by ADR 0035, OPA replicas with consistent bundle distribution, kopf operator leader election, NATS JetStream clustering with explicit replication factor per stream, OpenSearch master / data tier separation with replica counts.
- **Per-chokepoint fail-open vs fail-closed policy**, surfaced as configuration: e.g., when OPA is unavailable, do callbacks fail closed (deny all) or fail open with audit-only? Per chokepoint, the call is not currently made.
- **Graceful degradation paths**: define what the platform does when Langfuse is down (continue serving, queue traces, lose traces?), when OpenSearch is down (audit ingestion continues per ADR 0034 since Postgres + S3 are the system of record — the open question is the user-visible behavior of search and dashboards while OpenSearch is unavailable or rebuilding, not audit durability), when NATS is down (degraded eventing or hard fail?). LiteLLM rolling-restart drain semantics (readiness drop before SIGTERM, preStop sleep, raised termination grace period) are already specified by ADR 0035; the deferred work is the broader fail-open vs fail-closed posture per chokepoint.
- **Cross-region or cross-AZ replication patterns** for stateful components.
- **Backup-restore automation and DR exercise schedule**, making the v1.0 one-time drill (F2) into an ongoing cadence with measured RTO / RPO.
- **Multi-cluster federation** when redundancy spans clusters (cross-references backlog 3.13).

## 2. Continuous container security

### v1.0 stance

The CI/CD pipeline scans images at build time using Trivy or Grype and blocks merge on critical CVEs. The pipeline is designed security-first (scoped tokens, SHA-pinned actions, OIDC federation, separate runners — see overview section 8). Image signing, runtime verification, and continuous re-scanning are not in v1.0.

### Future enhancement

- **Admission-time enforcement of scan artifacts**: an image is admitted to the cluster only if it is associated with a recent passing scan. Implemented as an OPA Gatekeeper constraint plus a scan-artifact registry. Drift between scan and admission is a security concern; v1.0 trusts the pipeline produced an acceptable image at build time and doesn't re-verify at admission.
- **Continuous re-scanning of running images**: re-scan running images against new CVE data on a schedule (daily or on CVE feed update). Alert when a previously-clean image has a new critical CVE. Optionally trigger HolmesGPT for triage.
- **Image signing and signature verification at admission**: Sigstore / Cosign or Connaisseur. Cross-references backlog 3.9 (image signature verification was lost when we chose OPA over Kyverno).
- **SBOM generation and SBOM-based policy decisions**: emit SBOMs at build, store with the image, allow admission-time policy on SBOM contents (license, package source, etc.).
- **Continuous scanning extended to non-image artifacts**: skill artifacts, OPA bundles, MCP server containers, Crossplane Composition functions.

## 3. Operations metrics and SLO management

### v1.0 stance

The architecture commits to observability infrastructure (Tempo, Mimir, Loki, Langfuse) and per-component dashboards plus alert rules for failure conditions (see overview section 6.5 and section 11). It does **not** commit to SLO definitions, error budgets, reliability metrics, or any SRE-style operational measurement framework. The collection paths exist; the policy and dashboards on top of them are deferred.

### Future enhancement

- **Identification of platform-level SLIs and SLOs**: latency targets for the gateway, sandbox cold-start budgets, audit ingestion lag, end-to-end agent-request availability. Per-component SLOs as a deliverable from each Workstream A component (currently removed from standard deliverables).
- **Error budget policy and burn-rate alerting**: how budget burn translates into action — slowdown, freeze on changes, escalation.
- **Reliability dashboards**: daily / weekly summary of platform reliability against SLOs.
- **Automated SLO breach response via HolmesGPT**: alerts route through Knative Eventing as CloudEvents (under the `platform.observability.*` namespace introduced in section 6.7), trigger a Knative Trigger filtering for diagnostic-relevant alerts, dispatch to HolmesGPT, which uses its component toolsets to investigate and either propose a remediation (auto-PR / suggestion card) or, where OPA permits, execute it autonomously. This is a natural extension of the AlertManager → HolmesGPT flow already shipping in v1.0 (overview section 6.7), but specific SLO-driven flows and the trust model for autonomous remediation are deferred.
- **Cost-effectiveness metrics tied to reliability targets**: dollars spent per nine of availability, cost of error-budget consumption.

## 4. Chat-platform drop-in integrations beyond Mattermost

### v1.0 stance

ADR 0036 ships Mattermost (Team Edition, free) via Knative Eventing using a generic chat-platform driver pattern: a Knative Trigger filters events, an OPA-routed adapter applies policy, and a chat-driver component renders to the platform-specific protocol. The driver pattern is the integration boundary; only the Mattermost driver is delivered in v1.0.

### Future enhancement

- **Microsoft Teams driver**: drop-in component implementing the same chat-driver shape, talking the Teams Bot Framework / Graph API on the platform side.
- **Slack driver**: drop-in component implementing the same chat-driver shape, talking Slack's Web API / Events API / Socket Mode on the platform side.
- **Custom enterprise chat platform drivers**: the driver pattern accommodates bespoke / on-prem chat platforms; additions are component-implementation work, not architecture work.

The Knative Trigger / OPA-routed adapter / chat-driver shape is architecturally supported today. These additions are component-implementation work — no new architectural decisions required.

## 5. Periodic / scheduled platform-health test execution

### v1.0 stance

ADR 0011 keeps testing CLI-orchestrated; the CLI is invoked from CI for unit and integration runs. Periodic checks (smoke tests, drift detection, capacity baselining) today run via scheduled CI invocation of the same CLI — i.e., a CI cron triggers the CLI, the CLI runs the test, results flow back through CI reporting.

### Future enhancement

- **Declarative `HealthCheck` / `ScheduledTest` XRDs** reconciled by an in-cluster runner, for tests intended to run periodically rather than per-change. Removes the dependency on CI for the schedule and surfaces test status as cluster-native resources alongside the components they exercise.
- Cross-references backlog § 3.7 (Test CRDs more generally — declarative test execution as a felt need).

## 6. Documentation-before-implementation discipline

This is a **process commitment**, not a deferred capability. Recorded here so it isn't forgotten as the component-deliverable templates are formalized.

- **Documentation for a component is constructed BEFORE implementation.** The documentation serves as the source of test cases for acceptance — the spec defines the acceptance contract, the implementation is verified against the documented contract.
- **For components in a chain of unimplemented dependencies**, the documentation includes mock-out instructions for those dependencies so the component is implementable and testable independently of when its dependencies land.
- **Specifications and design output concrete test expectations** that are reviewed before implementation begins. Review cycles operate on the documented contract, not on post-hoc justification of code already written.
- **To be formalized in the component-deliverable templates** (Workstream A, B). The standard deliverable list for a component includes the documentation artifact as a gate before implementation work starts.

This is a process item, not a deferred capability.

## 7. Commercial web-scraping fallback

### v1.0 stance

ADR 0020 ships generic in-cluster web-search and web-scrape MCP servers. IP-block risk (target sites blocking the cluster's egress IP) is **accepted** for v1.0 — the in-cluster generic scrapers are sufficient for expected workload, and the cost / complexity of a commercial scraper is not justified pre-deployment.

### Future enhancement

- **Commercial-grade scraper MCP server** (Firecrawl, a Google Search API equivalent, or similar) added when IP-blocking becomes operationally painful. Same MCP-server shape as the generic scraper; selection between them is a routing / fallback decision.
- **Budget enforcement** on the commercial path — commercial scrapers are metered and a runaway agent could burn budget quickly. Cross-references ADR 0020 budget-deferral notes (commercial scraper budget enforcement is part of the deferred budgeting story, not v1.0).

## 8. LiteLLM hot-reload (upstream contribution)

### v1.0 stance

ADR 0035 accepts **rolling-restart-as-staged-restart** for LiteLLM. LiteLLM has no built-in hot-reload of `config.yaml` or log level, so changes to gateway routing, budget configuration, or log verbosity require pod replacement. v1.0 treats a rolling restart as the supported reload mechanism.

### Future enhancement

- **SIGHUP-style config reload contributed to (or adopted from) LiteLLM upstream**, so log-level / budget / route changes don't require pod replacement. Improves diagnostic agility — turning up log verbosity on a misbehaving gateway shouldn't require a restart that perturbs the very behavior under investigation.
- If upstream lands a reload mechanism first, adopt it; otherwise contribute the patch. Either path resolves the v1.0 limitation.

## 9. GitOps progressive-delivery promotion (Kargo)

**Resolved in v1.0 — see ADR 0040.** Kargo lands as the typed promotion pipeline for dev → staging → prod with verification gates. Tombstone retained for traceability against earlier deferral.

## 10. Multi-cluster Kargo promotion across geographic regions / availability zones

### v1.0 stance

ADR 0026 commits to an independent-cluster topology, with a single region per environment. Kargo (ADR 0040) promotes within that single-region-per-environment shape: dev → staging → prod, each environment one cluster — this is the target end-state of ADR 0040's phased trajectory, not the v1.0 initial deployment (v1.0 starts with one Stage; additional Stages are added incrementally).

### Future enhancement

- **Cross-region / cross-AZ Kargo promotion** for redundancy or geographic distribution: extend the Kargo pipeline to promote across regional cluster sets, with verification gates appropriate to the multi-region case (per-region health checks, staged regional rollout, region-aware rollback).
- Cross-references backlog § 3.13 (multi-cluster federation trigger). The trigger to revisit is the same: a use case requiring cross-cluster awareness, redundancy, or shared resources.

## 11. Unifying `OidcRoleMapping` CRD

### v1.0 stance

Per-service OIDC role-mapping configuration (Grafana `grafana.ini`, OpenSearch `roles_mapping.yml`, ArgoCD, LiteLLM, LibreChat, Mattermost, Langfuse) lives in each service's native config in Git. These are not platform CRDs and therefore fall outside ADR 0039's editor scope (which targets platform CRDs); edits flow through Git/PR with schema-aware validation and are promoted via Kargo per ADR 0040. Each service speaks its own config schema; mappings are maintained N times.

### Future enhancement

- **Platform-level `OidcRoleMapping` CRD** that declares "this platform claim → that service role" once, with a controller that translates to each service's native config. Reduces N service-specific editors to one declarative surface.
- **Trigger**: operational pain from drift across services — when N service mappings get out of sync more than once, the unification cost is justified. Until then, the per-service editor approach is the lower-overhead path.

## 12. Additional substrate Compositions (AKS, GCP, on-prem variants)

### v1.0 stance

ADR 0041 ships kind + AWS Compositions. The XRD shape is substrate-agnostic; the Compositions are substrate-specific.

### Future enhancement

- **One Composition per XRD per additional substrate** (AKS, GCP / GKE, on-prem variants such as vSphere or bare-metal). The XRD shape itself is unchanged — adding a substrate is implementation work, not architecture work.
- Architectural hooks already exist (XRD/Composition split per ADR 0041). Captured here for visibility so the work isn't mistaken for new architectural decisions when it comes up.

## 13. Cross-layer policy-conflict analysis and policy drift detection

### v1.0 stance

ADR 0038 ships the per-layer policy simulator: given a hypothetical request as input, it shows the per-layer decisions (Kubernetes RBAC, OPA Gatekeeper admission, OPA gateway / agent-runtime authorization, etc.) that the platform would produce for that single request. This is reactive — it answers "what happens to this specific request?" — and is the v1.0 commitment.

### Future enhancement

- **Proactive cross-layer conflict analysis**: tooling that scans the complete OPA bundle plus the RBAC binding set plus Gatekeeper constraints together, surfacing conflicting rules, unreachable rules, and rules that one layer permits while another silently denies. ADR 0038 explicitly defers this.
- **Policy drift detection**: continuous comparison between declared policy (the bundle / RBAC / constraints in Git) and the policy actually being evaluated at runtime, flagging divergence (e.g., a stale bundle on a replica, a Gatekeeper constraint that failed to install, an RBAC binding edited out-of-band). Also explicitly deferred by ADR 0038.
- Both items extend ADR 0038's per-layer simulator into a whole-bundle / whole-system analysis surface; the simulator's data model is reusable, but the analysis algorithms and drift-comparison machinery are net-new work.

## 14. Other deferred items (cross-references)

These are tracked elsewhere but fit the "future enhancement" framing. Listed here for visibility:

- **Multi-cluster federation** — backlog 3.13. Not in v1.0; each cluster is an independent install. Trigger to revisit: a use case requiring cross-cluster awareness or shared resources.
- **Multi-CI support beyond GitHub Actions** — backlog 3.14. v1.0 ships GitHub Actions only.
- **Adding SDKs beyond LangGraph and Langchain Deep Agents** — backlog 3.15. Multi-SDK harness shape is preserved in v1.0; LangGraph (supported low-level SDK) and Langchain Deep Agents (opinionated default) ship per ADR 0019.
- **Power-user personal agents in LibreChat** — backlog 3.12. LibreChat is locked down in v1.0.
- **Allure / ReportPortal for test reporting** — backlog 3.4. Not added in v1.0; CI output + Grafana metrics cover MVP.
- **SPIFFE identity** — backlog 3.5. Not needed for v1.0's independent-cluster model.
- **Knowledge base interactive UI beyond LibreChat** — backlog 3.10. LibreChat plus the Interactive Access Agent is the v1.0 access path.
- **HolmesGPT autonomous remediation expansion** — backlog 3.11. v1.0 starts with read-only diagnostics + auto-PR / suggestion cards; autonomous narrow-scope remediations come later as trust is established.
- **M-of-N approvers and complex approval routing** — backlog 3.3. The generalized approval system in v1.0 is "request a level, OPA may elevate, anyone with that level decides"; richer routing is added on demand.
- **Crossplane Operation type for day-2 patterns** — backlog 3.6. v1.0 is ad-hoc.
- **Custom load-testing harness** — backlog 3.8. v1.0 stress probes via Chainsaw / Playwright concurrent invocation are sufficient until proven otherwise.
- **Test CRDs** — backlog 3.7. v1.0 uses CLI orchestration; CRDs added only if declarative test execution becomes a felt need.

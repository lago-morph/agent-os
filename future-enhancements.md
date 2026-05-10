# Future Enhancements

Capabilities consciously deferred past v1.0. These are not "alternatives we may revisit" (that's the role of `architecture-backlog.md` section 3) — they are committed-future work, scoped out of v1.0 but acknowledged as needed for a fully production-grade platform.

Each entry includes (a) the v1.0 stance, (b) what the future enhancement adds, and (c) cross-references to related backlog and ADR-candidate items where relevant.

## 1. Redundancy and degraded operations

### v1.0 stance

The architecture deploys each component in its single-instance default. Failure of a chokepoint (LiteLLM gateway, OPA, the kopf operator, the NATS broker, OpenSearch, Postgres) produces a hard error; there is no graceful degradation path defined, no fail-open vs fail-closed policy committed per chokepoint, and no cross-AZ or cross-region replication beyond what the underlying managed services provide by default. Workstream F (Production readiness) includes a one-time DR drill (F2), but ongoing redundancy is not in scope.

### Future enhancement

- **HA topologies for chokepoints**: LiteLLM gateway clustering, OPA replicas with consistent bundle distribution, kopf operator leader election, NATS JetStream clustering with explicit replication factor per stream, OpenSearch master / data tier separation with replica counts.
- **Per-chokepoint fail-open vs fail-closed policy**, surfaced as configuration: e.g., when OPA is unavailable, do callbacks fail closed (deny all) or fail open with audit-only? Per chokepoint, the call is not currently made.
- **Graceful degradation paths**: define what the platform does when Langfuse is down (continue serving, queue traces, lose traces?), when OpenSearch is down (continue, queue audit, lose audit?), when NATS is down (degraded eventing or hard fail?).
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

## 4. Other deferred items (cross-references)

These are tracked elsewhere but fit the "future enhancement" framing. Listed here for visibility:

- **Multi-cluster federation** — backlog 3.13. Not in v1.0; each cluster is an independent install. Trigger to revisit: a use case requiring cross-cluster awareness or shared resources.
- **Multi-CI support beyond GitHub Actions** — backlog 3.14. v1.0 ships GitHub Actions only.
- **Adding SDKs beyond Langchain Deep Agents** — backlog 3.15. Multi-SDK harness shape is preserved in v1.0; only Langchain Deep Agents ships.
- **Power-user personal agents in LibreChat** — backlog 3.12. LibreChat is locked down in v1.0.
- **Allure / ReportPortal for test reporting** — backlog 3.4. Not added in v1.0; CI output + Grafana metrics cover MVP.
- **SPIFFE identity** — backlog 3.5. Not needed for v1.0's independent-cluster model.
- **Knowledge base interactive UI beyond LibreChat** — backlog 3.10. LibreChat plus the Interactive Access Agent is the v1.0 access path.
- **HolmesGPT autonomous remediation expansion** — backlog 3.11. v1.0 starts with read-only diagnostics + auto-PR / suggestion cards; autonomous narrow-scope remediations come later as trust is established.
- **M-of-N approvers and complex approval routing** — backlog 3.3. The generalized approval system in v1.0 is "request a level, OPA may elevate, anyone with that level decides"; richer routing is added on demand.
- **Crossplane Operation type for day-2 patterns** — backlog 3.6. v1.0 is ad-hoc.
- **Custom load-testing harness** — backlog 3.8. v1.0 stress probes via Chainsaw / Playwright concurrent invocation are sufficient until proven otherwise.
- **Test CRDs** — backlog 3.7. v1.0 uses CLI orchestration; CRDs added only if declarative test execution becomes a felt need.

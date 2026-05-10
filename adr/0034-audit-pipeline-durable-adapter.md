# ADR 0034: Audit pipeline — durable adapter, Postgres + S3 system of record, OpenSearch advisory only

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Architecture-overview.md § 6.5 commits the platform to broad audit emission from every component (gateway, egress, agent runtime, ARK, UIs) and § 6.3 establishes the storage rule of thumb that anything in OpenSearch must be reproducible from a primary source. Section 6.3 further notes that audit logs flow into OpenSearch for searchable short-term retention and are archived to object storage for long-term retention. Architecture-backlog.md § 1.13 defers the retention *policy* to Workstream F but leaves the *path* — how events get to those stores and which one is authoritative — to be settled at the architecture level.

The diagram in § 6.5 currently shows components writing audit events to the OpenSearch audit index directly. That conflicts with the § 6 reproducibility invariant in architecture-backlog.md ("anything in OpenSearch must be reproducible from a primary source"): if components write straight to OpenSearch, OpenSearch is acting as a system of record. We also need a single emission path so that components do not each carry storage-specific code, and we need the path to honour the dual-mode hosting rule (kind locally, AWS-managed services in cloud — ADR 0033, ADR 0009, ADR 0014).

## Decision

Audit emission is mediated by a single platform **audit adapter** — a Python library (consistent with architecture-backlog.md § 6: "All custom code is Python") that every component links against. No component writes audit records directly to Postgres, S3, or OpenSearch.

The adapter posts to a single deployable **audit endpoint** component. The endpoint owns its own dynamic log-level toggle (per ADR 0035) so verbosity can be raised per-tenant or per-event-class without redeploying callers.

The system of record is Postgres + S3:

- **In-flight rows live in Postgres.** The audit endpoint writes received events into a Postgres `audit_events` table.
- **On AWS**, a batch CronJob runs roughly every 5 minutes: it aggregates pending rows, writes them as an immutable object to S3, verifies the S3 write (object exists, byte count and checksum match), and only then deletes the corresponding rows from Postgres. **S3 is the durable archive.**
- **On kind**, S3 is not provisioned; Postgres alone is the system of record and the batch step is disabled. This matches ADR 0033's dual-mode hosting expectation that kind is functionally complete without cloud-managed services.

OpenSearch is **advisory fanout only**: an asynchronous indexer ships events to OpenSearch for query and dashboards. The audit OpenSearch index is rebuildable from S3 (on AWS) or Postgres (on kind) and is therefore consistent with the architecture-backlog.md § 6 reproducibility invariant. OpenSearch is never the system of record; if it is unavailable, audit ingestion still succeeds.

A new Crossplane XRD, **`AuditLog`**, wraps the whole pipeline (consistent with the XR pattern in ADR 0021). One `AuditLog` claim provisions:

- Postgres backing store (RDS on AWS, in-cluster Postgres on kind — per ADR 0014 and ADR 0033).
- The S3 bucket and its lifecycle policy (AWS only).
- The OpenSearch indexer configuration (AWS-managed OpenSearch on AWS, in-cluster OpenSearch on kind — per ADR 0009 and ADR 0033).
- The 5-minute batch CronJob (AWS only).
- The audit endpoint Deployment.

The retention durations, redaction rules, and lifecycle specifics remain deferred to Workstream F under architecture-backlog.md § 1.13; this ADR fixes the topology, not the policy.

## Consequences

- The § 6.5 diagram should be read as logical: components emit through the audit adapter to the audit endpoint, which fans out to Postgres (authoritative), S3 (archive, AWS only), and OpenSearch (advisory). The diagram update is a documentation follow-up, not an architectural change.
- Components no longer carry storage-specific audit code. Adding a new component is a matter of importing the adapter; ADR 0011's pattern of test results streaming to OpenSearch + OTel is preserved because the adapter, not the component, is what talks to OpenSearch.
- The reproducibility invariant (architecture-backlog.md § 6) is honoured: OpenSearch contents are always rebuildable from S3 (AWS) or Postgres (kind).
- Failure modes are bounded. If OpenSearch is down, audit still ingests; advisory index lags and is rebuilt. If S3 write or verification fails, the batch leaves rows in Postgres and retries — no data loss, only growth in the in-flight table until S3 recovers. If Postgres is down, the audit endpoint fails closed for that component; this is the same failure surface as any other Postgres-backed control-plane operation (ADR 0014).
- Egress from the audit endpoint to S3 and AWS-managed OpenSearch goes through the Envoy egress proxy per ADR 0003; the endpoint is not a special case.
- The dynamic log-level toggle on the audit endpoint (ADR 0035) lets operators raise verbosity for incident response without redeploying every caller, since callers only know about the endpoint.
- Workstream F retention work (architecture-backlog.md § 1.13) now has a concrete surface to attach to: S3 lifecycle policy on the `AuditLog` XRD, plus OpenSearch index rollover. No further architectural decision is required to begin that work.
- **Revisit triggers**: a regulatory regime that requires synchronous durable archive (S3 write before ack) reopens the 5-minute batch cadence; a requirement that OpenSearch hold authoritative records (e.g. for legal hold) would invalidate the advisory-only stance and is explicitly out of scope here.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.5](../architecture-overview.md#65-observability-architecture) (observability and audit), [§ 6.3](../architecture-overview.md#63-memory-and-data-architecture) (memory and data architecture, audit retention paragraph)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.13](../architecture-backlog.md#113-audit-retention-policy-deferred-to-workstream-f) (audit retention policy deferred to Workstream F), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs) (reproducibility invariant; all custom code is Python)
- [ADR 0003](./0003-envoy-egress-proxy.md) (Envoy egress proxy — egress path for S3 and managed OpenSearch)
- [ADR 0009](./0009-opensearch-search-vector-store.md) (OpenSearch as search/vector store; advisory role here)
- [ADR 0011](./0011-three-layer-testing-cli-orchestration.md) (three-layer testing — test results stream via the same OpenSearch + OTel paths)
- [ADR 0014](./0014-postgres-primary-opensearch-retrieval.md) (Postgres as primary; OpenSearch as retrieval optimization only)
- [ADR 0021](./0021-grafanadashboard-xrs.md) (Crossplane-composed XR pattern, applied here as the `AuditLog` XRD)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (initial implementation targets AWS + kind; dual-mode hosting rule)
- [ADR 0035](./0035-dynamic-log-and-trace-toggle.md) (dynamic log-level toggle on the audit endpoint)

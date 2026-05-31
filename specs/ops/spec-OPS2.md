# Component Spec: OPS2 — Disaster Recovery (RETIRED)

> Status: Retired
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS2

---

## Retirement Notice

OPS2 (Disaster Recovery) has been **retired** and is no longer part of the
agent-os scope.

The concerns previously covered by this component are all baseline /
platform / cluster responsibilities that sit outside the agent-os boundary:

- Backup-policy attachment
- RPO / RTO targets
- etcd-snapshot vs GitOps-based recovery
- Cross-region failover
- Tenant restore granularity

These are owned by the underlying platform/cluster operations team, not by the
agentic execution platform.

See `_meta/reviews/DECISIONS-LOG.md`, decisions #13–#17 (all out-of-scope).

# Portfolio Notes

KernelCafe is a compact platform-engineering project built from a real homelab rather than a greenfield demo.

The public repository is intentionally sanitized, but the architecture reflects the production implementation and the failure modes that were actually tested while building it.

## What makes the design interesting

### Talos + Omni instead of a traditional mutable Linux fleet

The cluster runs Talos Linux and is managed through Sidero Omni. Nodes are treated as replaceable appliances rather than pets. Omni lives outside the cluster so management survives a total Kubernetes failure.

### GitOps is the normal operating model

Argo CD owns platform and application reconciliation. The public app-of-apps tree shows dependency ordering with sync waves while still acknowledging that some disaster-recovery steps need explicit health gates rather than blind ordering.

### Storage is designed as a separate failure domain

Longhorn uses a dedicated encrypted SSD on each node. Replica/data traffic runs over a secondary physical interface using Multus and Whereabouts. This deliberately trades peak throughput for predictable isolation from application traffic.

### Secrets are split by audience

Human credentials and break-glass material remain outside Kubernetes. HashiCorp Vault stores machine/application secrets, while External Secrets Operator presents only the required values to workloads as Kubernetes Secrets.

The scheduled Vault backup identity is intentionally able to read Raft snapshots but not restore them.

### Observability includes the network edge

Prometheus, Grafana, Loki and Alloy provide cluster metrics and logs. Network and ingress telemetry are treated as part of platform observability instead of a separate afterthought. The `KernelCafe // WAR ROOM` dashboard is a sanitized version of the production NOC view.

### Disaster recovery is dependency-aware

The restore path is not simply "apply all manifests again". Vault state, seal material, ESO and stateful workloads have an explicit recovery order, and backups are only considered proven after restore testing.

## Representative engineering tradeoffs

| Decision | Why | Tradeoff |
| --- | --- | --- |
| Schedule workloads on control planes | Small homogeneous cluster; better hardware utilization | Less isolation than dedicated control-plane hardware |
| Dedicated Longhorn NIC | Keeps storage replication away from normal app traffic | Secondary link can cap rebuild throughput |
| Longhorn 2/3 replica classes | Match redundancy cost to workload importance | More capacity consumed by critical volumes |
| Vault + ESO | Central machine-secret authority with Kubernetes-native delivery | Adds a bootstrap dependency during DR |
| Omni outside Kubernetes | Cluster lifecycle remains manageable after cluster loss | Requires one additional management host |
| Manual break-glass restore | Limits destructive credentials in automation | Recovery requires an operator and tested runbook |

## Failure-oriented validation

The production platform has been developed around verification rather than assuming that a successful deployment means a resilient deployment. Examples include:

- controlled Longhorn failover tests;
- dedicated storage-network traffic validation;
- Vault Raft snapshot creation and checksum verification;
- off-cluster backup propagation;
- External Secrets reconciliation tests;
- Pod Security hardening of backup jobs;
- dependency-aware disaster-recovery documentation;
- full-history secret scanning before publishing this reference repository.

## Public vs production

The private production repository remains the operational source of truth. This public repository intentionally changes or omits:

- real DNS names and IP ranges;
- hardware serials and machine IDs;
- backup hosts and paths;
- SSH fingerprints and remote-access details;
- BMC information;
- recovery endpoints;
- credentials, tokens, hashes, kubeconfigs and private keys.

The goal is to preserve the engineering decisions and useful examples without publishing an attack map of the live environment.

# Dependency and deployability audit

This public repository intentionally pins infrastructure dependencies rather than tracking floating tags. Pins are reviewed against upstream release channels before being changed.

Audit date: 2026-09-15

| Component | Public pin | Audit result |
| --- | --- | --- |
| Longhorn | 1.12.1 | Keep. Matches the KernelCafe production baseline and current v1.12 maintenance line. |
| Vault Helm | 0.34.1 | Keep. Official chart currently pairs this with Vault 2.0.4. |
| External Secrets Operator | 2.10.0 | Keep. Current upstream release. |
| MetalLB | 0.16.1 | Updated from 0.15.3 to the reviewed stable chart. |
| cert-manager | v1.21.2 | Keep. Current documented upstream chart. |
| Traefik | 41.5.0 | Updated from 40.2.0. Chart values were migrated from `logs.general` to the v41 `log` syntax. Review CRD upgrade notes before applying over an existing installation. |
| kube-prometheus-stack | 90.1.1 | Keep. Current KernelCafe/public pin; upstream releases rapidly, so newer patch releases should be reviewed rather than blindly chased. |
| Alloy | 1.11.0 | Updated from 1.2.1. This chart carries Alloy 1.18.0. |
| Loki | 18.13.1 | Migrated to the Grafana Community chart using Monolithic mode, TSDB v13, filesystem storage and explicit persistent-volume retention. |

## Loki migration

The public Loki reference now uses the community-maintained chart repository and `deploymentMode: Monolithic`, replacing the legacy `SingleBinary` naming from the 6.x chart line. The example deliberately runs one replica with `commonConfig.replication_factor: 1` and TSDB v13.

Filesystem storage is intentional for this small, single-replica homelab/reference deployment. It keeps the example self-contained and durable through a Longhorn PVC, but it is not an HA storage design. Larger or highly available Loki installations should use a supported external object store and an appropriate deployment topology.

The StatefulSet PVC policy is explicitly retained on deletion and scale-down. This avoids relying on chart defaults for log-data retention. Alloy continues to write through the chart-provided `loki-gateway` service.

## Multus and Whereabouts

The repository contains the sanitized `NetworkAttachmentDefinition` configuration layer, but intentionally does not vendor or track upstream installation manifests yet. Multus and Whereabouts are cluster-level CNI components and should be pinned to reviewed release artifacts. Their installation should be added separately once the manifests and Talos/Kubernetes compatibility are validated.

## Upgrade policy

A newer version is not automatically a better public example. Major/minor upgrades that change CRDs, chart repositories, deployment modes, storage behavior, or values schemas should be isolated into their own PR and rendered/tested before merge. Renovate is useful for discovery; it is not a substitute for this review.

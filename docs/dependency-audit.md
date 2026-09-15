# Dependency and deployability audit

This public repository intentionally pins infrastructure dependencies rather than tracking floating tags. Pins are reviewed against upstream release channels before being changed.

Audit date: 2026-09-14

| Component | Public pin | Audit result |
| --- | --- | --- |
| Longhorn | 1.12.1 | Keep. Matches the KernelCafe production baseline and current v1.12 maintenance line. |
| Vault Helm | 0.34.1 | Keep. Official chart currently pairs this with Vault 2.0.4. |
| External Secrets Operator | 2.10.0 | Keep. Current upstream release. |
| MetalLB | 0.16.1 | Updated from 0.15.3 to the current stable chart. |
| cert-manager | v1.21.2 | Keep. Current documented upstream chart. |
| Traefik | 41.5.0 | Updated from 40.2.0. Chart values were migrated from `logs.general` to the v41 `log` syntax. Review CRD upgrade notes before applying over an existing installation. |
| kube-prometheus-stack | 90.1.1 | Keep. Current KernelCafe/public pin; upstream releases rapidly, so newer patch releases should be reviewed rather than blindly chased. |
| Alloy | 1.11.0 | Updated from 1.2.1. This chart carries Alloy 1.18.0. |
| Loki | 6.45.2 | Migration required; do not blindly bump. The OSS Loki chart moved from Grafana's original chart repository to `grafana-community/helm-charts` in March 2026 and the community chart has since introduced breaking deployment-mode changes. |

## Loki migration gate

The existing Loki manifest is intentionally left unchanged in this audit PR. It uses the old Grafana repository and `deploymentMode: SingleBinary`. The current community chart renamed SingleBinary to Monolithic as of chart 12.0.0. A safe migration therefore needs a dedicated PR that renders the new chart, translates values, verifies persistent-storage behavior, and confirms Alloy's gateway endpoint before changing the public example.

## Multus and Whereabouts

The repository contains the sanitized `NetworkAttachmentDefinition` configuration layer, but intentionally does not vendor or track upstream installation manifests yet. Multus and Whereabouts are cluster-level CNI components and should be pinned to reviewed release artifacts. Their installation should be added separately once the manifests and Talos/Kubernetes compatibility are validated.

## Upgrade policy

A newer version is not automatically a better public example. Major/minor upgrades that change CRDs, chart repositories, deployment modes, storage behavior, or values schemas should be isolated into their own PR and rendered/tested before merge. Renovate is useful for discovery; it is not a substitute for this review.

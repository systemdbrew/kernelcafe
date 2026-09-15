# KernelCafe

[![Validate](https://github.com/systemdbrew/kernelcafe/actions/workflows/validate.yml/badge.svg)](https://github.com/systemdbrew/kernelcafe/actions/workflows/validate.yml)
[![Gitleaks](https://github.com/systemdbrew/kernelcafe/actions/workflows/gitleaks.yml/badge.svg)](https://github.com/systemdbrew/kernelcafe/actions/workflows/gitleaks.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

KernelCafe is a public, sanitized reference implementation of my home Kubernetes platform.

It mirrors the architecture and engineering decisions used by the private production GitOps repository while intentionally omitting operational identifiers, production addressing, recovery endpoints, hardware serial numbers, credentials and other environment-specific details.

> **Public reference, not production source of truth.** The manifests are intentionally realistic enough to demonstrate the platform, but values that would map the live environment are replaced or omitted.

## What this project demonstrates

- Talos Linux managed through Sidero Omni
- upstream Kubernetes
- Argo CD app-of-apps GitOps
- Longhorn replicated storage
- dedicated storage networking with Multus + Whereabouts
- HashiCorp Vault + External Secrets Operator
- Traefik + MetalLB ingress
- cert-manager architecture
- Prometheus / Grafana monitoring
- Loki + Grafana Alloy logging
- a NOC-style `KernelCafe // WAR ROOM` dashboard
- Renovate dependency automation
- Gitleaks full-history secret scanning
- YAML/Kustomize CI validation
- dependency-aware backup and disaster-recovery design

For the engineering rationale and tradeoffs, see [docs/portfolio.md](docs/portfolio.md).

## Architecture

```text
                Dedicated management host
                         Omni
                          |
                          v
                     Talos nodes
                          |
                          v
                       GitHub
                          |
                          v
                       Argo CD
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
      Platform         Security      Observability
      --------         --------      -------------
      MetalLB          Vault         Prometheus
      Traefik          ESO           Grafana
      Multus                          Loki / Alloy
          |                               |
          +---------------+---------------+
                          |
                          v
                  Talos Kubernetes
             3 control-plane + 1 worker
                          |
                +---------+---------+
                |                   |
                v                   v
           Applications          Longhorn
                                     |
                            dedicated storage NIC
                          Multus + Whereabouts
```

Omni deliberately lives outside Kubernetes so a total cluster failure does not also remove the management plane.

## Public vs production

This repository preserves the reusable design while changing or omitting production-specific details.

| Published here | Kept private |
| --- | --- |
| platform architecture | production domain names |
| sanitized Argo CD applications | real RFC1918/VLAN addressing |
| example Vault + ESO configuration | hardware serials / Omni machine IDs |
| Longhorn storage design | backup hosts and filesystem paths |
| documentation-safe networking | Tailscale/BMC details |
| sanitized dashboards | SSH host fingerprints |
| DR dependency model | recovery endpoints and credentials |

Documentation examples use values such as `example.net`, `10.10.x.x` and placeholder machine IDs. They are not the production values.

## Cluster profile

The platform model uses small x86 nodes with:

- 3 control-plane nodes
- 1 worker node
- control-plane scheduling enabled
- a separate system NVMe device
- one dedicated SSD per node for Longhorn
- a primary Kubernetes/application NIC
- a dedicated Longhorn replication NIC
- TPM-backed encryption for Longhorn backing storage

## GitOps layout

```text
.
├── README.md
├── argocd/
│   ├── root-application.yaml
│   ├── kernelcafe-project.yaml
│   └── apps/
├── docs/
│   ├── architecture.md
│   ├── disaster-recovery.md
│   ├── portfolio.md
│   └── secrets.md
├── examples/
│   ├── cluster.yaml
│   └── inventory.yaml
├── infrastructure/
│   ├── external-secrets/
│   ├── longhorn/
│   ├── metallb/
│   ├── monitoring/dashboards/
│   ├── networking/
│   ├── observability/
│   ├── traefik/
│   └── vault/
├── CONTRIBUTING.md
├── SECURITY.md
├── LICENSE
└── .github/workflows/
    ├── gitleaks.yml
    └── validate.yml
```

The Argo CD tree demonstrates sync waves, Helm sources, Kustomize-managed configuration and the dependency relationships used by the real platform.

## Design principles

1. **Git is the declarative source of truth.** Routine platform changes go through GitOps.
2. **Management survives cluster loss.** Omni is outside Kubernetes.
3. **Storage failure domains matter.** Longhorn replicas are spread across physical nodes.
4. **Storage traffic is isolated.** Longhorn uses a secondary network instead of competing with normal application traffic.
5. **Secrets do not belong in Git.** Vault owns workload secrets; ESO performs delivery.
6. **Human and machine secrets are separated.** Human/break-glass material stays outside Kubernetes.
7. **Critical components are pinned and reviewed.** Automation can propose updates without blindly applying them.
8. **Backups are not proven until restore testing succeeds.**
9. **Least privilege is preferred.** Backup identities do not receive restore permissions.
10. **Observability is part of the platform.** Metrics, logs, network telemetry and ingress visibility belong in the same operational picture.

## Secrets architecture

```text
Human / operator
     |
     +---- password manager / offline recovery
     |
     v
HashiCorp Vault
     |
     | Kubernetes auth
     v
External Secrets Operator
     |
     v
Kubernetes Secret
     |
     v
Workload
```

See [docs/secrets.md](docs/secrets.md).

## Storage architecture

```text
Dedicated SSD
     |
     v
TPM-backed LUKS2
     |
     v
     XFS
     |
     v
/var/mnt/longhorn
     |
     v
  Longhorn
```

Longhorn replication traffic uses a dedicated secondary interface through Multus and Whereabouts. The reference design uses two replicas for normal state and three replicas for critical platform state.

See [infrastructure/longhorn/README.md](infrastructure/longhorn/README.md).

## Observability

The public observability example includes Prometheus/Grafana, Loki, Alloy and a sanitized version of the NOC-style `KernelCafe // WAR ROOM` dashboard. The layout and query patterns are preserved without publishing real hosts, domains, addresses or production service inventory.

See [infrastructure/observability/README.md](infrastructure/observability/README.md) and [the War Room dashboard](infrastructure/monitoring/dashboards/kernelcafe-war-room.yaml).

## Disaster recovery

Recovery is dependency-aware rather than "apply everything and hope":

```text
Talos / Kubernetes
        |
        v
     Argo CD
        |
        v
     Longhorn
        |
        v
       Vault
        |
        v
restore verified Vault snapshot
        |
        v
unseal + validate Raft
        |
        v
External Secrets Operator
        |
        v
workload secrets
        |
        v
applications
```

See [docs/disaster-recovery.md](docs/disaster-recovery.md).

## CI and dependency hygiene

Two independent CI checks protect the public repository:

- **Validate** lints YAML and renders every committed Kustomize entry point.
- **Gitleaks** scans Git history for accidental credential material.

`renovate.json` enables dependency discovery and digest pinning while deliberately disabling automerge for critical platform components.

## Security boundary

This repository intentionally does **not** contain:

- production domain names
- hardware serial numbers
- Omni machine UUIDs
- production RFC1918/VLAN addressing
- Tailscale addresses
- SSH host fingerprints
- production backup destinations
- BMC information
- Vault tokens or unseal material
- kubeconfigs or Talos credentials
- Cloudflare tokens
- production htpasswd values

See [SECURITY.md](SECURITY.md) before reporting a possible sensitive-data exposure.

## Contributing

Contributions are welcome when they preserve the sanitization boundary and improve the reusable design. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

KernelCafe is available under the [MIT License](LICENSE).

## Status

This repository is derived from a functioning homelab platform rather than a hypothetical architecture. The production implementation is continuously updated and tested privately; reusable lessons and sanitized examples are published here.

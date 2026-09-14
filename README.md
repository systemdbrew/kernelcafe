# KernelCafe

KernelCafe is a public, sanitized reference implementation of my home Kubernetes platform.

It mirrors the architecture and engineering decisions used by the private production GitOps repository while intentionally omitting operational identifiers, private addressing, recovery endpoints, hardware serial numbers, credentials, and other environment-specific details.

## What this project demonstrates

- Talos Linux managed through Sidero Omni
- upstream Kubernetes
- Argo CD app-of-apps GitOps
- Longhorn replicated storage
- dedicated storage networking with Multus + Whereabouts
- HashiCorp Vault + External Secrets Operator
- Traefik + MetalLB ingress
- cert-manager
- Prometheus / Grafana monitoring
- Loki + Grafana Alloy logging
- security-focused Grafana dashboards
- Renovate dependency automation
- Gitleaks secret scanning
- backup and disaster-recovery design

## Architecture

```text
                    GitHub
                       |
                       v
                    Argo CD
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
    Platform        Security      Observability
    --------        --------      -------------
    MetalLB         Vault         Prometheus
    Traefik         ESO           Grafana
    cert-manager                   Loki / Alloy
    Multus

                       |
                       v
               Talos Kubernetes
         3 control-plane + 1 worker
                       |
             +---------+---------+
             |                   |
             v                   v
        Application          Storage
        workloads            Longhorn
                                 |
                      dedicated secondary NIC
                      Multus + Whereabouts
```

Omni is deliberately hosted outside the Kubernetes cluster so a total cluster failure does not also remove the management plane.

## Repository model

This repository is documentation/reference material rather than the live production source of truth.

Values that would identify or expose the production environment are replaced with documentation-safe examples such as:

- `example.net`
- `10.10.0.0/16`
- placeholder Omni machine IDs
- generic backup targets

Secrets are never committed. Workload credentials are expected to live in Vault and are materialized into Kubernetes Secrets by External Secrets Operator.

## Cluster profile

The real platform uses small x86 nodes with:

- 3 control-plane nodes
- 1 worker node
- control-plane scheduling enabled
- one system NVMe device per node
- one dedicated SSD for Longhorn
- one primary application NIC
- one dedicated Longhorn replication NIC
- TPM-backed encryption for Longhorn backing storage

The exact production serial numbers, machine IDs and addresses are intentionally absent here.

## GitOps layout

```text
.
├── README.md
├── docs/
│   ├── architecture.md
│   ├── secrets.md
│   └── disaster-recovery.md
├── examples/
│   ├── cluster.yaml
│   └── inventory.yaml
├── infrastructure/
│   ├── external-secrets/
│   ├── longhorn/
│   ├── networking/
│   └── vault/
└── .github/
    └── workflows/
        └── gitleaks.yml
```

The private production repository contains the complete Argo CD application tree, workload manifests, dashboards, automation and operational runbooks. This public repository focuses on the reusable design rather than publishing a map of the live environment.

## Design principles

1. **Git is declarative source of truth.** Routine platform changes go through GitOps.
2. **Management survives cluster loss.** Omni is outside the cluster.
3. **Storage failure domains matter.** Longhorn replicas are spread across physical nodes.
4. **Storage traffic is isolated.** Longhorn uses a secondary network instead of competing with normal application traffic.
5. **Secrets do not belong in Git.** Vault owns machine/application secrets; ESO performs delivery.
6. **Human and machine secrets are separated.** A password manager is used for human credentials and break-glass recovery material; Vault serves workloads.
7. **Critical components are pinned and reviewed.** Renovate can propose updates, but storage and cluster upgrades are deliberately reviewed.
8. **Backups are not considered proven until restore testing succeeds.**
9. **Least privilege is preferred.** Backup identities can create snapshots but cannot restore them.
10. **Observability is part of the platform, not an afterthought.**

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

Each cluster node exposes a dedicated encrypted SSD to Longhorn:

```text
SSD
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

Longhorn replication traffic uses a dedicated secondary interface through Multus and Whereabouts.

See [infrastructure/longhorn/README.md](infrastructure/longhorn/README.md).

## Disaster recovery

The recovery order is intentionally dependency-aware:

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
restore Vault snapshot
        |
        v
unseal / validate Raft
        |
        v
External Secrets Operator
        |
        v
application secrets
        |
        v
workloads
```

See [docs/disaster-recovery.md](docs/disaster-recovery.md).

## Security

This repository uses Gitleaks in CI to scan repository history for accidental credentials.

The public repository intentionally does **not** contain:

- production domain names
- hardware serial numbers
- Omni machine UUIDs
- production RFC1918 addressing
- Tailscale addresses
- SSH host fingerprints
- production backup destinations
- BMC information
- Vault tokens or unseal material
- kubeconfigs or Talos credentials
- Cloudflare tokens
- htpasswd values

## Status

This repository is derived from a functioning homelab platform rather than a hypothetical architecture. The production implementation is continuously updated and tested privately; reusable lessons and sanitized examples are published here.

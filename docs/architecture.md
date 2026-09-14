# Architecture

## Management plane

Sidero Omni runs on a small dedicated Linux host outside the Kubernetes cluster.

That separation is intentional: Kubernetes can be destroyed and rebuilt without losing the system responsible for Talos machine lifecycle and cluster management.

## Kubernetes topology

The reference topology is:

```text
cp01  control-plane + workloads + Longhorn
cp02  control-plane + workloads + Longhorn
cp03  control-plane + workloads + Longhorn
wk01  workloads + Longhorn
```

Control-plane nodes are schedulable because the nodes are homogeneous and the cluster is small.

## Networking

Two node networks are used:

```text
Primary network
  Kubernetes API
  pod/application traffic
  ingress and normal services

Storage network
  Longhorn replica/data traffic
  Multus secondary attachment
  Whereabouts IPAM
```

The examples use documentation-only ranges:

```text
primary: 10.10.20.0/24
storage: 10.10.50.0/24
service VIPs: 10.10.90.0/24
```

These are not the production addresses.

## Ingress

MetalLB provides LoadBalancer addresses.

Traefik runs multiple replicas and is the primary ingress controller. cert-manager issues certificates using DNS-01 validation.

## Storage

Longhorn uses a dedicated SSD on every node. The backing device is encrypted before Longhorn uses it.

Two StorageClasses are used conceptually:

- `longhorn-standard`: two replicas for normal stateful workloads
- `longhorn-critical`: three replicas for important platform state

## Observability

The platform uses:

- Prometheus for metrics
- Grafana for visualization
- Loki for logs
- Alloy for collection
- UniFi Poller for network telemetry

A NOC-style "War Room" dashboard combines cluster, network and service health into one view.

## GitOps

Argo CD reconciles the platform using an app-of-apps model.

A simplified order is:

```text
Multus / networking
Longhorn
Vault
External Secrets Operator
cert-manager / MetalLB
Traefik
monitoring / logging
applications
```

Disaster recovery has additional manual health gates because ordering resources is not enough to guarantee that Vault is initialized, restored and unsealed before secret consumers start.

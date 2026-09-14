# Longhorn design

KernelCafe uses Longhorn for replicated Kubernetes persistent storage.

## Node storage

Each node provides a dedicated SSD:

```text
dedicated SSD
    |
    v
TPM-backed LUKS2
    |
    v
XFS
    |
    v
/var/mnt/longhorn
```

The system disk is separate.

## Storage classes

The reference design uses:

```text
longhorn-standard  -> 2 replicas
longhorn-critical  -> 3 replicas
```

Critical platform services such as Vault use the three-replica class.

## Dedicated storage network

Longhorn replica/data traffic uses a second physical NIC through Multus.

Documentation example:

```text
primary NIC   -> 10.10.20.0/24
storage NIC   -> 10.10.50.0/24
Whereabouts   -> 10.10.50.220-239
```

The production subnet is intentionally not published.

## Failure behavior

The design allows controller-managed RWO workloads to recover on a surviving node after a real node outage, assuming Longhorn replicas remain healthy and suitable compute capacity exists.

A controlled failover test should be performed before considering the storage platform production-ready.

## Tradeoff

The secondary storage NIC may be slower than the primary application NIC. That is an intentional tradeoff: predictable isolation is more valuable here than maximizing replica throughput.

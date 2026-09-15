# Multus + Whereabouts storage network

KernelCafe separates Longhorn replication traffic from the primary Kubernetes/application network. Multus attaches a second interface to participating pods and Whereabouts allocates addresses from the dedicated storage subnet.

The public example intentionally uses sanitized values:

- primary interface on the node: adapt `eth1` to your hardware
- storage subnet: `10.10.50.0/24`
- dynamic pod range: `10.10.50.220-10.10.50.239`
- gateway: `10.10.50.1`
- NetworkAttachmentDefinition: `longhorn-system/longhorn-storage`

Multus and Whereabouts must already be installed cluster-wide before this NetworkAttachmentDefinition is useful. The exact installation manifests should be pinned to reviewed upstream releases rather than copied blindly into a public example.

After installation, validate cross-node traffic on the secondary interface before directing stateful storage replication over it. A successful attachment on one node does not prove the storage network works between nodes.

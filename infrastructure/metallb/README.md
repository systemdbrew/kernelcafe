# MetalLB BGP edge

KernelCafe uses MetalLB BGP to advertise Kubernetes `LoadBalancer` VIPs to the upstream gateway instead of extending the service/edge VLAN onto the Talos hosts.

This directory is a **sanitized reference** derived from the production design. The addresses shown here are documentation values and do not identify the live network.

## Design

```text
Upstream gateway / AS64512 / 10.10.20.1
        |
        | eBGP over the node network
        |
MetalLB speakers / AS64513
        |
        `-- advertise selected service VIPs as /32 routes
              10.10.90.10/32
              10.10.90.11/32
```

The gateway learns more-specific `/32` service routes from MetalLB. Kubernetes nodes therefore do not need an interface on the edge/service VLAN just to make a `LoadBalancer` VIP reachable.

## Resources

- `ipaddresspool.yaml` defines the deliberately small set of service VIPs MetalLB may allocate.
- `bgp-peer.yaml` defines the eBGP relationship between MetalLB speakers and the upstream gateway.
- `bgp-advertisement.yaml` permits the VIP pool to be advertised over BGP.

The reference uses private ASNs `64512` and `64513`. These are suitable for illustrating the design but should be replaced to match the routing domain where the example is deployed.

## Failure detection

The reference requests a 1-second BGP keepalive and 3-second hold timer. The production design uses fast BGP failure detection so a dead node does not remain a viable next hop for several minutes after a hard failure.

Whether timers this aggressive are appropriate depends on the upstream router and network. Validate timer support and failure behavior before copying them into another environment.

## Route filtering

The upstream router should apply an inbound policy that accepts **only the approved service VIP prefix(es)** from the Kubernetes BGP peers. Do not accept arbitrary routes from the cluster.

Conceptually:

```text
MetalLB speakers  --->  upstream router
     AS64513              AS64512
        |
        `--- only approved /32 service VIPs are accepted
```

The production router policy is intentionally not published here because it contains environment-specific addressing and peer inventory.

## High availability

Each eligible MetalLB speaker can establish a BGP session with the upstream router. MetalLB and the router determine the active paths for advertised service VIPs; if a node disappears, its BGP session and associated next hop are withdrawn.

This keeps edge reachability in the routing layer instead of modifying Talos host networking to attach the service VLAN directly.

## Verify

After applying the resources in a test environment:

```bash
kubectl -n metallb-system get bgppeers.metallb.io
kubectl -n metallb-system get ipaddresspools.metallb.io
kubectl -n metallb-system get bgpadvertisements.metallb.io
kubectl get bgpsessionstates.frrk8s.metallb.io -A -o wide
```

Also verify on the upstream router that every expected peer is established and that only the intended `/32` VIP routes are learned.

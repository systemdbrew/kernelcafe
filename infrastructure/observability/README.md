# Observability stack

KernelCafe treats observability as part of the platform rather than an optional add-on.

## Components

- **Prometheus** collects cluster and application metrics.
- **Grafana** provides dashboards and alerting views.
- **Loki** stores logs.
- **Grafana Alloy** discovers Kubernetes workloads and ships logs to Loki.
- **UniFi Poller** and ingress logs can be added as external telemetry sources.

## War Room

The public War Room dashboard is intentionally sanitized. It demonstrates the NOC layout and query patterns without exposing production hostnames, IPs, domains, client addresses, or backup infrastructure.

The dashboard focuses on:

- cluster node health
- pod readiness
- CPU and memory pressure
- persistent-volume health
- ingress errors
- recent warning/error logs

The production dashboard contains additional environment-specific panels that are intentionally omitted from this public repository.

# Contributing

Thanks for taking an interest in KernelCafe.

This repository is a sanitized reference implementation of a real homelab platform. Contributions should improve the reusable architecture without reintroducing production-specific information.

## Before opening a pull request

- Keep secrets and credentials out of Git.
- Use documentation-only domains such as `example.net`.
- Use documentation-only private ranges already established in this repository (`10.10.x.x`).
- Do not add hardware serial numbers, Omni machine UUIDs, Tailscale addresses, SSH fingerprints, backup destinations, BMC details or recovery material.
- Prefer pinned versions for critical platform components.
- Keep examples realistic enough to be useful, but generic enough to be safely reusable.
- Run the repository validation and Gitleaks checks.

## Change style

Small focused pull requests are preferred. Explain the architectural reason for a change when it affects storage, networking, secrets, ingress, observability or disaster recovery.

For dependency updates, include upstream compatibility notes when the change affects Talos, Kubernetes, Longhorn, Vault, Argo CD or another stateful/cluster-critical component.

## Validation

The CI workflow checks YAML syntax and renders every committed Kustomize entry point. Gitleaks scans repository history for accidental secret material.

A passing CI run does not prove that a manifest is safe for a production cluster. This repository is reference material; adapt it to your own environment and threat model.

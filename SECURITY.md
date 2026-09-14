# Security Policy

KernelCafe is a sanitized public reference implementation. It is not the live production source of truth and should not contain credentials, production endpoints, recovery material, private keys, or other sensitive operational data.

## Reporting a security issue

Please do not open a public issue for a suspected credential leak, private key, token, or disclosure that could affect a real system.

Use GitHub's private security-advisory / private vulnerability-reporting flow when available. If that is not available, contact the maintainer through the GitHub profile rather than posting the sensitive value publicly.

When reporting a possible secret exposure, include the file path and commit SHA but do **not** copy the secret into the report unless the private reporting channel explicitly requires it.

## Scope

Useful reports include:

- credentials or private keys accidentally committed to this repository;
- production hostnames, IPs, hardware identifiers, backup targets, SSH fingerprints or recovery endpoints that bypass the sanitization boundary;
- unsafe example configurations that would predictably expose credentials;
- CI or dependency configuration that materially weakens the repository's security controls.

General Kubernetes hardening suggestions and architecture discussions are welcome as normal issues when they do not contain sensitive information.

## Repository controls

The repository uses Gitleaks to scan Git history for common secret patterns. Example manifests use documentation-only identifiers and networks. Secret values are intentionally represented by Vault / External Secrets references rather than plaintext values.

No automated scanner should be treated as a guarantee: review is still required before publishing changes copied from a live environment.

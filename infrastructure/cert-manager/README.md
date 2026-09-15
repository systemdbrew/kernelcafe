# cert-manager reference

This example mirrors the production pattern without publishing a real domain, email address, or Cloudflare credential.

The `ClusterIssuer` uses Let's Encrypt production ACME with a Cloudflare DNS-01 solver. The example expects a Kubernetes Secret named `cloudflare-api-token` in the `cert-manager` namespace with a key named `api-token`.

In KernelCafe production, workload/API credentials are owned by Vault and delivered to Kubernetes by External Secrets Operator. The ACME account key referenced by `privateKeySecretRef` is intentionally controller-managed by cert-manager rather than stored in Git or migrated into Vault.

For a public deployment, replace `admin@example.net` and provide the Cloudflare API token through your own secret-management system. Do not commit the token to this repository.

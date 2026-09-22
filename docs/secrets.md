# Secrets architecture

KernelCafe separates human secrets from workload secrets.

## Human secrets

Human credentials, recovery material, emergency procedures and other operator-only information belong in a password manager and in an independent offline recovery copy where appropriate.

They are not stored in Kubernetes or Git.

## Machine and workload secrets

HashiCorp Vault is the authoritative store for machine/application credentials.

Vault uses:

- 3 replicas
- integrated Raft storage
- persistent volumes on the critical Longhorn StorageClass
- Kubernetes authentication
- independent Transit auto-unseal in production; the portable public manifest intentionally omits provider-specific Transit configuration and therefore requires an operator-supplied seal/unseal design
- a file audit device
- no Agent Injector
- no CSI provider

External Secrets Operator reads Vault with a dedicated read-only policy and creates ordinary Kubernetes Secrets for workloads.

The sanitized `infrastructure/vault/values.yaml` is intentionally portable: it does not contain the production Transit provider address, CA, key name, token Secret, or seal stanza. Production architecture is documented in [`vault-operations.md`](vault-operations.md). Do not interpret the absence of that environment-specific configuration from the public values file as production using Shamir for routine restarts.

## ESO flow

```text
Kubernetes ServiceAccount token
             |
             v
       Vault Kubernetes auth
             |
             v
       read-only ESO policy
             |
             v
      ClusterSecretStore
             |
             v
       ExternalSecret
             |
             v
       Kubernetes Secret
```

The important point is that application manifests reference logical secret paths, not secret values.

## Example Vault policy

```hcl
path "secret/data/kernelcafe/*" {
  capabilities = ["read"]
}

path "secret/metadata/kernelcafe/*" {
  capabilities = ["read", "list"]
}
```

## Human Vault administration

Routine human administration should use a named authentication method and short-lived token rather than the initialization root token.

A practical small-environment model is:

```text
userpass admin
  -> admin policy
  -> short-lived renewable token

initial root token
  -> offline break-glass only
```

The production environment uses a short initial TTL and bounded maximum TTL for the admin identity.

## Vault backups

A dedicated Kubernetes identity can read the Raft snapshot endpoint. It cannot restore snapshots.

Conceptually:

```hcl
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}
```

That separation prevents the scheduled backup job from having destructive restore permissions.

## What never belongs in this repository

- Vault root tokens
- admin tokens
- unseal keys
- Kubernetes service-account JWTs
- SSH private keys
- API tokens
- passwords
- htpasswd hashes used by production
- decoded Kubernetes Secret values

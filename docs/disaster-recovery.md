# Disaster recovery

This document describes the architecture of the recovery process without publishing production recovery endpoints or identifiers.

## Recovery assets

A complete recovery set requires more than Git.

Important assets include:

1. the GitOps repository
2. Talos/Omni recovery capability
3. Longhorn or application backups where applicable
4. verified Vault Raft snapshots
5. independently-held Vault and Transit-provider recovery material
6. credentials required to bootstrap access to external systems

These assets should not all share the same failure domain.

## Vault recovery

A Vault Raft snapshot contains Vault's encrypted storage state. It does not replace the seal/unseal material.

The conceptual restore sequence is:

```text
build Kubernetes
     |
     v
restore/unseal Transit provider if required
     |
     v
deploy Vault with Transit seal configuration
     |
     v
restore verified Raft snapshot
     |
     v
verify application Vault auto-unseals
     |
     v
validate Raft quorum
     |
     v
enable ESO consumers
```

Snapshot restore is a manual break-glass operation.

Production uses an independent Transit Vault for application-Vault auto-unseal. Recovery therefore has an explicit dependency: the Transit provider must be available and unsealed before application Vault members that depend on it are restarted. The public manifests omit provider-specific addresses, certificates, key names and credentials.

The scheduled backup ServiceAccount should have snapshot-read permission only.

## Full platform recovery order

```text
1. restore management access
2. build Talos/Kubernetes
3. bootstrap Argo CD
4. restore platform networking and Longhorn
5. deploy Vault
6. restore a verified Vault snapshot
7. verify the Transit provider is available, then verify Vault auto-unseals and validate Raft
8. validate Kubernetes auth
9. start External Secrets Operator
10. validate ClusterSecretStore
11. allow ExternalSecrets to recreate workload Secrets
12. bring applications online
13. validate monitoring and backup jobs
```

## Restore testing

A backup is not considered fully proven until it can be restored.

Vault restore drills should happen in an isolated disposable environment. The test Vault must not be connected to production ESO or production consumers.

The drill should confirm:

- snapshot checksum is valid
- snapshot restore succeeds
- the independent Transit provider can auto-unseal the restored state; break-glass recovery material is available if the provider itself must be recovered
- Raft becomes healthy
- expected KV metadata exists
- no secret values are printed into logs during validation

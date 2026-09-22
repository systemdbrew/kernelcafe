# Vault operations

KernelCafe runs a three-member HashiCorp Vault HA cluster backed by integrated Raft storage and persistent volumes.

## Upgrade model

Vault uses the Helm chart's `OnDelete` StatefulSet strategy so upgrades remain deliberate and preserve Raft quorum. Never replace more than one member at a time.

For an image or pod-template update:

1. Confirm all three pods are Ready and unsealed.
2. Confirm one node is active, two are standby, and Raft committed/applied indexes are caught up.
3. Delete one standby pod.
4. Wait for it to return on the new StatefulSet revision and verify it is unsealed.
5. Confirm the upgraded member is Ready, standby, and caught up with Raft.
6. Repeat for the second standby.
7. Delete the old active member last.
8. Confirm all pods use the desired image/revision, exactly one member is active, all are unsealed, and Raft indexes are caught up.

Useful checks:

```bash
kubectl -n vault get sts vault
kubectl -n vault get pods -l app.kubernetes.io/name=vault \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,REVISION:.metadata.labels.controller-revision-hash,IMAGE:.spec.containers[*].image'

for pod in vault-0 vault-1 vault-2; do
  echo "===== $pod ====="
  kubectl -n vault exec "$pod" -- vault status | \
    grep -E 'Sealed|Version|HA Mode|Raft Committed|Raft Applied'
done
```

Do not delete a second member until the previously replaced member is Ready, unsealed, and caught up.

## Independent Transit auto-unseal

Production KernelCafe uses a separate, independently operated Vault Transit service for auto-unseal. The public repository intentionally omits the provider address, certificate material, key name, token, and environment-specific Kubernetes Secret.

The important architecture is:

```text
Application Vault (3-node Raft)
        |
        | Transit seal
        | renewable periodic credential
        v
Independent Transit Vault
```

The Transit provider must not be the same Vault cluster it unseals; that would create a circular dependency during a full restart.

A practical credential design is a renewable orphan periodic token with:

- a least-privilege policy limited to the required Transit encrypt/decrypt paths;
- the standard `default` policy retained so normal token self-renewal is available;
- no explicit maximum TTL;
- a period long enough to tolerate a reasonable provider outage;
- `disable_renewal = "false"` in the Transit seal stanza.

The production deployment uses a multi-day renewal period. Exact production addresses, key names, tokens, and recovery material are deliberately not published.

Keep the independent Transit provider's own recovery mechanism operational. If the provider restarts sealed, restore/unseal it before restarting application Vault members that depend on it.

## Renewal monitoring

A Transit token can continue to work for an already-running Vault cluster until its lease expires, while a later pod restart fails to auto-unseal. For that reason, validate renewal rather than testing only a single encrypt/decrypt operation.

The independent provider should record successful `auth/token/renew-self` events alongside Transit encrypt/decrypt activity. Alert or investigate repeated renewal failures before the periodic token expires.

Never print tokens during diagnostics. Compare protected copies using byte counts or hashes when necessary.

## Raft backups

A production-grade backup job should not report success merely because `vault operator raft snapshot save` returned. KernelCafe's backup flow validates the snapshot, transfers it to separate storage, verifies SHA-256 at the destination, and only then reports the job successful.

A useful operational lesson: when a Kubernetes Job uses an init container to create a snapshot, a failed init container can leave the later transfer container in `PodInitializing`. Diagnose the init-container logs first.

After changes to the seal provider or its credential, run a manual backup and verify the entire snapshot -> transfer -> checksum path before considering the recovery complete.

## Monitoring

Vault monitoring is managed as standalone GitOps resources in `infrastructure/vault/monitoring.yaml`. Useful alerts cover:

- a server remaining sealed;
- no active HA server;
- more than one server reporting active;
- missing Vault telemetry targets;
- a pending manual `OnDelete` rollout;
- failed or stale backup jobs where backup automation is deployed.

The generic StatefulSet rollout alert should account for Vault's deliberate `OnDelete` strategy so an intentional manual rollout is not mistaken for an unhealthy StatefulSet.

## Recovery material

Auto-unseal does not eliminate the need for recovery material. Keep recovery shares, root recovery credentials, Transit-provider recovery material, and any credential-store decryption key outside Git and in appropriately separated offline storage.

Do not rotate recovery shares, root tokens, or Transit keys merely because a pod failed to start. First determine whether the failure is the provider, network/TLS path, token lease, token policy, or seal configuration.

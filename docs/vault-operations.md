# Vault operations

KernelCafe runs a three-member HashiCorp Vault HA cluster backed by integrated
Raft storage and Longhorn persistent volumes.

## Upgrade model

Vault intentionally uses the Helm chart's `OnDelete` StatefulSet strategy.
Argo CD may update the StatefulSet pod template, but Kubernetes must not replace
all Vault members automatically while Shamir sealing is in use.

For an image or pod-template update:

1. Confirm all three pods are Ready and unsealed.
2. Confirm one node is active, two are standby, and Raft committed/applied
   indexes are caught up.
3. Delete one standby pod.
4. Wait for it to return on the new StatefulSet revision.
5. If Shamir sealing is still configured, unseal it with the required key
   shares. Never place unseal shares in Git, shell history, Kubernetes
   manifests, or chat/log output.
6. Confirm the upgraded member is Ready, standby, and caught up with Raft.
7. Repeat for the second standby.
8. Delete the old active member. The upgraded members should elect a new
   active node.
9. Unseal the recreated final member and verify it rejoins as Ready.
10. Confirm all pods use the same image/revision, exactly one member is active,
    all are unsealed, and Raft indexes are caught up.

Useful checks:

```bash
kubectl -n vault get sts vault
kubectl -n vault get pods -l app.kubernetes.io/name=vault \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,REVISION:.metadata.labels.controller-revision-hash,IMAGE:.spec.containers[*].image'

for pod in vault-0 vault-1 vault-2; do
  echo "===== $pod ====="
  kubectl -n vault exec "$pod" -- vault status
  echo
done
```

Do not delete a second member until the previously replaced member is Ready,
unsealed, and caught up.

## Monitoring

Vault monitoring is managed as standalone GitOps resources in
`infrastructure/vault/monitoring.yaml` and applied by the `vault-monitoring`
Argo CD application. The `ServiceMonitor` selects the headless `vault-internal`
service, allowing Prometheus to scrape each of the three Vault members
individually on the named `http` port. Vault's listener permits unauthenticated
access to the metrics endpoint.

Vault-specific alerts cover:

- a server remaining sealed;
- no active HA server;
- more than one server reporting active;
- fewer than three healthy Vault telemetry targets;
- a pending manual `OnDelete` rollout.

The upstream `KubeStatefulSetUpdateNotRolledOut` rule is disabled and replaced
with a KernelCafe version that excludes only `vault/vault` so Vault's deliberate
upgrade workflow is not reported as a generic failed rollout. The separate
`VaultManualRolloutPending` warning remains visible after 30 minutes and points
to the controlled standby-first procedure above.

The monitoring design was verified against a three-member cluster: Prometheus
reported all three `vault-internal` targets UP, all three members unsealed,
exactly one active member, and all five Vault alert rules loaded and healthy.

## Auto-unseal

Auto-unseal is the preferred future state, but it must not be configured until
KernelCafe has an independent trusted seal provider. Integrated Raft storage is
not itself an auto-unseal provider, and using this same Vault cluster as its own
Transit seal creates a circular dependency.

Suitable providers include a supported cloud KMS/HSM or a separate,
independently operated Vault Transit service. The seal provider's credentials
must be supplied through an appropriate workload identity or Kubernetes Secret
that is not committed to this repository.

Migration procedure once a provider is selected:

1. Back up Vault Raft storage and verify the snapshot is recoverable.
2. Configure the new seal stanza and provider identity/credentials.
3. Follow HashiCorp's seal-migration procedure; do not simply replace the
   Shamir stanza and restart every node.
4. Migrate one member at a time and preserve Raft quorum.
5. Restart a migrated member and verify it unseals without operator key entry.
6. Verify all three members, HA leadership, Raft replication, telemetry, and
   External Secrets consumers.
7. Only after successful migration consider changing the StatefulSet rollout
   strategy. Keeping `OnDelete` remains acceptable even with auto-unseal when
   deliberate Vault upgrades are preferred.

Until an independent seal provider is chosen and provisioned, Shamir remains
the intentional seal mechanism.

# CloudNativePG PostgreSQL Clusters

CloudNativePG is installed as a cluster-wide operator by Flux from `infrastructure/controllers/cloudnative-pg.yaml`.
It can manage PostgreSQL clusters in any namespace.

CSI snapshot support is also managed by Flux. The repo references the upstream
`kubernetes-csi/external-snapshotter` repository at tag `v7.0.2` from
`infrastructure/controllers/csi-snapshot-controller.yaml`, then reconciles:

- `client/config/crd` for `snapshot.storage.k8s.io/v1` CRDs
- `deploy/kubernetes/snapshot-controller` into `kube-system`

This keeps upstream snapshot-controller manifests out of the repo while still
letting Flux reconcile them declaratively.

## Backup Model

The default pattern is snapshot-only:

- CloudNativePG creates `Backup` and `VolumeSnapshot` objects through `ScheduledBackup`.
- The cluster uses `backup.volumeSnapshot.className: longhorn-backup-vsc`.
- `longhorn-backup-vsc` is a Longhorn CSI `VolumeSnapshotClass` with `parameters.type: bak`.
- Longhorn stores the resulting backups in the configured Longhorn backup target.

This setup does not provide PITR because WAL archiving is intentionally out of
scope for v1. For production workloads that need low RPO or point-in-time
restore, add the CloudNativePG Barman Cloud Plugin and WAL archiving later.

## Adding a PostgreSQL Cluster

1. Copy `docs/examples/cloudnative-pg-cluster.yaml` to `apps/base/<purpose>-postgres.yaml`.
2. Replace the example namespace, cluster name, database name, owner, storage sizes, LoadBalancer IPs, FQDNs, and Secret name.
3. Create or sync the referenced Secret out-of-band, preferably through Infisical.
4. Add the new file to `apps/base/kustomization.yaml`.
5. Commit and push. Flux will deploy it through the existing `apps` Kustomization.

Recommended naming:

- Namespace: `postgres-<purpose>`, for example `postgres-grafana`
- Cluster: `<purpose>-db`, for example `grafana-db`
- ScheduledBackup: `<cluster>-daily-snapshot`

## External Access

CloudNativePG creates internal `ClusterIP` services by default:

- `<cluster>-rw` for the current primary
- `<cluster>-ro` for read-only replicas
- `<cluster>-r` for any instance

For external PostgreSQL access, use `managed.services.additional` in the cluster
spec. The example creates two CNPG-managed `LoadBalancer` services:

- `example-db-rw-lb` at `example-db-rw.kube.local.tnndev.com` for writes
- `example-db-ro-lb` at `example-db-ro.kube.local.tnndev.com` for read-only traffic

Each exposed database endpoint needs a unique MetalLB IP because PostgreSQL is
raw TCP and cannot be routed by HTTP hostname through the shared HTTP Gateway.
Point the FQDN at the assigned MetalLB IP in DNS. The `external-dns` annotation
is included for a future ExternalDNS setup, but it has no effect unless that
controller is installed.

## Backup Retention

`infrastructure/configs/cnpg-backup-retention.yaml` runs a daily cleanup CronJob
at `03:30`, after the default `02:00` database snapshot schedule.

The cleanup job deletes only completed CNPG `Backup` objects that are:

- older than 7 days
- owned by a `ScheduledBackup`
- associated with a `ScheduledBackup` labeled `homelab.tnndev.com/cnpg-retention=7d`

Each retained `ScheduledBackup` should also carry the annotation
`homelab.tnndev.com/cnpg-retention: 7d` to make the retention intent explicit.

Example:

```yaml
metadata:
  labels:
    homelab.tnndev.com/cnpg-retention: 7d
  annotations:
    homelab.tnndev.com/cnpg-retention: 7d
spec:
  backupOwnerReference: self
```

Cluster examples use `snapshotOwnerReference: backup`, so deleting the CNPG
`Backup` garbage-collects the related `VolumeSnapshot`. Because
`longhorn-backup-vsc` uses `deletionPolicy: Delete`, the corresponding Longhorn
backup content is deleted as part of cleanup.

## Restore Notes

Snapshot-only recovery restores to a specific retained backup, not an arbitrary
point in time. Start from a new cluster manifest and reference an existing CNPG
`Backup`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: restored-db
  namespace: postgres-example
spec:
  instances: 1
  bootstrap:
    recovery:
      backup:
        name: <backup-name>
  storage:
    storageClass: longhorn
    size: 10Gi
  walStorage:
    storageClass: longhorn
    size: 10Gi
```

Kubernetes Secrets are not included in CloudNativePG backups. Keep database
credentials recoverable through Infisical or another external secret source.

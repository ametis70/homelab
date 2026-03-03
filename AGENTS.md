# Agent Instructions

## Critical Rules - DO NOT VIOLATE

1. **NEVER commit or push without explicitly asking the user first**
2. **NEVER apply manifests to the cluster without explicitly asking the user first**
3. **NEVER delete files without checking their contents first and asking the user**
4. **Always respect the existing directory structure**

## Cluster Commands

Always prepend `kube` to any command that requires cluster authentication:

```bash
# Correct
kube kubectl get pods
kube flux reconcile kustomization flux-system
kube helm list

# Wrong - will fail
kubectl get pods
flux reconcile kustomization flux-system
helm list
```

Commands that need `kube` prefix:
- `kubectl`
- `flux`
- `helm`
- Any other command that interacts with the Kubernetes API

## Directory Structure

```
homelab/
├── apps/                    # Application deployments
│   └── <namespace>/         # Namespace name (e.g., home-assistant, external-secrets)
│       ├── kustomization.yaml   # Main kustomization, references apps and components
│       └── <app>/               # App name (often same as namespace)
│           ├── ks.yaml              # Flux Kustomization resource
│           └── resources/
│               ├── kustomization.yaml   # Kustomize kustomization for resources
│               ├── helm-release.yaml    # HelmRelease (if using Helm)
│               ├── helm-repository.yaml # HelmRepository (if using Helm)
│               ├── ocirepository.yaml   # OCIRepository (if using OCI)
│               └── <other-manifests>.yaml
├── components/              # Reusable Kustomize components
│   └── namespace/           # Namespace component
├── flux/                    # Flux system configuration
│   └── cluster/
│       └── ks.yaml          # Root Flux Kustomization
└── disabled/                # Disabled apps (not deployed)
```

### Apps Directory Details

Each app follows the pattern `apps/<namespace>/<app>/`:

```
apps/<namespace>/
├── kustomization.yaml           # Main kustomization, references ./*/ks.yaml and components
└── <app>/                       # One or more apps per namespace
    ├── ks.yaml                  # Flux Kustomization resource
    └── resources/
        ├── kustomization.yaml   # Kustomize kustomization for resources
        └── *.yaml               # K8s manifests (HelmRelease, PVC, IngressRoute, etc.)
```

- A namespace can have multiple apps (e.g., `external-secrets/external-secrets/` and `external-secrets/cluster/`)
- The namespace-level `kustomization.yaml` references all `<app>/ks.yaml` files
- Each `<app>/ks.yaml` is a Flux Kustomization pointing to its `resources/` directory

## Validation and Dry Runs

### Kustomize Build (dry run)

```bash
# Validate kustomization builds correctly
kustomize build apps/<namespace>/

# With Flux substitutions (if using postBuild)
kustomize build apps/<namespace>/ | envsubst
```

### Flux Dry Run

```bash
# Dry run a Flux Kustomization
kube flux diff kustomization <name> --path ./apps/<namespace>/
```

### Kubeconform Validation

```bash
# Validate manifests against Kubernetes schemas
kustomize build apps/<namespace>/ | kubeconform -strict -summary

# With CRD schemas for Flux, etc.
kustomize build apps/<namespace>/ | kubeconform -strict -summary \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

## Storage Classes

Three Longhorn storage classes, each with different replication, backup, and reclaim policies:

| Storage Class | Replicas | Reclaim Policy | Recurring Jobs | Use Case |
|---|---|---|---|---|
| `longhorn` (default) | 3 | Retain | Daily snapshots (6 AM, retain 7), weekly backups (Sun 6:15 AM, retain 4), snapshot cleanup every 6h | General app data |
| `longhorn-single` | 1 (strict-local) | Retain | Weekly backup only (Sun 6:15 AM, retain 4) | CNPG postgres clusters |
| `longhorn-ephemeral` | 1 | Delete | Snapshot cleanup every 12h only | Redis caches, Prometheus, Loki, ML caches |

Recurring jobs are defined in `apps/longhorn/longhorn/resources/jobs.yaml`. Jobs are assigned to storage classes via `recurringJobSelector` in the SC parameters (group `default` for longhorn, individual job for longhorn-single, group `ephemeral` for longhorn-ephemeral).

## Flux Pruning and PVC Protection

Flux pruning controls whether Flux deletes resources from the cluster when they are removed from git. The strategy in this repo is:

- **Flux Kustomizations (`ks.yaml`)**: Most use `prune: false` for safety. Some stateless/infra apps use `prune: true`.
- **PVCs**: ALL durable PVCs MUST have the annotation `kustomize.toolkit.fluxcd.io/prune: disabled`, regardless of the Kustomization's prune setting. This is a safety net to prevent accidental data loss if prune is ever enabled.
- **Ephemeral PVCs** (e.g., Redis on `longhorn-ephemeral`): Do NOT need the prune-disabled annotation since their data is disposable.

Example durable PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-config
  namespace: app
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: longhorn
```

All PVCs are dynamically provisioned (no static PVs). Longhorn storage classes use `reclaimPolicy: Retain` to prevent accidental PV deletion when a PVC is removed.

## CNPG Postgres Backups (Barman)

CNPG clusters use the barman-cloud plugin for continuous backup to S3-compatible storage (Garage). The backup infrastructure is defined as reusable Kustomize components in `components/cnpg-cluster/`.

### Component Structure

- **`components/cnpg-cluster/base/`** — Shared resources: ObjectStore (S3 config), ExternalSecret (S3 credentials), ScheduledBackups, and patches for storage class (`longhorn-single`), pod security context, and WAL archive checks.
- **`components/cnpg-cluster/normal/`** — Steady-state operation. Patches the Cluster to remove `bootstrap` (no-op on running clusters) and add the barman-cloud plugin as WAL archiver + volumeSnapshot backup config.
- **`components/cnpg-cluster/restore/`** — Restore mode. Patches the Cluster to bootstrap via `recovery` from the latest barman backup. Removes `initdb` bootstrap. Uses `replacements` to auto-fill the backup source name from the cluster's namespace/name.
- **`components/cnpg-cluster/init/`** — Initial creation. For bootstrapping a brand-new cluster with `initdb` (no restore).

### Backup Flow (normal operation)

Each CNPG cluster with the `normal` component gets:
1. **Continuous WAL archiving** via barman-cloud plugin → S3 (Garage)
2. **Daily barman backup** at 4:15 AM (`cnpg-backup-barman-daily`)
3. **Volume snapshots** (currently suspended: `cnpg-backup-volumesnapshot-daily`)
4. **Retention**: 1 month of backups in the ObjectStore

### Restore Procedure

To restore a CNPG cluster from its latest barman backup:

1. Verify the backup exists: `kube kubectl get backups.postgresql.cnpg.io -n <namespace>`
2. Stop app workloads: `kube kubectl scale deployment/<app> -n <namespace> --replicas=0`
3. Change component from `normal` to `restore` in `apps/<namespace>/<app>/resources/kustomization.yaml`
4. Delete the cluster: `kube kubectl delete cluster.postgresql.cnpg.io <name>-postgres -n <namespace>` (PVC auto-deletes)
5. Commit and push the change
6. Flux reconciles → CNPG creates new cluster → full recovery from barman backup
7. Verify healthy: `kube kubectl get cluster.postgresql.cnpg.io -n <namespace>` (wait for "Cluster in healthy state")
8. Switch component back to `normal` (re-enables WAL archiving and backups)
9. Commit, push, and reconcile
10. Restart app workloads: `kube kubectl scale deployment/<app> -n <namespace> --replicas=1`

**Important**: After restoring, you MUST switch back to `normal` — otherwise the cluster has no WAL archiving or ongoing backups, and a future Flux reconciliation would attempt to re-bootstrap from backup.

## Before Making Changes

1. Read and understand the existing structure
2. Check what files exist before modifying or deleting
3. Validate changes with dry runs
4. **ASK the user before committing, pushing, or applying**

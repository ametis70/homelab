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

## Before Making Changes

1. Read and understand the existing structure
2. Check what files exist before modifying or deleting
3. Validate changes with dry runs
4. **ASK the user before committing, pushing, or applying**

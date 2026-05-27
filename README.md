# hub-gitops

Hub-only Helm chart. Install once on the cluster that runs ACM + hub OpenShift GitOps. After install, Argo CD manages the rest automatically.

> For a full end-to-end setup guide see [RUNBOOK.md](RUNBOOK.md).

## What it installs

- `ManagedClusterSet` (one per entry in `clusterSets`) — `rosa-platform` for ROSA spokes, `management` for the hub itself
- `ManagedClusterSetBinding` (one per set) — exposes each set to `openshift-gitops` namespace
- `Placement` (one per entry in `placements`) — selects clusters by `platform/clusterType` label within the named clusterSet
- `GitOpsCluster` (one per Placement) — registers matched clusters in hub Argo CD
- `ApplicationSet` (one per entry in `applicationSets`) — fans out `helm-catalog/spoke-argocd-bootstrap` to matched clusters

Bootstrap chart lives in the **helm-catalog** repo. This chart only installs Argo CD and ACM objects.

## Install

```bash
# From a local checkout of hub-gitops/
helm install hub-platform . \
  --namespace openshift-gitops \
  --create-namespace
```

Override values at install time or edit [values.yaml](values.yaml) first:

```bash
helm install hub-platform . \
  --namespace openshift-gitops \
  --create-namespace \
  --set global.helmCatalogRepoURL=https://github.com/YOUR_ORG/helm-catalog.git \
  --set global.platformAppsRepoURL=https://github.com/YOUR_ORG/platform-apps.git \
  --set global.platformConfigRepoURL=https://github.com/YOUR_ORG/platform-config.git
```

## Upgrade

```bash
helm upgrade hub-platform . --namespace openshift-gitops
```

## Uninstall

```bash
helm uninstall hub-platform --namespace openshift-gitops
```

Note: ACM `ManagedClusterSet` resources have a finalizer. Verify no managed clusters depend on them before uninstalling.

## Key values ([values.yaml](values.yaml))

| Key | Purpose |
|-----|---------|
| `global.helmCatalogRepoURL` | Source for `spoke-argocd-bootstrap` chart |
| `global.platformAppsRepoURL` | Passed into bootstrap chart (root app source) |
| `global.platformConfigRepoURL` | Passed into bootstrap chart (config source) |
| `clusterSets` | List of `{name}` — one `ManagedClusterSet` + `ManagedClusterSetBinding` per entry |
| `placements` | List of `{name, clusterSet, clusterTypes[]}` — one `Placement` + `GitOpsCluster` per entry |
| `applicationSets` | List of `{name, placement, requeueAfterSeconds, extraValues?}` — one `ApplicationSet` per entry |
| `applicationSets[].extraValues` | Optional key/value pairs merged into the bootstrap chart's helm values (e.g. `installOpenShiftGitOps: false` for the hub) |

## Required managed cluster labels

### Spoke clusters

Apply after import so the ApplicationSet starts selecting them:

```bash
oc label managedcluster <name> \
  cluster.open-cluster-management.io/clusterset=rosa-platform \
  platform/clusterType=dev \
  platform/clusterGroup=<group> \
  platform/region=<region> \
  platform/version=main \
  --overwrite
```

### Hub cluster (self-management)

```bash
oc label managedcluster local-cluster \
  cluster.open-cluster-management.io/clusterset=management \
  platform/clusterType=hub \
  platform/clusterGroup=eng \
  platform/region=eu-west-1 \
  platform/version=main \
  --overwrite
```

## Adding a new environment tier (e.g. preprod bootstrap)

1. Add an entry to `applicationSets` in `values.yaml` pointing at the appropriate placement.
2. Run `helm upgrade hub-platform . --namespace openshift-gitops`.

## Local render (dry-run / review before installing)

```bash
helm template hub-platform . --namespace openshift-gitops
```

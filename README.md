# hub-gitops

Hub-only Helm chart. Install once on the cluster that runs ACM + hub OpenShift GitOps. After install, Argo CD manages the rest automatically.

## What it installs

- `ManagedClusterSet` — groups all ROSA spokes (`rosa-platform`)
- `ManagedClusterSetBinding` — exposes the set to `openshift-gitops` namespace
- `Placement` (one per environment tier) — selects spokes by `platform/clusterType` label
- `GitOpsCluster` (one per Placement) — registers matched clusters in hub Argo CD
- `ApplicationSet` (`spoke-argocd-bootstrap`) — fans out `helm-catalog/spoke-argocd-bootstrap` to each selected spoke

Bootstrap charts live in the **helm-catalog** repo. This chart only installs Argo CD and ACM objects.

## Install

```bash
# From a local checkout
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

Note: ACM `ManagedClusterSet` resources have a finalizer. Verify no managed clusters depend on it before uninstalling.

## Key values ([values.yaml](values.yaml))

| Key | Purpose |
|-----|---------|
| `global.helmCatalogRepoURL` | Source for `spoke-argocd-bootstrap` chart |
| `global.platformAppsRepoURL` | Passed into bootstrap chart (spoke root app source) |
| `global.platformConfigRepoURL` | Passed into bootstrap chart (spoke config source) |
| `clusterSet.name` | ACM ManagedClusterSet name |
| `placements` | List of `{name, clusterTypes[]}` — one Placement + GitOpsCluster per entry |
| `applicationSet.placement` | Which Placement drives the bootstrap fan-out |

## Required managed cluster labels

Apply after install so the ApplicationSet starts selecting spokes:

```bash
oc label managedcluster <name> \
  cluster.open-cluster-management.io/clusterset=rosa-platform \
  platform/clusterType=dev|preprod|prod \
  platform/clusterGroup=<group> \
  platform/region=<region> \
  platform/version=main \
  --overwrite
```

## Adding preprod/prod bootstrap

Change `applicationSet.placement` in `values.yaml` or add a second ApplicationSet template, then run `helm upgrade`.

## Local render (dry-run / review before installing)

```bash
helm template hub-platform . --namespace openshift-gitops
```

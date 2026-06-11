# hub-gitops

Hub-only Helm chart. Install once on the cluster that runs ACM + hub OpenShift GitOps. After install, Argo CD manages the rest automatically.

> For a full end-to-end setup guide see [RUNBOOK.md](RUNBOOK.md).

## What it installs

- `ManagedClusterSet` (one per entry in `clusterSets`) — `dev`, `preprod`, `prod` for ROSA spokes; `management` for the hub itself
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

> **Important — this chart is NOT managed by Argo CD.**
> Unlike spoke workloads, `hub-gitops` is a manually installed Helm release. Changes to `values.yaml` or any template (ApplicationSets, Placements, ClusterSets) are **not picked up automatically** when you push to Git. You must run `helm upgrade` explicitly after every change.
>
> Common changes that require an upgrade:
> - Adding or removing an entry in `applicationSets`, `clusterSets`, or `placements`
> - Changing `clusterType`, `region`, `prune`, or `requeueAfterSeconds` on an ApplicationSet
> - Updating any of the three `global.*RepoURL` values
>
> After upgrading, the updated `ApplicationSet` resources are re-applied immediately. Any in-flight `Application` objects they manage (e.g. `platform-bootstrap-digital`) will be regenerated on the next ApplicationSet reconcile cycle (`requeueAfterSeconds`, default 180s) — or trigger an immediate refresh using the commands in [Forcing ApplicationSet reconciliation](RUNBOOK.md#forcing-applicationset-reconciliation) in the runbook.

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
| `clusterSets` | List of `{name}` — one `ManagedClusterSet` + `ManagedClusterSetBinding` per entry. Sets: `dev`, `preprod`, `prod`, `management` |
| `placements` | List of `{name, clusterSet, clusterTypes[]}` — one `Placement` + `GitOpsCluster` per entry |
| `applicationSets` | List of `{name, placement, requeueAfterSeconds, prune, extraValues?}` — one `ApplicationSet` per entry |
| `applicationSets[].prune` | `true` = bootstrap Application is deleted when cluster label is removed. Default `false`. Set `false` for prod to prevent accidental teardown |
| `applicationSets[].extraValues` | Optional key/value pairs merged into the bootstrap chart's Helm values (e.g. `installOpenShiftGitOps: false` for the hub) |

## Required managed cluster labels

### Spoke clusters

Apply after import so the correct ApplicationSet starts selecting the cluster:

```bash
oc label managedcluster <name> \
  cluster.open-cluster-management.io/clusterset=<dev|preprod|prod> \
  platform/clusterType=<dev|preprod|prod> \
  platform/clusterGroup=<payments|digital|mortgages> \
  platform/region=eu-west-1 \
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

## Sync and prune behaviour

Each `ApplicationSet` in `values.yaml` has `automated.selfHeal: true` (always on) and a configurable `prune` toggle:

| Tier | Default `prune` | Rationale |
|------|----------------|-----------|
| `bootstrap-dev` | `true` | Clean up stale resources automatically |
| `bootstrap-preprod` | `true` | Mirror dev behaviour for realism |
| `bootstrap-prod` | `false` | Never auto-delete prod workloads — manual action required |
| `hub-self-management` | `true` | Hub manages itself; removing the label is intentional |

To override: edit `applicationSets[].prune` in `values.yaml` and run `helm upgrade`.

## Adding a new environment tier (e.g. staging)

1. Add a new entry to `clusterSets`, `placements`, and `applicationSets` in `values.yaml`.
2. Run `helm upgrade hub-platform . --namespace openshift-gitops`.

## Local render (dry-run / review before installing)

```bash
helm template hub-platform . --namespace openshift-gitops
```

# Platform GitOps — End-to-End Runbook

This document walks through standing up the full two-tier GitOps platform from scratch: a hub ROSA cluster running ACM and OpenShift GitOps, and one or more spoke ROSA clusters that are bootstrapped and self-managed automatically.

---

## Architecture overview

```
Hub cluster (ACM + OpenShift GitOps)
│
├── hub-gitops Helm chart (installed manually once)
│   ├── ManagedClusterSets: dev / preprod / prod / management
│   ├── Placements (one per tier)
│   ├── GitOpsClusters (link placements → Argo CD)
│   └── ApplicationSets
│       ├── bootstrap-dev       → installs GitOps + root app on each dev spoke
│       ├── bootstrap-preprod   → installs GitOps + root app on each preprod spoke
│       ├── bootstrap-prod      → installs GitOps + root app on each prod spoke
│       └── hub-self-management → installs root app on hub (GitOps already present)
│
└── Root Application on hub (via hub-self-management)
    └── platform-apps chart → deploys ACS, alertmanager, operators on hub

Spoke cluster (bootstrapped automatically)
└── Root Application (deployed by spoke-argocd-bootstrap)
    └── platform-apps chart → deploys workloads defined in platform-config
```

### Config merge order (four layers, later overrides earlier)

```
default.yaml
  → regions/<region>/region.yaml
    → clusters/<clusterType>/common.yaml
      → clusters/<clusterType>/<clusterGroup>/clusterdef.yaml
```

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| `oc` CLI | Logged in to both clusters |
| `helm` CLI | v3.x |
| `git` CLI | |
| Hub ROSA cluster | ACM operator and OpenShift GitOps operator already installed |
| Spoke ROSA cluster | Untouched — GitOps will be installed by the bootstrap |
| 4 GitHub repos created | `hub-gitops`, `platform-apps`, `platform-config`, `helm-catalog` |

---

## Part 1 — Hub setup

### Step 1 — Push the repos to GitHub

Each subdirectory under `gitops-rework/` maps to one GitHub repo.

```bash
# From gitops-rework/

# hub-gitops
cd hub-gitops
git init && git add . && git commit -m "initial"
git remote add origin https://github.com/YOUR_ORG/hub-gitops.git
git push -u origin main

# platform-apps
cd ../platform-apps
git init && git add . && git commit -m "initial"
git remote add origin https://github.com/YOUR_ORG/platform-apps.git
git push -u origin main

# platform-config
cd ../platform-config
git init && git add . && git commit -m "initial"
git remote add origin https://github.com/YOUR_ORG/platform-config.git
git push -u origin main

# helm-catalog
cd ../helm-catalog
git init && git add . && git commit -m "initial"
git remote add origin https://github.com/YOUR_ORG/helm-catalog.git
git push -u origin main
```

---

### Step 2 — Update repo URLs in values files

Edit these two files with your actual GitHub org before installing:

**`hub-gitops/values.yaml`** — the three `global.*RepoURL` fields:

```yaml
global:
  helmCatalogRepoURL:    https://github.com/YOUR_ORG/helm-catalog.git
  platformAppsRepoURL:   https://github.com/YOUR_ORG/platform-apps.git
  platformConfigRepoURL: https://github.com/YOUR_ORG/platform-config.git
```

**`helm-catalog/spoke-argocd-bootstrap/values.yaml`** — the two repo URLs:

```yaml
platformAppsRepoURL:   https://github.com/YOUR_ORG/platform-apps.git
platformConfigRepoURL: https://github.com/YOUR_ORG/platform-config.git
```

Commit and push both changes after editing.

---

### Step 3 — Install the hub-gitops Helm chart

Log in to the hub and install from a local checkout of `hub-gitops/`:

```bash
oc login <hub-api-url> --token=<hub-token>

helm install hub-platform . \
  --namespace openshift-gitops \
  --create-namespace
```

This creates:
- Four `ManagedClusterSets` (`dev`, `preprod`, `prod`, `management`)
- Four `ManagedClusterSetBindings`
- Four `Placements` (dev / preprod / prod / hub)
- Four `GitOpsClusters`
- Four `ApplicationSets` (`bootstrap-dev`, `bootstrap-preprod`, `bootstrap-prod`, `hub-self-management`)

Dry-run first if you want to review:

```bash
helm template hub-platform . --namespace openshift-gitops
```

---

### Step 4 — Label local-cluster for hub self-management

This triggers the `hub-self-management` ApplicationSet to deploy the root app-of-apps on the hub itself:

```bash
oc label managedcluster local-cluster \
  cluster.open-cluster-management.io/clusterset=management \
  platform/clusterType=hub \
  platform/clusterGroup=eng \
  platform/region=eu-west-1 \
  platform/version=main \
  --overwrite
```

> This must be done after Step 3 — the `management` ManagedClusterSet must exist before `local-cluster` can join it.

---

### Step 5 — Verify hub self-management

```bash
# hub-self-management ApplicationSet should have generated one Application
oc get applicationset hub-self-management -n openshift-gitops

# Root Application for the hub should be visible
oc get application platform-bootstrap-local-cluster -n openshift-gitops

# platform-apps child Applications (ACS, alertmanager, operators) deploy from here
oc get application -n openshift-gitops
```

---

## Part 2 — Spoke cluster onboarding

### Step 6 — Create a cluster definition for your spoke

Each cluster needs exactly one `clusterdef.yaml` under `platform-config/clusters/<type>/<group>/`.
The folder name becomes the `clusterGroup` label value.

```
platform-config/clusters/
  dev/
    payments/clusterdef.yaml    ← payments dev cluster
    digital/clusterdef.yaml     ← digital dev cluster
    mortgages/clusterdef.yaml   ← mortgages dev cluster
  preprod/
    payments/clusterdef.yaml    ← payments preprod cluster
    ...
```

Copy the nearest example and edit:

```bash
# Example: new preprod cluster for the "biscuits" team
mkdir -p platform-config/clusters/preprod/biscuits
cp platform-config/clusters/preprod/payments/clusterdef.yaml \
   platform-config/clusters/preprod/biscuits/clusterdef.yaml
```

Minimum fields to update in the new file:

| Field | Value |
|-------|-------|
| `clusterGroup` | `biscuits` (matches folder name) |
| `clusterName` | `biscuits-preprod-eu-west-1` (or your chosen name) |
| `region` | `eu-west-1` (must match a directory under `regions/`) |
| `stackrox.clusterName` | `biscuits-preprod` |
| `rosa.machinePools.platform.clusterID` | fill in after cluster import (Step 7) |

```bash
# Find the ROSA infrastructure ID (needed if enabling MachinePools)
oc get infrastructure cluster \
  -o jsonpath='{.status.infrastructureName}' \
  --context <spoke-context>
```

Commit and push `platform-config`.

---

### Step 7 — Import the spoke cluster into ACM

#### Option A — ACM console (easiest)

1. Open the ACM console → **Infrastructure → Clusters → Import cluster**
2. Enter the spoke's API URL and kubeconfig / token
3. ACM will create a `ManagedCluster` resource named after the cluster

#### Option B — CLI

```bash
cat <<EOF | oc apply -f -
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: <spoke-cluster-name>
spec:
  hubAcceptsClient: true
EOF
```

Follow the import instructions that ACM surfaces (it will print a `kubectl` command to run on the spoke).

---

### Step 8 — Label the spoke ManagedCluster

Once the spoke shows `Available=True` in ACM, apply the platform labels so the Placement can select it:

```bash
oc label managedcluster <spoke-cluster-name> \
  cluster.open-cluster-management.io/clusterset=<dev|preprod|prod> \
  platform/clusterType=<dev|preprod|prod> \
  platform/clusterGroup=<group>        \
  platform/region=eu-west-1            \
  platform/version=main                \
  --overwrite
```

**Label → folder mapping:**

| Label | Must match |
|-------|-----------|
| `clusterset` | A `ManagedClusterSet` name: `dev`, `preprod`, or `prod` |
| `platform/clusterType` | Same as `clusterset` value (`dev`, `preprod`, `prod`) |
| `platform/clusterGroup` | Folder name under `clusters/<clusterType>/` (e.g. `payments`, `biscuits`) |
| `platform/region` | Folder name under `regions/` (e.g. `eu-west-1`) |
| `platform/version` | Git branch / tag for chart revisions (usually `main`) |

**Example — biscuits preprod:**

```bash
oc label managedcluster biscuits-preprod-1 \
  cluster.open-cluster-management.io/clusterset=preprod \
  platform/clusterType=preprod \
  platform/clusterGroup=biscuits \
  platform/region=eu-west-1 \
  platform/version=main \
  --overwrite
```

---

### Step 9 — Watch the spoke rollout

```bash
# bootstrap-preprod ApplicationSet should have generated an Application for the spoke
oc get application platform-bootstrap-biscuits-preprod-1 -n openshift-gitops -w
```

Switch to the spoke cluster to follow the bootstrap sequence:

```bash
oc login <spoke-api-url> --token=<spoke-token>

# OpenShift GitOps operator subscription (created by bootstrap)
oc get subscription openshift-gitops-operator -n openshift-operators

# Once GitOps is up, the root Application appears
oc get application -n openshift-gitops

# platform-apps child Applications deploy from the root app
oc get application -n openshift-gitops
```

#### Expected sequence

```
bootstrap-<tier> ApplicationSet fires
  → Subscription + Namespace created on spoke (sync-wave -2, -1)
  → OpenShift GitOps operator installs (~2-3 min)
  → Root Application created in openshift-gitops namespace (sync-wave 1)
  → platform-apps Helm chart renders child Applications
  → Child Applications sync their workloads
```

---

## Adding a new cluster (summary)

1. Create `platform-config/clusters/<type>/<group>/clusterdef.yaml` — one folder = one cluster
2. Commit and push `platform-config`
3. Import the cluster into ACM (Step 7)
4. Apply the labels (Step 8) — the correct `bootstrap-<tier>` ApplicationSet selects it automatically within `requeueAfterSeconds` (180s)

No changes to `hub-gitops` or any ApplicationSet are needed for a new cluster of the same type.

---

## Troubleshooting

### ApplicationSet not generating Applications

```bash
# Check the Placement is binding to clusters
oc get placementdecision -n openshift-gitops

# Verify labels on the ManagedCluster
oc get managedcluster <name> -o jsonpath='{.metadata.labels}' | jq
```

Common causes:
- Wrong `cluster.open-cluster-management.io/clusterset` label — must match the `ManagedClusterSet` name exactly (`dev`, `preprod`, `prod`, or `management`)
- `ManagedClusterSetBinding` missing — re-run `helm upgrade hub-platform .`
- `platform/clusterType` label value does not match any `placement.clusterTypes` entry

### Spoke GitOps operator stuck pending

```bash
oc get installplan -n openshift-operators --context <spoke-context>
```

The subscription uses `installPlanApproval: Automatic` so it should self-approve. If stuck, check that `openshift-marketplace` is healthy on the spoke:

```bash
oc get catalogsource -n openshift-marketplace --context <spoke-context>
```

### Root Application created but child apps missing

The root Application sources `platform-apps` as a Helm chart. Check that the `valueFiles` paths exist in `platform-config` for the cluster's type/group combination:

```bash
# Expected files (adjust for your cluster):
platform-config/default.yaml
platform-config/regions/eu-west-1/region.yaml
platform-config/clusters/dev/common.yaml
platform-config/clusters/dev/payments/clusterdef.yaml
```

`ignoreMissingValueFiles: true` is set, so missing files are silently skipped — but a missing `clusterdef.yaml` means no cluster-specific overrides will apply. The cluster may still work but with only defaults from `common.yaml`.

### Argo CD repo credentials

If your GitHub repos are private, add repository credentials in the hub Argo CD before installing:

```bash
# Via Argo CD CLI
argocd repo add https://github.com/YOUR_ORG/platform-config.git \
  --username <user> --password <token>

# Or create a Secret directly
oc create secret generic platform-config-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/YOUR_ORG/platform-config.git \
  --from-literal=password=<token> \
  --from-literal=username=<user> \
  -n openshift-gitops
oc label secret platform-config-repo \
  argocd.argoproj.io/secret-type=repository \
  -n openshift-gitops
```

Repeat for each of the four repos.

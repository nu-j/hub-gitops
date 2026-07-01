# Platform GitOps — End-to-End Runbook

This document walks through standing up the full two-tier GitOps platform from scratch: a management ROSA cluster running ACM and OpenShift GitOps, and one or more spoke ROSA clusters that are bootstrapped and self-managed automatically.

---

## Architecture overview

```
Management cluster (ACM + OpenShift GitOps)
│
├── hub-gitops Helm chart (installed manually once)
│   ├── ManagedClusterSets: dev / preprod / prod / management
│   ├── Placements (one per tier)
│   ├── GitOpsClusters (link placements → Argo CD)
│   └── ApplicationSets
│       ├── bootstrap-dev       → installs GitOps + root app on each dev spoke
│       ├── bootstrap-preprod   → installs GitOps + root app on each preprod spoke
│       ├── bootstrap-prod      → installs GitOps + root app on each prod spoke
│       └── hub-self-management → installs root app on management cluster (GitOps already present)
│
└── Root Application on management cluster (via hub-self-management)
    └── platform-apps chart → deploys ACS, alertmanager, operators on management cluster

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
| Hub management ROSA cluster | ACM operator and OpenShift GitOps operator already installed |
| Spoke ROSA cluster | Untouched — GitOps will be installed by the bootstrap |
| 4 GitHub repos created | `hub-gitops`, `platform-apps`, `platform-config`, `helm-catalog` |

---

## Part 1 — Management cluster setup

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

Log in to the management cluster and install from a local checkout of `hub-gitops/`:

```bash
oc login <management-api-url> --token=<management-token>

helm install hub-platform . \
  --namespace openshift-gitops \
  --create-namespace
```

This creates:
- Four `ManagedClusterSets` (`dev`, `preprod`, `prod`, `management`)
- Four `ManagedClusterSetBindings`
- Four `Placements` (dev / preprod / prod / management)
- Four `GitOpsClusters`
- Four `ApplicationSets` (`bootstrap-dev`, `bootstrap-preprod`, `bootstrap-prod`, `hub-self-management`)

Dry-run first if you want to review:

```bash
helm template hub-platform . --namespace openshift-gitops
```

---

### Step 4 — Label local-cluster for management self-management

This triggers the `hub-self-management` ApplicationSet to deploy the root app-of-apps on the management cluster itself:

```bash
oc label managedcluster local-cluster \
  cluster.open-cluster-management.io/clusterset=management \
  platform/clusterType=management \
  platform/clusterGroup=eng-hub \
  platform/region=eu-west-1 \
  platform/version=main \
  --overwrite
```

> This must be done after Step 3 — the `management` ManagedClusterSet must exist before `local-cluster` can join it.

---

### Step 5 — Verify management self-management

```bash
# hub-self-management ApplicationSet should have generated one Application
oc get applicationset hub-self-management -n openshift-gitops

# Root Application for the management cluster should be visible
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

Two distinct categories of label are applied here — understanding the difference matters:

| Label | Purpose | Required? |
|-------|---------|-----------|
| `cluster.open-cluster-management.io/clusterset` | **Functional** — ACM uses this to match the cluster to a `Placement`, which triggers the correct `bootstrap-<tier>` ApplicationSet | Yes — nothing fires without it |
| `platform/clusterType`, `platform/clusterGroup`, `platform/region` | **Informational** — for `oc describe`, ACM console filtering, and visual inspection on-cluster. These are NOT used by the ApplicationSet to resolve platform-config paths (that is handled by `hub-gitops/values.yaml` in Git) | Recommended for operability |

```bash
oc label managedcluster <spoke-cluster-name> \
  cluster.open-cluster-management.io/clusterset=<dev|preprod|prod> \
  platform/clusterType=<dev|preprod|prod> \
  platform/clusterGroup=<group>        \
  platform/region=eu-west-1            \
  platform/version=main                \
  --overwrite
```

> **Convention**: the ManagedCluster `name` must equal the `clusterGroup` (e.g. a cluster named `digital` resolves `clusters/dev/digital/clusterdef.yaml`). The `name` is set at import and cannot change without re-importing.

**Label → folder mapping:**

| Label | Must match |
|-------|-----------|
| `clusterset` | A `ManagedClusterSet` name: `dev`, `preprod`, or `prod` |
| `platform/clusterType` | Informational — matches the `clusterset` value |
| `platform/clusterGroup` | Informational — matches the ManagedCluster `name` and the folder under `clusters/<clusterType>/` |
| `platform/region` | Informational — matches a directory under `regions/` |
| `platform/version` | Informational — Git branch / tag used when reading platform-config |

**Example — digital dev:**

```bash
oc label managedcluster digital \
  cluster.open-cluster-management.io/clusterset=dev \
  platform/clusterType=dev \
  platform/clusterGroup=digital \
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

Once all Applications are `Synced / Healthy`, verify the merged config that was applied to the spoke:

```bash
# Inspect the fully merged platform-config values in effect on this cluster.
# This ConfigMap is written by the platform-config-snapshot Application on every sync.
oc get configmap platform-config-snapshot -n openshift-gitops \
  -o jsonpath='{.data.values\.yaml}' | yq
```

Run the same command on the management cluster to compare its effective config:

```bash
oc get configmap platform-config-snapshot -n openshift-gitops \
  -o jsonpath='{.data.values\.yaml}' \
  --context <management-context> | yq
```

---

## Adding a new cluster (summary)

1. Create `platform-config/clusters/<type>/<group>/clusterdef.yaml` — folder name must match the ManagedCluster name you will use at import
2. Commit and push `platform-config`
3. Import the cluster into ACM (Step 7)
4. Apply the `clusterset` label (Step 8) — the correct `bootstrap-<tier>` ApplicationSet selects it automatically within `requeueAfterSeconds` (180s)

No changes to `hub-gitops` or any ApplicationSet are needed for a new cluster of the same type and region.

---

## Troubleshooting

### ApplicationSet not generating Applications

```bash
# Check the Placement is binding to clusters
oc get placementdecision -n openshift-gitops

# Verify the clusterset label on the ManagedCluster (this is the functional trigger)
oc get managedcluster <name> -o jsonpath='{.metadata.labels}' | jq
```

Common causes:
- Wrong `cluster.open-cluster-management.io/clusterset` label — must match the `ManagedClusterSet` name exactly (`dev`, `preprod`, `prod`, or `management`). This is the only label the ApplicationSet depends on functionally.
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

If the cluster is up, the fastest way to see which values were actually merged is the config snapshot ConfigMap:

```bash
oc get configmap platform-config-snapshot -n openshift-gitops \
  -o jsonpath='{.data.values\.yaml}' | yq '.platform.argocdApps'
```

If `platform-config-snapshot` itself is missing, it means either the Application has not synced yet or `platformConfigSnapshot.enabled` is `false` in the merged values — check with:

```bash
oc get application platform-config-snapshot -n openshift-gitops
```

### Argo CD repo credentials

If your GitHub repos are private, add repository credentials in the management cluster's Argo CD before installing:

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

---

## Forcing ApplicationSet reconciliation

ApplicationSets re-evaluate on a timer (`requeueAfterSeconds: 180`). To trigger an immediate reconcile without waiting:

```bash
# Kick a specific ApplicationSet — fastest option
oc annotate applicationset bootstrap-dev \
  argocd.argoproj.io/refresh=normal \
  -n openshift-gitops --overwrite

# After pushing changes to hub-gitops or platform-apps, hard-refresh the hub root app
# to force a cache-busting git fetch and re-render of all ApplicationSet resources
oc annotate application eng-hub-aoa \
  argocd.argoproj.io/refresh=hard \
  -n openshift-gitops --overwrite

# Force-refresh a specific bootstrap Application (e.g. after re-labelling a cluster)
oc annotate application platform-bootstrap-digital \
  argocd.argoproj.io/refresh=hard \
  -n openshift-gitops --overwrite
```

Via the Argo CD CLI:

```bash
# Refresh an ApplicationSet
argocd appset get bootstrap-dev --refresh

# Hard-refresh a generated Application
argocd app get platform-bootstrap-digital --hard-refresh
```

**When to use each:**

| Situation | Command |
|-----------|---------|
| Cluster labelled but ApplicationSet hasn't fired yet | `annotate applicationset ... refresh=normal` |
| `hub-gitops` values changed (`helm upgrade` already run) | `annotate application eng-hub-aoa refresh=hard` |
| Bootstrap Application exists but has stale rendered values | `annotate application platform-bootstrap-<name> refresh=hard` |
| Spoke root Application not picking up platform-config changes | Sync the spoke root Application directly from the spoke Argo CD UI |

---

## Part 3 — Team self-service onboarding

Each team on a spoke cluster gets their own isolated Argo CD instance, scoped to their namespaces.
The platform team registers the team in `platform-config`; the team manages all applications
themselves from that point.

### How it works

```
platform-config/clusters/dev/digital/clusterdef.yaml
  teams:
    my-team:
      gitopsRepoURL: https://github.com/YOUR_ORG/my-team-gitops.git
        │
        ▼
platform-apps (spoke root app) renders team-argocd-app.yaml
  → Application: team-argocd-my-team (in openshift-gitops)
        │
        ▼
helm-catalog/team-argocd deploys to spoke:
  wave -2  Namespace: my-team-gitops
  wave -1  ClusterRoleBinding: team SA can create namespaces
  wave  0  ArgoCD CR: operator provisions team ArgoCD stack
  wave  1  AppProject: restricts team to my-team-* namespaces
  wave  2  Application: my-team-root → my-team-gitops.git
        │
        ▼
Team ArgoCD reconciles my-team-root
  → reads my-team-gitops.git/values.yaml
  → generates child Applications (ubiquitous-journey, pet-battle/test, pet-battle/stage)
        │
        ▼
Team's bootstrap Application deploys helm-catalog/bootstrap-project
  → creates my-team-tools, my-team-pet-battle-dev, my-team-pet-battle-test, my-team-pet-battle-stage
  → each namespace labelled: argocd.argoproj.io/managed-by: my-team-gitops
  → OpenShift GitOps operator auto-grants team ArgoCD SA deploy rights in each namespace
```

### Step 10 — Register the team in platform-config

Add the team to the relevant cluster's `clusterdef.yaml`. The key is the team name — it drives
all derived names (`<team>-gitops` namespace, `<team>-*` AppProject destinations):

```bash
# Edit platform-config/clusters/dev/digital/clusterdef.yaml
```

```yaml
teams:
  my-team:
    gitopsRepoURL: https://github.com/YOUR_ORG/my-team-gitops.git
    targetRevision: main
    # adGroupName: MY_TEAM_AD_GROUP    # optional: grants team UI access to their ArgoCD
```

`teams` is a **map** — entries at `common.yaml` and `clusterdef.yaml` layers are merged
together, so a team defined at the environment level coexists with cluster-specific teams.

Commit and push `platform-config`. The spoke root Application will pick up the new entry
on its next sync (or force it with a hard refresh — see Forcing ApplicationSet reconciliation).

---

### Step 11 — Verify team provisioning

```bash
# On the spoke cluster — platform ArgoCD should have created the team Application
oc get application team-argocd-my-team -n openshift-gitops

# Team ArgoCD namespace should exist
oc get namespace my-team-gitops

# Team ArgoCD instance should be provisioned by the operator (~2-3 min after namespace appears)
oc get argocd my-team-gitops -n my-team-gitops

# Root Application should appear once the team ArgoCD is ready
oc get application my-team-root -n my-team-gitops

# All Applications generated from the team's values.yaml
oc get application -n my-team-gitops
```

---

### Step 12 — Verify team namespace access

Once the team's bootstrap Application has synced, confirm the namespaces are created
and that the team ArgoCD SA has access to them:

```bash
# Namespaces created by bootstrap-project (my-team-tools + system × env)
oc get namespace -l platform/team=my-team

# Confirm the managed-by label is present on each namespace
oc get namespace my-team-tools -o jsonpath='{.metadata.labels}' | jq

# Operator-created RoleBinding granting team SA deploy rights (auto-created from label)
oc get rolebinding -n my-team-tools | grep argocd
```

---

### Adding a new system for an existing team

The team commits to their own `my-team-gitops` repo — **no platform team action required**.

In `my-team-gitops/ubiquitous-journey/values-tooling.yaml`, the team adds to the `systems` list:

```yaml
systems:
  - name: pet-battle
    envs: [dev, test, stage]
    operatorgroup: true
  - name: tournament-service      # ← new system
    envs: [dev, test]
    operatorgroup: false
```

This generates `my-team-tournament-service-dev` and `my-team-tournament-service-test`
automatically on the next bootstrap Application sync, with the `managed-by` label applied
so the team ArgoCD SA gains deploy rights immediately.

---

### Removing a team

1. Remove the team entry from `clusterdef.yaml` and push
2. The platform ArgoCD will prune `team-argocd-my-team` Application
3. The `my-team-gitops` namespace and its ArgoCD instance are deleted
4. Team namespaces (`my-team-*`) are **not** automatically deleted — they were created
   by the team's own ArgoCD and must be cleaned up manually:

```bash
# Review what will be deleted
oc get namespace -l platform/team=my-team

# Delete when confirmed safe
oc delete namespace -l platform/team=my-team
```

---

### Troubleshooting — team ArgoCD

**Team ArgoCD not provisioned after namespace appears**

The OpenShift GitOps operator must be configured to watch namespaces beyond `openshift-gitops`.
Check the operator is running the required version (1.10+ recommended) and the namespace has
the correct label:

```bash
oc get namespace my-team-gitops -o jsonpath='{.metadata.labels}' | jq \
  '."argocd.argoproj.io/managed-by-cluster-argocd"'
# Expected: "openshift-gitops"
```

If the label is present but no ArgoCD CR was provisioned, check operator logs:

```bash
oc logs -n openshift-gitops \
  deploy/openshift-gitops-operator-controller-manager \
  --tail=50
```

**Team Applications stuck in Unknown / Missing**

The team ArgoCD SA needs the ClusterRoleBinding created by `helm-catalog/team-argocd`.
Verify it exists:

```bash
oc get clusterrolebinding team-my-team-argocd-namespace-manager
```

If missing, force a re-sync of the `team-argocd-my-team` Application from the platform ArgoCD.

**Team can't deploy into a namespace**

Check the `managed-by` label on the namespace and whether the operator created a RoleBinding:

```bash
# Label must be present
oc get namespace my-team-pet-battle-dev \
  -o jsonpath='{.metadata.labels.argocd\.argoproj\.io/managed-by}'
# Expected: my-team-gitops

# Operator should have created this RoleBinding automatically
oc get rolebinding -n my-team-pet-battle-dev | grep my-team-gitops
```

If the label is missing, the team's `bootstrap-project` Application did not sync correctly.
Check `oc get application bootstrap -n my-team-gitops` and review sync errors.

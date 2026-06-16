# Architecture Diagrams

Reference diagrams for the two-tier hub-spoke GitOps platform.
See [RUNBOOK.md](RUNBOOK.md) for operational procedures and [README.md](README.md) for chart reference.

---

## Diagram 1 — The two-tier push/pull model

The hub **pushes** a bootstrap Application to each spoke via ACM and ApplicationSets.
The spoke's own Argo CD then **pulls** all further configuration independently.

```mermaid
flowchart TD
    subgraph hub [Hub Cluster]
        ACM["ACM Placement\nplacement-dev"]
        AppSet["ApplicationSet\nbootstrap-dev\nhub-gitops/templates/\napplicationset-spoke-argocd-bootstrap.yaml"]
        HubArgoCD["Hub Argo CD\nopenshift-gitops"]
        BootstrapApp["Application\nplatform-bootstrap-digital\n(lives on HUB, targets SPOKE)"]
    end

    subgraph git [Git Repositories]
        HelmCatalog["helm-catalog.git\nspoke-argocd-bootstrap/"]
        PlatformApps["platform-apps.git\ntemplates/spoke/\ntemplates/shared/"]
        PlatformConfig["platform-config.git\nclusters/dev/digital/\nclusterdef.yaml"]
    end

    subgraph spoke [Spoke Cluster — digital-dev]
        SpokeArgoCD["Spoke Argo CD\nopenshift-gitops"]
        RootApp["Application\ndigital-dev-aoa\n(root app-of-apps)"]
        BannerApp["Application\ncluster-banner"]
        GuardrailsApp["Application\ncluster-guardrails"]
        Banner["ConsoleNotification\ngreen dev banner"]
        Template["Template: project-request\n+ ResourceQuota\n+ LimitRange\n+ NetworkPolicy"]
    end

    ACM -->|"cluster joins\nclusterset=dev"| AppSet
    AppSet -->|"PUSH — creates"| BootstrapApp
    BootstrapApp -->|"sources chart"| HelmCatalog
    BootstrapApp -->|"deploys to spoke"| RootApp

    RootApp -->|"PULL — sources"| PlatformApps
    RootApp -->|"values from"| PlatformConfig
    RootApp -->|"generates"| BannerApp
    RootApp -->|"generates"| GuardrailsApp

    BannerApp -->|"sources chart"| HelmCatalog
    GuardrailsApp -->|"sources chart"| HelmCatalog
    BannerApp -->|"applies"| Banner
    GuardrailsApp -->|"applies"| Template
```

---

## Diagram 2 — Config merge chain → how values reach a chart

Four YAML layers are merged (last-key-wins) before being passed as Helm values to a chart.
This example traces how the cluster-banner colour and enabled flag flow to the spoke.

```mermaid
flowchart LR
    subgraph platformConfig [platform-config.git]
        D["default.yaml\nclusterBannerConfig:\n  enabled: true\nbgColor: varies"]
        R["regions/eu-west-1/\nregion.yaml\n(no banner override)"]
        C["clusters/dev/\ncommon.yaml\nbgColor: '#28a745'\n(green)"]
        CD["clusters/dev/digital/\nclusterdef.yaml\nclusterGroup: digital\nclusterName: digital-dev-eu-west-1"]
    end

    Merged["Merged values\n(last-key-wins)\nenabled: true\nbgColor: '#28a745'\nclusterGroup: digital"]

    subgraph platformApps [platform-apps.git]
        Helper["templates/_helpers.tpl\ndefault.valueFiles\n(4-layer list)"]
        BannerTpl["templates/shared/\ncluster-banner-app.yaml\n→ Application CR"]
    end

    subgraph helmCatalog [helm-catalog.git]
        Chart["cluster-banner/\ntemplates/\nconsole-notification.yaml"]
    end

    Spoke["ConsoleNotification\non spoke cluster\n(green banner)"]

    D -->|"layer 1"| Merged
    R -->|"layer 2"| Merged
    C -->|"layer 3"| Merged
    CD -->|"layer 4"| Merged
    Helper -->|"defines file order"| BannerTpl
    Merged -->|"injected as valuesObject"| BannerTpl
    BannerTpl -->|"Application.spec.source\npath: cluster-banner"| Chart
    Chart -->|"renders"| Spoke
```

---

## Diagram 3 — Detailed file references: cluster-guardrails end-to-end

Traces every file touched from the ACM `oc label` command to the on-cluster resources,
using cluster-guardrails as the example.

```mermaid
flowchart TD
    Label["oc label managedcluster digital\nclusterset=dev"]

    subgraph hubGitops [hub-gitops chart]
        Values["hub-gitops/values.yaml\napplicationSets:\n  - name: bootstrap-dev\n    clusterType: dev\n    region: eu-west-1"]
        AppSetTpl["hub-gitops/templates/\napplicationset-spoke-argocd-bootstrap.yaml\n→ ApplicationSet bootstrap-dev"]
    end

    subgraph bootstrapChart [helm-catalog/spoke-argocd-bootstrap]
        RootAppTpl["templates/root-application.yaml\n→ Application digital-dev-aoa\nsources:\n  - platform-apps.git\n  - platform-config.git (ref)"]
    end

    subgraph configLayers [platform-config.git — merged values]
        Default["default.yaml\nclusterGuardrails:\n  enabled: false"]
        DevCommon["clusters/dev/common.yaml\nclusterGuardrails:\n  enabled: true"]
        Clusterdef["clusters/dev/digital/clusterdef.yaml\nnetworkPolicy:\n  denyOtherNamespaces: false"]
    end

    subgraph appsTpl [platform-apps.git]
        GuardrailsTpl["templates/spoke/\ncluster-guardrails-app.yaml\n→ Application cluster-guardrails\nif clusterGuardrails.enabled\n   AND clusterType != hub"]
    end

    subgraph guardChart [helm-catalog/cluster-guardrails]
        RBAC["templates/rbac.yaml\nwave -1\nClusterRole + ClusterRoleBinding"]
        ProjTpl["templates/project-template.yaml\nwave 0\nTemplate in openshift-config\n+ ResourceQuota\n+ LimitRange\n+ NetworkPolicy"]
        ProjCfg["templates/project-config.yaml\nwave 1\nSSA patch:\nproject.config.openshift.io/cluster"]
    end

    Label -->|"ACM matches\nplacement-dev"| AppSetTpl
    Values -->|"configures"| AppSetTpl
    AppSetTpl -->|"creates Application\nplatform-bootstrap-digital"| RootAppTpl
    RootAppTpl -->|"multi-source\nvalueFiles chain"| Default
    Default -->|"merge layer 1"| DevCommon
    DevCommon -->|"merge layer 3\nenabled: true"| Clusterdef
    Clusterdef -->|"merge layer 4\ndenyOtherNamespaces: false"| GuardrailsTpl
    GuardrailsTpl -->|"Application\npath: cluster-guardrails"| RBAC
    RBAC -->|"wave -1"| ProjTpl
    ProjTpl -->|"wave 0"| ProjCfg
```

---

## Diagram 4 — Sync wave ordering within a spoke Application

Argo CD applies resources in ascending wave order within a single Application sync.
This example shows the cluster-guardrails wave sequence.

```mermaid
flowchart LR
    W_2["wave -2\nNamespace\nopenshift-devspaces\n(devspaces chart)"]
    W_1["wave -1\nClusterRole\nClusterRoleBinding\n(RBAC for Argo CD SA)"]
    W0["wave 0\nTemplate\nproject-request\n(openshift-config)"]
    W1["wave 1\nproject.config.openshift.io\n/cluster SSA patch"]

    W_2 --> W_1 --> W0 --> W1
```

---

## Key concepts

**Push model (hub → spoke)**
The hub cluster is the only entity that writes to spokes at bootstrap time.
ACM Placement selects clusters; the ApplicationSet fans out one bootstrap Application per matched cluster.
The platform team controls which clusters are bootstrapped by managing `oc label managedcluster`.

**Pull model (spoke → Git)**
After bootstrap, each spoke's Argo CD polls Git directly.
The hub plays no further role in managing spoke application state.
This means a spoke continues to function correctly even if the hub is unavailable.

**Four-layer config merge**
```
default.yaml  →  regions/<r>/region.yaml  →  clusters/<type>/common.yaml  →  clusters/<type>/<group>/clusterdef.yaml
```
Each later layer overrides earlier keys. A feature is toggled on at the environment layer (`common.yaml`)
and tuned or disabled at the cluster layer (`clusterdef.yaml`).

**clusterGroup convention**
The `ManagedCluster` name in ACM is used as `clusterGroup`. This is also the folder name under
`clusters/<type>/` in `platform-config`. Convention: one folder = one cluster.

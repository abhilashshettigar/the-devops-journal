# blog-1: LGTM observability stack on Kubernetes with Argo CD GitOps

A hands-on example of deploying the **LGTM** stack — **L**oki (logs), **G**rafana
(dashboards), **T**empo (traces) — plus **Argo CD** itself, using a pure
**GitOps** workflow. Every component is wrapped in a local Helm chart and
generated as an Argo CD `Application` by a single `ApplicationSet`, so the
cluster state is reconciled directly from this Git repository.

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │                 monitoring                   │
                    │                                              │
   Git push ──►  Argo CD ──sync──►  Grafana  ◄──datasources──┐    │
                (ApplicationSet)    Loki (logs)  ────────────┘    │
                                     Tempo (traces)                │
                    └─────────────────────────────────────────────┘
```

- **app-of-apps**: the root `Application` (`argocd-apps`) watches
  `blog-1/argocd-apps/apps/`, which holds the `ApplicationSet` and the
  `observability` `AppProject`.
- **ApplicationSet**: a single `ApplicationSet` generates one child
  `Application` per component (`argocd`, `grafana`, `loki`, `tempo`) from a list
  generator.
- Each child `Application` points at the local wrapper chart
  (`blog-1/<component>/`) in this repo on branch `main`.
- **AppProject**: the `observability` project scopes the child Applications to
  this repository and the `monitoring` namespace.
- Sync is **automated** with `prune` + `selfHeal`, so drift is corrected and
  deleted manifests are cleaned up.
- Everything lands in a single `monitoring` namespace.

## Layout

```
blog-1/
├── grafana/            # wrapper chart: grafana/grafana + Loki/Tempo datasources
├── loki/               # wrapper chart: grafana/loki (SingleBinary, filesystem)
├── tempo/              # wrapper chart: grafana/tempo (SingleBinary, local storage)
├── argocd/             # wrapper chart: argo/argo-cd
└── argocd-apps/        # kustomize root that declares the GitOps applications
    ├── kustomization.yaml   # bootstrap bundle: namespace + root app
    ├── namespace.yaml       # monitoring namespace
    ├── root-app.yaml        # app-of-apps root Application (project: default)
    └── apps/
        ├── kustomization.yaml
        ├── project.yaml         # AppProject "observability"
        └── applicationset.yaml  # ApplicationSet -> argocd/grafana/loki/tempo
```

## Prerequisites

- A Kubernetes cluster and a working `kubectl` context
- `helm` v3+ and the two upstream chart repos:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

## Bootstrap

Argo CD cannot manage itself before it exists, so it is installed once
manually; after that the `argocd` Application keeps it in sync.

```bash
# 1. Create the namespace
kubectl create namespace monitoring

# 2. Install Argo CD (initial seed)
helm install argocd argo/argo-cd -n monitoring -f blog-1/argocd/values.yaml

# 3. Hand over to GitOps: apply the app-of-apps root
kubectl apply -k blog-1/argocd-apps
```

The root `Application` applies the `observability` `AppProject` and the
`ApplicationSet`. The ApplicationSet then generates the Grafana, Loki, Tempo,
and Argo CD child Applications, and Argo CD reconciles the whole stack from Git.

### Why `root-app.yaml`?

Argo CD bootstraps from a single `Application`, not an `ApplicationSet`: the
ApplicationSet controller creates child `Application`s, but something still has
to install the `ApplicationSet` and `AppProject` themselves. That is
`root-app.yaml`.

```
root-app.yaml (Application, project: default)
      │ watches blog-1/argocd-apps/apps/
      ▼
project.yaml + applicationset.yaml
      │ ApplicationSet generates
      ▼
argocd / grafana / loki / tempo (Applications)
      │ each syncs its wrapper chart
      ▼
Helm-rendered resources in the monitoring namespace
```

Keeping the root `Application` means the `ApplicationSet` and `AppProject` are
themselves reconciled from Git: add or remove a component in the generator
list, or change the project, and Argo CD applies it on the next sync — no
kubectl. If you drop `root-app.yaml`, you would apply `project.yaml` and
`applicationset.yaml` once with `kubectl apply`; the four child Applications
stay fully GitOps-managed, but the `ApplicationSet`/`AppProject` themselves
would only change when you re-apply them manually.

## Accessing the UIs

```bash
# Grafana (admin / admin)
kubectl -n monitoring port-forward svc/grafana 3000:80

# Argo CD
kubectl -n monitoring port-forward svc/argocd-server 8080:443

# Loki
kubectl -n monitoring port-forward svc/loki 3100:3100

# Tempo (OTLP HTTP on 4318, query API on 3200)
kubectl -n monitoring port-forward svc/tempo 3200:3200
```

## Making a change (the GitOps loop)

1. Edit a `values.yaml` under `blog-1/<component>/`.
2. Commit and push to `main`.
3. Argo CD detects the new revision and syncs automatically.

To add a new component: add a wrapper chart and append an element to the list
generator in `blog-1/argocd-apps/apps/applicationset.yaml`, then push. The root
Application syncs the ApplicationSet, which generates the new Application.

## Notes

- These values are **demo-grade** (no persistence, `admin`/`admin` Grafana
  credentials, single replicas, filesystem/local storage). Do not use them
  as-is in production.
- Chart dependencies are pinned in each wrapper chart's `Chart.yaml` and
  resolved via `Chart.lock`. Argo CD runs `helm dependency build` at sync time.

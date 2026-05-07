# git-ops-miscro

GitOps repository for ITP microservice deployments.
Managed by dev teams and Jenkins CI/CD.

Argo CD watches this repository. Service charts in this repo reference shared
Helm charts published from `git-infra-miscro` to a Helm OCI registry such as
GitHub Container Registry (GHCR).

Recommended flow:

```text
git-infra-miscro GitHub repo
  -> publishes Helm charts to GHCR

git-ops-miscro GitHub repo
  -> stores Argo CD config, service values, and environment image tags
  -> references GHCR chart versions
  -> Argo CD deploys to Kubernetes
```

## Structure

```text
argocd/                 Argo CD config (ApplicationSet + projects)
teams/itp/              all ITP services
  project-itp/
    {service}/
      app.yaml          service metadata for ApplicationSet
      Chart.yaml        chart dependency and version
      environments/
        dev/values.yaml dev overrides for infra chart defaults
        prod/values.yaml prod overrides for infra chart defaults
```

`argocd/applicationset.yaml` discovers services by scanning:

```text
teams/itp/project-itp/*/app.yaml
```

That means you do not need to edit the ApplicationSet every time you add a new
service. Add a service folder with its own `app.yaml`, `Chart.yaml`, and
environment values.

Example `app.yaml`:

```yaml
service: account-service
chartPath: teams/itp/project-itp/account-service
wave: "2"
```

Recommended sync waves:

| Wave | Services |
|---|---|
| `"0"` | configserver |
| `"1"` | eurekaservice |
| `"2"` | backend services such as account-service, customer-service, pipeline-service, itp-identity-service |
| `"3"` | BFF services such as admin-bff, front-bff |
| `"4"` | gateway-server |
| `"5"` | frontend apps such as e-banking-front, next-shadcn-dashboard |

Sync waves provide deployment order metadata. Services should still use
readiness probes and retry their dependencies because microservices can start at
different speeds.

Example service chart dependency:

```yaml
dependencies:
  - name: base
    version: "1.0.0"
    repository: "oci://ghcr.io/seang454/git-infra-miscro/charts"
```

Change the chart version only when you want that service to use a newer shared
chart from `git-infra-miscro`.

Environment value files must override the dependency chart name. For example,
`account-service` depends on `base`, so its dev values use `base:`:

```yaml
base:
  deployments:
    api:
      image:
        repository: your-registry/account-service
        tag: "dev-latest"
```

The shared defaults for `base` live in `git-infra-miscro/charts/base/values.yaml`.

## Adding a New Service

```bash
mkdir -p teams/itp/project-itp/my-new-service/environments/{dev,prod}
# add app.yaml + Chart.yaml + environments/*/values.yaml
# Argo CD auto-detects and deploys automatically
```

## Deploying

Argo CD watches this repo automatically.
Jenkins updates `environments/{env}/values.yaml` after each build.

The shared chart source and default values are not copied into this repo. They
are packaged and published by `git-infra-miscro`, then pulled by Helm/Argo CD
from GHCR using the version in each service `Chart.yaml`.

## Apply Argo CD Config

Bootstrap the root application once:

```bash
kubectl apply -f argocd/root-application.yaml
```

After that, Argo CD manages the `argocd/` folder from Git. The root application
syncs `projects/itp-project.yaml` first, then `applicationset.yaml`, which
generates one Application per service and environment.

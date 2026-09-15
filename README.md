# eas-weather-rs-k8s

Kubernetes deployment for the [eas-weather-rs](https://github.com/shyuen/eas-weather-rs)
EAS alert microservice, using a combination of **Helm** (app templating) and
**Kustomize** (environment overlays).

## Layout

```
charts/eas-weather-rs/   Helm chart (Deployment: migrate init-container + server, ConfigMap, Service, Ingress)
overlays/
  base/                  Reference deployment (default values, default namespace)
  dev/                   Development overlay (tag: dev, debug logging, 1 replica)
  staging/               Staging overlay (tag: staging, JSON logging, 2 replicas, resources)
  prod/                  Production overlay (tag: prod, JSON logging, 2 replicas, ingress, resources)
```

Kustomize is the entry point: `helmCharts` renders the chart from the local
`charts/` directory; each overlay supplies its own `valuesInline`.

## Prerequisites

- `kubectl` (or `kustomize` + `helm`)
- An existing k8s Secret containing the app's secrets, mounted by the chart as files.
  Note: a Secret is mounted but not created by these overlays (handle out-of-band):

```bash
kubectl -n eas-weather-rs-dev create secret generic eas-weather-rs-secrets \
  --from-literal=conn_url='mysql://user:pass@db-host:3306/eas_weather' \
  --from-literal=api_key='<your-api-key>' \
  --from-literal=jwt_key='<your-jwt-key>'
```

## Deploy

```bash
# Development
kustomize build overlays/dev --enable-helm --load-restrictor=LoadRestrictionsNone | kubectl apply -f -

# Production
kustomize build overlays/prod --enable-helm --load-restrictor=LoadRestrictionsNone | kubectl apply -f -
```

`--load-restrictor=LoadRestrictionsNone` is required because `helmCharts.chartHome`
points at the shared `charts/` directory outside each overlay's root.

## Image bumps and deployment

The app repo's `ci.yml` `publish` job (runs on main push; also `workflow_dispatch`) builds
the image, publishes `<env>` + `<env>-<sha>` tags to GHCR and tags the codebase repo, then
clones this repo with a PAT (`secrets.EWRS_DEPLOY_REPO_PAT` — `repo` scope), yq-updates
`overlays/<env>/kustomization.yaml` `valuesInline.image.tag`, and pushes to `main`. The
static tags in `overlays/*` (`dev`, `staging`, `prod`) are the initial values until the
first publish.

Deployment is ArgoCD's job: it watches this repo's `main` and syncs the target overlay, so
there is deliberately no `kubectl apply`/deploy workflow here. `.github/workflows/verify.yaml`
runs `helm lint` + a full `kustomize build` of every overlay on each push/PR to keep `main`
green for ArgoCD.

Config-only changes need no pipeline: edit `valuesInline` (env vars, replicas, probes, ...)
in an overlay, push — verify.yaml renders it, ArgoCD applies it, and the `checksum/config`
annotation on the pod template rolls the workload. The app repo's CI is only involved when a
new image tag is being published.

## Customising an environment

Environment differences (image tag, replicas, logging, ingress, resources) are
expressed as `valuesInline` in the overlay's `kustomization.yaml`, mirroring the
chart's `values.yaml`. For example, to tweak the production image tag:

```bash
kubectl -n eas-weather-rs-prod set image deploy/eas-weather-rs \
  server=ghcr.io/shyuen/eas-weather-rs-server:1.2.3
```

## Verify the chart directly (without Kustomize)

```bash
helm lint charts/eas-weather-rs
helm template eas-weather-rs charts/eas-weather-rs -n eas-weather-rs-dev \
  --set image.repository=ghcr.io/shyuen/eas-weather-rs-server --set image.tag=dev
```
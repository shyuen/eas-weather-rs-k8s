# eas-weather-rs-k8s

Kubernetes deployment for the [eas-weather-rs](https://github.com/shyuen/eas-weather-rs)
EAS alert microservice, using a combination of **Helm** (app templating) and
**Kustomize** (environment overlays).

## Layout

```
charts/eas-weather-rs/   Helm chart (Deployment, init-container migrate, Service, Ingress)
overlays/
  base/                  Reference deployment (default values, default namespace)
  dev/                   Development overlay (tag: dev, debug logging, 1 replica)
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
# AGENTS.md

Kubernetes deployment for the [eas-weather-rs](https://github.com/shyuen/eas-weather-rs)
EAS alert microservice, using **Helm** (app templating) + **Kustomize** (environment overlays).

## Repo layout

```
charts/eas-weather-rs/
  Chart.yaml            Chart metadata (name: eas-weather-rs, version 0.1.0)
  values.yaml           Default chart values; source of truth for config keys
  templates/
    _helpers.tpl        name/fullname/labels/selectorLabels/serviceAccountName helpers
    deployment.yaml     Deployment: migrate init-container + server container
    service.yaml        ClusterIP Service on port 8080
    ingress.yaml        Ingress (rendered only if ingress.enabled)
    serviceaccount.yaml ServiceAccount (rendered only if serviceAccount.create)
overlays/
  base/                 Reference deployment (tags: latest, default namespace)
  dev/                  ns eas-weather-rs-dev, tag: dev, debug logging, 1 replica
  prod/                 ns eas-weather-rs-prod, tag: prod, JSON logging, 2 replicas,
                        ingress + cert-manager annotation, resource requests/limits
```

## Architecture

- Kustomize is the **entry point**: each overlay's `kustomization.yaml` uses a single
  `helmCharts` entry referencing the local `charts/` directory via `helmGlobals.chartHome: ../../charts`.
- Per-environment differences (image tag, replicas, logging, ingress, resources) go into the
  overlay's `helmCharts[].valuesInline`, which overrides chart defaults.
- **Secrets are NOT created by these overlays.** An existing k8s Secret
  (`eas-weather-rs-secrets`, name set via `valuesInline.secrets.secretName`) is mounted as a
  file volume read-only at `/etc/eas`. Handle the Secret out-of-band (e.g. External Secrets Operator).
- The app reads sensitive config from file paths, not env vars. The template wires:
  - `EAS_WEATHER_RS__WEBSERVER__API_KEY_FILE`
  - `EAS_WEATHER_RS__WEBSERVER__JWT_KEY_FILE`
  - `EAS_WEATHER_RS__DATABASE__CONN_URL_FILE`
- Other app config is passed as `EAS_WEATHER_RS__<SECTION>__<KEY>` env vars, overriding the app's
  `config/default.toml`. Values come from `values.yaml` `config:` / `extraEnv:`.

## Commands

Deploy (Kustomize is the deployment mechanism; charts are rendered, not installed via helm):

```bash
# Development
kustomize build overlays/dev --enable-helm --load-restrictor=LoadRestrictionsNone | kubectl apply -f -

# Production
kustomize build overlays/prod --enable-helm --load-restrictor=LoadRestrictionsNone | kubectl apply -f -
```

Verify the chart directly:

```bash
helm lint charts/eas-weather-rs
helm template eas-weather-rs charts/eas-weather-rs -n eas-weather-rs-dev \
  --set image.repository=ghcr.io/shyuen/eas-weather-rs-server --set image.tag=dev
```

## Conventions & gotchas

- `--load-restrictor=LoadRestrictionsNone` is **required** for kustomize builds because
  `chartHome` points outside each overlay's root.
- Image: both `server` and `migrate` binaries live in the same image
  (`ghcr.io/shyuen/eas-weather-rs-server:<tag>`). Image tag is the environment marker
  (`latest` / `dev` / `prod`).
- Bump `version`/`appVersion` in `Chart.yaml` when the deployment template materially changes.
- Deployment image helper: `{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}`.
  Prefer setting `image.tag` explicitly in the overlay over relying on the AppVersion default.
- To add a new env var, extend `values.yaml` `config:` and the server container's `env:` block in
  `deployment.yaml` using the `EAS_WEATHER_RS__SECTION__KEY` pattern; for one-off overrides use `extraEnv`.
- Health probes are configured via `values.yaml` `probes:` (startup/liveness/readiness, one route each);
  their `path` is composed under `config.webserver.basePath` in `deployment.yaml`. The app's `/health/*`
  routes require no API key.
- New environments: add an `overlays/<env>/kustomization.yaml` mirroring `overlays/dev`, set the
  namespace and `valuesInline`.

### Helm values vs Kustomize features (where env differences go)

- If the chart exposes a knob, a per-env difference goes in the overlay's `helmCharts[].valuesInline`
  (image tag, replicas, logging, probes, resources, ingress). This keeps the chart the single contract
  and each overlay a readable diff against chart defaults.
- Use Kustomize-native features only for **non-chart / environment-level** concerns:
  - `namespace:` (already the pattern) and extra cluster concerns.
  - `resources:`, `configMapGenerator`/`secretGenerator` for per-env objects the chart doesn't own
    (note: the app's Secret is deliberately kept out of overlays).
  - `commonLabels`/`commonAnnotations` for uniform sweeps across all rendered objects.
  - `patches:` / `images:` / `replicas:` only as a last resort for chart gaps — they depend on the
    rendered output and silently break if the chart changes. Prefer exposing a new chart value instead.

## Verification

- Run `helm lint charts/eas-weather-rs` after chart edits.
- Run `kustomize build overlays/<env> --enable-helm --load-restrictor=LoadRestrictionsNone` to
  confirm each environment still renders.
- Namespaces used: `eas-weather-rs-dev`, `eas-weather-rs-prod`.
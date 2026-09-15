# AGENTS.md

Kubernetes deployment for the [eas-weather-rs](https://github.com/shyuen/eas-weather-rs)
EAS alert microservice, using **Helm** (app templating) + **Kustomize** (environment overlays).

## Repo layout

```
charts/eas-weather-rs/
  Chart.yaml            Chart metadata (name: eas-weather-rs, version 0.4.0)
  values.yaml           Default chart values; source of truth for config keys
  templates/
    _helpers.tpl        name/fullname/labels/selectorLabels/serviceAccountName helpers
    deployment.yaml     Deployment: migrate init-container + server container
    configmap.yaml      Renders config.toml (read-only mount; secret file-path refs, no values)
    service.yaml        ClusterIP Service on port 8080
    ingress.yaml        Ingress (rendered only if ingress.enabled)
    serviceaccount.yaml ServiceAccount (rendered only if serviceAccount.create)
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
- **Non-secret config lives in a ConfigMap** (`<fullname>-config`) as one rendered `config.toml`,
  mounted read-only at `/etc/ewrs` and loaded by BOTH the migrate init-container and the server
  container via `EAS_WEATHER_RS__APP__CONFIG_FILE`. The app merges it section-over-section over its
  built-in `config/default.toml`, so only keys set in `values.yaml` `config:` override app defaults;
  the full app config surface (webserver, logging, database) is available with no template changes.
- **Secrets never enter the ConfigMap.** `database.conn_url_file`, `webserver.api_key_file` and
  `webserver.jwt_key_file` are file *paths* into the mounted Secret volume (`/etc/eas`), injected
  into the rendered config from `secrets:` and documented as secret references. The Secret values
  themselves are read by the app from files under the mountPath.
- Config precedence in the app: CLI > env vars > config file > default.toml > code defaults.

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
- To add app config, add keys under `values.yaml` `config:` (section = TOML section, key = the
  app's snake_case TOML key) — no template change is needed; the ConfigMap generator in
  `templates/configmap.yaml` emits everything. `service.port` feeds `[webserver] port` and the
  three secret mount file-path keys are injected from `secrets:`. One-off env overrides on the
  server container use `extraEnv` (`env:`, which beats the config file in app precedence).
- Health probes are configured via `values.yaml` `probes:` (startup/liveness/readiness, one route each);
  their `path` is composed under `config.webserver.base_path` in `deployment.yaml`. The app's `/health/*`
  routes require no API key.
- New environments: add an `overlays/<env>/kustomization.yaml` mirroring `overlays/dev`, set the
  namespace and `valuesInline`.

## Verification

- Run `helm lint charts/eas-weather-rs` after chart edits.
- Run `kustomize build overlays/<env> --enable-helm --load-restrictor=LoadRestrictionsNone` to
  confirm each environment still renders.
- Namespaces used: `eas-weather-rs-dev`, `eas-weather-rs-prod`.

## Workflow

- **Always commit changes you make** (the user expects a commit per change set). Commit only
  when the work is verified (see Verification). If asked, push and open a PR on a branch.
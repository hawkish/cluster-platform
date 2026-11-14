# cluster-platform

The ArgoCD "app of apps" for the home k3s cluster. Every `Application` in [`apps/`](apps/) is created and synced by ArgoCD. The values for the Helm charts are in [`values/`](values/).

The cluster itself, ArgoCD, sealed-secrets, and all secrets are managed in the `nix` repo (`modules/nixos/opt/k3s.nix`). That module ships the root `Application` that points at this repo's `apps/` directory.

## Apps

| App | Chart | Namespace | Notes |
| --- | --- | --- | --- |
| `loki` | `grafana/loki` 7.6.0 | `monitoring` | Single binary, filesystem storage on a 20Gi PVC, 30-day retention |
| `alloy` | `grafana/alloy` 1.16.0 | `monitoring` | DaemonSet. Ships the logs of the pods on its node and the node's systemd journal to Loki |
| `prometheus` | `prometheus-community/prometheus` 29.47.2 | `monitoring` | Server only, 10Gi PVC, 15-day retention. Scrapes RabbitMQ and ArgoCD |
| `grafana` | `grafana-community/grafana` 14.4.1 | `monitoring` | UI on `http://192.168.1.10:30300`. Data sources: Loki (default) and Prometheus. Dashboards from `dashboards/` via the sidecar |
| `rabbitmq` | `cloudpirates/rabbitmq` 0.21.32 | `rabbitmq` | 3 clustered brokers. Management UI on `http://192.168.1.10:30082`, AMQP on `30672`, metrics on `30692` |
| `pythia` | plain manifests in [`pythia/`](pythia/) | `monitoring` | Read-only Loki MCP server for Claude Code, on `http://192.168.1.10:30310/mcp` behind a bearer token |

## Add an app

1. Add an `Application` to `apps/`. For a Helm chart, use a multi-source `Application`, as in `apps/loki.yaml`, and put its values in `values/<app>.yaml`.
2. Commit and push to `main`. ArgoCD picks up the change on its next poll, within three minutes.

Don't commit secret values. Add the secret to sops in the `nix` repo and sync it into the cluster with a oneshot service, like `grafana-admin-password.service`, or commit a sealed secret.

## pythia

[pythia](https://github.com/hawkish/pythia) (a private repo) is a read-only MCP server over Loki. With it, Claude Code can read the cluster's pod logs and node journals itself instead of asking you to paste from Grafana. [`apps/pythia.yaml`](apps/pythia.yaml) syncs the plain manifests in [`pythia/`](pythia/). They live here rather than in the pythia repo because this repo decides what runs, and the pythia repo only produces images.

### Roll out a release

pythia's CI pushes `ghcr.io/hawkish/pythia:vX.Y.Z` for each release. Nothing changes in the cluster until you set `newTag` in [`pythia/kustomization.yaml`](pythia/kustomization.yaml) to that tag, then commit and push.

### Secrets

The `nix` repo's `pythia-secrets.service` (in `modules/nixos/opt/k3s.nix`) reconciles two Secrets from sops, so no token is committed here:

- **`pythia-token`:** the bearer token, from the `pythia_token` sops secret. pythia refuses to start without it. Rotating the token restarts the Deployment.
- **`ghcr-pull`:** the imagePullSecret for the private image, from `argocd_access_token`. That token has `read:packages` as well as `repo`.

### Connect Claude Code

```bash
claude mcp add --transport http --scope user pythia http://192.168.1.10:30310/mcp --header 'Authorization: Bearer ${PYTHIA_TOKEN}'
```

zsh exports `PYTHIA_TOKEN` on `mini` from the same `pythia_token` sops secret. The single quotes keep the variable unexpanded in `~/.claude.json`, so Claude Code reads it at startup, and a rotated token only needs a new shell.

Its `patterns` tool needs `pattern_ingester.enabled` in [`values/loki.yaml`](values/loki.yaml).

## RabbitMQ

RabbitMQ runs as 3 clustered brokers from [CloudPirates](https://github.com/CloudPirates-io/helm-charts)' chart, pulled from `oci://registry-1.docker.io/cloudpirates/rabbitmq` with [`values/rabbitmq.yaml`](values/rabbitmq.yaml) layered on top. Bitnami's free RabbitMQ chart and images were discontinued, and CloudPirates' chart is a community-maintained replacement with the same values schema.

The nix repo's `rabbitmq-admin-password.service` (in `modules/nixos/opt/k3s.nix`, server host only) reconciles everything secret from sops, so no password or hash is committed here:

- **`rabbitmq-credentials` Secret:** the admin password and Erlang cookie, from the `rabbitmq_admin_password` and `rabbitmq_erlang_cookie` sops secrets. `auth.existingSecret` points at it, so Helm never generates a random password that drifts on every ArgoCD re-render.
- **`rabbitmq-definitions` Secret:** `defs.json`, with the `agenda` quorum queue and the `admin` user's password hash. `definitions.existingSecret` points at it. When the chart renders its own definitions ConfigMap instead, it adds an `admin` user with `password: ""` whenever `auth.username` isn't in `definitions.users`, and the auto-reload sidecar re-imports that on every change, wiping the real admin password.
- **The `agenda` user:** created with `rabbitmqctl`, its password synced from the `rabbitmq_agenda_password` sops secret. It can configure and read only `^agenda$`, and write to `^(agenda|amq\.default)$`, because publishing to a queue by name goes through the default exchange.

To change the `agenda` queue's declaration, edit the `rabbitmq-admin-password` script in `k3s.nix` and rebuild the server host.

### Passwords

- **Admin:** the `rabbitmq_admin_password` sops secret in the nix repo's `secrets/nixos/secrets.json`. Log in to the management UI at `http://192.168.1.10:30082` as `admin`.
- **`agenda` user:** the `rabbitmq_agenda_password` sops secret. zsh exports it as `RABBITMQ_AGENDA_PASSWORD` on `nixos` and `nixos2` only, the hosts that can reach the broker. To rotate it, update the secret in both `nixos` and `nixos2`'s sops files and rebuild. Rebuilding the server host re-runs the oneshot, which updates the broker. Open a new shell on each host to pick up the new value.

## Log in to Grafana

Open `http://192.168.1.10:30300` and sign in as `admin`. The password is the sops secret `grafana_admin_password` in the `nix` repo (`secrets/nixos/secrets.json`).

## Dashboards

Dashboards are JSON files in [`dashboards/`](dashboards/). [`dashboards/kustomization.yaml`](dashboards/kustomization.yaml) turns each one into a ConfigMap labelled `grafana_dashboard: "1"`, and the Grafana `Application` syncs them as its third source. Grafana's dashboard sidecar loads every ConfigMap with that label and reloads Grafana when one changes, without a pod restart.

| File | Dashboard | Origin |
| --- | --- | --- |
| `rabbitmq-overview.json` | RabbitMQ-Overview: queue depth, message rates, connections, channels, and memory and disk per broker | [10991](https://grafana.com/grafana/dashboards/10991), revision 15 |
| `argocd-operational-overview.json` | ArgoCD / Operational / Overview: reconcile times, git requests, Kubernetes API calls, and gRPC traffic | [19993](https://grafana.com/grafana/dashboards/19993), revision 9 |
| `argocd-application-overview.json` | ArgoCD / Application / Overview: sync and health status for each app | [19974](https://grafana.com/grafana/dashboards/19974), revision 8 |

The files are exports of those revisions with the data source set to the `prometheus` UID, so they don't depend on grafana.com at runtime.

The RabbitMQ dashboard reads the metrics that `metrics.enabled` in `values/rabbitmq.yaml` turns on. The ArgoCD dashboards read the `argocd` scrape job in `values/prometheus.yaml`, which adds the `cluster="home"` label they filter on.

### Save a change made in the UI

Grafana can't save over a dashboard that comes from a file, and Grafana has no persistent volume, so changes made only in the UI are lost on the next restart. To keep a change:

1. In the dashboard, select **Export** > **Export as JSON**, and then copy or download the JSON.
2. Overwrite the dashboard's file in `dashboards/`. Keep its `uid` so links still work.
3. Commit and push. ArgoCD updates the ConfigMap, and the sidecar reloads the dashboard within about a minute.

### Add a dashboard

1. Build it in the UI, or download one from grafana.com, and save the JSON to `dashboards/<name>.json`. Point its panels at the data source UID `prometheus` or `loki`.
2. Add an entry for it under `configMapGenerator` in `dashboards/kustomization.yaml`.
3. If it needs new metrics, add a scrape job to `values/prometheus.yaml`.
4. Commit and push.

Keep each file under about 900 KB. A ConfigMap can't exceed 1 MiB.

## Query logs

In Grafana, select **Explore**, choose the **Loki** data source, and try these queries:

```logql
{namespace="rabbitmq"}
{namespace="argocd", app="argocd-server"}
{job="systemd-journal", unit="k3s.service"}
{job="systemd-journal", unit=~"(argocd|rabbitmq|grafana)-admin-password.service"}
```

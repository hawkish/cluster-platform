# cluster-platform

The ArgoCD "app of apps" for the home k3s cluster. Every `Application` in [`apps/`](apps/) is created and synced by ArgoCD. The values for the Helm charts are in [`values/`](values/).

The cluster itself, ArgoCD, sealed-secrets, and all secrets are managed in the `nix` repo (`modules/nixos/opt/k3s.nix`). That module ships the root `Application` that points at this repo's `apps/` directory.

## Apps

| App | Chart | Namespace | Notes |
| --- | --- | --- | --- |
| `loki` | `grafana/loki` 7.6.0 | `monitoring` | Single binary, filesystem storage on a 20Gi PVC, 30-day retention |
| `alloy` | `grafana/alloy` 1.16.0 | `monitoring` | DaemonSet. Ships the logs of the pods on its node and the node's systemd journal to Loki |
| `prometheus` | `prometheus-community/prometheus` 29.47.2 | `monitoring` | Server only, 10Gi PVC, 15-day retention. Scrapes RabbitMQ and ArgoCD |
| `grafana` | `grafana-community/grafana` 14.4.1 | `monitoring` | UI on `http://192.168.1.10:30300`. Data sources: Loki (default) and Prometheus. Dashboards: RabbitMQ-Overview, ArgoCD Operational and Application overviews |
| `rabbitmq` | `cloudpirates/rabbitmq` 0.21.32 | `rabbitmq` | Values are in the `agent-board` repo |

## Add an app

1. Add an `Application` to `apps/`. For a Helm chart, use a multi-source `Application`, as in `apps/loki.yaml`, and put its values in `values/<app>.yaml`.
2. Commit and push to `main`. ArgoCD picks up the change on its next poll, within three minutes.

Don't commit secret values. Add the secret to sops in the `nix` repo and sync it into the cluster with a oneshot service, like `grafana-admin-password.service`, or commit a sealed secret.

## Log in to Grafana

Open `http://192.168.1.10:30300` and sign in as `admin`. The password is the sops secret `grafana_admin_password` in the `nix` repo (`secrets/nixos/secrets.json`).

## Dashboards

Grafana downloads its dashboards from grafana.com when the pod starts, pinned to a revision in `values/grafana.yaml`:

- **RabbitMQ-Overview** ([10991](https://grafana.com/grafana/dashboards/10991), revision 15): queue depth, message rates, connections, channels, and memory and disk per broker. It reads the metrics Prometheus scrapes from RabbitMQ's `rabbitmq_prometheus` plugin, which `agent-board`'s `rabbitmq/values.yaml` enables with `metrics.enabled`.

- **ArgoCD / Operational / Overview** ([19993](https://grafana.com/grafana/dashboards/19993), revision 9): ArgoCD's own health, including reconcile times, git requests, Kubernetes API calls, and gRPC traffic.
- **ArgoCD / Application / Overview** ([19974](https://grafana.com/grafana/dashboards/19974), revision 8): sync and health status for each app.

Both ArgoCD dashboards come from the argo-cd-mixin. They read the `argocd` scrape job in `values/prometheus.yaml`, which collects every ArgoCD pod's `metrics` port and adds the `cluster="home"` label the dashboards filter on. ArgoCD itself is installed by the `nix` repo, but its pods expose metrics by default, so no change there is needed.

To add one, add an entry under `dashboards.default` in `values/grafana.yaml`, and add a scrape job to `values/prometheus.yaml` if the dashboard needs new metrics.

## Query logs

In Grafana, select **Explore**, choose the **Loki** data source, and try these queries:

```logql
{namespace="rabbitmq"}
{namespace="argocd", app="argocd-server"}
{job="systemd-journal", unit="k3s.service"}
{job="systemd-journal", unit=~"(argocd|rabbitmq|grafana)-admin-password.service"}
```

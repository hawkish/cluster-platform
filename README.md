# cluster-platform

The ArgoCD "app of apps" for the home k3s cluster. Every `Application` in [`apps/`](apps/) is created and synced by ArgoCD. The values for the Helm charts are in [`values/`](values/).

The cluster itself, ArgoCD, sealed-secrets, and all secrets are managed in the `nix` repo (`modules/nixos/opt/k3s.nix`). That module ships the root `Application` that points at this repo's `apps/` directory.

## Apps

| App | Chart | Namespace | Notes |
| --- | --- | --- | --- |
| `loki` | `grafana/loki` 7.6.0 | `monitoring` | Single binary, filesystem storage on a 20Gi PVC, 30-day retention |
| `alloy` | `grafana/alloy` 1.16.0 | `monitoring` | DaemonSet. Ships the logs of the pods on its node and the node's systemd journal to Loki |
| `grafana` | `grafana-community/grafana` 14.4.1 | `monitoring` | UI on `http://192.168.1.10:30300`. Loki is the default data source |
| `rabbitmq` | `cloudpirates/rabbitmq` 0.21.32 | `rabbitmq` | Values are in the `agent-board` repo |

## Add an app

1. Add an `Application` to `apps/`. For a Helm chart, use a multi-source `Application`, as in `apps/loki.yaml`, and put its values in `values/<app>.yaml`.
2. Commit and push to `main`. ArgoCD picks up the change on its next poll, within three minutes.

Don't commit secret values. Add the secret to sops in the `nix` repo and sync it into the cluster with a oneshot service, like `grafana-admin-password.service`, or commit a sealed secret.

## Log in to Grafana

Open `http://192.168.1.10:30300` and sign in as `admin`. The password is the sops secret `grafana_admin_password` in the `nix` repo (`secrets/nixos/secrets.json`).

## Query logs

In Grafana, select **Explore**, choose the **Loki** data source, and try these queries:

```logql
{namespace="rabbitmq"}
{namespace="argocd", app="argocd-server"}
{job="systemd-journal", unit="k3s.service"}
{job="systemd-journal", unit=~"(argocd|rabbitmq|grafana)-admin-password.service"}
```

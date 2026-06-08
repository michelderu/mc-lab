# Mission Control Helm values

Pinned chart defaults and lab overrides for Mission Control. Full walkthrough: [`docs/02-setup-mc.md`](../../docs/02-setup-mc.md).

**Prerequisites**

- 📂 Run commands from the **repository root**.
- KinD cluster **`mc`** is running ([KinD setup](../../docs/01-setup-kind.md)).

| File | Role |
|------|------|
| `values.yaml` | Pinned upstream chart defaults (`helm show values …`) |
| `overrides.yaml` | Lab changes: Dex, MinIO/Loki, platform `nodeSelector`s, optional Grafana (`grafana.enabled`) |

➡️ **Install:** [`docs/02-setup-mc.md`](../../docs/02-setup-mc.md) — cert-manager, registry login, `helm install` / upgrade, UI access.

## Pin chart version and default values

Regenerate `values.yaml` when you change `MC_CHART_VERSION` in `.env` (after [registry login](../../docs/02-setup-mc.md#registry-login-and-chart-version)):

```bash
set -a && source .env && set +a

helm registry login registry.replicated.com \
  --username "$MC_REGISTRY_USERNAME" \
  --password "$MC_REGISTRY_PASSWORD"

helm show values "$MC_CHART" --version "$MC_CHART_VERSION" > manifests/mission-control/values.yaml
```

Keep lab-specific settings in `overrides.yaml` only. Use the same `MC_CHART` and `MC_CHART_VERSION` from `.env` on `helm install` and `helm upgrade`.

> ⚠️ **MinIO / Loki:** Loki uses `<release>-minio.<namespace>.svc.cluster.local:9000` and Secret `mission-control-minio` (`rootUser` / `rootPassword`). Do not trim `loki.loki.schemaConfig.configs` in overrides — Helm replaces whole lists.

## Dex login (optional override)

Default lab user in `overrides.yaml`:

- `mission-control@example.com` / `cassandra`

Custom password:

```bash
echo 'your-password-here' | htpasswd -BinC 10 admin | cut -d: -f2
```

Set the hash under `dex.config.staticPasswords` in `overrides.yaml`, then [upgrade Mission Control](../../docs/02-setup-mc.md#upgrade-mission-control).

## Grafana (optional)

Pinned `values.yaml` keeps `grafana.enabled: false`. This lab sets `grafana.enabled: true` in `overrides.yaml` (Mimir + Loki datasources provisioned by the chart; platform `nodeSelector` is set there). To disable or change, edit `overrides.yaml` and [upgrade Mission Control](../../docs/02-setup-mc.md#upgrade-mission-control).

➡️ **Access:** [`docs/05-observability.md`](../../docs/05-observability.md)

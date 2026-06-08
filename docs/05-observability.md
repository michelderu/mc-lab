# Observability — Mission Control UI and Grafana

View HCD health and metrics after [HCD setup](03-setup-hcd.md). Optional prior step: [CQL and Data API](04-cql-data-api.md).

**Prerequisites**

- 📂 Run from the **repository root**.
- Mission Control installed in **`mission-control`** ([Mission Control setup](02-setup-mc.md)).
- HCD cluster **Ready** in **`database`** (any **`PROFILE`**).

```bash
set -a && source .env && set +a   # PROFILE when checking cluster-specific views
```

## What is already running

The lab Helm install (`values.yaml` + `overrides.yaml`) deploys the observability pipeline on **platform** nodes:

| Component | Role |
|-----------|------|
| **Mimir** | Metrics storage and query |
| **Loki** | Log aggregation |
| **Vector aggregator** | Collects and routes telemetry |
| **Alertmanager** | Alert routing |
| **MinIO** | Object storage for Mimir/Loki |
| **Grafana** | Dashboards (enabled in `overrides.yaml`) |

Pinned `values.yaml` keeps `grafana.enabled: false`; this lab turns Grafana on in [`overrides.yaml`](../manifests/mission-control/overrides.yaml). Mimir and Loki datasources are provisioned by the chart.

Verify platform pods:

```bash
kubectl get pods -n mission-control | grep -E 'mimir|loki|aggregator|grafana|minio'
```

---

## Mission Control UI — Observability

Built-in views for cluster health, capacity, and activity — no extra install.

### Port-forward and open

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. Open `https://localhost:8080` ([Mission Control login](02-setup-mc.md#access-the-ui)).
2. **Home** → project **`database`** → cluster **`<PROFILE>`** (same name as your KinD **`PROFILE`**: `two-dcs`, `three-racks`, or `minimal`).
3. Open **Observability**.

Use this for day-to-day checks on any topology. Metrics and logs shown here are backed by the in-cluster Mimir/Loki stack.

### What to look for

- Cassandra / HCD pod health and resource use
- Replication and topology alignment with your **`PROFILE`**
- Recent events or alerts surfaced by the UI

If **Observability** is empty, confirm database pods are Ready and the observability pods in **`mission-control`** are Running (see [Troubleshooting](#troubleshooting)).

---

## Grafana

Grafana ships with the lab install when `grafana.enabled: true` in `overrides.yaml` (default for this repo).

### Confirm Grafana is running

```bash
kubectl get pods -n mission-control -l app.kubernetes.io/name=grafana
kubectl get svc -n mission-control | grep grafana
```

If no Grafana pod exists, you may have installed before Grafana was enabled — [re-enable or upgrade](#enable-or-disable-grafana).

### Access Grafana

Terminal 1 (leave running):

```bash
kubectl port-forward svc/mission-control-grafana -n mission-control 3000:80
```

Open `https://localhost:3000`.

Admin credentials:

```bash
kubectl get secret mission-control-grafana -n mission-control -o jsonpath='{.data.admin-user}' | base64 -d; echo
kubectl get secret mission-control-grafana -n mission-control -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

### Using Grafana in the lab

1. **Dashboards** — Mission Control–bundled dashboards should appear under **Dashboards** (search for Cassandra / HCD / Mission Control).
2. **Explore** — Query **Mimir** (metrics) or **Loki** (logs) if you need ad-hoc investigation.
3. **Configuration** → **Data sources** — Confirm Mimir and Loki are present and healthy.

Charts with no data usually mean telemetry has not reached Mimir/Loki yet, or HCD pods are not Ready — wait for rollout, then refresh.

---

## Enable or disable Grafana

Grafana is controlled in `manifests/mission-control/overrides.yaml`:

```yaml
grafana:
  enabled: true   # set false to disable
```

After editing, upgrade Mission Control ([upgrade steps](02-setup-mc.md#upgrade-mission-control)):

```bash
set -a && source .env && set +a

helm upgrade mission-control "$MC_CHART" \
  -f manifests/mission-control/values.yaml \
  -f manifests/mission-control/overrides.yaml \
  --namespace mission-control \
  --version "$MC_CHART_VERSION"
```

---

## Pipeline checks (CLI)

```bash
kubectl get pods -n mission-control | grep -E 'mimir|loki|aggregator|grafana'
kubectl get missioncontrolcluster -n database
kubectl get pods -n database -l cassandra.datastax.com/cluster="${PROFILE}"
kubectl logs -n mission-control deploy/mission-control-mimir-distributor --tail=30
```

---

## Troubleshooting

| Symptom | Check |
|---------|--------|
| No project **`database`** in UI | Namespace labels in [HCD setup](03-setup-hcd.md#create-the-namespace-for-the-database-to-land-on) |
| Cluster missing in UI | `kubectl get missioncontrolcluster -n database`; name must match **`PROFILE`** |
| UI Observability empty | HCD pods Ready; Mimir/Vector/Loki pods Ready in **`mission-control`** |
| Observability empty; distributor `at least N live replicas required` | Keep chart defaults in `values.yaml` (3 ingesters). Do not set `ingester.replicas: 1` in overrides; `helm upgrade` after fixing |
| No Grafana pod | `grep -A2 '^grafana:' manifests/mission-control/overrides.yaml`; run [helm upgrade](02-setup-mc.md#upgrade-mission-control) |
| Grafana login fails | Secret `mission-control-grafana` in **`mission-control`** |
| Grafana charts empty | **Data sources** in Grafana; observability pods Ready; allow a few minutes after HCD install |
| Grafana pod pending | Platform nodes: `kubectl get nodes -l mission-control.datastax.com/role=platform` |

➡️ **Next:** [Backup and restore](06-backup-restore.md)

## See also

- [`manifests/mission-control/README.md`](../manifests/mission-control/README.md) — Grafana and Helm overlays
- [DataStax Mission Control documentation](https://docs.datastax.com/en/mission-control/)

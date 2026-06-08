# CQL and Data API

Connect to HCD with **CQL** (`cqlsh`, drivers, Mission Control console) and **Data API** (HTTP/JSON) after [HCD setup](03-setup-hcd.md).

**Prerequisites**

- 📂 Run from the **repository root**.
- Steps 1–3 complete; Cassandra pods **Ready** in namespace **`database`**.
- Same **`PROFILE`** as KinD (`minimal`, `three-racks`, or `two-dcs`).

Load `.env` when commands use **`PROFILE`**:

```bash
set -a && source .env && set +a
```

## Lab naming

| `PROFILE` | Cluster (`MissionControlCluster`) | DC1 | DC2 (`two-dcs` only) |
|-----------|-----------------------------------|-----|---------------------|
| `minimal` | `minimal` | `dc1` | — |
| `three-racks` | `three-racks` | `dc1` | — |
| `two-dcs` | `two-dcs` | `dc1` | `dc2` |

| Item | Value |
|------|--------|
| Namespace | `database` |
| Superuser Secret | `superuser` |
| Data API CR (dc1) | `<PROFILE>-data-api` |
| Data API Service (dc1) | `<PROFILE>-data-api-data-api-cip` on port **30080** |

The cluster manifest sets `spec.dataApi: {}` and includes a **`DataApi`** CR for dc1. CQL is always available on the datacenter **headless service** (port **9042**); an optional **CQL gateway** (`CqlConnectivity`) is not in the default manifest — add it from the UI if you need cql-router.

---

## CQL

### Ways to connect

| Method | Best for |
|--------|----------|
| **`scripts/cqlsh.sh`** | Quick CLI from the repo (recommended) |
| **Mission Control CQL console** | Browser `cqlsh` (no local client) |
| **Headless service + port-forward** | Local `cqlsh` or drivers on your machine |
| **CQL gateway** (optional) | External LoadBalancer / Ingress via UI |

### Credentials

```bash
kubectl get secret superuser -n database -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret superuser -n database -o jsonpath='{.data.password}' | base64 -d; echo
```

### ⌨️ CLI — `cqlsh.sh`

Uses **`PROFILE`** from `.env` (cluster name) and datacenter **`dc1`** by default:

```bash
./scripts/cqlsh.sh
./scripts/cqlsh.sh --dc dc2          # two-dcs only (second datacenter)
./scripts/cqlsh.sh --exec "DESCRIBE KEYSPACES;"
```

The script `exec`s into a running Cassandra pod and runs `cqlsh` against the in-cluster headless service.

### 🖥️ Mission Control UI — CQL console

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. Open `https://localhost:8080` ([Mission Control login](02-setup-mc.md#access-the-ui)).
2. **Home** → project **`database`** → cluster **`<PROFILE>`** → **CQL console**.
3. Pick datacenter **`dc1`** (on **`two-dcs`**, use **`dc2`** for the second DC).
4. Log in with **`superuser`** credentials.

Mission Control creates a **`cqlsh-pod`** per cluster in **`database`**.

### ⌨️ Port-forward headless service (local `cqlsh`)

List services:

```bash
kubectl get svc -n database -l app=cassandra
```

For **`two-dcs`**, dc1 is typically reached via a service named like **`two-dcs-dc1-service`** (port **9042**). Replace the name if your cluster differs:

```bash
kubectl port-forward svc/two-dcs-dc1-service -n database 9042:9042

cqlsh -u "$(kubectl get secret superuser -n database -o jsonpath='{.data.username}' | base64 -d)" \
  -p "$(kubectl get secret superuser -n database -o jsonpath='{.data.password}' | base64 -d)" \
  127.0.0.1 9042
```

> **KinD:** Lab clusters use **internode encryption**. If local `cqlsh` fails, use **`scripts/cqlsh.sh`** or the UI console ([CQL console docs](https://docs.datastax.com/en/mission-control/databases/use-cql-console.html)).

### Optional — CQL gateway (cql-router)

Not required for the lab defaults. To expose dc1 outside the cluster:

1. Port-forward the UI (see above).
2. **Connect** → **CQL** → **Add Gateway** → datacenter **`dc1`**, **LoadBalancer**, size **1**, port **9042**.

On KinD, LoadBalancer **EXTERNAL-IP** often stays `<pending>` without [MetalLB](https://kind.sigs.k8s.io/docs/user/loadbalancer/). Prefer the headless service or UI console.

---

## Data API

### What is deployed

| Piece | Role |
|-------|------|
| `dataApi: {}` on `MissionControlCluster` | Enables Data API for the cluster |
| **`DataApi` CR** | Gateway for **`dc1`** (in `mission-control-cluster-${PROFILE}.yaml`) |

Verify after HCD is Ready:

```bash
kubectl get dataapi -n database
kubectl get pods,svc -n database | grep -E 'data-api|stargate'
```

### Port-forward and test

Terminal 1 (leave running):

```bash
kubectl port-forward svc/${PROFILE}-data-api-data-api-cip -n database 30080:30080
```

In-cluster URL (no port-forward): `http://${PROFILE}-data-api-data-api-cip.database.svc.cluster.local:30080`

Terminal 2 — **`findKeyspaces`**:

```bash
DATA_API_USER=$(kubectl get secret superuser -n database -o jsonpath='{.data.username}' | base64 -d)
DATA_API_PASS=$(kubectl get secret superuser -n database -o jsonpath='{.data.password}' | base64 -d)
TOKEN="Cassandra:$(printf '%s' "$DATA_API_USER" | base64 | tr -d '\n'):$(printf '%s' "$DATA_API_PASS" | base64 | tr -d '\n')"

curl -sS -L -X POST "http://127.0.0.1:30080/v1" \
  -H "Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"findKeyspaces": {}}'
```

A JSON body listing keyspaces (possibly empty) means the gateway works. Token format: [HCD Data API auth](https://docs.datastax.com/en/hyper-converged-database/1.2/api-reference/dataapiclient.html#generate-token).

### 🖥️ Mission Control UI

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. **Home** → project **`database`** → cluster **`<PROFILE>`** → **Connect** → **APIs**.
2. Confirm gateway for **`dc1`** (ClusterIP **30080**) or **Add Gateway** if you installed without the bundled manifest.

For **`two-dcs`**, add a second gateway in the UI for **`dc2`** if you need Data API on the second datacenter.

### Remove Data API gateway (CLI)

Edit `manifests/hcd/mission-control-cluster-${PROFILE}.yaml` (remove the `DataApi` document) and re-apply, or delete the CR:

```bash
kubectl delete dataapi "${PROFILE}-data-api" -n database
```

---

## Troubleshooting

| Symptom | Check |
|---------|--------|
| Auth failed | `kubectl get secret superuser -n database` |
| `cqlsh.sh` no pod | `kubectl get pods -n database`; cluster name matches **`PROFILE`** |
| Wrong DC | `./scripts/cqlsh.sh --dc dc2` on **`two-dcs`**; datacenter name **`dc2`** |
| No Data API pod | `kubectl get dataapi "${PROFILE}-data-api" -n database -o yaml`; wait for dc1 Ready |
| `curl` connection refused | Port-forward to **`${PROFILE}-data-api-data-api-cip`** still running |
| No project in UI | Namespace labels: [HCD setup](03-setup-hcd.md#create-the-namespace-for-the-database-to-land-on) |

➡️ **Next:** [Observability](05-observability.md) — Mission Control UI and Grafana.

## See also

- [DataStax CQL console](https://docs.datastax.com/en/mission-control/databases/use-cql-console.html)
- [DataStax Data API](https://docs.datastax.com/en/mission-control/databases/get-started-data-api.html)

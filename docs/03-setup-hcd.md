# Setting up HCD

Deploy **Hyper-Converged Database (HCD)** with a `MissionControlCluster` custom resource.

**Prerequisites**

- 📂 Run from the **repository root**.
- [KinD setup](01-setup-kind.md) and [Mission Control setup](02-setup-mc.md) are complete.
- Same **`PROFILE`** as KinD ([topology profiles](01-setup-kind.md#topology-profiles): `two-dcs`, `three-racks`, or `minimal`).

> 💡 **DNS-1035 names:** `MissionControlCluster` must start with a letter (`two-dcs`, not `2-dcs`).

### Naming (cluster vs datacenter)

| Layer | Example (`PROFILE=two-dcs`) | Used for |
|-------|----------------------------|----------|
| `MissionControlCluster` | `two-dcs` | UI cluster name, K8ssandra cluster prefix |
| `CassandraDatacenter` CR (`metadata.name`) | `dc1`, `dc2` | Pods (`dc1-rack1-sts-0`), labels, Medusa jobs |
| Cassandra ring (`datacenterName`) | `dc1`, `dc2` | `nodetool`, `NetworkTopologyStrategy` |

CQL service: **`two-dcs-dc1-service`** (cluster + DC CR). Avoid repeating the profile in the DC CR name (`two-dcs-dc1` produced `two-dcs-two-dcs-dc1-service`).

## Pick your manifest

All profiles use namespace **`database`**. The manifest file name matches **`PROFILE`**:

| `PROFILE` | Manifest | `MissionControlCluster` | Datacenters | Cassandra pods |
|-----------|----------|-------------------------|-------------|----------------|
| `two-dcs` | `manifests/hcd/mission-control-cluster-two-dcs.yaml` | `two-dcs` | `dc1`, `dc2` | 6 |
| `three-racks` | `manifests/hcd/mission-control-cluster-three-racks.yaml` | `three-racks` | `dc1` | 3 |
| `minimal` | `manifests/hcd/mission-control-cluster-minimal.yaml` | `minimal` | `dc1` | 1 |

Each cluster manifest includes a **Data API** gateway for dc1 (`dataApi: {}` + `DataApi` CR). CQL access is via the datacenter headless service — see [CQL and Data API](04-cql-data-api.md).

| Item | Value |
|------|--------|
| Database namespace | `database` |
| Mission Control namespace | `mission-control` |
| `MissionControlCluster` name | same as **`PROFILE`** |
| Superuser Secret | `superuser` (namespace `database`; created by the operator) |
| Medusa storage Secret | `medusa-bucket-key` (namespace `database`; **you create this before apply**) |
| Data API (dc1) | `<PROFILE>-data-api` → Service `<PROFILE>-data-api-data-api-cip:30080` |

Lab manifests reference Medusa storage (`storageSecretRef: medusa-bucket-key`). Without that Secret, Medusa sidecars fail with `MountVolume.SetUp failed … secret "medusa-bucket-key" not found`.

➡️ **Next:** [CQL and Data API](04-cql-data-api.md) · [Observability](05-observability.md)

## Install

### Create the namespace for the database to land on

```bash
kubectl create namespace database

kubectl label namespace database mission-control.datastax.com/is-project=true
kubectl annotate namespace database mission-control.datastax.com/project-name=database
```

The `label` and `annotation` ensure that Mission Control UI shows the namespace as well.

### Prepare Medusa credentials Secret (required before HCD apply)

Mission Control ships **MinIO** in namespace **`mission-control`**. Cluster manifests point Medusa at Secret **`medusa-bucket-key`** in **`database`** — create it **before** `kubectl apply` on `MissionControlCluster`.

```bash
MINIO_USER=$(kubectl get secret mission-control-minio -n mission-control -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASS=$(kubectl get secret mission-control-minio -n mission-control -o jsonpath='{.data.rootPassword}' | base64 -d)

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: medusa-bucket-key
  namespace: database
type: Opaque
stringData:
  credentials: |-
    [default]
    aws_access_key_id = ${MINIO_USER}
    aws_secret_access_key = ${MINIO_PASS}
EOF

kubectl get secret medusa-bucket-key -n database
```

> 💡 The S3 bucket **`medusa-backups`** is only required when you run [backups](06-backup-restore.md). HCD rollout needs the Secret only.

### 🖥️ Mission Control UI

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. Open `https://localhost:8080` ([Mission Control login](02-setup-mc.md#access-the-ui)).
2. Open project **`database`**.
3. **Create Cluster** and match your KinD **`PROFILE`**, or skip if you use the CLI manifest.

### ⌨️ CLI

```bash
set -a && source .env && set +a   # PROFILE from .env (same as KinD setup)

kubectl apply -f manifests/hcd/mission-control-cluster-${PROFILE}.yaml
```

## Watch rollout

```bash
watch kubectl get pods -n database -o wide --sort-by=.spec.nodeName
```

| `PROFILE` | What to expect |
|-----------|----------------|
| `minimal` | One Cassandra pod on the database worker |
| `three-racks` | Three pods (`dc1-rack1-sts-0`, `rack2`, `rack3`) |
| `two-dcs` | DC1 racks ready first, then DC2 (`dc2-rack*-sts-0`) |

## Reconciliation and troubleshooting

```bash
kubectl get missioncontrolcluster -n database
kubectl describe missioncontrolcluster "${PROFILE}" -n database
kubectl get k8ssandracluster,cassandradatacenter -n database
kubectl get pods -n database -o wide --sort-by=.spec.nodeName
kubectl logs -n mission-control deploy/mission-control-operator --tail=50
```

## Update / upgrade HCD

Edit the manifest, then re-apply with the same **`PROFILE`**:

```bash
set -a && source .env && set +a   # PROFILE from .env (same as KinD setup)

kubectl apply -f manifests/hcd/mission-control-cluster-${PROFILE}.yaml
```

For large topology changes, deleting and recreating the `MissionControlCluster` may be simpler.

## Remove an HCD cluster

```bash
set -a && source .env && set +a   # PROFILE from .env (same as KinD setup)

kubectl delete missioncontrolcluster "${PROFILE}" -n database
kubectl delete namespace database
```

Or delete the whole KinD cluster ([KinD setup](01-setup-kind.md)).

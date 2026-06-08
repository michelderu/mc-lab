# Backup and restore (Mission Control + Medusa)

Run a backup and restore workflow for HCD using the **Mission Control UI** and verify underlying **Medusa** resources.

**Prerequisites**

- 📂 Run from the **repository root**.
- Steps 1–3 complete ([HCD setup](03-setup-hcd.md)); Cassandra pods **Ready** in **`database`**.
- 💡 Optional: [CQL / Data API](04-cql-data-api.md), [Observability](05-observability.md).

| Item | Value |
|------|--------|
| Namespace | `database` |
| Cluster | **`<PROFILE>`** (`two-dcs`, `three-racks`, `minimal`) |
| DC1 (Medusa / CDC name) | **`dc1`** |
| DC2 (`two-dcs` only) | **`dc2`** |
| Cassandra DC names (replication, repairs) | `dc1`, `dc2` |
| Superuser Secret | `superuser` |
| Medusa storage Secret | `medusa-bucket-key` |
| `MedusaConfiguration` | `database-backup` (in cluster manifest) |
| Backup prefix per cluster | same as **`PROFILE`** (`minimal`, `three-racks`, `two-dcs`) |
| MinIO bucket | `medusa-backups` |
| MinIO host (in-cluster) | `mission-control-minio.mission-control.svc.cluster.local:9000` |

Each `manifests/hcd/mission-control-cluster-${PROFILE}.yaml` defines **`MedusaConfiguration` `database-backup`** (shared MinIO settings) and references it from `MissionControlCluster` with a per-profile **`prefix`**. Secret **`medusa-bucket-key`** must exist before HCD apply ([HCD setup — Medusa Secret](03-setup-hcd.md#create-medusa-credentials-secret-required-before-hcd-apply)). You still need bucket **`medusa-backups`** in MinIO before backups run.

➡️ Prior step: [Observability](05-observability.md)

---

## Configure object storage (once per lab)

### 1) Create bucket `medusa-backups`

Get MinIO credentials:

```bash
MINIO_USER=$(kubectl get secret mission-control-minio -n mission-control -o jsonpath='{.data.rootUser}' | base64 -d)
echo $MINIO_USER
MINIO_PASS=$(kubectl get secret mission-control-minio -n mission-control -o jsonpath='{.data.rootPassword}' | base64 -d)
echo $MINIO_PASS
```

#### 🖥️ MinIO console

```bash
kubectl port-forward svc/mission-control-minio-console -n mission-control 9001:9001
```

1. Open `http://localhost:9001`.
2. Log in with `MINIO_USER` / `MINIO_PASS`.
3. **Buckets** → **Create Bucket** → name **`medusa-backups`**.

#### ⌨️ CLI (`mc` or `aws`)

```bash
kubectl port-forward svc/mission-control-minio -n mission-control 9000:9000
```

With MinIO Client:

```bash
mcli alias set local http://localhost:9000 "$MINIO_USER" "$MINIO_PASS"
mcli mb --ignore-existing local/medusa-backups
mcli ls local
```

With AWS CLI:

```bash
AWS_ACCESS_KEY_ID="$MINIO_USER"
AWS_SECRET_ACCESS_KEY="$MINIO_PASS"
aws --endpoint-url http://localhost:9000 s3api create-bucket --bucket medusa-backups
aws --endpoint-url http://localhost:9000 s3api list-buckets
```

> Keep the port-forward on **9000** running while using `mc` / `aws` against `localhost:9000`.

### 2) Create Medusa credentials Secret

Should already exist from [HCD setup](03-setup-hcd.md#create-medusa-credentials-secret-required-before-hcd-apply). If missing, repeat the `kubectl apply` in that section (uses `MINIO_USER` / `MINIO_PASS` from step 1).

### 3) Medusa backup configuration

The cluster manifest applies **`MedusaConfiguration` `database-backup`** and wires the cluster via `medusaConfigurationRef`. No separate UI step is required for a standard lab install.

#### ⌨️ CLI (verify)

```bash
set -a && source .env && set +a

kubectl get medusaconfiguration database-backup -n database
kubectl get missioncontrolcluster "${PROFILE}" -n database -o jsonpath='{.spec.k8ssandra.medusa.medusaConfigurationRef.name}{" prefix="}{.spec.k8ssandra.medusa.storageProperties.prefix}{"\n"}'
kubectl get secret medusa-bucket-key -n database
```

If Medusa was added or changed after install, re-apply the profile manifest:

```bash
kubectl apply -f manifests/hcd/mission-control-cluster-${PROFILE}.yaml
```

#### 🖥️ UI (optional)

Use **Settings** → **Backup Configuration** only if you are **not** using the manifest `MedusaConfiguration`, or to compare with UI-created config. Manifest values: S3-compatible, bucket `medusa-backups`, region `us-east-1`, host `mission-control-minio.mission-control.svc.cluster.local`, port `9000`, **Secure** / **Verify SSL** off.

---

## Walkthrough: backup → data loss → restore

Validates:

1. Sample data
2. Backup (UI or `MedusaBackupJob`)
3. Track Medusa resources
4. Simulate data loss
5. Restore
6. Validate rows

> **Lab scope:** proves Medusa on KinD. In production, define schedules, retention, and cross-DC restore drills.

Use **`PROFILE=two-dcs`** below for two-datacenter replication. On **`minimal`** or **`three-racks`**, use `'dc1': 1` (or `3`) only and skip DC2 steps.

---

## 1) Create sample data

### ⌨️ CLI (`scripts/cqlsh.sh`)

Uses **`PROFILE`** from `.env`, secret **`superuser`**, and dc1 by default ([CQL guide](04-cql-data-api.md)).

```bash
set -a && source .env && set +a

# two-dcs — replication across dc1 and dc2
./scripts/cqlsh.sh --exec "CREATE KEYSPACE IF NOT EXISTS lab_restore WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3, 'dc2': 3} AND durable_writes = true;"
./scripts/cqlsh.sh --exec "CREATE TABLE IF NOT EXISTS lab_restore.users (id int PRIMARY KEY, name text);"
./scripts/cqlsh.sh --exec "INSERT INTO lab_restore.users (id, name) VALUES (1, 'alice');"
./scripts/cqlsh.sh --exec "INSERT INTO lab_restore.users (id, name) VALUES (2, 'bob');"
./scripts/cqlsh.sh --exec "SELECT * FROM lab_restore.users;"
```

On **`minimal`** or **`three-racks`**, use one DC only — set the `dc1` replication factor to your rack count (**`1`** for minimal, **`3`** for three-racks):

```bash
# minimal
./scripts/cqlsh.sh --exec "CREATE KEYSPACE IF NOT EXISTS lab_restore WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 1} AND durable_writes = true;"

# three-racks
./scripts/cqlsh.sh --exec "CREATE KEYSPACE IF NOT EXISTS lab_restore WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3} AND durable_writes = true;"
```

Then run the `CREATE TABLE`, `INSERT`, and `SELECT` lines above unchanged.

Run a **full repair** on `lab_restore` so replicas are consistent before backup.

> **Reaper, not `CassandraTask`:** Mission Control runs anti-entropy repairs through [**Reaper**](https://docs.datastax.com/en/mission-control/reference/platform-components.html) (control-plane repair manager). The UI **Repairs** view submits repair runs to Reaper. Do **not** use `CassandraTask` with `command: repair` — that is not the supported repair path for user keyspaces in this lab (see [CassandraTask](https://docs.datastax.com/en/mission-control/reference/cassandra-task.html) for ops like `restart`, `replacenode`, `cleanup`). Do not run `nodetool repair` on Cassandra pods by hand.

### 🖥️ Mission Control UI (Reaper)

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. Open `https://localhost:8080` ([login](02-setup-mc.md#access-the-ui)).
2. **Home** → project **`database`** → cluster **`<PROFILE>`** → **Repairs**.
3. **Run repair** (or equivalent manual repair action).
4. **Keyspace:** `lab_restore`.
5. Suggested options (match the UI labels): **Parallelism** `DATACENTER_AWARE`, **Repair threads** `1`.
6. On **`two-dcs`**: run repair for datacenter **`dc1`**, wait until it succeeds, then repeat for **`dc2`**. On single-DC profiles, one run for **`dc1`** is enough.

Repair is complete when the UI shows success for each datacenter.

### ⌨️ CLI — check Reaper (optional)

Reaper is enrolled by the k8ssandra/Mission Control operators when the cluster is Ready. Confirm a Reaper pod exists (name pattern varies by version):

```bash
kubectl get pods -n database | grep -i reaper
kubectl get pods -n mission-control | grep -i reaper
```

---

## 2) Start a backup

### 🖥️ UI

1. **Home** → **`database`** → **`<PROFILE>`** → **Backups**.
2. **Create backup** / **Run backup now**.
3. Datacenter **`dc1`** → submit.
4. Note the backup name for restore.

### ⌨️ CLI (`MedusaBackupJob`)

```bash
BACKUP_JOB_NAME="backup-lab-$(date -u +%Y-%m-%dT%H-%M-%S | tr '[:upper:]' '[:lower:]')z"

kubectl apply -f - <<EOF
apiVersion: medusa.k8ssandra.io/v1alpha1
kind: MedusaBackupJob
metadata:
  name: ${BACKUP_JOB_NAME}
  namespace: database
spec:
  backupType: full
  cassandraDatacenter: ${PROFILE}-dc1
EOF

kubectl get medusabackupjob "${BACKUP_JOB_NAME}" -n database -w
```

> `backupType: differential` only after a recent **full** backup for that datacenter.

---

## 3) Track backup status

```bash
kubectl get medusabackupjob -n database
kubectl get medusabackup -n database
kubectl get medusabackupjob "${BACKUP_JOB_NAME}" -n database -o yaml   # replace name
```

Complete when the job has `finishTime`, `MedusaBackup.status.status` is `SUCCESS`, and the UI shows success.

Note the **`MedusaBackup`** name for restore (often the same as `BACKUP_JOB_NAME` when you created the job via CLI):

```bash
kubectl get medusabackup -n database
```

---

## 4) Simulate data loss

```bash
set -a && source .env && set +a

./scripts/cqlsh.sh --exec "TRUNCATE lab_restore.users;"
./scripts/cqlsh.sh --exec "SELECT * FROM lab_restore.users;"
```

---

## 5) Restore

### 🖥️ UI

1. **Backups** → select the backup from step 2.
2. **Restore** → datacenter **`dc1`** → submit.

### ⌨️ CLI (`MedusaRestoreJob`)

Restore needs the **`MedusaBackup`** resource name (not the UI display label). List backups, then create a restore job targeting the same datacenter you backed up.

```bash
set -a && source .env && set +a

# Name of the MedusaBackup object from step 3 (CLI backups often match BACKUP_JOB_NAME)
kubectl get medusabackup -n database
BACKUP_NAME="backup-lab-2026-06-03t12-00-00z"   # replace with your backup NAME column

RESTORE_JOB_NAME="restore-lab-$(date -u +%Y-%m-%dT%H-%M-%S | tr '[:upper:]' '[:lower:]')z"

kubectl apply -f - <<EOF
apiVersion: medusa.k8ssandra.io/v1alpha1
kind: MedusaRestoreJob
metadata:
  name: ${RESTORE_JOB_NAME}
  namespace: database
spec:
  cassandraDatacenter: ${PROFILE}-dc1
  backup: ${BACKUP_NAME}
EOF
```

`spec.cassandraDatacenter` is the **CassandraDatacenter CR name** (`dc1`, `dc2`, …). In this lab the CR name matches the ring name (`datacenterName`).

Watch progress:

```bash
kubectl get medusarestorejob "${RESTORE_JOB_NAME}" -n database -w

kubectl wait --for=condition=complete "medusarestorejob/${RESTORE_JOB_NAME}" -n database --timeout=7200s

kubectl get medusarestore -n database
kubectl get medusarestorejob "${RESTORE_JOB_NAME}" -n database -o yaml
```

Restore is complete when the job finishes successfully and the UI (if open) shows success. Medusa restores **one datacenter per job** — this lab walkthrough restores **`dc1`** only.

To retry a failed restore:

```bash
kubectl delete medusarestorejob "${RESTORE_JOB_NAME}" -n database
```

Then fix the backup/Medusa issue and apply a new `MedusaRestoreJob` with the same `backup` name.

---

## 6) Validate

```bash
set -a && source .env && set +a

./scripts/cqlsh.sh --exec "SELECT * FROM lab_restore.users;"
```

Expect rows for **alice** and **bob**.

---

## Cleanup (optional)

```bash
set -a && source .env && set +a

./scripts/cqlsh.sh --exec "DROP TABLE IF EXISTS lab_restore.users;"
./scripts/cqlsh.sh --exec "DROP KEYSPACE IF EXISTS lab_restore;"
```

---

## Troubleshooting

| Symptom | Check |
|---------|--------|
| No backup in UI | Cluster Ready; Medusa enabled on `MissionControlCluster` |
| Backup fails | `medusa-bucket-key` in **`database`**; bucket exists; MinIO reachable |
| Missing secret on Medusa pods | Create [step 2](#2-create-medusa-credentials-secret) secret |
| No `MedusaBackup` object | `kubectl get events -n database --sort-by=.lastTimestamp` |
| Wrong DC in job | `cassandraDatacenter` must be **`dc1`**, not `dc1` |
| Restore job not found | `spec.backup` must match a `MedusaBackup` **metadata.name** from `kubectl get medusabackup -n database` |
| Restore stuck / failed | `kubectl describe medusarestorejob "${RESTORE_JOB_NAME}" -n database`; Medusa pod logs in **`database`** |
| Restore OK, no rows | Backup ID and target DC; Reaper repair succeeded before backup |
| Repair fails in UI | Cluster Ready; `kubectl get pods -n database \| grep -i reaper`; retry from **Repairs** |
| `CassandraTask` repair ignored | Use UI **Repairs** (Reaper), not `command: repair` on `CassandraTask` |
| `cqlsh.sh` no pod | `kubectl get pods -n database`; cluster label must match **`PROFILE`**; try `./scripts/cqlsh.sh --dc dc1` |

## See also

- [**Restore DC1 and rebuild corrupt DC2**](../mops/01-restore-and-rebuild.md) — advanced `two-dcs` MOP after you have a valid DC1 backup
- [DataStax backup documentation](https://docs.datastax.com/en/mission-control/)

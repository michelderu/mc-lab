# Datacenter canary upgrades (lab MOP)

Manual **datacenter canary** procedure for **`PROFILE=two-dcs`**. Mission Control does not yet offer a single workflow for canary rollouts across topologies (single DC, single rack, multi-DC, and so on). Until that exists, apply a change to **one datacenter first**, validate it, then promote the same change to **cluster scope** so the operators roll the remaining datacenters.

**Canary datacenter:** **`dc1`** (`dc1`, first entry in `spec.k8ssandra.cassandra.datacenters`). **Remainder:** **`dc2`** (`dc2`).

**Prerequisites**

- [Mission Control lab](../README.md) steps 1–6 complete.
- KinD cluster and HCD deployed from [`manifests/hcd/mission-control-cluster-two-dcs.yaml`](../manifests/hcd/mission-control-cluster-two-dcs.yaml) (namespace **`database`**, cluster **`two-dcs`**).
- `kubectl` can patch `MissionControlCluster` in **`database`**.

> ⚠️ **Version upgrades:** Read **HCD** release notes for your target `serverVersion` before Procedure B. SSTable format changes can make downgrade non-trivial.

---

## Cluster layout

| Item | Value |
|------|--------|
| `MissionControlCluster` | `two-dcs` (namespace `database`) |
| Canary DC (ring / CR) | `dc1` / `dc1` — patch `datacenters[0]` |
| Remainder DC (ring / CR) | `dc2` / `dc2` |
| Topology | 3 racks per DC, 1 node per rack (`size: 3`) |
| Database | HCD (`serverType: hcd`; cluster `serverVersion` in manifest, currently **`1.2.5`**) |

---

## How canary merging works

- **Cluster-level** settings under `spec.k8ssandra.cassandra` apply to every datacenter unless overridden.
- **Per-datacenter** settings under `spec.k8ssandra.cassandra.datacenters[n]` are **merged** with cluster-level values; you do not need to repeat the full `cassandraYaml` or `serverVersion` when canarying a single field.
- **Promotion:** after the canary DC is healthy, **remove** the per-DC override and set the same values at **cluster** level. Operators rolling-restart datacenters that do not already match; the canary DC is **skipped**.

---

## Architecture

```
Initial:  cluster-level config/version  ──►  all DCs
Step 1:   override on datacenters[0]   ──►  rolling restart dc1 only
Step 3:   cluster-level + remove [0]   ──►  rolling restart dc2 only (dc1 skipped)
```

---

## Prerequisites

**Environment variables** (repository root; `set -a && source .env && set +a` sets `PROFILE=two-dcs`):

```bash
NAMESPACE="database"
CLUSTER_NAME="two-dcs"

CANARY_DC_CR="dc1"
CANARY_DC_RING="dc1"
REMAINDER_DC_CR="dc2"
REMAINDER_DC_RING="dc2"

# Match manifests/hcd/mission-control-cluster-two-dcs.yaml before the exercise
SERVER_VERSION_CURRENT="1.2.5"
SERVER_VERSION_TARGET="1.2.6"   # set to a HCD version you have tested in this lab

CANARY_SEED_POD="$(kubectl get pods -n ${NAMESPACE} \
  -l "cassandra.datastax.com/datacenter=${CANARY_DC_CR},cassandra.datastax.com/cluster=${CLUSTER_NAME}" \
  -o jsonpath='{.items[0].metadata.name}')"
```

**Watch helper** (reuse in validation steps):

```bash
watch_dc_rollout() {
  local dc_cr="$1"
  kubectl get cassandradatacenter "${dc_cr}" -n "${NAMESPACE}"
  kubectl get pods -n "${NAMESPACE}" \
    -l "cassandra.datastax.com/datacenter=${dc_cr},app.kubernetes.io/name=cassandra"
  kubectl wait --for=condition=ready "cassandradatacenter/${dc_cr}" \
    -n "${NAMESPACE}" --timeout=1800s
}
```

---

## Process overview

Two procedures share the same four-step shape:

| Procedure | Canary on `datacenters[0]` (`dc1`) | Promote to cluster |
|-----------|------------------------------------------|--------------------|
| **A — Configuration** | `config.cassandraYaml` | `spec.k8ssandra.cassandra.config` |
| **B — Server version** | `serverVersion` | `spec.k8ssandra.cassandra.serverVersion` |

KinD rollouts are often **10–20 minutes per datacenter** depending on load and image pull; plan accordingly.

---

# Procedure A — Configuration change (canary)

**Example:** change `compaction_throughput_mb_per_sec` from **`0`** (cluster-wide) to **`200`** on **`dc1`**, then promote cluster-wide.

### Optional baseline (lab exercise)

Set a cluster-wide value before canarying (skip if your cluster already has the baseline you want):

```bash
kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type merge \
  --patch-file /dev/stdin <<'EOF'
spec:
  k8ssandra:
    cassandra:
      config:
        cassandraYaml:
          compaction_throughput_mb_per_sec: 0
          dynamic_snitch: false
EOF
```

Wait for any rolling restart to finish before starting the canary.

---

### A.1 — Deploy change to canary DC (`dc1`)

Add the override on **`datacenters[0]`** only. JSON patch avoids replacing the entire `datacenters` array (which would drop `dc2`).

```bash
kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type=json \
  --patch-file /dev/stdin <<'EOF'
[
  {
    "op": "add",
    "path": "/spec/k8ssandra/cassandra/datacenters/0/config",
    "value": {
      "cassandraYaml": {
        "compaction_throughput_mb_per_sec": 200
      }
    }
  }
]
EOF
```

Target shape:

```yaml
# spec.k8ssandra.cassandra.datacenters[0].config
cassandraYaml:
  compaction_throughput_mb_per_sec: 200
```

> 💡 If `config` already exists on `datacenters[0]`, use `"op": "replace"` instead of `"add"`, or edit [`mission-control-cluster-two-dcs.yaml`](../manifests/hcd/mission-control-cluster-two-dcs.yaml) and `kubectl apply -f …`.

> 💡 Add `--dry-run=client` on `kubectl patch` to validate without applying.

---

### A.2 — Validate canary DC is online

Operators perform a **rolling restart of `dc1` only** with the merged configuration.

```bash
watch_dc_rollout "${CANARY_DC_CR}"
```

```bash
kubectl get pods -n ${NAMESPACE} \
  -l "cassandra.datastax.com/datacenter=${CANARY_DC_CR}" \
  -o wide
kubectl exec -it "${CANARY_SEED_POD}" -n ${NAMESPACE} -c cassandra -- nodetool status
```

**✓ Validation:** All **`dc1`** pods are `Running` and ready; `CassandraDatacenter` is ready. If something fails, **keep changes scoped to `dc1`** (fix or roll back the per-DC block) before promoting.

---

### A.3 — Promote configuration to cluster scope

1. Set the value at **cluster** level.
2. **Remove** the per-DC override on `datacenters[0]`.

```bash
kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type merge \
  --patch-file /dev/stdin <<'EOF'
spec:
  k8ssandra:
    cassandra:
      config:
        cassandraYaml:
          compaction_throughput_mb_per_sec: 200
EOF

kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type=json \
  --patch='[{"op": "remove", "path": "/spec/k8ssandra/cassandra/datacenters/0/config"}]'
```

After promotion:

```yaml
# spec.k8ssandra.cassandra.config.cassandraYaml.compaction_throughput_mb_per_sec: 200
# (remove spec.k8ssandra.cassandra.datacenters[0].config)
```

Update [`mission-control-cluster-two-dcs.yaml`](../manifests/hcd/mission-control-cluster-two-dcs.yaml) if you want Git to match the cluster.

---

### A.4 — Validate remainder DC (`dc2`)

A rolling restart runs on **`dc2`**; **`dc1`** already matches and is **skipped**.

```bash
watch_dc_rollout "${REMAINDER_DC_CR}"
```

```bash
kubectl get pods -n ${NAMESPACE} \
  -l "cassandra.datastax.com/datacenter=${REMAINDER_DC_CR}" \
  -o wide
kubectl exec -it "${CANARY_SEED_POD}" -n ${NAMESPACE} -c cassandra -- nodetool status
```

**✓ Validation:** All pods in both datacenters are ready; no errors on `K8ssandraCluster` / `MissionControlCluster`.

---

# Procedure B — HCD server version upgrade (canary)

Upgrade **`dc1`** first, then promote `serverVersion` to cluster scope and remove the per-DC override.

### Caveats

1. Read **HCD** release notes for `${SERVER_VERSION_TARGET}` before patching.
2. If the release changes **SSTable format**, new data and compactions use the new format before `upgradesstables` finishes. Rolling back may need extra steps — check release notes.

---

### B.1 — Deploy version to canary DC (`dc1`)

```bash
kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type=json \
  --patch="[{\"op\": \"add\", \"path\": \"/spec/k8ssandra/cassandra/datacenters/0/serverVersion\", \"value\": \"${SERVER_VERSION_TARGET}\"}]"
```

> 💡 Use `"op": "replace"` if `serverVersion` is already set on `datacenters[0]`.

---

### B.2 — Validate canary DC is online

Rolling restart applies the new version to **`dc1`** only.

```bash
watch_dc_rollout "${CANARY_DC_CR}"
```

```bash
kubectl get missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  -o jsonpath='{.spec.k8ssandra.cassandra.datacenters[0].serverVersion}{"\n"}'
kubectl exec -it "${CANARY_SEED_POD}" -n ${NAMESPACE} -c cassandra -- nodetool version 2>/dev/null || true
```

**✓ Validation:** Canary pods ready; per-DC `serverVersion` is `${SERVER_VERSION_TARGET}`. On failure, stay on per-DC overrides until resolved.

---

### B.3 — Promote version to cluster scope

```bash
kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type merge \
  --patch "{\"spec\":{\"k8ssandra\":{\"cassandra\":{\"serverVersion\":\"${SERVER_VERSION_TARGET}\"}}}}"

kubectl patch missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  --type=json \
  --patch='[{"op": "remove", "path": "/spec/k8ssandra/cassandra/datacenters/0/serverVersion"}]'
```

Update [`mission-control-cluster-two-dcs.yaml`](../manifests/hcd/mission-control-cluster-two-dcs.yaml) (`serverVersion: "${SERVER_VERSION_TARGET}"`) so Git matches the cluster.

---

### B.4 — Validate remainder DC (`dc2`)

Rolling restart on **`dc2`**; **`dc1`** is skipped.

```bash
watch_dc_rollout "${REMAINDER_DC_CR}"
```

```bash
kubectl get pods -n ${NAMESPACE} \
  -l "cassandra.datastax.com/datacenter=${REMAINDER_DC_CR}" \
  -o wide
kubectl get missioncontrolcluster ${CLUSTER_NAME} -n ${NAMESPACE} \
  -o jsonpath='{.spec.k8ssandra.cassandra.serverVersion}{"\n"}'
kubectl exec -it "${CANARY_SEED_POD}" -n ${NAMESPACE} -c cassandra -- nodetool status
```

**✓ Validation:** Cluster-level `serverVersion` is `${SERVER_VERSION_TARGET}`; both DCs healthy in `nodetool status`.

---

## Rollback

**During canary (A.1 / B.1):** Remove the per-DC override (JSON `remove` on `datacenters[0]` paths) or set fields back to cluster defaults. Wait for **`dc1`** to stabilize before changing cluster-level spec.

**After promotion (A.3 / B.3):** Reverting cluster-level fields rolls **every** datacenter that no longer matches — not a single-DC canary. If SSTable format changed, use release-note downgrade steps or [backup and restore](../docs/06-backup-restore.md) in the lab.

**Configuration only:** Restore previous `cassandraYaml` at cluster scope and remove stale per-DC blocks so merges stay predictable.

---

## See also

* [Lab runbook](../README.md) · [HCD setup](../docs/03-setup-hcd.md)
* [`manifests/hcd/mission-control-cluster-two-dcs.yaml`](../manifests/hcd/mission-control-cluster-two-dcs.yaml) — full cluster manifest; `kubectl apply -f …` after editing is an alternative to JSON patches
* [MOP 01 — Restore and rebuild](01-restore-and-rebuild.md) — datacenter removal/re-add (not a canary rollout)
* [Backup and restore](../docs/06-backup-restore.md) — if you need to recover after a failed version change

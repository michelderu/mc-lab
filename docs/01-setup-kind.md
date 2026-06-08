# Setting up KinD

Create the local **KinD** cluster `mc`, apply topology labels for HCD scheduling, and verify nodes before Mission Control.

**Prerequisites**

- 📂 Run all commands from the **repository root** (`kind/`, `scripts/` paths are relative to it).
- Docker, KinD, `kubectl`, `helm`, and `htpasswd` on the host (see [preflight](#preflight)).
- Cluster name **`mc`** (kubectl context `kind-mc`).

➡️ **Next:** [Mission Control setup](02-setup-mc.md)

## Preflight

```bash
./scripts/preflight-check.sh
```

## Topology profiles

Set **`PROFILE`** once in `.env` (`two-dcs`, `three-racks`, or `minimal`) — the same value is used in [HCD setup](03-setup-hcd.md). Default in `.env.example`: **`two-dcs`**.

| Profile | KinD config | HCD manifest | Cassandra pods |
|---------|-------------|----------------|----------------|
| **two-dcs** (default) | `kind/kind-cluster-two-dcs.yaml` | `manifests/hcd/mission-control-cluster-two-dcs.yaml` | 6 |
| **three-racks** | `kind/kind-cluster-three-racks.yaml` | `manifests/hcd/mission-control-cluster-three-racks.yaml` | 3 |
| **minimal** | `kind/kind-cluster-minimal.yaml` | `manifests/hcd/mission-control-cluster-minimal.yaml` | 1 |

| Label | Meaning |
|-------|---------|
| `mission-control.datastax.com/role` | `platform` or `database` (set at KinD join) |
| `topology.kubernetes.io/region` | Datacenter (`dc1`, `dc2`) — from `scripts/apply-topology-labels.sh` |
| `topology.kubernetes.io/zone` | Rack (`rack1`, …) |

See also [`kind/README.md`](../kind/README.md) for config file names.

> 📦 **Resources:** **minimal** = 3 KinD nodes · **three-racks** = 6 · **two-dcs** = 9 (most RAM).

> Switching `PROFILE` later: delete the KinD cluster or namespace **`database`** before re-running with a different profile.

## Lab config

```bash
cp .env.example .env   # PROFILE=two-dcs | three-racks | minimal
```

Now edit `.env` ensuring the topology profile is set.

```bash
set -a && source .env && set +a
```

## Create the cluster and apply the topology labels

```bash
kind create cluster --name mc --config kind/kind-cluster-${PROFILE}.yaml
./scripts/apply-topology-labels.sh
```

| `PROFILE` | KinD nodes | HCD layout (step 3) |
|-----------|------------|---------------------|
| `minimal` | 3 | 1 DC, 1 rack, 1 pod |
| `three-racks` | 6 | 1 DC, 3 racks, 3 pods |
| `two-dcs` | 9 | 2 DCs × 3 racks, 6 pods |

## Verify

```bash
kubectl config use-context kind-mc
docker ps --filter name=mc-
kubectl cluster-info
kubectl get nodes -o wide
```

Confirm topology labels:

```bash
kubectl get nodes -L mission-control.datastax.com/role,topology.kubernetes.io/region,topology.kubernetes.io/zone
```

A `two-dcs` topology would show:

```text
NAME               STATUS   ROLES           AGE   VERSION   ROLE       REGION   ZONE
mc-control-plane   Ready    control-plane   45h   v1.35.0                       
mc-worker          Ready    <none>          45h   v1.35.0   platform            
mc-worker2         Ready    <none>          45h   v1.35.0   platform            
mc-worker3         Ready    <none>          45h   v1.35.0   database   dc1      rack1
mc-worker4         Ready    <none>          45h   v1.35.0   database   dc1      rack2
mc-worker5         Ready    <none>          45h   v1.35.0   database   dc1      rack3
mc-worker6         Ready    <none>          45h   v1.35.0   database   dc2      rack1
mc-worker7         Ready    <none>          45h   v1.35.0   database   dc2      rack2
mc-worker8         Ready    <none>          45h   v1.35.0   database   dc2      rack3
```

> 💡 **ROLES vs ROLE:** **ROLES** is the Kubernetes node role; **ROLE** is `mission-control.datastax.com/role`.

## Pause, stop, or delete the cluster

| Option | Keeps | Frees host RAM? | Use when |
|--------|--------|-----------------|----------|
| **Pause** (`docker pause`) | Full cluster state | No | Short break |
| **Stop** (`docker stop`) | Disk inside node containers | Yes | Overnight |
| **Delete** (`kind delete cluster`) | Nothing | Yes | Full reset |

**Pause / resume:**

```bash
ids=$(docker ps -q --filter "name=mc-")
[ -n "$ids" ] && docker pause $ids
# resume:
[ -n "$ids" ] && docker unpause $ids
```

**Stop / start** (do not `docker rm` the nodes):

```bash
ids=$(docker ps -q --filter "name=mc-")
[ -n "$ids" ] && docker stop $ids
# start:
ids=$(docker ps -aq --filter "name=mc-")
[ -n "$ids" ] && docker start $ids
kubectl config use-context kind-mc
kubectl get nodes
```

**Delete cluster** (removes all namespaces, PVCs, Mission Control, HCD):

```bash
kind delete cluster --name mc
```

After delete, redo **Create the cluster**, then ➡️ [Mission Control setup](02-setup-mc.md).

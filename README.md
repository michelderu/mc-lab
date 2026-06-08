# Mission Control lab (KinD + HCD)

Local lab for **Mission Control** on **KinD** with embedded MinIO and **HCD** clusters pinned to platform/database topology labels.

Run every command from the **repository root**.

## Runbook (in order)

| Step | Guide | What you do |
|------|--------|-------------|
| **1** | [**KinD setup**](docs/01-setup-kind.md) | Preflight, cluster `mc`, topology labels |
| **2** | [**Mission Control setup**](docs/02-setup-mc.md) | Registry, cert-manager, Helm install, UI |
| **3** | [**HCD setup**](docs/03-setup-hcd.md) | `MissionControlCluster` deploy and rollout |
| **4** | [**CQL and Data API**](docs/04-cql-data-api.md) | `cqlsh`, UI console, Data API `curl` |
| **5** | [**Observability**](docs/05-observability.md) | MC UI metrics, Grafana |
| **6** | [**Backup and restore**](docs/06-backup-restore.md) | Medusa + MinIO walkthrough |
| **MOP** | [**Restore & rebuild DC2**](mops/01-restore-and-rebuild.md) | Optional: corrupt DC2 / survivor restore (`two-dcs` only) |

## Lab conventions

Pick one topology profile in **`.env`** (same value for steps 1–3):

```bash
cp .env.example .env   # set PROFILE=two-dcs | three-racks | minimal
set -a && source .env && set +a
```

| Profile | KinD config | HCD manifest | Cassandra pods |
|---------|-------------|----------------|----------------|
| **two-dcs** (default) | `kind/kind-cluster-two-dcs.yaml` | `manifests/hcd/mission-control-cluster-two-dcs.yaml` | 6 |
| **three-racks** | `kind/kind-cluster-three-racks.yaml` | `manifests/hcd/mission-control-cluster-three-racks.yaml` | 3 |
| **minimal** | `kind/kind-cluster-minimal.yaml` | `manifests/hcd/mission-control-cluster-minimal.yaml` | 1 |

| Item | Value |
|------|--------|
| Database namespace | `database` |
| Mission Control namespace | `mission-control` |
| `MissionControlCluster` | same as **`PROFILE`** (`two-dcs`, `three-racks`, or `minimal`) |
| Superuser Secret | `superuser` (namespace `database`) |

> Switching `PROFILE` later: delete the KinD cluster or namespace **`database`** before re-running with a different profile.

## CQL and Data API

See [**docs/04-cql-data-api.md**](docs/04-cql-data-api.md). Quick start: `./scripts/cqlsh.sh` and port-forward `${PROFILE}-data-api-data-api-cip` on **30080**.

## Repo layout

| Path | Purpose |
|------|---------|
| [`docs/`](docs/01-setup-kind.md) | KinD, Mission Control, and HCD setup guides |
| [`kind/`](kind/README.md) | KinD cluster YAML per profile |
| [`manifests/mission-control/`](manifests/mission-control/README.md) | Pinned Helm `values.yaml` + lab `overrides.yaml` |
| [`manifests/hcd/`](manifests/hcd/) | `MissionControlCluster` and optional gateways |
| [`scripts/`](scripts/) | Preflight, topology labels, `cqlsh` |
| [`mops/`](mops/README.md) | Advanced procedures (failed DC rebuild) |
| [`concepts/`](concepts/README.md) | Architecture and wiring reference |

## Concepts (reference)

- [`concepts/deployment-structure.md`](concepts/deployment-structure.md) — Helm + operator flow
- [`concepts/software-components-wiring.md`](concepts/software-components-wiring.md) — component map
- [`concepts/understanding-a-deployment-quickly.md`](concepts/understanding-a-deployment-quickly.md) — triage checklist

Mission Control is the management plane for DataStax Cassandra/HCD on Kubernetes: UI, APIs, and operators that reconcile `MissionControlCluster` resources into running database topologies.

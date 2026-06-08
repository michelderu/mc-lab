# HCD MissionControlCluster manifests

**Prerequisites**

- 📂 Paths in the runbooks are relative to the **repository root**.
- Namespace **`database`** for all profiles.

```bash
set -a && source .env && set +a   # PROFILE in .env (match KinD)

# namespace + medusa-bucket-key first — docs/03-setup-hcd.md
kubectl apply -f manifests/hcd/mission-control-cluster-${PROFILE}.yaml
```

| `PROFILE` | File | Layout |
|-----------|------|--------|
| `two-dcs` (default) | `mission-control-cluster-two-dcs.yaml` | 2 DCs × 3 racks |
| `three-racks` | `mission-control-cluster-three-racks.yaml` | 1 DC, 3 racks |
| `minimal` | `mission-control-cluster-minimal.yaml` | 1 DC, 1 rack |

| Add-on | Notes |
|--------|--------|
| Medusa | `MedusaConfiguration` `database-backup` + per-profile `prefix` in each cluster manifest |
| Data API (dc1) | Included in each `mission-control-cluster-*.yaml` (`dataApi: {}` + `DataApi` CR) |
| CQL | Headless service on **9042**; optional gateway via UI — [`docs/04-cql-data-api.md`](../../docs/04-cql-data-api.md) |

➡️ **Install:** [`docs/03-setup-hcd.md`](../../docs/03-setup-hcd.md) · **Connect:** [`docs/04-cql-data-api.md`](../../docs/04-cql-data-api.md) · **Backups:** [`docs/06-backup-restore.md`](../../docs/06-backup-restore.md) · **MOP:** [`mops/01-restore-and-rebuild.md`](../../mops/01-restore-and-rebuild.md) (`two-dcs` only)

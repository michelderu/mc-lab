# Mission Control lab — MOPs

Operational procedures beyond the main [runbook](../README.md).

| MOP | When to use |
|-----|-------------|
| [**01-restore-and-rebuild.md**](01-restore-and-rebuild.md) | **`PROFILE=two-dcs`**: DC1 restored from Medusa backup, **DC2 corrupt** — rebuild failed datacenter |
| [**02-upgrade-db-canary.md**](02-upgrade-db-canary.md) | **`PROFILE=two-dcs`**: manual **datacenter canary** for config or server version (DC1 first, then cluster-wide) |

**Prerequisites for MOP 01:** Complete [backup and restore](../docs/06-backup-restore.md) first (`lab_restore`, valid backup on `dc1`).

**Prerequisites for MOP 02:** Healthy `two-dcs` cluster from the main lab runbook; no Medusa restore required.

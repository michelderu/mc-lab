# KinD cluster configs

KinD YAML for this lab. Full walkthrough: [`docs/01-setup-kind.md`](../docs/01-setup-kind.md).

**Prerequisites**

- 📂 Run commands from the **repository root**.

```bash
set -a && source .env && set +a   # PROFILE in .env

kind create cluster --name mc --config kind/kind-cluster-${PROFILE}.yaml
./scripts/apply-topology-labels.sh
```

| `PROFILE` | File |
|-----------|------|
| `two-dcs` (default) | [`kind-cluster-two-dcs.yaml`](kind-cluster-two-dcs.yaml) |
| `three-racks` | [`kind-cluster-three-racks.yaml`](kind-cluster-three-racks.yaml) |
| `minimal` | [`kind-cluster-minimal.yaml`](kind-cluster-minimal.yaml) |

`role` labels are in each YAML (`kubeadmConfigPatches`). `topology.kubernetes.io/region` / `zone` are applied by the script.

# Concepts

Architecture and wiring reference for this lab — not install steps.

**Prerequisites**

- 📂 Step-by-step runbooks live under [`docs/`](../docs/01-setup-kind.md) (commands from the **repository root**; **`PROFILE`** in `.env` for steps 1–3 — see [lab conventions](../README.md#lab-conventions)).

## Runbooks

| Step | Guide | Topic |
|------|-------|--------|
| 1 | [`../docs/01-setup-kind.md`](../docs/01-setup-kind.md) | KinD cluster, topology labels |
| 2 | [`../docs/02-setup-mc.md`](../docs/02-setup-mc.md) | Helm install, upgrade, UI |
| 3 | [`../docs/03-setup-hcd.md`](../docs/03-setup-hcd.md) | HCD deploy, upgrade |

## Deep dives

- [`deployment-structure.md`](deployment-structure.md) — Helm-first deployment and reconciliation flow
- [`software-components-wiring.md`](software-components-wiring.md) — component inventory and YAML wiring
- [`understanding-a-deployment-quickly.md`](understanding-a-deployment-quickly.md) — fast triage checklist

## Config assets

- [`../kind/README.md`](../kind/README.md) — KinD YAML profiles
- [`../manifests/mission-control/README.md`](../manifests/mission-control/README.md) — Helm `values.yaml` + `overrides.yaml`
- [`../manifests/hcd/`](../manifests/hcd/) — `MissionControlCluster` examples

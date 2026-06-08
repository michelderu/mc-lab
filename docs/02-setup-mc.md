# Setting up Mission Control

Install Mission Control on the KinD cluster from [KinD setup](01-setup-kind.md) using **embedded MinIO**, pinned Helm values, and platform `nodeSelector`s.

**Prerequisites**

- 📂 Run from the **repository root**.
- KinD cluster **`mc`** is running ([KinD setup](01-setup-kind.md)).

➡️ **Next:** [HCD setup](03-setup-hcd.md) — use the same **`PROFILE`** as KinD.

## What you install

- Mission Control operator, UI, APIs
- **Dex** (local static login in this lab)
- **cert-manager** (TLS for MC and database certs)
- **Loki / Mimir** with in-cluster **MinIO**
- Operators pinned to `mission-control.datastax.com/role=platform` nodes

Chart files: `manifests/mission-control/values.yaml` (pinned upstream) + `manifests/mission-control/overrides.yaml` (lab changes). See [`manifests/mission-control/README.md`](../manifests/mission-control/README.md).

## Install cert-manager

Required before Mission Control.

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version "v1.16.1" \
  --set crds.enabled=true \
  --set 'extraArgs[0]=--enable-certificate-owner-ref=true'

kubectl get pods -n cert-manager
```

## Registry login and chart version

Set registry credentials and chart version in `.env` (see `.env.example`: `MC_REGISTRY_*`, `MC_CHART`, `MC_CHART_VERSION`).
After any change to `.env`, reload your environment variables with:

```bash
set -a && source .env && set +a
```

## Pin chart version and default values

`manifests/mission-control/values.yaml` is upstream chart defaults for a pinned release. Regenerate when you change the Mission Control version:

```bash
set -a && source .env && set +a   # PROFILE from .env (same as KinD setup)

helm registry login registry.replicated.com \
  --username "$MC_REGISTRY_USERNAME" \
  --password "$MC_REGISTRY_PASSWORD"

helm show values "$MC_CHART" --version "$MC_CHART_VERSION" > manifests/mission-control/values.yaml
```

Keep lab-specific settings in `manifests/mission-control/overrides.yaml` only (Dex, MinIO/Loki, platform `nodeSelector`s, Grafana). Dex and Grafana: [`manifests/mission-control/README.md`](../manifests/mission-control/README.md).

## Install Mission Control

```bash
set -a && source .env && set +a   # PROFILE from .env (same as KinD setup)

helm install mission-control "$MC_CHART" \
  -f manifests/mission-control/values.yaml \
  -f manifests/mission-control/overrides.yaml \
  --namespace mission-control \
  --create-namespace \
  --version "$MC_CHART_VERSION"

watch kubectl get pods -n mission-control -o wide --sort-by=.spec.nodeName
```

Platform pods should land on workers with `role=platform`. For `two-dcs` topology it should look like:

```text
NAME                                                        READY   STATUS      RESTARTS        AGE   IP            NODE         NOMINATED NODE   READINESS GATES
mission-control-grafana-6678bc699d-4wccf                    3/3     Running     0               43h   10.244.1.26   mc-worker    <none>           <none>
mission-control-k8ssandra-operator-85fbf7bb87-b9h79         1/1     Running     1 (4h54m ago)   45h   10.244.1.9    mc-worker    <none>           <none>
replicated-f65f69fd5-7tld5                                  1/1     Running     0               45h   10.244.1.4    mc-worker    <none>           <none>
mission-control-aggregator-0                                1/1     Running     0               45h   10.244.1.18   mc-worker    <none>           <none>
mission-control-cass-operator-5fd4999d77-sb9bb              1/1     Running     1 (4h54m ago)   45h   10.244.1.8    mc-worker    <none>           <none>
mission-control-crd-patcher-gcpb4                           0/1     Completed   0               43h   10.244.1.25   mc-worker    <none>           <none>
loki-read-d48c8c6cb-92w7l                                   1/1     Running     0               45h   10.244.1.14   mc-worker    <none>           <none>
mission-control-crd-upgrader-6bchl                          0/1     Completed   0               43h   10.244.1.24   mc-worker    <none>           <none>
mission-control-mimir-ingester-2                            1/1     Running     0               45h   10.244.1.12   mc-worker    <none>           <none>
mission-control-operator-5b9fc8d6bf-hv6kh                   1/1     Running     1 (4h54m ago)   45h   10.244.1.10   mc-worker    <none>           <none>
mission-control-mimir-store-gateway-0                       1/1     Running     0               45h   10.244.1.19   mc-worker    <none>           <none>
mission-control-mimir-query-scheduler-99fdc87d-rbfrx        1/1     Running     0               45h   10.244.1.17   mc-worker    <none>           <none>
mission-control-mimir-alertmanager-0                        1/1     Running     0               45h   10.244.1.20   mc-worker    <none>           <none>
mission-control-mimir-querier-7846798965-lgr78              1/1     Running     0               45h   10.244.1.15   mc-worker    <none>           <none>
mission-control-mimir-distributor-65495575c4-cnzgn          1/1     Running     0               45h   10.244.1.16   mc-worker    <none>           <none>
mission-control-mimir-gateway-58ff6bd844-hsdmg              1/1     Running     0               45h   10.244.1.13   mc-worker    <none>           <none>
mission-control-mimir-ingester-0                            1/1     Running     0               45h   10.244.2.15   mc-worker2   <none>           <none>
mission-control-mimir-query-frontend-57d5c76f8d-z4gvr       1/1     Running     0               45h   10.244.2.20   mc-worker2   <none>           <none>
loki-backend-0                                              1/1     Running     0               45h   10.244.2.13   mc-worker2   <none>           <none>
loki-write-0                                                1/1     Running     0               45h   10.244.2.24   mc-worker2   <none>           <none>
mission-control-mimir-overrides-exporter-66578c7ffc-zcm6w   1/1     Running     0               45h   10.244.2.19   mc-worker2   <none>           <none>
mission-control-mimir-compactor-0                           1/1     Running     0               45h   10.244.2.12   mc-worker2   <none>           <none>
mission-control-mimir-querier-7846798965-rpwsk              1/1     Running     0               45h   10.244.2.21   mc-worker2   <none>           <none>
mission-control-mimir-ingester-1                            1/1     Running     0               45h   10.244.2.14   mc-worker2   <none>           <none>
mission-control-mimir-query-scheduler-99fdc87d-dcql6        1/1     Running     0               45h   10.244.2.23   mc-worker2   <none>           <none>
mission-control-loki-gateway-5fd99d7dc-2htp7                1/1     Running     0               45h   10.244.2.18   mc-worker2   <none>           <none>
mission-control-mimir-ruler-6c966b6948-c2m9d                1/1     Running     0               45h   10.244.2.22   mc-worker2   <none>           <none>
mission-control-kube-state-metrics-67665dd795-ldrjm         1/1     Running     3 (4h22m ago)   45h   10.244.2.4    mc-worker2   <none>           <none>
mission-control-minio-7d69ddc8fd-gs9jl                      1/1     Running     0               45h   10.244.2.11   mc-worker2   <none>           <none>
mission-control-dex-6c8c8dd5f4-622mc                        1/1     Running     3 (4h54m ago)   45h   10.244.2.17   mc-worker2   <none>           <none>
mission-control-ui-f8d4df66f-9wsj7                          1/1     Running     3 (4h23m ago)   45h   10.244.2.16   mc-worker2   <none>           <none>
mission-control-mimir-make-minio-buckets-5.4.0-nfls6        0/1     Completed   0               45h   10.244.10.3   mc-worker7   <none>           <none>
```

## Access the UI

```bash
kubectl port-forward svc/mission-control-ui -n mission-control 8080:8080
```

1. Open `https://localhost:8080`.
2. Log in with Dex credentials ([default lab user](../manifests/mission-control/README.md#dex-login-optional-override)).

Keep this port-forward running for later work (HCD UI, CQL, Data API, observability) whenever you need **Mission Control login**.

## Upgrade Mission Control

After editing `overrides.yaml` or refreshing pinned `values.yaml` for a new `MC_CHART_VERSION` in `.env`:

```bash
set -a && source .env && set +a

helm upgrade mission-control "$MC_CHART" \
  -f manifests/mission-control/values.yaml \
  -f manifests/mission-control/overrides.yaml \
  --namespace mission-control \
  --version "$MC_CHART_VERSION"
```

## Uninstall Mission Control (keep KinD)

Removes the Mission Control namespace and MC PVCs. **Does not** remove the database namespace (`database`).

```bash
helm uninstall mission-control -n mission-control
kubectl delete namespace mission-control
```

Reinstall from **Install Mission Control** above. HCD clusters in other namespaces remain until you delete them ([HCD setup](03-setup-hcd.md)).

## Deeper reference

- [`concepts/deployment-structure.md`](../concepts/deployment-structure.md)
- [`concepts/software-components-wiring.md`](../concepts/software-components-wiring.md)

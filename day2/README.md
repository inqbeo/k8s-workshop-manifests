# Day 2 — Lab manifests

Plain Kubernetes YAML for the Day-2 labs (Kubernetes basics). Built around the
demo app `ghcr.io/inqbeo/containerization-demo-webapp:0.1.1` — the same app from
Day 1, now deployed to Kubernetes, unchanged.

> Apply everything **into your own namespace**:
> `kubectl apply -n <your-namespace> -f <file>` (or set it once:
> `kubectl config set-context --current --namespace <your-namespace>`).

## Order (matches the day's blocks)

| File | Block | What it does |
|------|------:|--------------|
| `01-pod.yaml` | 3 | A bare Pod — fragile, no self-healing (delete it → it's gone). |
| `02-deployment.yaml` | 4 | A Deployment with liveness/readiness **probes**. Rolling update by editing `COMPANY_NAME` / `THEME`. |
| `03-configmap.yaml` | 5 | Config as a cluster object (`COMPANY_NAME`, `THEME`, `LOG_LEVEL`). |
| `04-secret.yaml` | 5 | The app password as a Secret (base64, **not** encrypted). |
| `05-pvc.yaml` | 5 | A `ReadWriteOnce` PVC for `/data` — data survives Pod restarts. |
| `06-deployment-app.yaml` | 5 | The “full” Deployment: `envFrom` ConfigMap+Secret, PVC mount, `fsGroup`, probes. Supersedes `02`. Switch its `strategy` to `Recreate` so a single RWO volume is never double-mounted. |
| `07-service.yaml` | 6 | A `ClusterIP` Service — reach the app in-cluster as `app:8080`. |
| `08-debug-pod.yaml` | 6 | A `network-multitool` pod to test service discovery by name / DNS. |
| `09-whoami.yaml` | 6 | A stateless, scalable `traefik/whoami` Deployment + Service — used to *see* a Service round-robin across replicas (the todo app can't: single RWO PVC). |

> **Day 2 stays on SQLite + a PVC.** A real database (Postgres as a StatefulSet)
> and switching the app over to it now live in **`../day3/`** — together with
> Ingress/Gateway, certificates and HA.

## Typical flow

```bash
NS=<your-namespace>

# Block 3 — a bare Pod
kubectl apply -n $NS -f 01-pod.yaml
kubectl -n $NS port-forward pod/app 8080:8080      # open http://localhost:8080
kubectl -n $NS delete pod app                       # ...and it's gone

# Block 4 — a Deployment (self-healing, scaling, rolling update, probes)
kubectl apply -n $NS -f 02-deployment.yaml
kubectl -n $NS get pods -w                           # watch it self-heal
# edit THEME: coral -> navy, then:
kubectl apply -n $NS -f 02-deployment.yaml
kubectl -n $NS rollout status deploy/app
kubectl -n $NS rollout undo deploy/app

# Block 5 — config, secret, persistent storage
kubectl apply -n $NS -f 03-configmap.yaml -f 04-secret.yaml -f 05-pvc.yaml
kubectl apply -n $NS -f 06-deployment-app.yaml       # full version (replaces 02)

# Block 6 — a Service + discovery by name
kubectl apply -n $NS -f 07-service.yaml -f 08-debug-pod.yaml
kubectl -n $NS exec -it netshoot -- curl http://app:8080/healthz
kubectl -n $NS exec -it netshoot -- nslookup app.$NS.svc.cluster.local
```

## Notes

- **Namespace-agnostic:** no `namespace:` is hard-coded — always apply with `-n`.
- **StorageClass:** `storageClassName` is commented out → the cluster default is
  used. On the workshop cluster that is **Longhorn**; uncomment to pin it.
- **Not production:** credentials are inline demo values. The clean, HA way
  (a real database via the CloudNativePG operator, plus public exposure) is
  **Day 3 / Day 4**.

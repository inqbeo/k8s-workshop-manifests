# Day 3 — Lab manifests

Day 3 builds on the Day-2 app: a **real database**, automated upkeep, getting
traffic **in** (Ingress / Gateway API + TLS), locking it down, and scaling.

All manifests are namespace-agnostic — apply them into your own namespace with
`kubectl apply -n <your-namespace> -f <file>`.

| File | Lab | Topic |
|------|-------|-------|
| `01-postgres-statefulset.yaml` | 9 | Postgres as a **StatefulSet** (own volume) + a Service `db`. |
| `02-switch-app-to-postgres.yaml` | 9 | Switch the app SQLite → Postgres: `DB_DRIVER` + `DATABASE_URL` change **and** the Deployment drops the SQLite `/data` PVC mount (state is in Postgres now). Apply → rollout, then `kubectl delete pvc app-data`. |
| `03-seed-job.yaml` | 9 | A **Job** that seeds a few demo TODOs into the DB (runs once → Completed). |
| `04-cleanup-cronjob.yaml` | 9 | A **CronJob** that deletes completed TODOs every 5 min (force a run with `kubectl create job --from=cronjob/cleanup`). |
| `05-gateway.yaml` | 10b | **Variant A — Gateway API (1/2):** your **own** Gateway (per namespace), HTTP/HTTPS listeners for your host + per-student cert (cert-manager annotation). Apply first. |
| `05-httproute.yaml` | 10b | **Variant A — Gateway API (2/2):** an HTTPRoute attaching the app Service to **your** Gateway. Apply after `05-gateway.yaml`. |
| `06-ingress.yaml` | 10a | **Variant B — classic Ingress:** the same goal via a single Ingress (TLS via cert-manager). |
| `07-networkpolicy.yaml` | 11 | A **NetworkPolicy**: only the app Pods may reach the DB on 5432. |
| `08-metrics-port.yaml` | 12b | The Service `app` gets a second, cluster-internal port **`metrics`** (8081) — since app 0.4.0 `/metrics` lives on its own listener, never on the public port. |
| `09-servicemonitor.yaml` | 12b | A **ServiceMonitor**: the platform's Prometheus scrapes the app's `/metrics` (selects Service `app` by label, port name `metrics`). |
| `10-prometheusrule.yaml` | 12b | *Going further* — a **PrometheusRule**: `TodoDatabaseDown` (db_up == 0 for 2 min, one alert per namespace) and a 5 % error-rate alert; both name the namespace. |

> The HA database (CloudNativePG operator), Kustomize/Helm and GitOps are
> **Day 4**.

```bash
NS=<your-namespace>
# Lab 9 — real DB + automated upkeep
kubectl apply -n $NS -f 01-postgres-statefulset.yaml
kubectl apply -n $NS -f 02-switch-app-to-postgres.yaml
kubectl -n $NS rollout status deploy/app         # data now lives in Postgres
kubectl -n $NS delete pvc app-data               # the SQLite volume is unused now
kubectl apply -n $NS -f 03-seed-job.yaml          # seed demo TODOs (a Job)
kubectl apply -n $NS -f 04-cleanup-cronjob.yaml   # tidy up on a schedule

# Lab 10 — public HTTPS URL (pick ONE variant; your namespace goes into the host — the lab guide does this via sed)
kubectl apply -n $NS -f 05-gateway.yaml           # Variant A (1/2) — your Gateway
kubectl apply -n $NS -f 05-httproute.yaml         # Variant A (2/2) — the HTTPRoute
# kubectl apply -n $NS -f 06-ingress.yaml         # Variant B — classic Ingress

# Lab 11 — only the app may reach the DB
kubectl apply -n $NS -f 07-networkpolicy.yaml

# Lab 12b — metrics (and, going further, alerts)
kubectl apply -n $NS -f 08-metrics-port.yaml
kubectl apply -n $NS -f 09-servicemonitor.yaml
kubectl apply -n $NS -f 10-prometheusrule.yaml
```

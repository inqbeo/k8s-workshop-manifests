# Day 3: Lab manifests

Day 3 builds on the Day-2 app: a **real database**, automated upkeep, getting
traffic **in** (Ingress / Gateway API + TLS), locking it down, scaling, and
**metrics** for the app and its database.

All manifests are namespace-agnostic: apply them into your own namespace with
`kubectl apply -n <your-namespace> -f <file>`.

| File | Lab | Topic |
|------|-------|-------|
| `01-postgres-statefulset.yaml` | 3.1 | Postgres as a **StatefulSet** (own volume) + a headless Service `db`. |
| `02-switch-app-to-postgres.yaml` | 3.1 | Switch the app SQLite → Postgres: `DB_DRIVER` + `DATABASE_URL` change **and** the Deployment drops the SQLite `/data` PVC mount (state is in Postgres now). Apply → rollout, then `kubectl delete pvc app-data`. |
| `03-seed-job.yaml` | 3.1 | A **Job** that seeds a few demo TODOs into the DB (runs once → Completed). |
| `04-cleanup-cronjob.yaml` | 3.1 | A **CronJob** that deletes completed TODOs every 5 min (force a run with `kubectl create job --from=cronjob/cleanup`). |
| `06-ingress.yaml` | 3.2 Part A | **Classic Ingress:** host → Service, TLS via cert-manager (ACME HTTP-01, issuer `letsencrypt` in name, ZeroSSL behind it). |
| `05-gateway.yaml` | 3.2 Part B | **Gateway API (1/2):** your **own** Gateway (per namespace), HTTP/HTTPS listeners for your host + per-student cert (cert-manager annotation). Apply first. |
| `05-httproute.yaml` | 3.2 Part B | **Gateway API (2/2):** an HTTPRoute attaching the app Service to **your** Gateway. Apply after `05-gateway.yaml`. |
| `07-networkpolicy.yaml` | 3.3 | A **NetworkPolicy**: only the app Pods may reach the DB on 5432. |
| `08-metrics-port.yaml` | 3.5 | The Service `app` gets a second, cluster-internal port **`metrics`** (8081): `/metrics` lives on its own listener, never on the public port. |
| `09-servicemonitor.yaml` | 3.5 | A **ServiceMonitor**: the platform's Prometheus scrapes the app's `/metrics` (selects Service `app` by label, port name `metrics`). |
| `10-prometheusrule.yaml` | 3.5 | *Going further*: a **PrometheusRule** with `TodoDatabaseDown` (db_up == 0 for 2 min, one alert per namespace) and a 5 % error-rate alert; both name the namespace. |
| `11-postgres-exporter.yaml` | 3.5 | The Postgres StatefulSet from `01` plus a **postgres_exporter sidecar** (`quay.io/prometheuscommunity/postgres-exporter:v0.20.1`, credentials from the existing `db-secret`) and a second Service port **`metrics`** (9187). Re-apply keeps the data. |
| `12-servicemonitor-postgres.yaml` | 3.5 | A **ServiceMonitor** for the exporter (`pg_up`, `pg_stat_database_*`) plus a second **NetworkPolicy** that admits the monitoring namespace on 9187 (the policy from `07` would block the scrape). |

> The HA database (CloudNativePG operator), Kustomize/Helm and GitOps are
> **Day 4**.

```bash
NS=<your-namespace>
# Lab 3.1: real DB + automated upkeep
kubectl apply -n $NS -f 01-postgres-statefulset.yaml
kubectl apply -n $NS -f 02-switch-app-to-postgres.yaml
kubectl -n $NS rollout status deploy/app         # data now lives in Postgres
kubectl -n $NS delete pvc app-data               # the SQLite volume is unused now
kubectl apply -n $NS -f 03-seed-job.yaml          # seed demo TODOs (a Job)
kubectl apply -n $NS -f 04-cleanup-cronjob.yaml   # tidy up on a schedule

# Lab 3.2: public HTTPS URL (pick ONE part; your namespace goes into the host, the lab guide does this via sed)
# kubectl apply -n $NS -f 06-ingress.yaml         # Part A: classic Ingress
kubectl apply -n $NS -f 05-gateway.yaml           # Part B (1/2): your Gateway
kubectl apply -n $NS -f 05-httproute.yaml         # Part B (2/2): the HTTPRoute

# Lab 3.3: only the app may reach the DB
kubectl apply -n $NS -f 07-networkpolicy.yaml

# Lab 3.5: metrics for the app, then the database (and, going further, alerts)
kubectl apply -n $NS -f 08-metrics-port.yaml
kubectl apply -n $NS -f 09-servicemonitor.yaml
kubectl apply -n $NS -f 11-postgres-exporter.yaml          # db-0 is replaced, 2/2 READY
kubectl apply -n $NS -f 12-servicemonitor-postgres.yaml
kubectl apply -n $NS -f 10-prometheusrule.yaml
```

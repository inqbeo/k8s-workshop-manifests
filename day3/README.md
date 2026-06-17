# Day 3 — Lab manifests

Day 3 builds on the Day-2 app: a **real database**, automated upkeep, getting
traffic **in** (Ingress / Gateway API + TLS), locking it down, and scaling.

All manifests are namespace-agnostic — apply them into your own namespace with
`kubectl apply -n <your-namespace> -f <file>`.

| File | Block | Topic |
|------|-------|-------|
| `01-postgres-statefulset.yaml` | 2 | Postgres as a **StatefulSet** (own volume) + a Service `db`. |
| `02-switch-app-to-postgres.yaml` | 2 | Switch the app SQLite → Postgres: `DB_DRIVER` + `DATABASE_URL` change **and** the Deployment drops the SQLite `/data` PVC mount (state is in Postgres now). Apply → rollout, then `kubectl delete pvc app-data`. |
| `03-seed-job.yaml` | 2 | A **Job** that seeds a few demo TODOs into the DB (runs once → Completed). |
| `04-cleanup-cronjob.yaml` | 2 | A **CronJob** that deletes completed TODOs every 5 min (force a run with `kubectl create job --from=cronjob/cleanup`). |
| `05-httproute.yaml` | 4 | **Variant A — Gateway API:** an HTTPRoute attaching the app to the shared Gateway (public HTTPS). |
| `06-ingress.yaml` | 4 | **Variant B — classic Ingress:** the same goal via an Ingress (TLS via cert-manager). |
| `07-networkpolicy.yaml` | 5 | A **NetworkPolicy**: only the app Pods may reach the DB on 5432. |

> The HA database (CloudNativePG operator), Kustomize/Helm, GitOps and Rancher
> are **Day 4**.

```bash
NS=<your-namespace>
# Block 2 — real DB + automated upkeep
kubectl apply -n $NS -f 01-postgres-statefulset.yaml
kubectl apply -n $NS -f 02-switch-app-to-postgres.yaml
kubectl -n $NS rollout restart deploy/app        # data now lives in Postgres
kubectl apply -n $NS -f 03-seed-job.yaml          # seed demo TODOs (a Job)
kubectl apply -n $NS -f 04-cleanup-cronjob.yaml   # tidy up on a schedule

# Block 4 — public HTTPS URL (pick ONE; your namespace goes into the host (the lab guide does this via sed))
kubectl apply -n $NS -f 05-httproute.yaml         # Variant A — Gateway API
# kubectl apply -n $NS -f 06-ingress.yaml         # Variant B — classic Ingress

# Block 5 — only the app may reach the DB
kubectl apply -n $NS -f 07-networkpolicy.yaml
```

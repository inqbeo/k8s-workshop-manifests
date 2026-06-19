# Day 4 — Lab manifests

Taking the **same app** from "hand-applied YAML" to **professionally operated**:
package it (Kustomize, then the published Helm chart), give it a **highly-available**
database with the **CloudNativePG** operator, and roll it out with **GitOps**.

> Apply into your **own namespace**:
> `kubectl apply -n <your-namespace> -f <file>` (or set it once:
> `kubectl config set-context --current --namespace <your-namespace>`).

## What's here

| Path | Lab | What it does |
|------|----:|--------------|
| `kustomize/base/` | 13 | The app as a Kustomize **base**: `deployment.yaml`, `service.yaml`, and a `configMapGenerator` for `app-config` (hash-suffixed). |
| `kustomize/overlays/prod/` | 13 | A **prod overlay**: a `replicas-3.yaml` strategic-merge patch + a `configMapGenerator` `behavior: merge` (COMPANY_NAME/THEME). Render with `kubectl kustomize`, apply with `kubectl apply -k`. |
| `values.yaml` | 14 | Helm values for the `todo` chart: external Postgres (via a `uri` Secret), ingress + host + cert. In Lab 16 this file moves into your Gitea repo for Flux. |
| `01-pg-cluster.yaml` | 15 | A **CloudNativePG `Cluster`** (`db`, 3 instances) → primary + 2 replicas, the `db-rw` Service and the `db-app` Secret (with a ready-to-use `uri`). |
| `02-seed-cnpg-job.yaml` | 15 | A **Job** that re-seeds the new cnpg database with demo TODOs (reads the `db-app` Secret, talks to `db-rw`). |
| `02-switch-app-to-cnpg.yaml` | 15 | *Deprecated / reference only* — the raw-manifest form of the DB switch. The lab switches the app with `helm upgrade` instead. |
| `gitops/` | 16 | The whole desired state for your **Gitea** repo — copy it in one shot (`cp gitops/* .`) alongside your own `values.yaml`: `cnpg-cluster.yaml` (the HA database, same Cluster as `01-pg-cluster.yaml`), `ocirepository.yaml` (chart source), `release.yaml` (HelmRelease using `values.yaml`), `kustomization.yaml` (ties them together; `values.yaml` → hash-suffixed `todo-values` ConfigMap), and `kustomizeconfig.yaml` (nameReference so the hash is rewritten into the HelmRelease's `valuesFrom` → editing values auto-triggers a Helm upgrade). |

## Not in this folder (by design)

- **The Helm chart** (Lab 14) lives on an OCI registry: `oci://ghcr.io/inqbeo/charts/todo`
  — you install it, you don't build it. (Its source is in the `todo-app/chart/` of the
  course repo.)
- **GitOps** (Lab 16): the chart stays on OCI, and **your `values.yaml`** lives in your
  per-student **Gitea** repo `todo-gitops`; Flux's `OCIRepository` / `GitRepository` /
  `HelmRelease` are pre-provisioned for your namespace.

## Typical flow

```bash
NS=<your-namespace>

# Lab 13 — Kustomize
kubectl kustomize kustomize/overlays/prod      # render (no apply)
kubectl apply -k kustomize/overlays/prod -n $NS

# Lab 14 — Helm (chart from OCI; uninstall before Lab 15 so cnpg owns the DB)
helm install todo oci://ghcr.io/inqbeo/charts/todo --version 0.1.2 -n $NS
helm uninstall todo -n $NS

# Lab 15 — HA Postgres via cnpg
kubectl apply -n $NS -f 01-pg-cluster.yaml
kubectl apply -n $NS -f 02-switch-app-to-cnpg.yaml
kubectl -n $NS rollout restart deploy/app
```

> The cnpg operator, cert-manager, Traefik, Longhorn and Flux are installed
> cluster-wide by the platform; you only need **namespace-admin**.

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
| `01-pg-cluster.yaml` | 15 | A **CloudNativePG `Cluster`** (`db`, 3 instances) → primary + 2 replicas, the `db-rw` Service and the `db-app` Secret (with a ready-to-use `uri`). |
| `02-switch-app-to-cnpg.yaml` | 15 | Re-point the app's `DATABASE_URL` at cnpg's `db-rw` by reading the `uri` key from the `db-app` Secret. |

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

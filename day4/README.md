# Day 4: Lab manifests

Taking the **same app** from "hand-applied YAML" to **professionally operated**:
package it (Kustomize, then the published Helm chart), give it a **highly-available**
database with the **CloudNativePG** operator, and roll it out with **GitOps**.

> Apply into your **own namespace**:
> `kubectl apply -n <your-namespace> -f <file>` (or set it once:
> `kubectl config set-context --current --namespace <your-namespace>`).

## What's here

| Path | Lab | What it does |
|------|----:|--------------|
| `kustomize/base/` | 4.1 | The app as a Kustomize **base**: `deployment.yaml`, `service.yaml`, `ingress.yaml` (generic host `app.example.invalid`), and a `configMapGenerator` for `app-config` (hash-suffixed). |
| `kustomize/overlays/prod/` | 4.1 | A **prod overlay**: a `replicas-3.yaml` strategic-merge patch, a **JSON patch** that sets the real Ingress host (`studentNN` placeholder, the guide seds it), + a `configMapGenerator` `behavior: merge` (COMPANY_NAME/THEME). Render with `kubectl kustomize`, apply with `kubectl apply -k`. |
| `values.yaml` | 4.2 | Helm values for the `todo` chart: external Postgres (via a `uri` Secret), ingress + host + cert. In Lab 4.4 this file moves into your Gitea repo for Flux. |
| `01-pg-cluster.yaml` | 4.3 | A **CloudNativePG `Cluster`** (`db`, 3 instances) → primary + 2 replicas, the `db-rw` Service and the `db-app` Secret (with a ready-to-use `uri`). |
| `02-seed-cnpg-job.yaml` | 4.3 | A **Job** that re-seeds the new cnpg database with demo TODOs (reads the `db-app` Secret, talks to `db-rw`). |
| `02-switch-app-to-cnpg.yaml` | 4.3 | *Deprecated / reference only*: the raw-manifest form of the DB switch. The lab switches the app with `helm upgrade` instead. |
| `gitops/` | 4.4 | The whole desired state for your **Gitea** repo. Copy it in one shot (`cp gitops/* .`) alongside your own `values.yaml`: `cnpg-cluster.yaml` (the HA database, same Cluster as `01-pg-cluster.yaml`), `ocirepository.yaml` (chart source), `release.yaml` (HelmRelease using `values.yaml`), `kustomization.yaml` (ties them together; `values.yaml` → hash-suffixed `todo-values` ConfigMap), and `kustomizeconfig.yaml` (nameReference so the hash is rewritten into the HelmRelease's `valuesFrom` → editing values auto-triggers a Helm upgrade). |

## Not in this folder (by design)

- **The Helm chart** (Lab 4.2) lives on an OCI registry: `oci://ghcr.io/inqbeo/charts/todo`.
  You install it, you don't build it. (Its source is in the `todo-app/chart/` of the
  course repo.)
- **The Flux bootstrap** (Lab 4.4): your per-student **Gitea** repo `todo-gitops` starts
  empty (README only). You fill it from `gitops/` + your `values.yaml`, then create the
  two pointers into it yourself with the `flux` CLI: a `GitRepository` (plus its
  credentials Secret) and a Flux `Kustomization`, all in **your** namespace. Only the
  Flux controllers and the `flux` ServiceAccount in your namespace are pre-provisioned.

## Typical flow

```bash
NS=<your-namespace>

# Lab 4.1: Kustomize
sed -i "s/studentNN/$NS/" kustomize/overlays/prod/kustomization.yaml   # your host
kubectl kustomize kustomize/overlays/prod      # render (no apply)
kubectl apply -k kustomize/overlays/prod -n $NS

# Lab 4.2: Helm (chart from OCI, pointed at the Day-3 Postgres; keep the release running)
kubectl -n $NS delete deploy,svc,ingress,httproute,gateway,configmap,servicemonitor \
  -l app=todo --ignore-not-found                # clean slate; the db (app=todo-db) stays
cp values.yaml ~/values.yaml && sed -i "s/studentNN/$NS/" ~/values.yaml
kubectl -n $NS create secret generic todo-db-uri \
  --from-literal=uri='postgres://todo:todo@db:5432/todo?sslmode=disable'
helm install todo oci://ghcr.io/inqbeo/charts/todo --version 0.6.0 -n $NS -f ~/values.yaml
#   (the Day-3 NetworkPolicy only admits app=todo; the lab extends it by JSON patch)

# Lab 4.3: HA Postgres via cnpg, then one helm upgrade
kubectl apply -n $NS -f 01-pg-cluster.yaml
sed -i 's/existingSecret: todo-db-uri/existingSecret: db-app/' ~/values.yaml
helm upgrade todo oci://ghcr.io/inqbeo/charts/todo --version 0.6.0 -n $NS -f ~/values.yaml
kubectl apply -n $NS -f 02-seed-cnpg-job.yaml   # re-seed the new (empty) database

# Lab 4.4: GitOps: tear down by hand, let Flux rebuild everything from Git
helm uninstall todo -n $NS && kubectl -n $NS delete cluster.postgresql.cnpg.io db
cd ~/todo-gitops && cp ~/values.yaml . && cp ~/manifests/day4/gitops/* .
sed -i "s/studentNN/$NS/" kustomization.yaml && git add -A && git commit -m app && git push
flux create secret git todo-gitops-auth -n $NS --url=<repo-url> --username=$NS --password='<pw>'
flux create source git todo -n $NS --url=<repo-url> --branch=main \
  --secret-ref=todo-gitops-auth --interval=1m
flux create kustomization todo -n $NS --target-namespace=$NS \
  --source=GitRepository/todo --path=./ --prune=true --service-account=flux --interval=1m
```

> The cnpg operator, cert-manager, Traefik, Longhorn and Flux are installed
> cluster-wide by the platform; you only need **namespace-admin**.

# k8s-workshop-manifests

Kubernetes manifests for the Inqbeo Kubernetes basics workshop, sorted by day.
They build up the same demo app (`ghcr.io/inqbeo/containerization-demo-webapp`)
step by step across the course.

```
day2/   Kubernetes basics — Pod, Deployment (+ probes), ConfigMap/Secret, PVC, Service & DNS
day3/   Workload types (StatefulSet/DaemonSet/Jobs) + a real database, Ingress / Gateway API,
        certificates, high availability, network policies
day4/   (coming) Kustomize / Helm, operators / CRDs (CloudNativePG), GitOps, Rancher
```

All manifests are plain YAML and **namespace-agnostic** — apply them into your
own namespace with `kubectl apply -n <your-namespace> -f <file>`. See each day's
`README.md` for the order and a walkthrough.

> Remote: `git@github.com:inqbeo/k8s-workshop-manifests.git`

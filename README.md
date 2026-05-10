# GitOpsTest — ArgoCD Examples

A hands-on learning repo for ArgoCD with three levels of complexity, all deploying a simple nginx web server.

## Prerequisites

| Tool | Purpose |
|------|---------|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Container runtime + WSL2 backend |
| [k3d](https://k3d.io/) | Lightweight k3s cluster in Docker |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Kubernetes CLI |
| [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) | Optional, but handy for interacting with ArgoCD |

## Cluster Setup

This repo assumes a k3d cluster named `TestK8SCluster`. If you don't have one yet:

```bash
k3d cluster create TestK8SCluster
kubectl cluster-info
```

---

## Installing ArgoCD

```bash
# Create the ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD (--server-side avoids the 262KB annotation limit on the ApplicationSets CRD)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts

# Wait until the server is ready (takes ~60s)
kubectl wait --for=condition=available --timeout=180s deployment/argocd-server -n argocd
```

## Accessing the ArgoCD UI

```bash
# Forward port 8080 on your machine to the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open **https://localhost:8080** in your browser (accept the self-signed certificate warning).

**Username:** `admin`

**Get the initial password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```

**Login with the CLI (optional):**

```bash
argocd login localhost:8080 --insecure --username admin
```

> After first login it is good practice to change the password via the UI (User Info → Update Password) or:
> ```bash
> argocd account update-password
> ```

---

## Repository Structure

ArgoCD `Application` manifests live in `apps/` — separate from the content folders they point to. This prevents ArgoCD from trying to apply its own Application definition when syncing.

```
GitOpsTest/
├── apps/
│   ├── basic-app.yaml          ← ArgoCD Application for basic/
│   ├── intermediate-app.yaml   ← ArgoCD Application for intermediate/
│   └── expert-app.yaml         ← ArgoCD Application for expert/
├── basic/                      ← raw Kubernetes YAML
├── intermediate/               ← Helm chart
└── expert/                     ← Kustomize base + overlays
```

## Examples Overview

| Folder | Technique | Difficulty |
|--------|-----------|------------|
| [`basic/`](#basic--raw-yaml) | Raw Kubernetes YAML | Beginner |
| [`intermediate/`](#intermediate--helm) | In-repo Helm chart | Intermediate |
| [`expert/`](#expert--kustomize) | Kustomize with overlays | Advanced |

---

## Basic — Raw YAML

**What it does:** Deploys a plain nginx `Deployment` and `Service` as raw Kubernetes manifests. ArgoCD watches the `basic/` folder and keeps the cluster in sync with what is committed here.

```
basic/
├── deployment.yaml
└── service.yaml
```

**Deploy:**

```bash
kubectl apply -f apps/basic-app.yaml
```

**Verify:**

```bash
kubectl get application basic-nginx -n argocd
kubectl get pods -n default -l app=nginx
```

**How it works:** The `Application` manifest tells ArgoCD to watch `path: basic` in this repo. Whenever you push a change to those files, ArgoCD detects the drift and syncs the cluster automatically (`syncPolicy.automated`).

---

## Intermediate — Helm

**What it does:** Deploys nginx via a minimal Helm chart stored inside this repo (`intermediate/my-app/`). Helm templating lets you parameterise replica count, image tag, and more via `values.yaml`.

```
intermediate/
└── my-app/                ← Helm chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
```

**Deploy:**

```bash
kubectl apply -f apps/intermediate-app.yaml
```

**Verify:**

```bash
kubectl get application intermediate-nginx -n argocd
kubectl get pods -n default -l app=intermediate-nginx
```

**Override a value without editing files:**

```bash
argocd app set intermediate-nginx -p replicaCount=3
```

**How it works:** The `Application` manifest sets `source.path: intermediate/my-app` and `source.helm.valueFiles: [values.yaml]`. ArgoCD runs `helm template` against the chart and applies the rendered manifests.

---

## Expert — Kustomize

**What it does:** Uses Kustomize to maintain a single `base/` configuration and two environment overlays (`dev` and `prod`) that patch replica counts and set separate namespaces — no duplication.

```
expert/
└── kustomize/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    └── overlays/
        ├── dev/              ← 1 replica, namespace: dev
        │   ├── kustomization.yaml
        │   └── replica-patch.yaml
        └── prod/             ← 2 replicas, namespace: prod
            ├── kustomization.yaml
            └── replica-patch.yaml
```

| Overlay | Replicas | Namespace | Name prefix |
|---------|----------|-----------|-------------|
| dev | 1 | dev | `dev-` |
| prod | 2 | prod | `prod-` |

**Deploy (dev overlay):**

```bash
kubectl apply -f apps/expert-app.yaml
```

ArgoCD will also create the `dev` namespace automatically (`syncOptions: CreateNamespace=true`).

**Verify:**

```bash
kubectl get application expert-nginx-dev -n argocd
kubectl get pods -n dev
```

**Switch to the prod overlay:**

Edit `apps/expert-app.yaml` and change `path` from `expert/kustomize/overlays/dev` to `expert/kustomize/overlays/prod` (and update the app name if you like), then:

```bash
kubectl apply -f apps/expert-app.yaml
```

**How it works:** ArgoCD detects that the `path` points to a folder containing a `kustomization.yaml` and automatically runs `kustomize build` before applying. The overlays compose on top of the base using strategic-merge patches.

---

## Clean Up

```bash
# Remove all three ArgoCD Applications (ArgoCD will prune the resources too)
kubectl delete -f apps/basic-app.yaml
kubectl delete -f apps/intermediate-app.yaml
kubectl delete -f apps/expert-app.yaml

# Uninstall ArgoCD
kubectl delete namespace argocd

# Delete the cluster entirely
k3d cluster delete TestK8SCluster
```

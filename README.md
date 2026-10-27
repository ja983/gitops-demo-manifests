# GitOps with ArgoCD

Deploy an application to Kubernetes using **GitOps**: a Git repository is the single source of truth, and **ArgoCD** continuously keeps the cluster in sync with it.

Built as part of the 2-week DevOps internship at **Elevate Labs**.

## What this project demonstrates

| Capability | How it is shown |
|---|---|
| Declarative deployment | `deployment.yaml` and `service.yaml` live in Git, not applied by hand |
| Automated sync | A `git push` changes the cluster with no `kubectl apply` |
| Prune | Resources deleted from Git are deleted from the cluster |
| Self-heal | A manual `kubectl` change is reverted back to what Git says |

## Architecture

```
Developer ──git push──▶ GitHub (manifests repo)
                              │  ArgoCD polls (~3 min) or webhook
                              ▼
                        ArgoCD (argocd namespace)
                              │  compares Git (desired) vs cluster (live)
                              ▼
                  Kubernetes cluster (Minikube)
                  namespace: demo ─ Deployment ─ Pods, Service
```

## Prerequisites

- WSL2 Ubuntu with Docker running
- `kubectl`, `minikube`
- A GitHub account

## Setup

### 1. Start the cluster
```bash
minikube start --driver=docker
kubectl get nodes
```

### 2. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment --all -n argocd --timeout=300s
```

### 3. Open the UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open https://localhost:8080 (accept the self-signed certificate warning). User: `admin`.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### 4. Push the manifests to the manifests repo
Put `manifests/deployment.yaml` and `manifests/service.yaml` in the `gitops-demo-manifests` repo on branch `main`.

### 5. Create the ArgoCD Application
Edit `repoURL` in `argocd/application.yaml`, then:
```bash
kubectl apply -f argocd/application.yaml
```
The app should reach **Synced** and **Healthy**.

## Demo scenarios

### A. Change in Git → cluster follows
Edit `deployment.yaml` (e.g. `replicas: 2` → `4`, or `image: nginx:1.25` → `nginx:1.27`), commit and push. ArgoCD detects the change and syncs automatically.
```bash
kubectl get pods -n demo -w
```

### B. Self-heal
```bash
kubectl scale deployment demo-app -n demo --replicas=10
kubectl get pods -n demo -w     # ArgoCD scales it back to the value in Git
```

### C. Prune
Delete `service.yaml` from Git and push. The Service disappears from the cluster.

## Troubleshooting

| Problem | Fix |
|---|---|
| Application stuck `OutOfSync` or `Unknown` | Check `repoURL`, branch and `path`; for a private repo add credentials in ArgoCD Settings → Repositories |
| `kubectl port-forward` stops | Re-run it; it must stay running in its own terminal |
| Pods `ImagePullBackOff` | Check the image tag exists and Minikube has internet access |
| Change in Git not picked up yet | ArgoCD polls about every 3 minutes; click **Refresh** in the UI |

## Cleanup
```bash
kubectl delete -f argocd/application.yaml
minikube delete
```

## Tools used
Minikube, kubectl, ArgoCD, GitHub, Docker, WSL2 Ubuntu

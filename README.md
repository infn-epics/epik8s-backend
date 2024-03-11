# ecs-k8s-backend

## Require ARGOCD installed
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd_values.yaml

```

Base services for ecs k8s installations.

* Elastic Search *
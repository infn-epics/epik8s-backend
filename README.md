# EPIK8 backend services
The base backend services needed to run epics services.


- __Elastic Search__  : needed for logs,alarms,save and restore
- __Mongo__  : needed for logbook (olog)
- __Kafka__  : for alarms

__NOTE:__
*Require ArgoCD installed*

Example start application on a cloud infn account.

```
kubectl apply -f examples/epik8-backend-cloudinfn.yaml
```

## ARGOCD installation instructions
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd_values.yaml

## once installed to get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

## then connect to https://<domain specified into argocd_values>
```

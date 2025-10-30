# EPIK8S Backend Services

A Helm chart for deploying essential backend services for EPICS (Experimental Physics and Industrial Control System) infrastructure using ArgoCD.

## Overview

This chart deploys three core backend services required for EPICS operations:

- **Elasticsearch** - For logs, alarms, save and restore functionality
- **Kibana** - Dashboard for Elasticsearch visualization
- **MongoDB** - Database for the logbook (Olog) service  
- **Kafka** - Message broker for alarm handling

## Prerequisites

- Kubernetes cluster (version 1.20+)
- ArgoCD installed and configured
- Ingress controller (nginx, HAProxy, or OpenShift routes)
- Helm 3.x

## Installation

### 1. Install ArgoCD (if not already installed)

Add the ArgoCD Helm repository:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Install ArgoCD:

```bash
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd_values.yaml
```

**Note:** Update `argocd_values.yaml` with your domain and ingress class before installation.

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access ArgoCD UI at: `https://<your-argocd-domain>`

### 2. Deploy Backend Services

#### Option A: Using ArgoCD Application Manifest

For production deployments, use the ArgoCD Application manifest:

```bash
kubectl apply -f examples/epik8s-backend-rke2.yaml
```

#### Option B: Using Helm Template for Testing

To preview the manifests:

```bash
helm template backend . --values examples/epik8s-backend-rke2-values.yaml
```

To install directly (bypasses ArgoCD):

```bash
helm install backend . --values examples/epik8s-backend-rke2-values.yaml
```

## Configuration

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `namespace` | Target namespace for all services | `backend` |
| `domain` | Base domain for ingress hosts | `apps.okd-datest.lnf.infn.it` |
| `ingressClassName` | Ingress class (use `""` for OpenShift) | `nginx` |
| `openshift` | Enable OpenShift specific configs | `false` |
| `size` | Default PV size | `10Gi` |

### Service-Specific Configuration

#### Elasticsearch

```yaml
elasticsearch:
  replicaCount: 1
  targetRevision: '21.2.5'
  kibanaEnabled: true
  resources:
    limits:
      cpu: 5
      memory: 4Gi
  kibana:
    username: "epics"
    password: "epics"
```

#### MongoDB

```yaml
mongo:
  replicaCount: 1
  targetRevision: '14.12.3'
  auth:
    enabled: false
  resources:
    limits:
      cpu: 2
      memory: 2Gi
```

#### Kafka

```yaml
kafka:
  replicaCount: 3
  targetRevision: '32.4.3'
  externalAccess:
    enabled: true
    service:
      type: LoadBalancer
      loadBalancerIPs:
        - 10.10.6.250
        - 10.10.6.251
        - 10.10.6.252
```

## Examples

The `examples/` directory contains sample configurations for different environments:

- `epik8-backend-cloudinfn.yaml` - Cloud INFN deployment
- `epik8-backend-openshift.yaml` - OpenShift deployment
- `epik8-backend-k8sELI.yaml` - ELI Beamlines deployment
- `epik8-backend-k8ssparc.yaml` - SPARC deployment
- `epik8s-backend-rke2.yaml` - RKE2 cluster deployment
- `epik8s-backend-rke2-values.yaml` - Helm values for RKE2

### Customizing for Your Environment

1. Copy an example file closest to your setup
2. Update the `domain` value to match your cluster
3. Adjust resource limits based on your requirements
4. Configure external access for Kafka if needed
5. Set appropriate `ingressClassName` for your ingress controller

## Accessing Services

After deployment, services will be available at:

- Elasticsearch: `https://elasticsearch.<your-domain>`
- Kibana: `https://kibana.<your-domain>`
- MongoDB: `mongodb.<namespace>.svc.cluster.local:27017`
- Kafka: `kafka.<namespace>.svc.cluster.local:9092`

## Monitoring

Check application status:

```bash
kubectl get applications -n argocd
```

View application details:

```bash
argocd app get elasticsearch
argocd app get mongodb
argocd app get kafka
```

Check pod status:

```bash
kubectl get pods -n backend
```

## Troubleshooting

### Applications not syncing

```bash
argocd app sync elasticsearch --force
argocd app sync mongodb --force
argocd app sync kafka --force
```

### Check application logs

```bash
kubectl logs -n backend -l app.kubernetes.io/name=elasticsearch
kubectl logs -n backend -l app.kubernetes.io/name=mongodb
kubectl logs -n backend -l app.kubernetes.io/name=kafka
```

### Kafka external access issues

Ensure:
1. LoadBalancer IPs are available and not in use
2. Number of IPs matches `replicaCount`
3. Firewall rules allow traffic on port 9092

## Upgrading

To upgrade service versions, update the `targetRevision` values:

```yaml
elasticsearch:
  targetRevision: '21.3.0'  # New version
```

Then sync the application:

```bash
argocd app sync elasticsearch
```

## Contributing

For issues and contributions, please visit:
- GitHub: https://github.com/infn-epics/epik8s-backend
- Baltig: https://baltig.infn.it/epics-containers/epik8-backend

## License

This project is maintained by the INFN EPICS Team.

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Bitnami Helm Charts](https://github.com/bitnami/charts)
- [EPICS Homepage](https://epics-controls.org/) 

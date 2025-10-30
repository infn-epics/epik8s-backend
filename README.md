# EPIK8S Backend Services

A Helm chart for deploying essential backend services for EPICS (Experimental Physics and Industrial Control System) infrastructure using ArgoCD.

## Overview

This chart deploys three core backend services required for EPICS operations:

- **Elasticsearch** - For logs, alarms, save and restore functionality
- **Kibana** - Dashboard for Elasticsearch visualization
- **MongoDB** - Database for the logbook (Olog) service  
- **Kafka** - Message broker for alarm handling (via Strimzi Operator)

## Prerequisites

- Kubernetes cluster (version 1.20+)
- ArgoCD installed and configured
- Ingress controller (nginx, HAProxy, or OpenShift routes)
- Helm 3.x
- For Kafka: Strimzi Operator will be automatically installed

## 🎉 Kafka via Strimzi Operator

**Kafka is now deployed using the Strimzi Operator** instead of Bitnami Helm charts. This provides:
- No dependency on Bitnami registry (which was archived Aug 28, 2025)
- Kubernetes-native Kafka management via CRDs
- Better scalability and upgrade capabilities
- Official Apache Kafka images from `quay.io/strimzi`

Elasticsearch and MongoDB still use Bitnami Helm charts.

## ⚠️ Important: Bitnami Registry Changes (Elasticsearch & MongoDB)

**As of August 28, 2025**, Bitnami has archived their public Docker Hub registry (`docker.io/bitnami`). This affects Elasticsearch and MongoDB deployments.

### What This Means

- Old Bitnami images from Docker Hub are no longer publicly available
- Helm charts are now hosted as OCI artifacts: `oci://registry-1.docker.io/bitnamicharts`
- Container images may need to be mirrored or alternative registries used

### Solutions

1. **Use Alternative Image Registry** (Recommended for production):
   - Mirror images to your own private registry
   - Update values to point to your registry:
   ```yaml
   kafka:
     image:
       registry: your-registry.example.com
       repository: bitnami/kafka
       tag: 3.8.0-debian-12-r5
   ```

2. **Subscribe to Bitnami Secure Images**:
   - Access to supported, hardened images
   - Enterprise support and long-term versions
   - More info: https://bitnami.com/

3. **Use Bitnami Legacy Registry** (Temporary only):
   - Unsupported legacy images
   - Not recommended for production
   - Will accumulate unpatched vulnerabilities

For more details, see: [Bitnami Migration Guide](https://community.broadcom.com/blogs/beltran-rueda-borrego/2025/08/18/how-to-prepare-for-the-bitnami-changes-coming-soon)

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

#### Kafka (Strimzi Operator)

```yaml
kafka:
  clusterName: "epics-kafka"
  version: "3.8.0"
  replicaCount: 3
  strimzi:
    operatorVersion: "0.43.0"
  externalAccess:
    enabled: true
    type: loadbalancer
    port: 9094
    bootstrap:
      loadBalancerIP: 10.10.6.250
    brokers:
      - broker: 0
        loadBalancerIP: 10.10.6.250
      - broker: 1
        loadBalancerIP: 10.10.6.251
      - broker: 2
        loadBalancerIP: 10.10.6.252
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
- Kafka (internal): `epics-kafka-kafka-bootstrap.<namespace>.svc.cluster.local:9092`
- Kafka (external): `<bootstrap-loadbalancer-ip>:9094`

## Monitoring

Check application status:

```bash
kubectl get applications -n argocd
```

View application details:

```bash
argocd app get elasticsearch
argocd app get mongodb
argocd app get strimzi-operator
argocd app get kafka-cluster
```

Check Kafka cluster status:

```bash
# Kafka custom resource
kubectl get kafka -n backend

# Detailed status
kubectl describe kafka epics-kafka -n backend

# Kafka pods
kubectl get pods -n backend -l strimzi.io/cluster=epics-kafka
```

Check all pod status:

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

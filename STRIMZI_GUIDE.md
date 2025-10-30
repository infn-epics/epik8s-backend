# Strimzi Kafka Operator Guide

This document explains how Kafka is deployed using the Strimzi Operator in this chart.

## What is Strimzi?

[Strimzi](https://strimzi.io/) is an open-source project that provides a Kubernetes-native way to run Apache Kafka. It uses Custom Resource Definitions (CRDs) to manage Kafka clusters, topics, and users.

### Benefits over Bitnami Kafka

1. **No Registry Issues**: Uses official images from `quay.io/strimzi` (no Bitnami dependency)
2. **Kubernetes-Native**: Declarative Kafka management via CRDs
3. **Production-Ready**: Battle-tested operator used by many organizations
4. **Rich Features**:
   - Automatic TLS certificate management
   - User and topic management via CRDs
   - Multiple authentication methods (SCRAM, OAuth, mTLS)
   - Cruise Control integration for rebalancing
   - Kafka Connect and Mirror Maker support

## Architecture

This chart deploys Kafka in two steps:

### 1. Strimzi Operator

ArgoCD Application that installs the Strimzi operator via Helm chart:
- Watches the specified namespace for Kafka CRs
- Manages Kafka cluster lifecycle
- Handles upgrades and configuration changes

### 2. Kafka Cluster

ArgoCD Application that deploys Kafka Custom Resources:
- `Kafka` CR defines the cluster configuration
- Operator reconciles the CR and creates:
  - Kafka broker StatefulSets
  - Zookeeper StatefulSet
  - Services for bootstrap and per-broker access
  - ConfigMaps and Secrets

## Configuration

### Basic Configuration

```yaml
kafka:
  clusterName: "epics-kafka"     # Kafka cluster name
  version: "3.8.0"                # Kafka version
  replicaCount: 3                 # Number of brokers
```

### Storage

```yaml
kafka:
  storage:
    type: persistent-claim
    size: 10Gi
    deleteClaim: false
    # storageClass: "fast-ssd"    # Optional: specify storage class
```

### Resources

```yaml
kafka:
  resources:
    limits:
      cpu: 2
      memory: 4Gi
    requests:
      cpu: 1
      memory: 2Gi
```

### External Access

Strimzi supports multiple external access types:

#### LoadBalancer (Default in this chart)

```yaml
kafka:
  externalAccess:
    enabled: true
    type: loadbalancer
    port: 9094
    bootstrap:
      loadBalancerIP: 10.10.6.250    # Bootstrap service IP
    brokers:
      - broker: 0
        loadBalancerIP: 10.10.6.250
      - broker: 1
        loadBalancerIP: 10.10.6.251
      - broker: 2
        loadBalancerIP: 10.10.6.252
```

Each broker gets its own LoadBalancer service.

#### NodePort

```yaml
kafka:
  externalAccess:
    enabled: true
    type: nodeport
    port: 9094
    # No need to specify IPs
```

#### Ingress

```yaml
kafka:
  externalAccess:
    enabled: true
    type: ingress
    port: 9094
    # Requires ingress controller with TLS passthrough
```

### Zookeeper Configuration

```yaml
kafka:
  zookeeper:
    replicaCount: 3
    storage:
      type: persistent-claim
      size: 5Gi
    resources:
      limits:
        cpu: 1
        memory: 2Gi
```

### Entity Operators

Strimzi includes Topic and User operators for managing Kafka resources:

```yaml
kafka:
  entityOperator:
    enabled: true
    topicOperator:
      resources:
        limits:
          cpu: 500m
          memory: 512Mi
    userOperator:
      resources:
        limits:
          cpu: 500m
          memory: 512Mi
```

## Deployment

### Using ArgoCD (Recommended)

```bash
kubectl apply -f examples/epik8s-backend-rke2.yaml
```

The ArgoCD application will:
1. Install Strimzi operator
2. Wait for operator to be ready
3. Deploy Kafka cluster via CR

### Using Helm Template

```bash
helm template backend . --values examples/epik8s-backend-rke2-values-strimzi.yaml
```

## Monitoring

### Check Operator Status

```bash
# Operator pod
kubectl get pods -n backend -l name=strimzi-cluster-operator

# Operator logs
kubectl logs -n backend -l name=strimzi-cluster-operator -f
```

### Check Kafka Cluster Status

```bash
# Kafka CR status
kubectl get kafka epics-kafka -n backend

# Detailed view
kubectl describe kafka epics-kafka -n backend

# Check conditions
kubectl get kafka epics-kafka -n backend -o jsonpath='{.status.conditions}' | jq
```

### Check Kafka Pods

```bash
# All Kafka pods
kubectl get pods -n backend -l strimzi.io/cluster=epics-kafka

# Kafka brokers
kubectl get pods -n backend -l strimzi.io/name=epics-kafka-kafka

# Zookeeper
kubectl get pods -n backend -l strimzi.io/name=epics-kafka-zookeeper

# Entity operators
kubectl get pods -n backend -l strimzi.io/name=epics-kafka-entity-operator
```

### Check Services

```bash
# Bootstrap service (for clients)
kubectl get svc -n backend epics-kafka-kafka-bootstrap

# External LoadBalancer services
kubectl get svc -n backend -l strimzi.io/cluster=epics-kafka,strimzi.io/kind=Kafka
```

## Accessing Kafka

### From Within Kubernetes

Use the bootstrap service:

```bash
epics-kafka-kafka-bootstrap.backend.svc.cluster.local:9092
```

### From Outside Kubernetes

Use the external LoadBalancer IP:

```bash
10.10.6.250:9094
```

## Managing Topics

### Using KafkaTopic CR

Create a topic declaratively:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: alarms
  namespace: backend
  labels:
    strimzi.io/cluster: epics-kafka
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000  # 7 days
    segment.bytes: 1073741824
```

Apply it:

```bash
kubectl apply -f topic.yaml
```

### List Topics

```bash
kubectl get kafkatopics -n backend
```

## Troubleshooting

### Operator Not Starting

```bash
# Check operator logs
kubectl logs -n backend -l name=strimzi-cluster-operator

# Check RBAC
kubectl auth can-i create kafkas --as=system:serviceaccount:backend:strimzi-cluster-operator -n backend
```

### Kafka Pods Not Starting

```bash
# Check Kafka CR events
kubectl describe kafka epics-kafka -n backend

# Check pod events
kubectl describe pod epics-kafka-kafka-0 -n backend

# Check pod logs
kubectl logs epics-kafka-kafka-0 -n backend -c kafka
```

### Storage Issues

```bash
# Check PVCs
kubectl get pvc -n backend -l strimzi.io/cluster=epics-kafka

# Check PVC events
kubectl describe pvc data-epics-kafka-kafka-0 -n backend
```

### LoadBalancer Not Getting IP

```bash
# Check service
kubectl describe svc epics-kafka-kafka-external-bootstrap -n backend

# Check if LoadBalancer IPs are available
kubectl get nodes -o wide

# For MetalLB users
kubectl get ipaddresspools -n metallb-system
```

## Upgrading

### Kafka Version Upgrade

1. Update values:
   ```yaml
   kafka:
     version: "3.9.0"  # New version
   ```

2. Commit and push (ArgoCD will sync)

3. Strimzi will perform rolling upgrade automatically

### Operator Upgrade

1. Update operator version:
   ```yaml
   kafka:
     strimzi:
       operatorVersion: "0.44.0"
   ```

2. ArgoCD will upgrade the operator
3. Operator will check if Kafka cluster needs updates

## Additional Resources

- [Strimzi Documentation](https://strimzi.io/docs/)
- [Strimzi GitHub](https://github.com/strimzi/strimzi-kafka-operator)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Strimzi Examples](https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples)

## Migration from Bitnami

If migrating from Bitnami Kafka:

1. **Backup topics and data**:
   ```bash
   # Export topic configurations
   kafka-topics.sh --bootstrap-server <old-kafka>:9092 --list > topics.txt
   ```

2. **Deploy Strimzi Kafka** (this chart)

3. **Recreate topics** using KafkaTopic CRs or kafka-topics.sh

4. **Migrate data** using Mirror Maker 2 or Kafka Connect

5. **Update client configurations** to point to new bootstrap servers

6. **Decommission old Kafka** after validation

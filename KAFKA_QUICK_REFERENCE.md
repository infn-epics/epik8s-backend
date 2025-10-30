# Quick Reference: Strimzi Kafka

## Connection Strings

### Internal (from pods in K8s)
```
epics-kafka-kafka-bootstrap.backend.svc.cluster.local:9092
```

### External (LoadBalancer)
```
10.10.6.250:9094
```

## Common Commands

### Check Status
```bash
# Kafka cluster status
kubectl get kafka -n backend

# Detailed info
kubectl describe kafka epics-kafka -n backend

# Pods
kubectl get pods -n backend -l strimzi.io/cluster=epics-kafka

# Services
kubectl get svc -n backend -l strimzi.io/cluster=epics-kafka
```

### View Logs
```bash
# Operator
kubectl logs -n backend -l name=strimzi-cluster-operator -f

# Kafka broker 0
kubectl logs epics-kafka-kafka-0 -n backend -c kafka -f

# Zookeeper 0
kubectl logs epics-kafka-zookeeper-0 -n backend -f
```

### Manage Topics
```bash
# List topics
kubectl get kafkatopics -n backend

# Create topic
kubectl apply -f - <<EOF
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
    retention.ms: 604800000
EOF

# Delete topic
kubectl delete kafkatopic alarms -n backend
```

### Scale Kafka
```bash
# Edit values.yaml
kafka:
  replicaCount: 5  # Scale to 5 brokers

# Or patch directly (not recommended)
kubectl patch kafka epics-kafka -n backend --type merge -p '{"spec":{"kafka":{"replicas":5}}}'
```

### Check External Access
```bash
# LoadBalancer services
kubectl get svc -n backend | grep external

# Test from outside
telnet 10.10.6.250 9094
```

### Troubleshooting
```bash
# Check events
kubectl get events -n backend --sort-by='.lastTimestamp' | grep kafka

# Check PVCs
kubectl get pvc -n backend -l strimzi.io/cluster=epics-kafka

# Describe problem pod
kubectl describe pod epics-kafka-kafka-0 -n backend

# Check operator RBAC
kubectl auth can-i create kafkas --as=system:serviceaccount:backend:strimzi-cluster-operator -n backend
```

## ArgoCD Commands

```bash
# Sync all
argocd app sync strimzi-operator kafka-cluster

# Check status
argocd app get strimzi-operator
argocd app get kafka-cluster

# View diff
argocd app diff kafka-cluster
```

## Configuration Files

- **Main config**: `values.yaml`
- **Examples**: `examples/epik8s-backend-rke2-values-strimzi.yaml`
- **Kafka CR**: `kafka-cluster/kafka.yaml`
- **ArgoCD apps**: `templates/kafka.yaml`

## Documentation

- **Complete Guide**: `STRIMZI_GUIDE.md`
- **Migration Guide**: `KAFKA_STRIMZI_MIGRATION.md`
- **Main README**: `README.md`
- **Strimzi Docs**: https://strimzi.io/docs/

## Key Differences from Bitnami

| Feature | Bitnami | Strimzi |
|---------|---------|---------|
| **Images** | docker.io/bitnami (archived) | quay.io/strimzi (active) |
| **Management** | Helm values | Kubernetes CRDs |
| **Topics** | Manual creation | KafkaTopic CR |
| **Users** | Manual creation | KafkaUser CR |
| **Upgrades** | Helm upgrade | Operator handles it |
| **External Access** | Complex config | Simple LoadBalancer config |
| **Monitoring** | Limited | Prometheus metrics built-in |

## Default Ports

- **9092**: Plain internal listener
- **9094**: External listener (LoadBalancer)
- **2181**: Zookeeper client port
- **9404**: Prometheus metrics (when enabled)

## Resource Names

- **Kafka pods**: `epics-kafka-kafka-{0,1,2}`
- **Zookeeper pods**: `epics-kafka-zookeeper-{0,1,2}`
- **Entity operator**: `epics-kafka-entity-operator-*`
- **Bootstrap service**: `epics-kafka-kafka-bootstrap`
- **External services**: `epics-kafka-kafka-{0,1,2}-external`

# Kafka Migration to Strimzi Operator - Summary

## What Changed

The backend chart now deploys **Kafka using the Strimzi Operator** instead of Bitnami Helm charts.

## Why?

1. **Bitnami Registry Archived**: docker.io/bitnami no longer available (Aug 28, 2025)
2. **Kubernetes-Native**: Better integration with Kubernetes via CRDs
3. **Production-Ready**: Strimzi is widely used and battle-tested
4. **No Image Issues**: Uses official Strimzi images from quay.io

## New Structure

### Before (Bitnami)
```yaml
kafka:
  targetRevision: '32.4.3'
  image:
    registry: your-registry.example.com  # Had to mirror images!
  externalAccess:
    controller:
      service:
        loadBalancerIPs: [...]
```

### After (Strimzi)
```yaml
kafka:
  clusterName: "epics-kafka"
  version: "3.8.0"
  replicaCount: 3
  strimzi:
    operatorVersion: "0.43.0"
  externalAccess:
    enabled: true
    bootstrap:
      loadBalancerIP: 10.10.6.250
    brokers:
      - broker: 0
        loadBalancerIP: 10.10.6.250
```

## Files Changed

### New Files
- `templates/kafka.yaml` - Strimzi operator + kafka-cluster ArgoCD apps
- `kafka-cluster/kafka.yaml` - Templated Kafka Custom Resource
- `STRIMZI_GUIDE.md` - Complete Strimzi documentation
- `examples/epik8s-backend-rke2-values-strimzi.yaml` - Strimzi example

### Modified Files
- `values.yaml` - New Strimzi configuration structure
- `README.md` - Added Strimzi documentation
- `NOTES.txt` - Updated for Strimzi status checks
- `IMPROVEMENTS.md` - Documented migration

### Backed Up
- `templates/kafka.yaml.bitnami.bak` - Old Bitnami template (for reference)

## How to Use

### 1. Update Your Values

Replace your Kafka configuration with Strimzi structure:

```yaml
kafka:
  clusterName: "epics-kafka"
  version: "3.8.0"
  replicaCount: 3
  
  externalAccess:
    enabled: true
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

### 2. Deploy

```bash
# Using ArgoCD
kubectl apply -f examples/epik8s-backend-rke2.yaml

# Using Helm template
helm template backend . --values your-values.yaml
```

### 3. Monitor

```bash
# Check operator
kubectl get pods -n backend -l name=strimzi-cluster-operator

# Check Kafka cluster
kubectl get kafka -n backend
kubectl describe kafka epics-kafka -n backend

# Check pods
kubectl get pods -n backend -l strimzi.io/cluster=epics-kafka
```

## Accessing Kafka

### Internal (from within K8s)
```
epics-kafka-kafka-bootstrap.backend.svc.cluster.local:9092
```

### External (LoadBalancer)
```
10.10.6.250:9094
```

## Key Features

### Supported
✅ Persistent storage with PVCs  
✅ External access (LoadBalancer, NodePort, Ingress)  
✅ Resource limits and requests  
✅ Multiple replicas (3+ recommended)  
✅ Zookeeper included  
✅ Topic management via CRDs  
✅ User management via CRDs  
✅ TLS encryption (optional)  
✅ SCRAM/OAuth authentication (optional)  

### Benefits Over Bitnami
- No image registry issues
- Declarative configuration via CRDs
- Automatic rolling upgrades
- Better monitoring integration
- Cruise Control support for rebalancing
- Kafka Connect and Mirror Maker CRDs

## Troubleshooting

### Operator Not Starting
```bash
kubectl logs -n backend -l name=strimzi-cluster-operator
```

### Kafka Pods Not Starting
```bash
kubectl describe kafka epics-kafka -n backend
kubectl logs epics-kafka-kafka-0 -n backend -c kafka
```

### LoadBalancer Not Getting IP
```bash
kubectl describe svc -n backend -l strimzi.io/cluster=epics-kafka
```

## Documentation

- **Complete Guide**: See `STRIMZI_GUIDE.md`
- **Values Reference**: See comments in `values.yaml`
- **Examples**: See `examples/epik8s-backend-rke2-values-strimzi.yaml`
- **Official Docs**: https://strimzi.io/docs/

## Migration from Bitnami

If you have existing Bitnami Kafka:

1. **Backup** your topics and data
2. **Deploy** Strimzi Kafka (this chart)
3. **Recreate** topics (or use Mirror Maker 2)
4. **Update** client applications
5. **Decommission** old Kafka

See `STRIMZI_GUIDE.md` section "Migration from Bitnami" for details.

## Need Help?

- Read `STRIMZI_GUIDE.md` for comprehensive documentation
- Check Strimzi docs: https://strimzi.io/docs/
- GitHub issues: https://github.com/strimzi/strimzi-kafka-operator/issues

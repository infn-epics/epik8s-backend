# Kafka Cluster Manifests

This directory contains the Kafka Custom Resource (CR) manifest that will be deployed by ArgoCD.

## Structure

```
kafka-cluster/
  └── kafka.yaml       # Templated Kafka CR
```

## How It Works

1. The `templates/kafka.yaml` file creates an ArgoCD Application called `kafka-cluster`
2. This application points to this directory in your git repository
3. ArgoCD renders the template with values from your Helm values file
4. The Strimzi operator watches for Kafka CRs and reconciles them

## Template Variables

The `kafka.yaml` file uses Helm templating with variables from `values.yaml`:

- `{{.Values.kafka.clusterName}}` - Kafka cluster name
- `{{.Values.kafka.version}}` - Kafka version
- `{{.Values.kafka.replicaCount}}` - Number of brokers
- `{{.Values.kafka.storage}}` - Storage configuration
- `{{.Values.kafka.resources}}` - Resource limits
- `{{.Values.kafka.externalAccess}}` - External access config
- `{{.Values.kafka.zookeeper}}` - Zookeeper config
- `{{.Values.namespace}}` - Target namespace

## Deploying

This directory is deployed automatically when you deploy the backend chart. The ArgoCD sync wave ensures the Strimzi operator is installed before the Kafka CR is created.

## Manual Testing

To test the Kafka CR template locally:

```bash
# From chart root
helm template backend . --values examples/epik8s-backend-rke2-values-strimzi.yaml --show-only kafka-cluster/kafka.yaml
```

## Customization

You can add additional manifests to this directory:

### KafkaTopic CR

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: alarms
  namespace: {{ .Values.namespace }}
  labels:
    strimzi.io/cluster: {{ .Values.kafka.clusterName }}
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000
```

### KafkaUser CR

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: epics-user
  namespace: {{ .Values.namespace }}
  labels:
    strimzi.io/cluster: {{ .Values.kafka.clusterName }}
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: alarms
          patternType: literal
        operations:
          - Read
          - Write
```

## More Info

- Parent chart README: `../README.md`
- Strimzi guide: `../STRIMZI_GUIDE.md`
- Migration guide: `../KAFKA_STRIMZI_MIGRATION.md`

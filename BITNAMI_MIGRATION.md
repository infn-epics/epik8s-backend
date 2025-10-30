# Bitnami Registry Migration Guide

## Background

On **August 28, 2025**, Bitnami archived their public Docker Hub registry (`docker.io/bitnami`). This affects all users of Bitnami Helm charts and container images.

**Official Announcement**: https://community.broadcom.com/blogs/beltran-rueda-borrego/2025/08/18/how-to-prepare-for-the-bitnami-changes-coming-soon

## What Changed

- ❌ **No longer available**: `docker.io/bitnami/*` images
- ✅ **Still available**: Bitnami Helm chart source code (Apache 2.0 license on GitHub)
- ✅ **New location**: Helm charts as OCI artifacts `oci://registry-1.docker.io/bitnamicharts`
- ⚠️ **Images**: Must be sourced from alternative registries

## Impact on This Chart

This chart deploys:
- Elasticsearch
- MongoDB  
- Kafka

All three services use Bitnami container images which are no longer publicly available.

## Solutions

### Option 1: Mirror Images to Your Registry (Recommended)

1. **Set up a private registry** (Harbor, Artifactory, ECR, GCR, ACR, etc.)

2. **Mirror the required images** before they're completely removed:
   ```bash
   # Example for Kafka 3.8.0
   docker pull docker.io/bitnami/kafka:3.8.0-debian-12-r5  # May still work temporarily
   docker tag docker.io/bitnami/kafka:3.8.0-debian-12-r5 your-registry.example.com/bitnami/kafka:3.8.0-debian-12-r5
   docker push your-registry.example.com/bitnami/kafka:3.8.0-debian-12-r5
   ```

3. **Update values.yaml**:
   ```yaml
   kafka:
     image:
       registry: your-registry.example.com
       repository: bitnami/kafka
       tag: 3.8.0-debian-12-r5
   
   # Repeat for elasticsearch and mongodb if needed
   ```

### Option 2: Use Bitnami Secure Images (BSI)

**Requires subscription** but provides:
- Hardened Photon Linux-based images
- Long-term support
- Enterprise support
- Reduced CVE count
- Private OCI registry

More info: https://bitnami.com/

### Option 3: Bitnami Legacy Registry (Temporary)

⚠️ **Not recommended for production**

- Unsupported legacy images
- Will accumulate security vulnerabilities
- Being phased out
- Use only as a temporary bridge

## Chart Updates Made

### 1. Helm Repository URLs

Changed from HTTP to OCI format:
```yaml
# Before
repoURL: 'registry-1.docker.io/bitnamicharts'

# After
repoURL: 'oci://registry-1.docker.io/bitnamicharts'
```

### 2. Image Override Support

Added `image` override capability in `kafka.yaml`:
```yaml
{{- if .Values.kafka.image }}
image:
{{ toYaml .Values.kafka.image | indent 10 }}
{{- end }}
```

### 3. Updated Documentation

- README.md includes warning and solutions
- values.yaml includes image configuration
- Example files updated with registry placeholders

## Migration Steps for Users

### For Development/Testing

1. **Update your values file**:
   ```yaml
   kafka:
     targetRevision: '30.1.6'  # Use older chart with fewer issues
     image:
       registry: your-registry.example.com
       repository: bitnami/kafka
       tag: 3.8.0-debian-12-r5
   ```

2. **Deploy**:
   ```bash
   helm template backend . --values your-values.yaml
   # or
   kubectl apply -f your-argocd-application.yaml
   ```

### For Production

1. **Set up image mirroring pipeline**
   - Automate pulling from available sources
   - Push to your internal registry
   - Set up vulnerability scanning

2. **Update CI/CD pipelines**
   - Reference your internal registry
   - Add image promotion workflows
   - Implement security scanning gates

3. **Consider BSI subscription**
   - Evaluate cost vs. maintenance effort
   - Test hardened Photon images
   - Plan migration timeline

## Finding Alternative Images

### Check what chart versions need

```bash
# For a specific chart version
helm show values bitnami/kafka --version 30.1.6 | grep -A 5 "^image:"
```

### Search for compatible tags

Look for images in:
- Your organization's registry
- Bitnami Secure Images (BSI) catalog
- Alternative providers (e.g., Docker Official Images, though features may differ)

## Troubleshooting

### Error: `ErrImagePull` or `manifest unknown`

**Cause**: Trying to pull from archived `docker.io/bitnami` registry

**Solution**: 
1. Override image registry in values
2. Use a mirrored or alternative image
3. Check `kubectl describe pod` for exact image being pulled

### Error: Chart validation fails

**Cause**: Chart version mismatch with values structure

**Solution**:
- Use chart version 30.x for older values structure
- Or update values to match version 32.x requirements (controller.service)

### Error: LoadBalancer validation fails

**Cause**: Kafka chart requires `loadBalancerIPs` length == `replicaCount`

**Solution**:
```yaml
kafka:
  replicaCount: 3
  externalAccess:
    controller:
      service:
        loadBalancerIPs:
          - 10.10.6.250
          - 10.10.6.251
          - 10.10.6.252  # 3 IPs for 3 replicas
```

## Timeline

- **Aug 28, 2025**: New images stopped being published to docker.io/bitnami
- **Sep 29, 2025**: Planned deletion of public catalog (postponed from original date)
- **Brownouts**: Series of 24-hour outages for specific images to raise awareness
- **Future**: Complete removal of legacy images

## Additional Resources

- [Bitnami Official Announcement](https://community.broadcom.com/blogs/beltran-rueda-borrego/2025/08/18/how-to-prepare-for-the-bitnami-changes-coming-soon)
- [Bitnami Charts GitHub](https://github.com/bitnami/charts)
- [Bitnami Secure Images](https://bitnami.com/)
- [ArgoCD OCI Helm Support](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#oci-helm-charts)

## Support

For issues specific to this chart:
- GitHub: https://github.com/infn-epics/epik8s-backend/issues

For Bitnami-specific questions:
- Bitnami GitHub: https://github.com/bitnami/charts/issues
- Bitnami Community: https://community.broadcom.com/tanzu/communities/community-home?CommunityKey=56a49fa1-c592-460c-aa05-019446f8102f

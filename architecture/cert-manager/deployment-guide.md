# Cert-Manager Deployment Guide with Kustomize

This guide will walk you through deploying cert-manager to Kubernetes using Kustomize with all configuration stored as committable code.

## Prerequisites

- Kubernetes cluster (1.21+)
- kubectl configured
- kustomize (built into kubectl 1.14+)

## Repository Location

All cert-manager files should be created in the **`solidity-security-infrastructure`** repository.

## Directory Structure

Create the following directory structure in the `solidity-security-infrastructure` repository:

```
solidity-security-infrastructure/
└── k8s/
    └── cert-manager/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── crds/
│   │   │   ├── kustomization.yaml
│   │   │   ├── certificaterequests.yaml
│   │   │   ├── certificates.yaml
│   │   │   ├── challenges.yaml
│   │   │   ├── clusterissuers.yaml
│   │   │   ├── issuers.yaml
│   │   │   └── orders.yaml
│   │   ├── rbac/
│   │   │   ├── kustomization.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── clusterrole.yaml
│   │   │   ├── clusterrolebinding.yaml
│   │   │   ├── role.yaml
│   │   │   ├── rolebinding.yaml
│   │   │   └── leader-election-rbac.yaml
│   │   ├── webhook/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── clusterrole.yaml
│   │   │   ├── clusterrolebinding.yaml
│   │   │   ├── role.yaml
│   │   │   ├── rolebinding.yaml
│   │   │   ├── mutating-webhook.yaml
│   │   │   └── validating-webhook.yaml
│   │   ├── cainjector/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── clusterrole.yaml
│   │   │   ├── clusterrolebinding.yaml
│   │   │   ├── role.yaml
│   │   │   └── rolebinding.yaml
│   │   └── controller/
│   │       ├── kustomization.yaml
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── servicemonitor.yaml
│   └── overlays/
│               ├── staging/
│       │   ├── kustomization.yaml
│       │   ├── resource-limits.yaml
│       │   └── letsencrypt-staging-issuer.yaml
│       └── production/
│           ├── kustomization.yaml
│           ├── resource-limits.yaml
│           ├── letsencrypt-prod-issuer.yaml
│           └── monitoring.yaml
```

## File Locations

**Repository**: `solidity-security-infrastructure`

All files should be created under: `solidity-security-infrastructure/k8s/cert-manager/`

## Deployment Steps

### Step 1: Create Base Namespace Configuration

**Repository**: `solidity-security-infrastructure`
**File Location**: `k8s/cert-manager/base/namespace.yaml`

### Step 2: Set Up Custom Resource Definitions (CRDs)

**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/base/crds/kustomization.yaml`
- `k8s/cert-manager/base/crds/certificaterequests.yaml`
- `k8s/cert-manager/base/crds/certificates.yaml`
- `k8s/cert-manager/base/crds/challenges.yaml`
- `k8s/cert-manager/base/crds/clusterissuers.yaml`
- `k8s/cert-manager/base/crds/issuers.yaml`
- `k8s/cert-manager/base/crds/orders.yaml`

### Step 3: Configure RBAC

**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/base/rbac/kustomization.yaml`
- `k8s/cert-manager/base/rbac/serviceaccount.yaml`
- `k8s/cert-manager/base/rbac/clusterrole.yaml`
- `k8s/cert-manager/base/rbac/clusterrolebinding.yaml`
- `k8s/cert-manager/base/rbac/role.yaml`
- `k8s/cert-manager/base/rbac/rolebinding.yaml`
- `k8s/cert-manager/base/rbac/leader-election-rbac.yaml`

### Step 4: Deploy Cert-Manager Controller

**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/base/controller/kustomization.yaml`
- `k8s/cert-manager/base/controller/deployment.yaml`
- `k8s/cert-manager/base/controller/service.yaml`
- `k8s/cert-manager/base/controller/servicemonitor.yaml` (for Prometheus monitoring)

### Step 5: Deploy Cert-Manager Webhook

**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/base/webhook/kustomization.yaml`
- `k8s/cert-manager/base/webhook/deployment.yaml`
- `k8s/cert-manager/base/webhook/service.yaml`
- `k8s/cert-manager/base/webhook/serviceaccount.yaml`
- `k8s/cert-manager/base/webhook/clusterrole.yaml`
- `k8s/cert-manager/base/webhook/clusterrolebinding.yaml`
- `k8s/cert-manager/base/webhook/role.yaml`
- `k8s/cert-manager/base/webhook/rolebinding.yaml`
- `k8s/cert-manager/base/webhook/mutating-webhook.yaml`
- `k8s/cert-manager/base/webhook/validating-webhook.yaml`

### Step 6: Deploy CA Injector

**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/base/cainjector/kustomization.yaml`
- `k8s/cert-manager/base/cainjector/deployment.yaml`
- `k8s/cert-manager/base/cainjector/serviceaccount.yaml`
- `k8s/cert-manager/base/cainjector/clusterrole.yaml`
- `k8s/cert-manager/base/cainjector/clusterrolebinding.yaml`
- `k8s/cert-manager/base/cainjector/role.yaml`
- `k8s/cert-manager/base/cainjector/rolebinding.yaml`

### Step 7: Create Base Kustomization

**Repository**: `solidity-security-infrastructure`
**File Location**: `k8s/cert-manager/base/kustomization.yaml`

### Step 8: Set Up Environment Overlays

#### Staging Environment
**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/overlays/staging/kustomization.yaml`
- `k8s/cert-manager/overlays/staging/resource-limits.yaml`
- `k8s/cert-manager/overlays/staging/letsencrypt-staging-issuer.yaml`

#### Production Environment
**Repository**: `solidity-security-infrastructure`
**File Locations**:
- `k8s/cert-manager/overlays/production/kustomization.yaml`
- `k8s/cert-manager/overlays/production/resource-limits.yaml`
- `k8s/cert-manager/overlays/production/letsencrypt-prod-issuer.yaml`
- `k8s/cert-manager/overlays/production/monitoring.yaml`

## Manual Deployment Commands

### Deploy to Staging

1. **Navigate to the infrastructure repository**:
   ```bash
   cd solidity-security-infrastructure
   ```

2. **Deploy CRDs first**:
   ```bash
   kubectl apply -k k8s/cert-manager/base/crds/
   ```

3. **Wait for CRDs to be established** (manually check status):
   ```bash
   kubectl wait --for condition=established --timeout=60s crd/certificates.cert-manager.io
   ```

4. **Deploy cert-manager**:
   ```bash
   kubectl apply -k k8s/cert-manager/overlays/staging/
   ```

5. **Verify deployment**:
   ```bash
   kubectl get pods -n cert-manager
   kubectl get crd | grep cert-manager
   ```

### Deploy to Production

1. **Navigate to the infrastructure repository**:
   ```bash
   cd solidity-security-infrastructure
   ```

2. **Deploy CRDs first**:
   ```bash
   kubectl apply -k k8s/cert-manager/base/crds/
   ```

3. **Wait for CRDs to be established** (manually check status):
   ```bash
   kubectl wait --for condition=established --timeout=60s crd/certificates.cert-manager.io
   ```

4. **Deploy cert-manager**:
   ```bash
   kubectl apply -k k8s/cert-manager/overlays/production/
   ```

5. **Verify deployment**:
   ```bash
   kubectl get pods -n cert-manager
   kubectl get crd | grep cert-manager
   ```

## Manual Verification Steps

### 1. Check Pod Status
```bash
kubectl get pods -n cert-manager
```

Expected output: All pods should be in `Running` status:
- cert-manager-controller
- cert-manager-webhook  
- cert-manager-cainjector

### 2. Verify CRDs
```bash
kubectl get crd | grep cert-manager
```

### 3. Check Logs
```bash
kubectl logs -n cert-manager deployment/cert-manager-controller
kubectl logs -n cert-manager deployment/cert-manager-webhook
kubectl logs -n cert-manager deployment/cert-manager-cainjector
```

### 4. Test with a ClusterIssuer
```bash
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-staging  # or letsencrypt-prod
```

## Manual Maintenance Commands

### Update cert-manager
1. **Update the image tags in the deployment files manually**
2. **Apply the changes**:
   ```bash
   cd solidity-security-infrastructure
   kubectl apply -k k8s/cert-manager/overlays/production/
   ```

### Remove cert-manager
1. **Remove cert-manager components**:
   ```bash
   cd solidity-security-infrastructure
   kubectl delete -k k8s/cert-manager/overlays/production/
   ```

2. **Remove CRDs** (this will delete all certificates!):
   ```bash
   kubectl delete -k k8s/cert-manager/base/crds/
   ```

## Manual Troubleshooting

### Common Issues
1. **CRDs not established**: Wait for CRDs to be ready before deploying main components
2. **Webhook not ready**: Check webhook service and endpoint status
3. **RBAC issues**: Verify all ClusterRoles and bindings are applied correctly

### Manual Debug Commands
```bash
# Check webhook configuration
kubectl get validatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration

# Check RBAC
kubectl auth can-i --list --as=system:serviceaccount:cert-manager:cert-manager

# Check events
kubectl get events -n cert-manager --sort-by='.lastTimestamp'
```

## File Creation Workflow

### Order of File Creation

1. **First, create the directory structure**:
   ```bash
   cd solidity-security-infrastructure
   mkdir -p k8s/cert-manager/base/{crds,rbac,webhook,cainjector,controller}
   mkdir -p k8s/cert-manager/overlays/{staging,production}
   ```

2. **Create files in this order**:
   - Create CRD files first
   - Create RBAC configurations
   - Create service account and deployment files
   - Create kustomization files
   - Create overlay configurations

3. **Commit to git after each major component**:
   ```bash
   git add k8s/cert-manager/base/crds/
   git commit -m "Add cert-manager CRDs"
   
   git add k8s/cert-manager/base/rbac/
   git commit -m "Add cert-manager RBAC configuration"
   
   # Continue for each component...
   ```

## Next Steps

After successful deployment:
1. Create Certificate resources for your applications
2. Set up monitoring and alerting for certificate expiration
3. Configure backup strategies for certificate data
4. Implement certificate rotation policies

This deployment approach ensures all cert-manager configuration is version-controlled and reproducible across environments.

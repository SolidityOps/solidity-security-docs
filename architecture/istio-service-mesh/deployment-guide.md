# Istio Deployment Guide with Kustomize

## Repository Structure

All files should be created in the **`solidity-security-infrastructure`** repository unless otherwise specified.

## Prerequisites

- Kubernetes cluster running
- kubectl configured
- nginx-ingress-controller installed
- cert-manager installed and configured

## Directory Structure Setup

```
solidity-security-infrastructure/
├── k8s/
│   ├── istio/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── namespace.yaml
│   │   │   ├── istio-system/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   ├── istiod/
│   │   │   │   │   ├── kustomization.yaml
│   │   │   │   │   ├── deployment.yaml
│   │   │   │   │   ├── service.yaml
│   │   │   │   │   ├── configmap.yaml
│   │   │   │   │   ├── serviceaccount.yaml
│   │   │   │   │   ├── clusterrole.yaml
│   │   │   │   │   ├── clusterrolebinding.yaml
│   │   │   │   │   ├── role.yaml
│   │   │   │   │   ├── rolebinding.yaml
│   │   │   │   │   ├── mutatingwebhookconfiguration.yaml
│   │   │   │   │   ├── validatingwebhookconfiguration.yaml
│   │   │   │   │   └── horizontalpodautoscaler.yaml
│   │   │   │   └── gateway/
│   │   │   │       ├── kustomization.yaml
│   │   │   │       ├── deployment.yaml
│   │   │   │       ├── service.yaml
│   │   │   │       ├── serviceaccount.yaml
│   │   │   │       ├── role.yaml
│   │   │   │       ├── rolebinding.yaml
│   │   │   │       └── horizontalpodautoscaler.yaml
│   │   │   ├── crds/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   ├── authorization-policy.yaml
│   │   │   │   ├── destination-rule.yaml
│   │   │   │   ├── envoy-filter.yaml
│   │   │   │   ├── gateway.yaml
│   │   │   │   ├── peer-authentication.yaml
│   │   │   │   ├── proxy-config.yaml
│   │   │   │   ├── request-authentication.yaml
│   │   │   │   ├── service-entry.yaml
│   │   │   │   ├── sidecar.yaml
│   │   │   │   ├── telemetry.yaml
│   │   │   │   ├── virtual-service.yaml
│   │   │   │   ├── wasm-plugin.yaml
│   │   │   │   └── workload-entry.yaml
│   │   │   └── rbac/
│   │   │       ├── kustomization.yaml
│   │   │       ├── clusterrole-istiod.yaml
│   │   │       ├── clusterrole-pilot-discovery.yaml
│   │   │       ├── clusterrole-reader.yaml
│   │   │       ├── clusterrolebinding-istiod.yaml
│   │   │       └── clusterrolebinding-pilot-discovery.yaml
│   │   ├── overlays/
│   │   │   ├── staging/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   ├── istiod-configmap-patch.yaml
│   │   │   │   ├── gateway-service-patch.yaml
│   │   │   │   └── resource-quotas.yaml
│   │   │   └── production/
│   │   │       ├── kustomization.yaml
│   │   │       ├── istiod-configmap-patch.yaml
│   │   │       ├── gateway-service-patch.yaml
│   │   │       ├── resource-quotas.yaml
│   │   │       ├── network-policies.yaml
│   │   │       └── pod-security-policies.yaml
│   │   └── platform-integration/
│   │       ├── kustomization.yaml
│   │       ├── gateway.yaml
│   │       ├── virtual-service.yaml
│   │       ├── destination-rules.yaml
│   │       ├── peer-authentication.yaml
│   │       └── authorization-policies.yaml
│   └── applications/
│       └── solidity-platform/
│           ├── kustomization.yaml
│           ├── api-service/
│           │   ├── kustomization.yaml
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   ├── virtual-service.yaml
│           │   └── destination-rule.yaml
│           ├── frontend/
│           │   ├── kustomization.yaml
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   ├── virtual-service.yaml
│           │   └── destination-rule.yaml
│           └── intelligence-engine/
│               ├── kustomization.yaml
│               ├── deployment.yaml
│               ├── service.yaml
│               ├── virtual-service.yaml
│               └── destination-rule.yaml
├── scripts/
│   ├── install-istio.sh
│   ├── verify-istio.sh
│   ├── configure-gateway.sh
│   └── inject-sidecar.sh
└── docs/
    ├── istio-setup.md
    ├── troubleshooting.md
    └── service-mesh-guide.md
```

## Installation Steps

### Step 1: Download Istio Resources

Create the installation script:
- **File**: `scripts/install-istio.sh`
- **Purpose**: Download Istio manifests and prepare for kustomization

### Step 2: Setup Base Istio Configuration

#### 2.1 Create Namespace
- **File**: `k8s/istio/base/namespace.yaml`
- **Purpose**: Create istio-system namespace

#### 2.2 Setup CRDs
Create all Custom Resource Definitions:
- **Directory**: `k8s/istio/base/crds/`
- **Files**: All CRD YAML files listed above
- **File**: `k8s/istio/base/crds/kustomization.yaml`

#### 2.3 Setup RBAC
Configure cluster-level permissions:
- **Directory**: `k8s/istio/base/rbac/`
- **Files**: All RBAC YAML files listed above
- **File**: `k8s/istio/base/rbac/kustomization.yaml`

#### 2.4 Setup Istiod (Control Plane)
Configure the Istio control plane:
- **Directory**: `k8s/istio/base/istio-system/istiod/`
- **Files**: All istiod component files listed above
- **File**: `k8s/istio/base/istio-system/istiod/kustomization.yaml`

#### 2.5 Setup Istio Gateway (Data Plane)
Configure the Istio ingress gateway:
- **Directory**: `k8s/istio/base/istio-system/gateway/`
- **Files**: All gateway component files listed above
- **File**: `k8s/istio/base/istio-system/gateway/kustomization.yaml`

#### 2.6 Base Kustomization
- **File**: `k8s/istio/base/istio-system/kustomization.yaml`
- **Purpose**: Combine istiod and gateway components

- **File**: `k8s/istio/base/kustomization.yaml`
- **Purpose**: Main base kustomization file

### Step 3: Environment-Specific Overlays

#### 3.1 Staging Environment
- **File**: `k8s/istio/overlays/staging/kustomization.yaml`
- **File**: `k8s/istio/overlays/staging/istiod-configmap-patch.yaml`
- **File**: `k8s/istio/overlays/staging/gateway-service-patch.yaml`
- **File**: `k8s/istio/overlays/staging/resource-quotas.yaml`

#### 3.2 Production Environment
- **File**: `k8s/istio/overlays/production/kustomization.yaml`
- **File**: `k8s/istio/overlays/production/istiod-configmap-patch.yaml`
- **File**: `k8s/istio/overlays/production/gateway-service-patch.yaml`
- **File**: `k8s/istio/overlays/production/resource-quotas.yaml`
- **File**: `k8s/istio/overlays/production/network-policies.yaml`
- **File**: `k8s/istio/overlays/production/pod-security-policies.yaml`

### Step 4: Platform Integration

#### 4.1 Istio Gateway Configuration
- **File**: `k8s/istio/platform-integration/kustomization.yaml`
- **File**: `k8s/istio/platform-integration/gateway.yaml`
- **File**: `k8s/istio/platform-integration/virtual-service.yaml`
- **File**: `k8s/istio/platform-integration/destination-rules.yaml`
- **File**: `k8s/istio/platform-integration/peer-authentication.yaml`
- **File**: `k8s/istio/platform-integration/authorization-policies.yaml`

### Step 5: Application Service Mesh Configuration

#### 5.1 API Service
- **File**: `k8s/applications/solidity-platform/api-service/kustomization.yaml`
- **File**: `k8s/applications/solidity-platform/api-service/deployment.yaml`
- **File**: `k8s/applications/solidity-platform/api-service/service.yaml`
- **File**: `k8s/applications/solidity-platform/api-service/virtual-service.yaml`
- **File**: `k8s/applications/solidity-platform/api-service/destination-rule.yaml`

#### 5.2 Frontend Service
- **File**: `k8s/applications/solidity-platform/frontend/kustomization.yaml`
- **File**: `k8s/applications/solidity-platform/frontend/deployment.yaml`
- **File**: `k8s/applications/solidity-platform/frontend/service.yaml`
- **File**: `k8s/applications/solidity-platform/frontend/virtual-service.yaml`
- **File**: `k8s/applications/solidity-platform/frontend/destination-rule.yaml`

#### 5.3 Intelligence Engine Service
- **File**: `k8s/applications/solidity-platform/intelligence-engine/kustomization.yaml`
- **File**: `k8s/applications/solidity-platform/intelligence-engine/deployment.yaml`
- **File**: `k8s/applications/solidity-platform/intelligence-engine/service.yaml`
- **File**: `k8s/applications/solidity-platform/intelligence-engine/virtual-service.yaml`
- **File**: `k8s/applications/solidity-platform/intelligence-engine/destination-rule.yaml`

#### 5.4 Platform Application Kustomization
- **File**: `k8s/applications/solidity-platform/kustomization.yaml`

### Step 6: Utility Scripts

#### 6.1 Verification Script
- **File**: `scripts/verify-istio.sh`
- **Purpose**: Verify Istio installation and configuration

#### 6.2 Gateway Configuration Script
- **File**: `scripts/configure-gateway.sh`
- **Purpose**: Configure Istio gateway integration with nginx

#### 6.3 Sidecar Injection Script
- **File**: `scripts/inject-sidecar.sh`
- **Purpose**: Enable automatic sidecar injection for namespaces

### Step 7: Documentation

#### 7.1 Setup Documentation
- **File**: `docs/istio-setup.md`
- **Purpose**: Detailed setup and configuration guide

#### 7.2 Troubleshooting Guide
- **File**: `docs/troubleshooting.md`
- **Purpose**: Common issues and solutions

#### 7.3 Service Mesh Guide
- **File**: `docs/service-mesh-guide.md`
- **Purpose**: Service mesh concepts and usage patterns

## Deployment Commands

### Install Base Istio Components

```bash
# Apply CRDs first
kubectl apply -k k8s/istio/base/crds/

# Apply RBAC
kubectl apply -k k8s/istio/base/rbac/

# Apply base Istio components
kubectl apply -k k8s/istio/base/
```

### Deploy Environment-Specific Configuration

```bash
# For staging
kubectl apply -k k8s/istio/overlays/staging/

# For production
kubectl apply -k k8s/istio/overlays/production/
```

### Configure Platform Integration

```bash
# Apply platform-specific Istio configuration
kubectl apply -k k8s/istio/platform-integration/

# Deploy applications with service mesh
kubectl apply -k k8s/applications/solidity-platform/
```

### Enable Sidecar Injection

```bash
# Enable automatic sidecar injection for application namespaces
./scripts/inject-sidecar.sh
```

## Integration with Existing Infrastructure

### nginx-ingress Integration
- Istio Gateway will work alongside nginx-ingress
- External traffic: nginx-ingress → Istio Gateway → Services
- Internal traffic: Direct service mesh communication

### cert-manager Integration
- cert-manager continues to manage SSL certificates
- Certificates are used by both nginx-ingress and Istio Gateway
- No additional certificate management required for Istio

## Validation Steps

1. **Verify Istio Installation**: Run `scripts/verify-istio.sh`
2. **Check Service Mesh**: Verify sidecar injection is working
3. **Test Traffic Flow**: Ensure requests flow through the mesh
4. **Monitor Metrics**: Confirm telemetry data collection
5. **Security Policies**: Validate mTLS and authorization policies

## Post-Installation Tasks

1. Configure observability (Prometheus, Grafana, Jaeger)
2. Set up traffic management policies
3. Configure security policies (mTLS, AuthZ)
4. Implement circuit breakers and retry policies
5. Setup monitoring and alerting for service mesh

This deployment approach provides:
- ✅ **GitOps-friendly**: All configuration in version control
- ✅ **Environment-specific**: Staging vs Production overlays
- ✅ **Modular**: Separate concerns with kustomize
- ✅ **Integration-ready**: Works with existing nginx/cert-manager
- ✅ **Scalable**: Easy to add new services to the mesh

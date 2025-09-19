# Nginx Kubernetes Deployment Guide with Kustomize

## Overview
This guide provides step-by-step instructions for deploying nginx ingress controller to Kubernetes using Kustomize for the Unified Solidity Security Platform. All configurations are managed through committable YAML files without Helm charts.

## Target Repository
**Repository**: `solidity-security-infrastructure`
**Reason**: Infrastructure components including ingress controllers belong in the infrastructure repository

## Prerequisites
- Kubernetes cluster (1.24+)
- kubectl configured with cluster access
- Kustomize (built into kubectl 1.14+)
- cert-manager installed (for SSL certificate management)

## Directory Structure in `solidity-security-infrastructure`
```
solidity-security-infrastructure/
├── k8s/
│   ├── base/
│   │   ├── nginx/
│   │   │   ├── kustomization.yaml
│   │   │   ├── namespace.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   ├── service-account.yaml
│   │   │   ├── cluster-role.yaml
│   │   │   ├── cluster-role-binding.yaml
│   │   │   ├── role.yaml
│   │   │   ├── role-binding.yaml
│   │   │   └── ingress-class.yaml
│   │   └── cert-manager/
│   ├── overlays/
│   │   ├── development/
│   │   │   ├── nginx/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   ├── nginx-config-patch.yaml
│   │   │   │   ├── resource-limits-patch.yaml
│   │   │   │   └── replica-patch.yaml
│   │   ├── staging/
│   │   │   ├── nginx/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   ├── nginx-config-patch.yaml
│   │   │   │   ├── resource-limits-patch.yaml
│   │   │   │   ├── replica-patch.yaml
│   │   │   │   └── ssl-redirect-patch.yaml
│   │   └── production/
│   │       ├── nginx/
│   │       │   ├── kustomization.yaml
│   │       │   ├── nginx-config-patch.yaml
│   │       │   ├── resource-limits-patch.yaml
│   │       │   ├── replica-patch.yaml
│   │       │   ├── ssl-redirect-patch.yaml
│   │       │   ├── rate-limit-patch.yaml
│   │       │   └── security-headers-patch.yaml
│   └── ingress/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   ├── api-ingress.yaml
│       │   ├── frontend-ingress.yaml
│       │   └── monitoring-ingress.yaml
│       ├── development/
│       │   ├── kustomization.yaml
│       │   ├── api-ingress-patch.yaml
│       │   └── frontend-ingress-patch.yaml
│       ├── staging/
│       │   ├── kustomization.yaml
│       │   ├── api-ingress-patch.yaml
│       │   ├── frontend-ingress-patch.yaml
│       │   └── ssl-certificate-patch.yaml
│       └── production/
│           ├── kustomization.yaml
│           ├── api-ingress-patch.yaml
│           ├── frontend-ingress-patch.yaml
│           ├── ssl-certificate-patch.yaml
│           ├── rate-limit-ingress-patch.yaml
│           └── security-ingress-patch.yaml
```

## Step-by-Step Manual Deployment

### Step 1: Create Base Directory Structure
**Repository**: `solidity-security-infrastructure`

Manually create the following directories:

1. Navigate to `solidity-security-infrastructure` repository
2. Create directory: `k8s/base/nginx`
3. Create directory: `k8s/overlays/development/nginx`
4. Create directory: `k8s/overlays/staging/nginx`
5. Create directory: `k8s/overlays/production/nginx`
6. Create directory: `k8s/ingress/base`
7. Create directory: `k8s/ingress/development`
8. Create directory: `k8s/ingress/staging`
9. Create directory: `k8s/ingress/production`

### Step 2: Create Base Nginx Resources
**Repository**: `solidity-security-infrastructure`
**Location**: `k8s/base/nginx/`

#### 2.1 Namespace Configuration
Manually create file: `k8s/base/nginx/namespace.yaml`

#### 2.2 Service Account and RBAC
Manually create files:
- `k8s/base/nginx/service-account.yaml`
- `k8s/base/nginx/cluster-role.yaml`
- `k8s/base/nginx/cluster-role-binding.yaml`
- `k8s/base/nginx/role.yaml`
- `k8s/base/nginx/role-binding.yaml`

#### 2.3 Nginx Controller Deployment
Manually create file: `k8s/base/nginx/deployment.yaml`

#### 2.4 Service Configuration
Manually create file: `k8s/base/nginx/service.yaml`

#### 2.5 ConfigMap for Nginx Configuration
Manually create file: `k8s/base/nginx/configmap.yaml`

#### 2.6 IngressClass Definition
Manually create file: `k8s/base/nginx/ingress-class.yaml`

#### 2.7 Base Kustomization
Manually create file: `k8s/base/nginx/kustomization.yaml`

### Step 3: Create Environment-Specific Overlays
**Repository**: `solidity-security-infrastructure`

#### 3.1 Development Environment
**Location**: `k8s/overlays/development/nginx/`

Manually create files:
- `k8s/overlays/development/nginx/kustomization.yaml`
- `k8s/overlays/development/nginx/nginx-config-patch.yaml`
- `k8s/overlays/development/nginx/resource-limits-patch.yaml`
- `k8s/overlays/development/nginx/replica-patch.yaml`

#### 3.2 Staging Environment
**Location**: `k8s/overlays/staging/nginx/`

Manually create files:
- `k8s/overlays/staging/nginx/kustomization.yaml`
- `k8s/overlays/staging/nginx/nginx-config-patch.yaml`
- `k8s/overlays/staging/nginx/resource-limits-patch.yaml`
- `k8s/overlays/staging/nginx/replica-patch.yaml`
- `k8s/overlays/staging/nginx/ssl-redirect-patch.yaml`

#### 3.3 Production Environment
**Location**: `k8s/overlays/production/nginx/`

Manually create files:
- `k8s/overlays/production/nginx/kustomization.yaml`
- `k8s/overlays/production/nginx/nginx-config-patch.yaml`
- `k8s/overlays/production/nginx/resource-limits-patch.yaml`
- `k8s/overlays/production/nginx/replica-patch.yaml`
- `k8s/overlays/production/nginx/ssl-redirect-patch.yaml`
- `k8s/overlays/production/nginx/rate-limit-patch.yaml`
- `k8s/overlays/production/nginx/security-headers-patch.yaml`

### Step 4: Create Ingress Resources
**Repository**: `solidity-security-infrastructure`

#### 4.1 Base Ingress Resources
**Location**: `k8s/ingress/base/`

Manually create files:
- `k8s/ingress/base/kustomization.yaml`
- `k8s/ingress/base/api-ingress.yaml`
- `k8s/ingress/base/frontend-ingress.yaml`
- `k8s/ingress/base/monitoring-ingress.yaml`

#### 4.2 Development Ingress Overlays
**Location**: `k8s/ingress/development/`

Manually create files:
- `k8s/ingress/development/kustomization.yaml`
- `k8s/ingress/development/api-ingress-patch.yaml`
- `k8s/ingress/development/frontend-ingress-patch.yaml`

#### 4.3 Staging Ingress Overlays
**Location**: `k8s/ingress/staging/`

Manually create files:
- `k8s/ingress/staging/kustomization.yaml`
- `k8s/ingress/staging/api-ingress-patch.yaml`
- `k8s/ingress/staging/frontend-ingress-patch.yaml`
- `k8s/ingress/staging/ssl-certificate-patch.yaml`

#### 4.4 Production Ingress Overlays
**Location**: `k8s/ingress/production/`

Manually create files:
- `k8s/ingress/production/kustomization.yaml`
- `k8s/ingress/production/api-ingress-patch.yaml`
- `k8s/ingress/production/frontend-ingress-patch.yaml`
- `k8s/ingress/production/ssl-certificate-patch.yaml`
- `k8s/ingress/production/rate-limit-ingress-patch.yaml`
- `k8s/ingress/production/security-ingress-patch.yaml`

### Step 5: Deploy Nginx Controller
**Repository**: `solidity-security-infrastructure`

#### 5.1 Development Deployment
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/overlays/development/nginx`
3. Verify deployment: `kubectl get pods -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx`
4. Check service status: `kubectl get svc -n nginx-ingress`

#### 5.2 Staging Deployment
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/overlays/staging/nginx`
3. Verify SSL redirect configuration: `kubectl describe configmap nginx-configuration -n nginx-ingress`

#### 5.3 Production Deployment
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/overlays/production/nginx`
3. Verify security headers and rate limiting: `kubectl describe configmap nginx-configuration -n nginx-ingress`

### Step 6: Deploy Ingress Resources
**Repository**: `solidity-security-infrastructure`

#### 6.1 Development Ingress
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/ingress/development`
3. Verify ingress creation: `kubectl get ingress -A`

#### 6.2 Staging Ingress
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/ingress/staging`
3. Check SSL certificate status: `kubectl get certificates -A`

#### 6.3 Production Ingress
1. Navigate to `solidity-security-infrastructure` repository root
2. Run: `kubectl apply -k k8s/ingress/production`
3. Verify all security configurations: `kubectl describe ingress -n production`

### Step 7: Verification and Testing

#### 7.1 Health Checks
1. Check nginx controller health: `kubectl get pods -n nginx-ingress -o wide`
2. Verify nginx controller logs: `kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx`
3. Test ingress controller readiness: `kubectl get deployment -n nginx-ingress`

#### 7.2 SSL Certificate Verification
1. Check certificate status: `kubectl get certificates -A`
2. Verify certificate details: `kubectl describe certificate -n <namespace> <certificate-name>`
3. Test SSL endpoint manually: `curl -I https://your-domain.com`

#### 7.3 Configuration Validation
1. Validate nginx configuration: `kubectl exec -n nginx-ingress <nginx-pod> -- nginx -t`
2. Check loaded configuration: `kubectl exec -n nginx-ingress <nginx-pod> -- cat /etc/nginx/nginx.conf`

### Step 8: Monitoring and Maintenance

#### 8.1 Monitoring Setup
1. Check nginx metrics endpoint: `kubectl port-forward -n nginx-ingress <nginx-pod> 10254:10254`
2. Access metrics: `curl http://localhost:10254/metrics`

#### 8.2 Log Monitoring
1. Monitor nginx access logs: `kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx -f`
2. Check error logs: `kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx --previous`

#### 8.3 Configuration Updates
1. Modify files in `solidity-security-infrastructure` repository
2. Apply configuration changes: `kubectl apply -k k8s/overlays/<environment>/nginx`
3. Restart nginx controller if needed: `kubectl rollout restart deployment/ingress-nginx-controller -n nginx-ingress`

### Step 9: Troubleshooting

#### 9.1 Common Issues
- Pod startup failures
- SSL certificate issues
- Ingress routing problems
- Configuration validation errors

#### 9.2 Debug Commands
1. Debug nginx controller: `kubectl describe pod -n nginx-ingress <nginx-pod>`
2. Check events: `kubectl get events -n nginx-ingress --sort-by='.lastTimestamp'`
3. Validate ingress resources: `kubectl describe ingress <ingress-name> -n <namespace>`

### Step 10: Cleanup (if needed)

#### 10.1 Remove Ingress Resources
1. Navigate to `solidity-security-infrastructure` repository root
2. Remove ingress resources: `kubectl delete -k k8s/ingress/<environment>`

#### 10.2 Remove Nginx Controller
1. Navigate to `solidity-security-infrastructure` repository root
2. Remove nginx controller: `kubectl delete -k k8s/overlays/<environment>/nginx`

## Configuration Notes

### Security Considerations
- All production deployments include security headers
- Rate limiting configured for production environment
- SSL redirect enforced in staging and production
- RBAC properly configured with minimal permissions

### Performance Tuning
- Resource limits adjusted per environment
- Replica counts scaled based on load requirements
- Nginx configuration optimized for each environment

### Maintenance
- Regular updates to nginx controller image
- SSL certificate rotation handled by cert-manager
- Configuration changes applied through Kustomize overlays
- All changes tracked in version control

## Integration with Platform Services
**Repository Integration**: This nginx deployment in `solidity-security-infrastructure` will route traffic to services deployed from other repositories:

### Service Routing Configuration
- **API services** (from `solidity-security-platform` repo) - routed through `/api/*` paths
- **Frontend** (from `solidity-security-platform` repo) - routed through `/` path  
- **Monitoring services** (from `solidity-security-monitoring` repo) - routed through `/monitoring/*` paths
- **Tool services** (from `solidity-security-tools` repo) - routed through `/tools/*` paths

### Repository Dependencies
The nginx configuration in `solidity-security-infrastructure` depends on:
- Service definitions from `solidity-security-platform`
- Monitoring endpoints from `solidity-security-monitoring` 
- Tool service endpoints from `solidity-security-tools`

The separation follows infrastructure best practices where networking and ingress components are managed separately from application code while maintaining clear routing to services across repositories.

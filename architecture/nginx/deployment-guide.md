# Nginx Kubernetes Deployment Guide with Kustomize

## Overview
This guide provides step-by-step instructions for deploying nginx ingress controller to Kubernetes using Kustomize for the Unified Solidity Security Platform. All configurations are managed through committable YAML files without Helm charts.

## Prerequisites
- Kubernetes cluster (1.24+)
- kubectl configured with cluster access
- Kustomize (built into kubectl 1.14+)
- cert-manager installed (for SSL certificate management)

## Directory Structure
```
k8s/
├── base/
│   ├── nginx/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── service-account.yaml
│   │   ├── cluster-role.yaml
│   │   ├── cluster-role-binding.yaml
│   │   ├── role.yaml
│   │   ├── role-binding.yaml
│   │   └── ingress-class.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── nginx-config-patch.yaml
│   │   ├── resource-limits-patch.yaml
│   │   └── replica-patch.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── nginx-config-patch.yaml
│   │   ├── resource-limits-patch.yaml
│   │   ├── replica-patch.yaml
│   │   └── ssl-redirect-patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── nginx-config-patch.yaml
│       ├── resource-limits-patch.yaml
│       ├── replica-patch.yaml
│       ├── ssl-redirect-patch.yaml
│       ├── rate-limit-patch.yaml
│       └── security-headers-patch.yaml
└── ingress/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── api-ingress.yaml
    │   ├── frontend-ingress.yaml
    │   └── monitoring-ingress.yaml
    ├── development/
    │   ├── kustomization.yaml
    │   ├── api-ingress-patch.yaml
    │   └── frontend-ingress-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── api-ingress-patch.yaml
    │   ├── frontend-ingress-patch.yaml
    │   └── ssl-certificate-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── api-ingress-patch.yaml
        ├── frontend-ingress-patch.yaml
        ├── ssl-certificate-patch.yaml
        ├── rate-limit-ingress-patch.yaml
        └── security-ingress-patch.yaml
```

## Step-by-Step Deployment

### Step 1: Create Base Directory Structure
Create the following directory structure in your repository:

```bash
mkdir -p k8s/base/nginx
mkdir -p k8s/overlays/{development,staging,production}
mkdir -p k8s/ingress/{base,development,staging,production}
```

### Step 2: Create Base Nginx Resources

#### 2.1 Namespace Configuration
Create file: `k8s/base/nginx/namespace.yaml`

#### 2.2 Service Account and RBAC
Create files:
- `k8s/base/nginx/service-account.yaml`
- `k8s/base/nginx/cluster-role.yaml`
- `k8s/base/nginx/cluster-role-binding.yaml`
- `k8s/base/nginx/role.yaml`
- `k8s/base/nginx/role-binding.yaml`

#### 2.3 Nginx Controller Deployment
Create file: `k8s/base/nginx/deployment.yaml`

#### 2.4 Service Configuration
Create file: `k8s/base/nginx/service.yaml`

#### 2.5 ConfigMap for Nginx Configuration
Create file: `k8s/base/nginx/configmap.yaml`

#### 2.6 IngressClass Definition
Create file: `k8s/base/nginx/ingress-class.yaml`

#### 2.7 Base Kustomization
Create file: `k8s/base/nginx/kustomization.yaml`

### Step 3: Create Environment-Specific Overlays

#### 3.1 Development Environment
Create files:
- `k8s/overlays/development/kustomization.yaml`
- `k8s/overlays/development/nginx-config-patch.yaml`
- `k8s/overlays/development/resource-limits-patch.yaml`
- `k8s/overlays/development/replica-patch.yaml`

#### 3.2 Staging Environment
Create files:
- `k8s/overlays/staging/kustomization.yaml`
- `k8s/overlays/staging/nginx-config-patch.yaml`
- `k8s/overlays/staging/resource-limits-patch.yaml`
- `k8s/overlays/staging/replica-patch.yaml`
- `k8s/overlays/staging/ssl-redirect-patch.yaml`

#### 3.3 Production Environment
Create files:
- `k8s/overlays/production/kustomization.yaml`
- `k8s/overlays/production/nginx-config-patch.yaml`
- `k8s/overlays/production/resource-limits-patch.yaml`
- `k8s/overlays/production/replica-patch.yaml`
- `k8s/overlays/production/ssl-redirect-patch.yaml`
- `k8s/overlays/production/rate-limit-patch.yaml`
- `k8s/overlays/production/security-headers-patch.yaml`

### Step 4: Create Ingress Resources

#### 4.1 Base Ingress Resources
Create files:
- `k8s/ingress/base/kustomization.yaml`
- `k8s/ingress/base/api-ingress.yaml`
- `k8s/ingress/base/frontend-ingress.yaml`
- `k8s/ingress/base/monitoring-ingress.yaml`

#### 4.2 Development Ingress Overlays
Create files:
- `k8s/ingress/development/kustomization.yaml`
- `k8s/ingress/development/api-ingress-patch.yaml`
- `k8s/ingress/development/frontend-ingress-patch.yaml`

#### 4.3 Staging Ingress Overlays
Create files:
- `k8s/ingress/staging/kustomization.yaml`
- `k8s/ingress/staging/api-ingress-patch.yaml`
- `k8s/ingress/staging/frontend-ingress-patch.yaml`
- `k8s/ingress/staging/ssl-certificate-patch.yaml`

#### 4.4 Production Ingress Overlays
Create files:
- `k8s/ingress/production/kustomization.yaml`
- `k8s/ingress/production/api-ingress-patch.yaml`
- `k8s/ingress/production/frontend-ingress-patch.yaml`
- `k8s/ingress/production/ssl-certificate-patch.yaml`
- `k8s/ingress/production/rate-limit-ingress-patch.yaml`
- `k8s/ingress/production/security-ingress-patch.yaml`

### Step 5: Deploy Nginx Controller

#### 5.1 Development Deployment
```bash
# Deploy nginx controller for development
kubectl apply -k k8s/overlays/development

# Verify deployment
kubectl get pods -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx

# Check service status
kubectl get svc -n nginx-ingress
```

#### 5.2 Staging Deployment
```bash
# Deploy nginx controller for staging
kubectl apply -k k8s/overlays/staging

# Verify SSL redirect configuration
kubectl describe configmap nginx-configuration -n nginx-ingress
```

#### 5.3 Production Deployment
```bash
# Deploy nginx controller for production
kubectl apply -k k8s/overlays/production

# Verify security headers and rate limiting
kubectl describe configmap nginx-configuration -n nginx-ingress
```

### Step 6: Deploy Ingress Resources

#### 6.1 Development Ingress
```bash
# Deploy development ingress resources
kubectl apply -k k8s/ingress/development

# Verify ingress creation
kubectl get ingress -A
```

#### 6.2 Staging Ingress
```bash
# Deploy staging ingress resources
kubectl apply -k k8s/ingress/staging

# Check SSL certificate status
kubectl get certificates -A
```

#### 6.3 Production Ingress
```bash
# Deploy production ingress resources
kubectl apply -k k8s/ingress/production

# Verify all security configurations
kubectl describe ingress -n production
```

### Step 7: Verification and Testing

#### 7.1 Health Checks
```bash
# Check nginx controller health
kubectl get pods -n nginx-ingress -o wide

# Verify nginx controller logs
kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx

# Test ingress controller readiness
kubectl get deployment -n nginx-ingress
```

#### 7.2 SSL Certificate Verification
```bash
# Check certificate status
kubectl get certificates -A

# Verify certificate details
kubectl describe certificate -n <namespace> <certificate-name>

# Test SSL endpoint
curl -I https://your-domain.com
```

#### 7.3 Configuration Validation
```bash
# Validate nginx configuration
kubectl exec -n nginx-ingress <nginx-pod> -- nginx -t

# Check loaded configuration
kubectl exec -n nginx-ingress <nginx-pod> -- cat /etc/nginx/nginx.conf
```

### Step 8: Monitoring and Maintenance

#### 8.1 Monitoring Setup
```bash
# Check nginx metrics endpoint
kubectl port-forward -n nginx-ingress <nginx-pod> 10254:10254
curl http://localhost:10254/metrics
```

#### 8.2 Log Monitoring
```bash
# Monitor nginx access logs
kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx -f

# Check error logs
kubectl logs -n nginx-ingress -l app.kubernetes.io/name=ingress-nginx --previous
```

#### 8.3 Configuration Updates
```bash
# Apply configuration changes
kubectl apply -k k8s/overlays/<environment>

# Restart nginx controller if needed
kubectl rollout restart deployment/ingress-nginx-controller -n nginx-ingress
```

### Step 9: Troubleshooting

#### 9.1 Common Issues
- Pod startup failures
- SSL certificate issues
- Ingress routing problems
- Configuration validation errors

#### 9.2 Debug Commands
```bash
# Debug nginx controller
kubectl describe pod -n nginx-ingress <nginx-pod>

# Check events
kubectl get events -n nginx-ingress --sort-by='.lastTimestamp'

# Validate ingress resources
kubectl describe ingress <ingress-name> -n <namespace>
```

### Step 10: Cleanup (if needed)

#### 10.1 Remove Ingress Resources
```bash
# Remove ingress resources
kubectl delete -k k8s/ingress/<environment>
```

#### 10.2 Remove Nginx Controller
```bash
# Remove nginx controller
kubectl delete -k k8s/overlays/<environment>
```

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

This nginx deployment will serve as the ingress controller for:
- API Gateway (Kong) - routed through `/api/*` paths
- Frontend React Application - routed through `/` path
- Monitoring Services - routed through `/monitoring/*` paths
- Tool Integration Services - routed through `/tools/*` paths

The configuration supports the platform's requirements for SSL termination, rate limiting, and security headers as specified in the technical development plan.

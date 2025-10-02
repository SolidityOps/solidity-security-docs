# Infrastructure Fixes and Workarounds

> **⚠️ LOCAL DEVELOPMENT ONLY - These fixes are NOT for production use**

## Overview

This document details all infrastructure fixes and workarounds applied to resolve deployment issues in the local development environment. These modifications ensure functionality while maintaining simplicity for local development.

## Database Services Fixes

### PostgreSQL Image Issues ❌➡️✅

#### **Problem**
Original Helm deployment failed with image pull errors:
```bash
Failed to pull image "docker.io/bitnami/postgresql:17.6.0-debian-12-r4":
Error response from daemon: manifest for bitnami/postgresql:17.6.0-debian-12-r4 not found
```

#### **Root Cause**
- Bitnami images with specific version tags were not available
- Helm chart specified non-existent image versions
- Platform compatibility issues with specified tags

#### **Solution Applied**
Replaced Bitnami deployment with official PostgreSQL image:

```yaml
# File: /Users/pwner/Git/ABS/postgresql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:15  # ✅ Official stable image
        env:
        - name: POSTGRES_PASSWORD
          value: "dev-password"
        - name: POSTGRES_DB
          value: "soliditysecurity"
        - name: POSTGRES_USER
          value: "postgres"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgresql-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgresql-storage
        emptyDir: {}  # ⚠️ LOCAL DEV ONLY - Use PVC in production
```

#### **Verification**
```bash
kubectl exec postgresql-<pod-id> -- psql -U postgres -d soliditysecurity -c "SELECT version();"
# Result: PostgreSQL 15.14 (Debian 15.14-1.pgdg13+1) ✅
```

### Redis Image Issues ❌➡️✅

#### **Problem**
Similar Bitnami Redis image pull failures:
```bash
Failed to pull image "docker.io/bitnami/redis:8.2.1-debian-12-r0":
manifest for bitnami/redis:8.2.1-debian-12-r0 not found
```

#### **Solution Applied**
Replaced with official Redis image:

```yaml
# File: /Users/pwner/Git/ABS/redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      role: master
  template:
    metadata:
      labels:
        app: redis
        role: master
    spec:
      containers:
      - name: redis
        image: redis:7  # ✅ Official stable image
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        emptyDir: {}  # ⚠️ LOCAL DEV ONLY - Use PVC in production
```

#### **Verification**
```bash
kubectl exec redis-master-<pod-id> -- redis-cli ping
# Result: PONG ✅
```

## ConfigMap Updates

### Database Connection Strings

Updated ConfigMap to use correct internal DNS names:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-config
  namespace: solidity-security
data:
  DATABASE_URL: "postgresql://postgres:dev-password@postgresql.default.svc.cluster.local:5432/soliditysecurity"
  REDIS_URL: "redis://redis-master.default.svc.cluster.local:6379"
  VAULT_URL: "http://vault.default.svc.cluster.local:8200"
  VAULT_TOKEN: "dev-root-token"
```

**Key Changes:**
- Added `.default.svc.cluster.local` for cross-namespace DNS resolution
- Changed from Bitnami service names to custom service names
- Maintained production-compatible environment variable names

## Docker Build Fixes

### Memory Allocation Issue ❌➡️✅

#### **Problem**
```bash
Docker Desktop has only 7837MB memory but you specified 12288MB
```

#### **Solution**
Reduced minikube memory allocation:
```bash
# Original (failed)
minikube start --memory=12288

# Fixed (working)
minikube start --driver=docker --memory=7000 --cpus=6 --disk-size=80g
```

### Docker Engine Connection ❌➡️✅

#### **Problem**
Docker daemon not running / wrong socket path

#### **Solution**
Used podman-machine with Docker API compatibility:
```bash
# Start podman machine
podman machine start podman-machine-default

# Set Docker host for minikube
export DOCKER_HOST='unix:///var/folders/b3/bj8j75yx1db_5cdbgbsl5f5c0000gn/T/podman/podman-machine-default-api.sock'
```

### Development Dependencies Issue ❌➡️✅

#### **Problem**
Service builds failing on missing development packages:
```bash
ERROR: Could not find a version that satisfies the requirement httpx-mock<1.0.0,>=0.10.1
```

#### **Solution**
Modified Dockerfiles to exclude dev dependencies in production builds:
```dockerfile
# Before (failing)
RUN pip install --user --no-cache-dir -r requirements/base.txt -r requirements/dev.txt

# After (working)
RUN pip install --user --no-cache-dir -r requirements/base.txt
```

**⚠️ Production Note**: Production builds should include proper dev dependency management.

## Namespace Strategy

### Applied Structure
```
default              # Infrastructure services (PostgreSQL, Redis, Vault)
├── postgresql       # Custom PostgreSQL deployment
├── redis-master     # Custom Redis deployment
└── vault-0          # Vault (Helm deployment)

monitoring           # Monitoring stack (Helm deployment)
├── monitoring-grafana
├── prometheus-monitoring-kube-prometheus-prometheus-0
└── alertmanager-monitoring-kube-prometheus-alertmanager-0

solidity-security    # Application services (future deployments)
└── service-config   # ConfigMap with database URLs

ingress-nginx        # Ingress controller (minikube addon)
kube-system          # Kubernetes system components
```

**Rationale**:
- Infrastructure in `default` for simplicity
- Applications in dedicated namespace for isolation
- Monitoring in separate namespace per Helm chart requirements

## Local Registry Configuration

### Setup
```bash
# Start local Docker registry
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  --restart unless-stopped \
  registry:2

# Verify
curl http://localhost:5000/v2/_catalog
```

### Integration
All built images use the `localhost:5000/` prefix for local development.

## Storage Simplifications

### **⚠️ LOCAL DEVELOPMENT ONLY**

All persistent storage uses `emptyDir` volumes:

```yaml
volumes:
- name: postgresql-storage
  emptyDir: {}  # Data lost on pod restart
```

**Production Requirements:**
- Use PersistentVolumeClaims (PVC)
- Configure appropriate StorageClasses
- Implement backup strategies
- Use StatefulSets for databases

## Security Relaxations

### **⚠️ LOCAL DEVELOPMENT ONLY**

1. **Vault**: Development mode with static token
2. **PostgreSQL**: Simple password in ConfigMap (not Secret)
3. **Redis**: No authentication enabled
4. **Container Security**: Running as non-root but simplified

**Production Requirements:**
- Proper secret management with Vault
- Encrypted secret storage
- Network policies
- Pod security standards
- RBAC implementation

## Performance Optimizations Applied

### Resource Allocation
```yaml
# Applied to minikube
--memory=7000    # Reduced from 12GB
--cpus=6         # Maintained
--disk-size=80g  # Maintained
```

### Image Optimization
- Multi-stage Docker builds maintained
- Dependency caching enabled
- Local registry for faster pulls

## Network Configuration

### Ingress Setup
```yaml
# File: /Users/pwner/Git/ABS/local-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: solidityops-ingress
  namespace: monitoring
spec:
  rules:
  - host: grafana.solidityops.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitoring-grafana
            port:
              number: 80
```

**Note**: Host resolution requires `/etc/hosts` entries or port-forwarding for local access.

## Monitoring and Observability

### Working Components
- ✅ Prometheus metrics collection
- ✅ Grafana dashboards (admin/admin)
- ✅ Alertmanager notifications
- ✅ Node and pod metrics

### Access Methods
```bash
# Port forwarding (recommended for local dev)
kubectl port-forward svc/monitoring-grafana 3001:80 -n monitoring &

# Ingress (requires host configuration)
echo "$(minikube ip) grafana.solidityops.local" | sudo tee -a /etc/hosts
```

## Troubleshooting Applied Fixes

### Common Issues Resolved

1. **Pod Pending State**: Resolved with correct resource allocation
2. **ImagePullBackOff**: Fixed with official Docker images
3. **DNS Resolution**: Fixed with full service DNS names
4. **Permission Denied**: Fixed Dockerfile user/directory permissions
5. **Build Failures**: Resolved dependency and shared library issues

### Diagnostic Commands Used
```bash
# Pod investigation
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>

# Resource checking
kubectl top nodes
kubectl top pods -A

# Network testing
kubectl exec -it <pod-name> -- nslookup <service-name>
```

## Rollback Procedures

### Infrastructure Services
```bash
# Remove custom deployments
kubectl delete -f postgresql-deployment.yaml
kubectl delete -f redis-deployment.yaml

# Reinstall with Helm (if needed)
helm install postgresql bitnami/postgresql [options]
```

### Application Services
```bash
# Remove local images
docker rmi localhost:5000/solidity-security-api-service:dev

# Clean local registry
docker stop registry && docker rm registry
```

---

**Applied Date**: October 2, 2025
**Environment**: Local Development minikube
**Status**: ✅ All fixes applied and tested
**Production Compatibility**: ❌ Requires production-specific configurations
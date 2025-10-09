# API Service Local Development Configuration

**Last Updated**: October 9, 2025

## Overview

This document describes the local development configuration for the API Service, including dependency management, Kubernetes deployment, and troubleshooting common issues.

## Dependencies

### Python Requirements

The API service uses Python 3.13 with the following critical dependencies:

#### Core Framework
```python
fastapi>=0.104.1,<1.0.0
uvicorn[standard]>=0.24.0,<1.0.0
pydantic>=2.5.0,<3.0.0
pydantic-settings>=2.1.0,<3.0.0
```

#### Database
```python
sqlalchemy>=2.0.23,<3.0.0
alembic>=1.13.0,<2.0.0
asyncpg>=0.29.0,<1.0.0  # PostgreSQL async driver
```

#### Redis
```python
redis>=5.0.1,<6.0.0
hiredis>=2.2.3,<3.0.0
```

#### Authentication (Critical)
```python
python-jose[cryptography]>=3.3.0,<4.0.0
bcrypt==4.2.1  # MUST be pinned to 4.2.1
python-multipart>=0.0.6,<1.0.0
```

**IMPORTANT**: bcrypt must be pinned to version 4.2.1 due to compatibility issues:
- bcrypt 5.x removed the `__about__` module
- passlib 1.7.4 (transitive dependency) requires `bcrypt.__about__`
- Using bcrypt 5.x causes: `AttributeError: module 'bcrypt' has no attribute '__about__'`

## Configuration Management

### Settings Architecture

The API service uses Pydantic Settings for configuration management:

**File**: `src/infrastructure/config.py`

```python
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # Configuration fields...
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret_key: str
    # ...

@lru_cache
def get_settings() -> Settings:
    """Get cached settings instance"""
    return Settings()
```

### Usage Pattern

When importing settings in application code:

```python
# CORRECT
from src.infrastructure.config import get_settings

settings = get_settings()
redis_url = str(settings.redis_url)  # Convert Pydantic types to strings

# INCORRECT
from src.infrastructure.config import settings  # Does not exist!
```

**Important Type Conversions**:
- `RedisDsn` objects must be converted to strings: `str(settings.redis_url)`
- `PostgresDsn` objects must be converted to strings: `str(settings.database_url)`

## Kubernetes Configuration

### Image Versioning

**DO NOT use `:latest` tags in production or local development**

Always use semantic versioning for Docker images:

```yaml
# Good
images:
- name: api-service
  newTag: 0.1.1-bcrypt-fix

# Bad
images:
- name: api-service
  newTag: latest  # Causes caching issues!
```

**Why versioned tags?**
1. Forces Kubernetes to recognize image changes
2. Prevents cache-related deployment issues
3. Enables rollback to specific versions
4. Provides audit trail
5. Enterprise production best practice

### Local Development Overlay

**File**: `k8s/overlays/local/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- namespace.yaml
- ../../base/

namespace: api-service-local

images:
- name: api-service
  newTag: 0.1.1-bcrypt-fix  # Use semantic versioning

labels:
- includeSelectors: true
  pairs:
    app.kubernetes.io/name: api-service
    app.kubernetes.io/version: 0.1.1-bcrypt-fix
    environment: local
```

### Deployment Patch

**File**: `k8s/overlays/local/deployment-patch.yaml`

Key configurations for local development:

```yaml
spec:
  replicas: 1  # Single replica for local dev
  template:
    spec:
      containers:
      - name: api-service
        resources:
          requests:
            memory: "64Mi"   # Minimal for local
            cpu: "50m"
          limits:
            memory: "256Mi"  # Conservative limit
            cpu: "200m"
        env:
        - name: REDIS_URL
          value: "redis://:redis-local-password@redis.redis-local.svc.cluster.local:6379"
          # IMPORTANT: Include password in URL for authentication
```

### Redis Authentication

Local Redis instance requires authentication. URL format:

```
redis://:password@host:port/db
```

Example:
```
redis://:redis-local-password@redis.redis-local.svc.cluster.local:6379/0
```

**Common Error**: If password is missing:
```
redis.exceptions.AuthenticationError: Authentication required
```

**Solution**: Update deployment-patch.yaml with authenticated URL.

## Resource Management

### Local Development Resources

For local Minikube development on MacBook (16GB RAM):

**Per Service**:
- Replicas: 1 (no HPA)
- Memory Request: 64Mi
- Memory Limit: 256Mi
- CPU Request: 50m
- CPU Limit: 200m

**Why?**
- Conserves limited laptop resources
- HPA unnecessary for local development
- Allows running full stack (8+ services) simultaneously
- Production uses different resource profiles

### Memory Optimization

If experiencing memory pressure:

1. **Check current usage**:
   ```bash
   kubectl top nodes
   ```

2. **Scale down services**:
   ```bash
   kubectl scale deployment <service> --replicas=1 -n <namespace>
   ```

3. **Delete HPAs in local namespaces**:
   ```bash
   kubectl get hpa -A | grep local
   kubectl delete hpa <hpa-name> -n <namespace>
   ```

4. **Disable Grafana** (if not actively monitoring):
   ```bash
   kubectl scale deployment grafana --replicas=0 -n monitoring-local
   ```

Expected local memory usage: **60-70%** of Minikube allocation

## Deployment Workflow

### Building and Deploying Changes

1. **Set Minikube Docker environment**:
   ```bash
   eval $(minikube docker-env)
   ```

2. **Build with versioned tag**:
   ```bash
   docker build -t api-service:0.1.2 .
   ```

3. **Update kustomization.yaml**:
   ```yaml
   images:
   - name: api-service
     newTag: 0.1.2
   ```

4. **Apply configuration**:
   ```bash
   kubectl apply -k k8s/overlays/local
   ```

5. **Verify deployment**:
   ```bash
   kubectl rollout status deployment/api-service -n api-service-local
   kubectl get pods -n api-service-local
   ```

6. **Check image is correct**:
   ```bash
   kubectl get pods -n api-service-local -o jsonpath='{.items[0].spec.containers[0].image}'
   ```

## Troubleshooting

### Issue: bcrypt AttributeError

**Error**:
```
AttributeError: module 'bcrypt' has no attribute '__about__'
password cannot be longer than 72 bytes, truncate manually if necessary
```

**Solution**:
1. Verify `requirements/base.txt` has: `bcrypt==4.2.1`
2. Rebuild Docker image
3. Redeploy with new image tag

### Issue: ImportError for settings

**Error**:
```
ImportError: cannot import name 'settings' from 'src.infrastructure.config'
```

**Solution**:
```python
# Change from:
from src.infrastructure.config import settings

# To:
from src.infrastructure.config import get_settings
settings = get_settings()
```

### Issue: Redis Type Error

**Error**:
```
AttributeError: 'RedisDsn' object has no attribute 'decode'
```

**Solution**:
```python
# Convert Pydantic type to string
redis_client = aioredis.from_url(
    str(settings.redis_url),  # Add str() conversion
    encoding="utf-8"
)
```

### Issue: Redis Authentication Error

**Error**:
```
redis.exceptions.AuthenticationError: Authentication required
```

**Solution**:
Update `k8s/overlays/local/deployment-patch.yaml`:
```yaml
- name: REDIS_URL
  value: "redis://:redis-local-password@redis.redis-local.svc.cluster.local:6379"
```

### Issue: Pods Using Old Image

**Symptoms**:
- Rebuilt Docker image but changes not reflected
- `kubectl describe pod` shows old image ID

**Solution**:
1. Use versioned tags instead of `:latest`
2. Update kustomization.yaml with new version
3. Apply configuration: `kubectl apply -k k8s/overlays/local`
4. Force rollout: `kubectl rollout restart deployment/api-service -n api-service-local`

### Issue: HPA Overriding Replica Count

**Symptoms**:
- Set `replicas: 1` but deployment has multiple pods
- `kubectl get hpa` shows active HPA

**Solution**:
```bash
kubectl delete hpa api-service-hpa -n api-service-local
kubectl scale deployment api-service --replicas=1 -n api-service-local
```

## Testing

### Port Forwarding

```bash
kubectl port-forward -n api-service-local svc/api-service 8000:8000
```

### API Health Check

```bash
curl http://localhost:8000/
# Expected: {"message":"Solidity Security API Service","status":"running","version":"0.1.0"}
```

### Authentication Test

1. **Register User**:
   ```bash
   curl -X POST http://localhost:8000/api/v1/auth/register \
     -H "Content-Type: application/json" \
     -d '{"email": "test@example.com", "password": "TestPassword123", "username": "testuser"}'
   ```

2. **Login**:
   ```bash
   curl -X POST http://localhost:8000/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "test@example.com", "password": "TestPassword123"}'
   ```

3. **Expected Response**:
   ```json
   {
     "access_token": "eyJhbGci...",
     "refresh_token": "eyJhbGci...",
     "token_type": "bearer",
     "expires_in": 10800
   }
   ```

### End-to-End Workflow Test

See `/Users/pwner/Git/ABS/TaskDocs/SolidityOps/api-test-results-summary.md` for comprehensive test scripts.

## Security Considerations

### Local Development Secrets

Local development uses simplified secrets:

```yaml
JWT_SECRET_KEY: "local-dev-jwt-secret-key-change-in-production"
SESSION_SECRET: "local-dev-session-secret-change-in-production"
REDIS_PASSWORD: "redis-local-password"
```

**NEVER use these values in staging or production environments.**

### Production Differences

Production configuration differs significantly:

1. **Secrets**: Managed via HashiCorp Vault and External Secrets Operator
2. **Replicas**: 3+ with HPA (min: 2, max: 10)
3. **Resources**: Much higher memory/CPU allocations
4. **TLS**: All communications encrypted
5. **Network Policies**: Strict ingress/egress rules

## References

- [Scanner Execution Architecture](./scanner-execution-architecture.md)
- [Kubernetes Deployment Guide](../local-development/kubernetes-setup.md)
- [API Documentation](../api/README.md)

---

**Maintainer**: Backend Team
**Last Verified**: October 9, 2025
**Next Review**: November 2025

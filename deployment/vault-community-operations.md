# HashiCorp Vault Community Edition Operations Guide

## Overview

This guide covers the operation of HashiCorp Vault Community Edition deployed in EKS with the `[service]-[overlay]` namespace convention.

## Namespace Configuration

### Vault Server Namespaces
| Environment | Namespace | Vault URL | Purpose |
|-------------|-----------|-----------|----------|
| Staging | `vault-staging` | `http://vault.vault-staging.svc.cluster.local:8200` | Vault server pods and storage |
| Production | `vault-production` | `http://vault.vault-production.svc.cluster.local:8200` | Vault server pods and storage |

### External Secrets Operator Namespaces
| Environment | Namespace | Purpose |
|-------------|-----------|----------|
| Staging | `external-secrets-staging` | External Secrets Operator for secret injection |
| Production | `external-secrets-production` | External Secrets Operator for secret injection |

## Architecture: Vault vs External Secrets

```
┌─────────────────────┐    ┌──────────────────────────┐    ┌─────────────────────┐
│   vault-staging     │    │  external-secrets-staging │    │  Application        │
│                     │    │                          │    │  Namespaces         │
│ ┌─────────────────┐ │    │ ┌────────────────────┐ │    │ ┌─────────────────┐ │
│ │ Vault Server    │◄┼────┼─┤ External Secrets     │ │    │ │ Kubernetes      │ │
│ │ - Secret Storage│ │    │ │ Operator             │ │    │ │ Secrets         │ │
│ │ - KV Engine     │ │    │ │ - Fetches from Vault │─┼────┼►│ - Used by Pods  │ │
│ │ - Auth Methods  │ │    │ │ - Creates K8s Secrets│ │    │ │ - Mounted as    │ │
│ └─────────────────┘ │    │ └────────────────────┘ │    │ │   Files/EnvVars │ │
└─────────────────────┘    └──────────────────────────┘    │ └─────────────────┘ │
                                                           └─────────────────────┘
```

**Key Distinction:**
- **Vault** (`vault-*` namespaces): The **secret storage backend** - stores actual secret data
- **External Secrets** (`external-secrets-*` namespaces): The **secret injection mechanism** - fetches secrets from Vault and creates Kubernetes secrets for applications

## Community Edition Limitations

**What's NOT available in Community Edition:**
- ❌ **Auto-unseal** (AWS KMS, Azure Key Vault, etc.)
- ❌ **Enterprise features** (namespaces, replication, etc.)
- ❌ **Advanced auth methods** (some enterprise-only methods)

**What IS available in Community Edition:**
- ✅ **Manual unseal** (Shamir's secret sharing)
- ✅ **Raft storage** (integrated storage)
- ✅ **Kubernetes auth** (for External Secrets integration)
- ✅ **KV secrets engine** (v1 and v2)
- ✅ **Web UI**
- ✅ **High availability** (with Raft)

## Deployment Architecture

### High Availability Setup
- **3-node Raft cluster** for HA
- **Integrated storage** (no external dependencies)
- **Manual unseal** required after restarts
- **Leader election** via Raft consensus

### Pod Distribution
```
vault-0.vault-internal  (Raft Node 1)
vault-1.vault-internal  (Raft Node 2)
vault-2.vault-internal  (Raft Node 3)
```

## Initial Setup Process

### 1. Deploy Infrastructure
```bash
# Deploy staging environment
kubectl apply -k k8s/overlays/staging/infrastructure

# Deploy production environment
kubectl apply -k k8s/overlays/production/infrastructure
```

### 2. Wait for Pods to Start
```bash
# Check pod status
kubectl get pods -n vault-staging
kubectl get pods -n vault-production
```

### 3. Initialize Vault (First Time Only)

**For Staging:**
```bash
# Initialize Vault (only run ONCE)
kubectl exec -it vault-0 -n vault-staging -- vault operator init

# IMPORTANT: Save the 5 unseal keys and root token securely!
# Example output:
# Unseal Key 1: <base64-key-1>
# Unseal Key 2: <base64-key-2>
# Unseal Key 3: <base64-key-3>
# Unseal Key 4: <base64-key-4>
# Unseal Key 5: <base64-key-5>
# Initial Root Token: <root-token>
```

**For Production:**
```bash
# Initialize Vault (only run ONCE)
kubectl exec -it vault-0 -n vault-production -- vault operator init
```

### 4. Unseal Vault Cluster

**Staging Environment:**
```bash
# Unseal vault-0 (requires 3 of 5 keys)
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <unseal-key-1>
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <unseal-key-2>
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <unseal-key-3>

# Unseal vault-1
kubectl exec -it vault-1 -n vault-staging -- vault operator unseal <unseal-key-1>
kubectl exec -it vault-1 -n vault-staging -- vault operator unseal <unseal-key-2>
kubectl exec -it vault-1 -n vault-staging -- vault operator unseal <unseal-key-3>

# Unseal vault-2
kubectl exec -it vault-2 -n vault-staging -- vault operator unseal <unseal-key-1>
kubectl exec -it vault-2 -n vault-staging -- vault operator unseal <unseal-key-2>
kubectl exec -it vault-2 -n vault-staging -- vault operator unseal <unseal-key-3>
```

**Production Environment:**
```bash
# Repeat the same process for production namespace
kubectl exec -it vault-0 -n vault-production -- vault operator unseal <prod-unseal-key-1>
kubectl exec -it vault-0 -n vault-production -- vault operator unseal <prod-unseal-key-2>
kubectl exec -it vault-0 -n vault-production -- vault operator unseal <prod-unseal-key-3>

# Continue for vault-1 and vault-2...
```

### 5. Configure Kubernetes Authentication

**Staging:**
```bash
# Login with root token
kubectl exec -it vault-0 -n vault-staging -- vault auth <staging-root-token>

# Enable Kubernetes auth
kubectl exec -it vault-0 -n vault-staging -- vault auth enable kubernetes

# Configure Kubernetes auth
kubectl exec -it vault-0 -n vault-staging -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$(kubectl exec vault-0 -n vault-staging -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert="$(kubectl exec vault-0 -n vault-staging -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"
```

### 6. Set Up External Secrets Operator Integration

**Purpose**: Configure Vault to allow the External Secrets Operator (running in `external-secrets-*` namespaces) to authenticate and fetch secrets.

**Create Policy for External Secrets Operator:**
```bash
# This policy allows the External Secrets Operator to read secrets
kubectl exec -it vault-0 -n vault-staging -- vault policy write external-secrets - <<EOF
path "secret/*" {
  capabilities = ["read", "list"]
}
EOF
```

**Create Role for External Secrets Operator:**
```bash
# This role binds the External Secrets Operator service account to the policy
kubectl exec -it vault-0 -n vault-staging -- vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets-staging \
  policies=external-secrets \
  ttl=24h

# Note: The service account 'external-secrets' runs in the 'external-secrets-staging' namespace,
# NOT in the vault-staging namespace. This is the separation of concerns.
```

### 7. Enable Secrets Engine

```bash
# Enable KV v2 secrets engine
kubectl exec -it vault-0 -n vault-staging -- vault secrets enable -path=secret kv-v2
```

## Daily Operations

### Check Vault Status
```bash
# Check if Vault is sealed/unsealed
kubectl exec -it vault-0 -n vault-staging -- vault status

# Check cluster status
kubectl exec -it vault-0 -n vault-staging -- vault operator raft list-peers
```

### Unseal After Pod Restart
```bash
# If pods restart, you need to unseal again
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <key-1>
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <key-2>
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <key-3>
```

### Access Vault UI
```bash
# Port-forward for local access
kubectl port-forward svc/vault 8200:8200 -n vault-staging

# Access at: http://localhost:8200
# Login with root token or other auth methods
```

## Backup and Recovery

### Raft Snapshots
```bash
# Create snapshot
kubectl exec -it vault-0 -n vault-staging -- vault operator raft snapshot save backup.snap

# Copy snapshot from pod
kubectl cp vault-staging/vault-0:backup.snap ./vault-backup-$(date +%Y%m%d).snap
```

### Disaster Recovery
1. **Deploy new Vault cluster**
2. **Initialize new cluster**
3. **Restore from snapshot:**
   ```bash
   kubectl cp ./vault-backup.snap vault-staging/vault-0:restore.snap
   kubectl exec -it vault-0 -n vault-staging -- vault operator raft snapshot restore restore.snap
   ```

## Security Considerations

### Unseal Key Management
- **Split keys across multiple operators**
- **Store keys in separate secure locations**
- **Never store all keys in the same place**
- **Consider using a key management system**

### Network Security
- **Vault runs on HTTP** internally (behind ALB with HTTPS)
- **Kubernetes network policies** can restrict access
- **Service mesh** can provide mTLS if needed

### Audit Logging
```bash
# Enable audit logging
kubectl exec -it vault-0 -n vault-staging -- vault audit enable file file_path=/vault/logs/audit.log
```

## Troubleshooting

### Common Issues

**Vault Sealed:**
```bash
# Check status
kubectl exec -it vault-0 -n vault-staging -- vault status

# If sealed, unseal with keys
kubectl exec -it vault-0 -n vault-staging -- vault operator unseal <key>
```

**Raft Cluster Issues:**
```bash
# Check cluster health
kubectl exec -it vault-0 -n vault-staging -- vault operator raft list-peers

# Check logs
kubectl logs vault-0 -n vault-staging
```

**External Secrets Not Working:**
```bash
# Check External Secrets Operator (runs in external-secrets-* namespace)
kubectl get pods -n external-secrets-staging
kubectl logs -n external-secrets-staging deployment/external-secrets

# Check ClusterSecretStore configuration
kubectl describe clustersecretstore vault-backend

# Check External Secret resources
kubectl describe externalsecret <secret-name> -n <namespace>

# Test Vault connectivity from External Secrets namespace
kubectl exec -n external-secrets-staging -it deployment/external-secrets -- \
  vault auth -address=http://vault.vault-staging.svc.cluster.local:8200 -method=kubernetes role=external-secrets

# Remember: External Secrets Operator runs in external-secrets-* namespaces,
# NOT in the vault-* namespaces where Vault servers run.
```

## Production Considerations

1. **Separate unseal keys** for staging and production
2. **Different root tokens** for each environment
3. **Regular snapshot backups**
4. **Monitor Vault health** with the monitoring stack
5. **Plan for manual unseal procedures** during maintenance
6. **Document emergency procedures** for the team

## Future Considerations

If you need enterprise features in the future:
- **Auto-unseal** with AWS KMS (requires Vault Enterprise)
- **Disaster Recovery** replication (requires Vault Enterprise)
- **Performance replication** (requires Vault Enterprise)
- **Namespaces** for multi-tenancy (requires Vault Enterprise)

For now, Vault Community Edition provides all the secret management capabilities needed for the Solidity Security Platform.
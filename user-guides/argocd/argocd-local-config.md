# ArgoCD Local Configuration Guide

Configure GitHub → ArgoCD (local) → Minikube Cluster GitOps Pipeline

## Overview

This guide walks through configuring ArgoCD running on your local minikube cluster to automatically deploy applications from GitHub repositories. You'll set up a complete GitOps workflow where changes to your GitHub repo automatically sync to your minikube cluster.

## Prerequisites

- Minikube cluster running with ArgoCD deployed
- GitHub repository with Kubernetes manifests
- ArgoCD accessible at `https://argocd-local.minikube.local`

## Architecture

```
GitHub Repository → ArgoCD (minikube) → Application Deployment (minikube)
     ↓                    ↓                         ↓
  Git Push           Sync Detection           Kubernetes Apply
```

## Step 1: Access ArgoCD UI

### Method 1: Web Browser (Recommended)
```bash
# Access ArgoCD UI directly
open https://argocd-local.minikube.local
```

### Method 2: Port Forward
```bash
# Port forward ArgoCD service
kubectl port-forward -n argocd-local svc/argocd-server 8080:80

# Access via localhost
open http://localhost:8080
```

### Login Credentials
- **Username**: `admin`
- **Password**: `admin`

## Step 2: Configure GitHub Repository Access

### For Public Repositories
ArgoCD can access public GitHub repositories without additional configuration.

### For Private Repositories

#### Option A: Personal Access Token
1. **Generate GitHub PAT**:
   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Create token with `repo` permissions
   - Copy the token

2. **Add Repository in ArgoCD UI**:
   - Go to Settings → Repositories
   - Click "Connect Repo"
   - **Type**: Git
   - **Repository URL**: `https://github.com/username/repo.git`
   - **Username**: Your GitHub username
   - **Password**: Your PAT token

#### Option B: SSH Key (Advanced)
1. **Generate SSH Key**:
```bash
# Generate SSH key for ArgoCD
ssh-keygen -t rsa -b 4096 -f ~/.ssh/argocd_rsa -N ""
```

2. **Add Public Key to GitHub**:
   - Copy `~/.ssh/argocd_rsa.pub` content
   - Add to GitHub Settings → SSH keys

3. **Add SSH Key to ArgoCD**:
   - In ArgoCD UI: Settings → Repositories
   - **Type**: Git
   - **Repository URL**: `git@github.com:username/repo.git`
   - **SSH Private Key**: Paste content of `~/.ssh/argocd_rsa`

## Step 3: Create ArgoCD Application

### Method 1: ArgoCD UI
1. **Click "New App"**
2. **General Settings**:
   - **Application Name**: `my-app`
   - **Project**: `default`
   - **Sync Policy**: `Automatic`

3. **Source Settings**:
   - **Repository URL**: `https://github.com/username/repo.git`
   - **Revision**: `HEAD` (or specific branch)
   - **Path**: `k8s/manifests` (path to your Kubernetes files)

4. **Destination Settings**:
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `default` (or target namespace)

5. **Click "Create"**

### Method 2: YAML Configuration
Create `application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd-local
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/username/repo.git
    targetRevision: HEAD
    path: k8s/manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Apply the configuration:
```bash
kubectl apply -f application.yaml
```

## Step 4: Repository Structure

Organize your GitHub repository:

```
your-repo/
├── k8s/
│   └── manifests/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── README.md
└── ...
```

### Sample Application Manifests

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: app
        image: nginx:1.20
        ports:
        - containerPort: 80
```

**service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

## Step 5: Configure Sync Policies

### Automatic Sync
For automatic deployment when code changes:

```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources not in Git
    selfHeal: true   # Correct drift from desired state
  syncOptions:
  - CreateNamespace=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

### Manual Sync
For manual approval workflow:

```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true
```

## Step 6: Monitor and Manage

### View Application Status
```bash
# List all applications
kubectl get applications -n argocd-local

# Get application details
kubectl describe application my-app -n argocd-local
```

### Sync Applications
```bash
# Manual sync via CLI
argocd app sync my-app

# Or use ArgoCD UI
# Go to Applications → Select app → Click "Sync"
```

### View Logs
```bash
# ArgoCD server logs
kubectl logs -n argocd-local deployment/argocd-server

# Application controller logs
kubectl logs -n argocd-local deployment/argocd-application-controller
```

## Step 7: GitOps Workflow

### Development Workflow
1. **Make changes** to Kubernetes manifests in your GitHub repo
2. **Commit and push** changes to the configured branch
3. **ArgoCD detects** changes (polls every 3 minutes by default)
4. **Automatic sync** deploys changes to minikube cluster
5. **Monitor** deployment in ArgoCD UI

### Manual Workflow
1. **Make changes** and push to GitHub
2. **Login to ArgoCD UI**
3. **Click "Refresh"** to detect changes
4. **Click "Sync"** to deploy changes
5. **Verify** deployment status

## Troubleshooting

### Common Issues

#### 1. Repository Access Denied
```bash
# Check repository credentials
kubectl get secret -n argocd-local
kubectl describe secret argocd-repo-server-tls -n argocd-local
```

#### 2. Application Sync Failed
```bash
# Check application events
kubectl describe application my-app -n argocd-local

# View detailed sync status
argocd app get my-app
```

#### 3. Network Issues
```bash
# Test GitHub connectivity from ArgoCD
kubectl exec -n argocd-local deployment/argocd-repo-server -- \
  curl -I https://github.com
```

#### 4. Permission Issues
```bash
# Check ArgoCD RBAC
kubectl describe clusterrole argocd-server
kubectl describe clusterrolebinding argocd-server
```

### Health Checks

#### Verify ArgoCD Components
```bash
# Check all ArgoCD resources
kubectl get all -n argocd-local

# Ensure pods are running
kubectl get pods -n argocd-local
```

#### Test Application Deployment
```bash
# Check target namespace
kubectl get all -n default

# Verify application resources
kubectl get deployments,services,ingress -n default
```

## Advanced Configuration

### Multi-Environment Setup
Configure different applications for different environments:

```yaml
# staging-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
spec:
  source:
    path: k8s/overlays/staging
  destination:
    namespace: staging

# production-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
spec:
  source:
    path: k8s/overlays/production
  destination:
    namespace: production
```

### Webhook Configuration
For faster sync times, configure GitHub webhooks:

1. **In GitHub repo**: Settings → Webhooks
2. **Payload URL**: `https://argocd-local.minikube.local/api/webhook`
3. **Content type**: `application/json`
4. **Events**: Push events

### Resource Hooks
Use ArgoCD hooks for complex deployment scenarios:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-job
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  # Job specification
```

## Security Best Practices

1. **Use least privilege** for ArgoCD service accounts
2. **Rotate credentials** regularly (PATs, SSH keys)
3. **Enable RBAC** for ArgoCD users and applications
4. **Use encrypted secrets** for sensitive data
5. **Monitor access logs** for suspicious activity

## Next Steps

- Set up monitoring with Prometheus/Grafana integration
- Configure multiple projects for team separation
- Implement advanced sync policies and health checks
- Explore ArgoCD CLI for automation scripts
- Set up disaster recovery procedures

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://opengitops.dev/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitHub Actions with ArgoCD](https://docs.github.com/en/actions)
# Local Deployment Guide - Dependency Monitoring

This guide covers deploying the dependency monitoring service to a local Kubernetes environment for development and testing purposes.

## Prerequisites

### Required Tools
- **kubectl**: Kubernetes command-line tool
- **minikube** or **Docker Desktop**: Local Kubernetes cluster
- **kustomize**: Kubernetes configuration management (included in kubectl 1.14+)
- **curl**: For API testing
- **jq**: JSON processing (optional, for formatted output)

### Verify Prerequisites
```bash
# Check kubectl
kubectl version --client

# Check cluster access
kubectl cluster-info

# Verify kustomize
kubectl kustomize --help

# Check available nodes
kubectl get nodes
```

## Deployment Steps

### 1. Prepare Local Environment

#### Start Local Kubernetes Cluster
```bash
# If using minikube
minikube start --memory=4096 --cpus=2

# If using Docker Desktop
# Enable Kubernetes in Docker Desktop settings

# Verify cluster is running
kubectl get nodes
```

#### Set Kubernetes Context
```bash
# List available contexts
kubectl config get-contexts

# Set to local context (adjust name as needed)
kubectl config use-context minikube
# or
kubectl config use-context docker-desktop
```

### 2. Validate Kubernetes Manifests

#### Check Kustomize Configuration
```bash
# Navigate to monitoring repository
cd /Users/pwner/Git/ABS/solidity-security-monitoring

# Validate base configuration
kubectl kustomize k8s/base/dependency-monitor/

# Validate local overlay
kubectl kustomize k8s/overlays/local/dependency-monitor/
```

#### Verify Required Files
```bash
# Check that all required files exist
ls -la k8s/base/dependency-monitor/
ls -la k8s/overlays/local/dependency-monitor/

# Expected files in base:
# - kustomization.yaml
# - deployment.yaml
# - service.yaml
# - configmap.yaml
# - serviceaccount.yaml
# - cronjob.yaml

# Expected files in local overlay:
# - kustomization.yaml
# - namespace.yaml
# - deployment-patch.yaml
# - configmap-patch.yaml
# - service_paths.json
# - service_languages.json
# - scan_schedule.yaml
```

### 3. Deploy Dependency Monitor

#### Create Namespace and Deploy Service
```bash
# Deploy the complete dependency monitoring service
kubectl apply -k k8s/overlays/local/dependency-monitor/

# Verify deployment status
kubectl get all -n monitoring-local

# Expected output should show:
# - namespace/monitoring-local
# - deployment.apps/dependency-monitor
# - service/dependency-monitor
# - configmap/dependency-monitor-config
# - serviceaccount/dependency-monitor
# - cronjob.apps/dependency-scanner
# - cronjob.apps/security-vulnerability-scanner
```

#### Monitor Deployment Progress
```bash
# Watch pods until they're running
kubectl get pods -n monitoring-local -w

# Check deployment status
kubectl rollout status deployment/dependency-monitor -n monitoring-local

# Verify services are ready
kubectl get svc -n monitoring-local
```

### 4. Verify Deployment Health

#### Check Pod Status
```bash
# Get detailed pod information
kubectl get pods -n monitoring-local -o wide

# Check pod logs
kubectl logs -f deployment/dependency-monitor -n monitoring-local

# Describe pod for troubleshooting
kubectl describe pod -l app=dependency-monitor -n monitoring-local
```

#### Verify ConfigMaps
```bash
# Check ConfigMap creation
kubectl get configmap -n monitoring-local

# View ConfigMap contents
kubectl describe configmap dependency-monitor-config -n monitoring-local

# Check service paths configuration
kubectl get configmap dependency-monitor-config -n monitoring-local -o jsonpath='{.data.service_paths\.json}' | jq .
```

#### Check CronJobs
```bash
# Verify CronJobs are created
kubectl get cronjob -n monitoring-local

# Check CronJob details
kubectl describe cronjob dependency-scanner -n monitoring-local
kubectl describe cronjob security-vulnerability-scanner -n monitoring-local

# View recent jobs
kubectl get jobs -n monitoring-local
```

### 5. Test Service Connectivity

#### Port Forward to Service
```bash
# Forward local port to service
kubectl port-forward svc/dependency-monitor 8080:80 -n monitoring-local

# Keep this running in one terminal, continue in another
```

#### Test Health Endpoints
```bash
# Test basic health
curl http://localhost:8080/

# Expected response:
# {
#   "status": "healthy",
#   "version": "1.0.0",
#   "uptime": "5m 32s",
#   "services_monitored": 12
# }

# Test metrics endpoint
curl http://localhost:8080/metrics

# Should return Prometheus metrics format

# Test services configuration
curl http://localhost:8080/services | jq .
```

### 6. Validate Multi-Language Scanning

#### Test Python Service Scan
```bash
# Scan a Python service
curl -X POST http://localhost:8080/scan/api-service \
  -H "Content-Type: application/json"

# Expected response with scan results
# {
#   "service": "api-service",
#   "language": "python",
#   "scan_time": "2024-10-04T12:00:00Z",
#   "summary": {...}
# }
```

#### Test Node.js Service Scan
```bash
# Scan a Node.js service
curl -X POST http://localhost:8080/scan/ui-core \
  -H "Content-Type: application/json"
```

#### Test Rust Service Scan
```bash
# Scan a Rust service
curl -X POST http://localhost:8080/scan/contract-parser \
  -H "Content-Type: application/json"
```

#### Test Custom Project Scan
```bash
# Scan custom project
curl -X POST http://localhost:8080/scan/custom \
  -H "Content-Type: application/json" \
  -d '{
    "service_name": "test-project",
    "language": "python",
    "project_path": "/Users/pwner/Git/ABS/solidity-security-api-service",
    "scan_options": {
      "include_dev_dependencies": true,
      "vulnerability_only": false
    }
  }'
```

### 7. Verify Prometheus Integration

#### Check Metrics Export
```bash
# Get dependency metrics
curl -s http://localhost:8080/metrics | grep dependency_

# Expected metrics:
# dependency_current_version_info
# dependency_latest_version_info
# dependency_versions_behind_total
# dependency_vulnerabilities_total
# service_dependency_count_total
```

#### Test Metrics with Service Scan
```bash
# Trigger a scan
curl -X POST http://localhost:8080/scan/api-service

# Wait a moment, then check updated metrics
curl -s http://localhost:8080/metrics | grep -E "(dependency_scan_duration|dependency_vulnerabilities_total)"
```

### 8. Verify Automated Scanning

#### Check CronJob Execution
```bash
# Manually trigger a CronJob
kubectl create job --from=cronjob/dependency-scanner manual-scan-$(date +%s) -n monitoring-local

# Watch job execution
kubectl get jobs -n monitoring-local -w

# Check job logs
kubectl logs job/manual-scan-<timestamp> -n monitoring-local
```

#### View Scheduled Jobs
```bash
# List all CronJobs and their schedules
kubectl get cronjob -n monitoring-local -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,SUSPEND:.spec.suspend,ACTIVE:.status.active,LAST-SCHEDULE:.status.lastScheduleTime

# Check recent job history
kubectl get jobs -n monitoring-local --sort-by=.metadata.creationTimestamp
```

## Troubleshooting

### Common Issues

#### Pod Startup Issues
```bash
# Check pod events
kubectl describe pod -l app=dependency-monitor -n monitoring-local

# Common issues and solutions:
# - ImagePullBackOff: Check image name and availability
# - CrashLoopBackOff: Check logs for application errors
# - Pending: Check resource availability and node capacity

# Check resource usage
kubectl top pods -n monitoring-local
kubectl top nodes
```

#### Service Connectivity Issues
```bash
# Check service endpoints
kubectl get endpoints -n monitoring-local

# Test service connectivity from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- /bin/sh
# From inside pod:
# wget -qO- http://dependency-monitor.monitoring-local.svc.cluster.local/

# Check service logs for errors
kubectl logs -f deployment/dependency-monitor -n monitoring-local
```

#### ConfigMap Issues
```bash
# Verify ConfigMap data
kubectl get configmap dependency-monitor-config -n monitoring-local -o yaml

# Check file permissions in pod
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- ls -la /app/config/

# Recreate ConfigMap if needed
kubectl delete configmap dependency-monitor-config -n monitoring-local
kubectl apply -k k8s/overlays/local/dependency-monitor/
```

#### Scanning Issues
```bash
# Check if tools are available in container
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- which pip
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- which npm
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- which cargo

# Test repository access
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- ls -la /app/repos/

# Check scanning logs
kubectl logs -f deployment/dependency-monitor -n monitoring-local | grep -E "(scan|error)"
```

### Performance Issues

#### Resource Limits
```bash
# Check current resource usage
kubectl top pod -l app=dependency-monitor -n monitoring-local

# View resource limits
kubectl describe deployment dependency-monitor -n monitoring-local | grep -A 6 "Limits"

# Update resource limits if needed
kubectl patch deployment dependency-monitor -n monitoring-local -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "dependency-monitor",
          "resources": {
            "limits": {"memory": "1Gi", "cpu": "500m"},
            "requests": {"memory": "512Mi", "cpu": "250m"}
          }
        }]
      }
    }
  }
}'
```

#### Scan Performance
```bash
# Monitor scan duration metrics
curl -s http://localhost:8080/metrics | grep dependency_scan_duration_seconds

# Check for scan errors
curl -s http://localhost:8080/metrics | grep dependency_scan_errors_total

# Test scan timeout settings
curl -X POST http://localhost:8080/scan/api-service \
  -H "Content-Type: application/json" \
  -d '{"timeout": 300}'
```

### Debug Mode

#### Enable Debug Logging
```bash
# Update deployment with debug environment
kubectl patch deployment dependency-monitor -n monitoring-local -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "dependency-monitor",
          "env": [{"name": "LOG_LEVEL", "value": "DEBUG"}]
        }]
      }
    }
  }
}'

# Watch debug logs
kubectl logs -f deployment/dependency-monitor -n monitoring-local
```

#### Interactive Debugging
```bash
# Get shell access to pod
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- /bin/bash

# Inside pod, test components manually:
# cd /app
# python -m src.collectors.python_collector --service api-service
# python -m src.collectors.nodejs_collector --service ui-core
# python -m src.collectors.rust_collector --service contract-parser
```

## Cleanup

### Remove Deployment
```bash
# Delete the dependency monitor
kubectl delete -k k8s/overlays/local/dependency-monitor/

# Verify cleanup
kubectl get all -n monitoring-local

# Delete namespace if empty
kubectl delete namespace monitoring-local
```

### Reset Local Environment
```bash
# Stop port forwarding (Ctrl+C in terminal)

# If using minikube, optionally restart
minikube stop
minikube start --memory=4096 --cpus=2
```

## Next Steps

After successful local deployment:

1. **Integrate with Prometheus**: Set up local Prometheus to scrape metrics
2. **Add Grafana Dashboards**: Import dependency monitoring dashboards
3. **Configure Alerts**: Set up basic alerting rules
4. **Test CI/CD Integration**: Configure pipeline integration
5. **Scale Testing**: Test with larger repository sets

The local deployment provides a complete dependency monitoring environment for development and testing of the SolidityOps platform.
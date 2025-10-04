# Dependency Monitoring Troubleshooting Guide

This guide provides comprehensive troubleshooting procedures for the dependency monitoring service, covering common issues, diagnostic tools, and resolution strategies.

## Quick Diagnosis

### Health Check Commands
```bash
# Quick service status check
kubectl get pods -l app=dependency-monitor -n monitoring-local

# Service health endpoint
curl http://localhost:8080/ || echo "Service not accessible"

# Metrics availability
curl -s http://localhost:8080/metrics | head -10 || echo "Metrics not available"

# Recent logs
kubectl logs --tail=50 deployment/dependency-monitor -n monitoring-local
```

### Critical Service Indicators
- ✅ **Pod Status**: `Running` with `Ready 1/1`
- ✅ **Health Endpoint**: Returns HTTP 200 with status "healthy"
- ✅ **Metrics Endpoint**: Returns Prometheus metrics
- ✅ **Recent Scans**: Shows recent successful scans in logs

## Common Issues and Solutions

### 1. Pod/Container Issues

#### Issue: Pod Stuck in `Pending` State
**Symptoms:**
```bash
kubectl get pods -n monitoring-local
# NAME                                 READY   STATUS    RESTARTS   AGE
# dependency-monitor-xxx-yyy           0/1     Pending   0          5m
```

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod -l app=dependency-monitor -n monitoring-local

# Check node resources
kubectl describe nodes
kubectl top nodes
```

**Common Causes & Solutions:**
- **Insufficient Resources**:
  ```bash
  # Reduce resource requests
  kubectl patch deployment dependency-monitor -n monitoring-local -p '{
    "spec": {
      "template": {
        "spec": {
          "containers": [{
            "name": "dependency-monitor",
            "resources": {
              "requests": {"memory": "256Mi", "cpu": "100m"}
            }
          }]
        }
      }
    }
  }'
  ```

- **Node Selector Issues**:
  ```bash
  # Remove node selector if present
  kubectl patch deployment dependency-monitor -n monitoring-local -p '{
    "spec": {
      "template": {
        "spec": {
          "nodeSelector": null
        }
      }
    }
  }'
  ```

#### Issue: Pod in `CrashLoopBackOff`
**Symptoms:**
```bash
kubectl get pods -n monitoring-local
# dependency-monitor-xxx-yyy           0/1     CrashLoopBackOff   3          5m
```

**Diagnosis:**
```bash
# Check container logs
kubectl logs deployment/dependency-monitor -n monitoring-local --previous

# Check container exit code
kubectl describe pod -l app=dependency-monitor -n monitoring-local | grep "Exit Code"
```

**Common Causes & Solutions:**
- **Missing Dependencies**:
  ```bash
  # Check if required tools are in container
  kubectl exec deployment/dependency-monitor -n monitoring-local -- which pip
  kubectl exec deployment/dependency-monitor -n monitoring-local -- which npm
  kubectl exec deployment/dependency-monitor -n monitoring-local -- which cargo
  ```

- **Configuration Errors**:
  ```bash
  # Verify ConfigMap data
  kubectl get configmap dependency-monitor-config -n monitoring-local -o yaml

  # Check file mounts
  kubectl exec deployment/dependency-monitor -n monitoring-local -- ls -la /app/config/
  ```

- **Permission Issues**:
  ```bash
  # Check file permissions
  kubectl exec deployment/dependency-monitor -n monitoring-local -- ls -la /app/repos/

  # Verify service account
  kubectl describe serviceaccount dependency-monitor -n monitoring-local
  ```

#### Issue: Pod in `ImagePullBackOff`
**Symptoms:**
```bash
kubectl get pods -n monitoring-local
# dependency-monitor-xxx-yyy           0/1     ImagePullBackOff   0          2m
```

**Diagnosis:**
```bash
# Check image pull events
kubectl describe pod -l app=dependency-monitor -n monitoring-local | grep -A 5 "Failed to pull image"

# Verify image name in deployment
kubectl get deployment dependency-monitor -n monitoring-local -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Solutions:**
- **Fix Image Name**:
  ```bash
  # Update to correct image
  kubectl patch deployment dependency-monitor -n monitoring-local -p '{
    "spec": {
      "template": {
        "spec": {
          "containers": [{
            "name": "dependency-monitor",
            "image": "solidity-security-dependency-monitor:latest"
          }]
        }
      }
    }
  }'
  ```

- **Build Image Locally**:
  ```bash
  # Build and load image (for local development)
  cd /Users/pwner/Git/ABS/solidity-security-monitoring/dependency-monitor
  docker build -t solidity-security-dependency-monitor:latest .

  # For minikube
  minikube image load solidity-security-dependency-monitor:latest
  ```

### 2. Service Connectivity Issues

#### Issue: Service Not Accessible via Port Forward
**Symptoms:**
```bash
kubectl port-forward svc/dependency-monitor 8080:80 -n monitoring-local
# Forwarding from 127.0.0.1:8080 -> 80
curl http://localhost:8080/
# curl: (7) Failed to connect to localhost port 8080: Connection refused
```

**Diagnosis:**
```bash
# Check service endpoints
kubectl get endpoints dependency-monitor -n monitoring-local

# Check if pod is ready
kubectl get pods -l app=dependency-monitor -n monitoring-local

# Test service from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -qO- http://dependency-monitor.monitoring-local.svc.cluster.local/
```

**Solutions:**
- **Check Pod Readiness**:
  ```bash
  # Wait for pod to be ready
  kubectl wait --for=condition=ready pod -l app=dependency-monitor -n monitoring-local --timeout=300s

  # Check readiness probe
  kubectl describe pod -l app=dependency-monitor -n monitoring-local | grep -A 5 "Readiness"
  ```

- **Verify Service Configuration**:
  ```bash
  # Check service ports
  kubectl get svc dependency-monitor -n monitoring-local -o yaml

  # Ensure service selector matches pod labels
  kubectl get pods -l app=dependency-monitor -n monitoring-local --show-labels
  ```

#### Issue: API Endpoints Return Errors
**Symptoms:**
```bash
curl http://localhost:8080/
# HTTP 500 Internal Server Error
```

**Diagnosis:**
```bash
# Check application logs
kubectl logs -f deployment/dependency-monitor -n monitoring-local | grep -E "(error|ERROR|exception)"

# Test basic endpoints
curl -v http://localhost:8080/health
curl -v http://localhost:8080/metrics
```

**Solutions:**
- **Check Application Configuration**:
  ```bash
  # Verify environment variables
  kubectl exec deployment/dependency-monitor -n monitoring-local -- env | grep -E "(LOG_|API_|SCAN_)"

  # Check configuration files
  kubectl exec deployment/dependency-monitor -n monitoring-local -- cat /app/config/service_paths.json
  ```

- **Enable Debug Logging**:
  ```bash
  # Set debug log level
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
  ```

### 3. Scanning Issues

#### Issue: Scan Failures for Specific Languages
**Symptoms:**
```bash
curl -X POST http://localhost:8080/scan/api-service
# {"error": "Failed to scan python dependencies", "details": "pip command not found"}
```

**Diagnosis:**
```bash
# Check if scanning tools are available
kubectl exec deployment/dependency-monitor -n monitoring-local -- which pip
kubectl exec deployment/dependency-monitor -n monitoring-local -- which pip-audit
kubectl exec deployment/dependency-monitor -n monitoring-local -- which npm
kubectl exec deployment/dependency-monitor -n monitoring-local -- which cargo

# Test tool execution
kubectl exec deployment/dependency-monitor -n monitoring-local -- pip --version
kubectl exec deployment/dependency-monitor -n monitoring-local -- npm --version
kubectl exec deployment/dependency-monitor -n monitoring-local -- cargo --version
```

**Solutions:**
- **Install Missing Tools**:
  ```bash
  # Install pip-audit in container (temporary fix)
  kubectl exec deployment/dependency-monitor -n monitoring-local -- pip install pip-audit

  # For permanent fix, rebuild container image with tools
  # Update Dockerfile to include:
  # RUN pip install pip-audit safety
  # RUN npm install -g npm-audit-resolver
  # RUN cargo install cargo-audit
  ```

- **Check Tool Configurations**:
  ```bash
  # Test pip-audit
  kubectl exec deployment/dependency-monitor -n monitoring-local -- pip-audit --help

  # Test npm audit
  kubectl exec deployment/dependency-monitor -n monitoring-local -- npm audit --help

  # Test cargo audit
  kubectl exec deployment/dependency-monitor -n monitoring-local -- cargo audit --version
  ```

#### Issue: Repository Access Problems
**Symptoms:**
```bash
curl -X POST http://localhost:8080/scan/api-service
# {"error": "Repository not found", "details": "Path /app/repos/solidity-security-api-service not accessible"}
```

**Diagnosis:**
```bash
# Check repository mounts
kubectl exec deployment/dependency-monitor -n monitoring-local -- ls -la /app/repos/

# Verify specific service path
kubectl exec deployment/dependency-monitor -n monitoring-local -- ls -la /app/repos/solidity-security-api-service/

# Check volume mounts in deployment
kubectl describe deployment dependency-monitor -n monitoring-local | grep -A 10 "Mounts"
```

**Solutions:**
- **Fix Volume Mounts**:
  ```bash
  # For local development, ensure correct host path
  kubectl patch deployment dependency-monitor -n monitoring-local -p '{
    "spec": {
      "template": {
        "spec": {
          "volumes": [{
            "name": "repos",
            "hostPath": {
              "path": "/Users/pwner/Git/ABS",
              "type": "Directory"
            }
          }]
        }
      }
    }
  }'
  ```

- **Update Service Configuration**:
  ```bash
  # Check ConfigMap paths
  kubectl get configmap dependency-monitor-config -n monitoring-local -o jsonpath='{.data.service_paths\.json}' | jq .

  # Update paths if incorrect
  kubectl patch configmap dependency-monitor-config -n monitoring-local -p '{
    "data": {
      "service_paths.json": "{\"api-service\":\"/app/repos/solidity-security-api-service\",\"ui-core\":\"/app/repos/solidity-security-ui-core\"}"
    }
  }'
  ```

### 4. Metrics and Monitoring Issues

#### Issue: Metrics Not Available
**Symptoms:**
```bash
curl http://localhost:8080/metrics
# curl: (7) Failed to connect to localhost port 8080: Connection refused
```

**Diagnosis:**
```bash
# Check if metrics endpoint is enabled
kubectl exec deployment/dependency-monitor -n monitoring-local -- curl -s localhost:8000/metrics | head

# Check application configuration
kubectl exec deployment/dependency-monitor -n monitoring-local -- env | grep METRICS
```

**Solutions:**
- **Verify Metrics Configuration**:
  ```bash
  # Check if metrics port is exposed
  kubectl get svc dependency-monitor -n monitoring-local -o yaml | grep -A 5 ports

  # Update service to expose metrics port if needed
  kubectl patch svc dependency-monitor -n monitoring-local -p '{
    "spec": {
      "ports": [
        {"name": "http", "port": 80, "targetPort": 8000},
        {"name": "metrics", "port": 8000, "targetPort": 8000}
      ]
    }
  }'
  ```

#### Issue: Missing or Incorrect Metrics
**Symptoms:**
```bash
curl -s http://localhost:8080/metrics | grep dependency_
# No output or unexpected metrics
```

**Diagnosis:**
```bash
# Check application logs for metric registration
kubectl logs deployment/dependency-monitor -n monitoring-local | grep -i metric

# Test scan to generate metrics
curl -X POST http://localhost:8080/scan/api-service
sleep 5
curl -s http://localhost:8080/metrics | grep dependency_
```

**Solutions:**
- **Trigger Scans to Generate Metrics**:
  ```bash
  # Run manual scans for all services
  for service in api-service ui-core dashboard findings analysis; do
    echo "Scanning $service..."
    curl -X POST http://localhost:8080/scan/$service
    sleep 2
  done

  # Check metrics after scans
  curl -s http://localhost:8080/metrics | grep -E "(dependency_|service_)"
  ```

### 5. CronJob and Scheduling Issues

#### Issue: CronJobs Not Running
**Symptoms:**
```bash
kubectl get cronjob -n monitoring-local
# NAME                            SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# dependency-scanner              0 6 * * *     False     0        <none>          1h
```

**Diagnosis:**
```bash
# Check CronJob status
kubectl describe cronjob dependency-scanner -n monitoring-local

# Check for recent jobs
kubectl get jobs -n monitoring-local --sort-by=.metadata.creationTimestamp

# Check CronJob events
kubectl get events -n monitoring-local --field-selector involvedObject.kind=CronJob
```

**Solutions:**
- **Manually Trigger CronJob**:
  ```bash
  # Create manual job from CronJob
  kubectl create job --from=cronjob/dependency-scanner manual-scan-$(date +%s) -n monitoring-local

  # Watch job execution
  kubectl get jobs -n monitoring-local -w

  # Check job logs
  kubectl logs job/manual-scan-<timestamp> -n monitoring-local
  ```

- **Fix CronJob Schedule**:
  ```bash
  # Update CronJob schedule for testing (every minute)
  kubectl patch cronjob dependency-scanner -n monitoring-local -p '{
    "spec": {
      "schedule": "* * * * *"
    }
  }'

  # Revert to original schedule after testing
  kubectl patch cronjob dependency-scanner -n monitoring-local -p '{
    "spec": {
      "schedule": "0 6 * * *"
    }
  }'
  ```

## Performance Troubleshooting

### High Memory Usage
**Diagnosis:**
```bash
# Check current memory usage
kubectl top pod -l app=dependency-monitor -n monitoring-local

# Check memory limits
kubectl describe deployment dependency-monitor -n monitoring-local | grep -A 5 Limits
```

**Solutions:**
```bash
# Increase memory limits
kubectl patch deployment dependency-monitor -n monitoring-local -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "dependency-monitor",
          "resources": {
            "limits": {"memory": "2Gi"},
            "requests": {"memory": "1Gi"}
          }
        }]
      }
    }
  }
}'

# Monitor memory usage over time
while true; do
  kubectl top pod -l app=dependency-monitor -n monitoring-local
  sleep 30
done
```

### Slow Scan Performance
**Diagnosis:**
```bash
# Check scan duration metrics
curl -s http://localhost:8080/metrics | grep dependency_scan_duration_seconds

# Monitor scan performance
curl -X POST http://localhost:8080/scan/api-service -w "Total time: %{time_total}s\n"
```

**Solutions:**
```bash
# Increase CPU limits
kubectl patch deployment dependency-monitor -n monitoring-local -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "dependency-monitor",
          "resources": {
            "limits": {"cpu": "1000m"},
            "requests": {"cpu": "500m"}
          }
        }]
      }
    }
  }
}'

# Enable concurrent scanning
kubectl patch deployment dependency-monitor -n monitoring-local -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "dependency-monitor",
          "env": [{"name": "CONCURRENT_SCANS", "value": "5"}]
        }]
      }
    }
  }
}'
```

## Debugging Tools and Techniques

### Log Analysis
```bash
# Follow logs in real-time
kubectl logs -f deployment/dependency-monitor -n monitoring-local

# Filter for specific patterns
kubectl logs deployment/dependency-monitor -n monitoring-local | grep -E "(ERROR|scan|metric)"

# Export logs for analysis
kubectl logs deployment/dependency-monitor -n monitoring-local > dependency-monitor.log

# Search for specific errors
grep -n "Failed to scan" dependency-monitor.log
```

### Interactive Debugging
```bash
# Get shell access to container
kubectl exec -it deployment/dependency-monitor -n monitoring-local -- /bin/bash

# Inside container, test components:
cd /app

# Test Python collector
python -m src.collectors.python_collector --service api-service --debug

# Test metrics export
python -c "from src.metrics.exporter import export_metrics; export_metrics()"

# Test API endpoints
curl localhost:8000/health
curl localhost:8000/services
```

### Network Debugging
```bash
# Test network connectivity from pod
kubectl exec deployment/dependency-monitor -n monitoring-local -- nslookup kubernetes.default

# Test external connectivity (if needed)
kubectl exec deployment/dependency-monitor -n monitoring-local -- ping 8.8.8.8

# Check service mesh connectivity (if applicable)
kubectl exec deployment/dependency-monitor -n monitoring-local -- curl -v http://prometheus.monitoring.svc.cluster.local:9090
```

## Recovery Procedures

### Service Recovery
```bash
# Restart deployment
kubectl rollout restart deployment/dependency-monitor -n monitoring-local

# Scale down and up
kubectl scale deployment dependency-monitor --replicas=0 -n monitoring-local
kubectl scale deployment dependency-monitor --replicas=1 -n monitoring-local

# Delete and recreate problematic pods
kubectl delete pod -l app=dependency-monitor -n monitoring-local
```

### Configuration Recovery
```bash
# Backup current configuration
kubectl get deployment dependency-monitor -n monitoring-local -o yaml > dependency-monitor-backup.yaml

# Reset to original configuration
kubectl apply -k k8s/overlays/local/dependency-monitor/

# Restore from backup if needed
kubectl apply -f dependency-monitor-backup.yaml
```

### Data Recovery
```bash
# Clear cached data (if persistent volumes are used)
kubectl exec deployment/dependency-monitor -n monitoring-local -- rm -rf /app/cache/*

# Reset metrics (restart metrics collection)
kubectl delete pod -l app=dependency-monitor -n monitoring-local

# Trigger fresh scans
curl -X POST http://localhost:8080/scan/all
```

## Prevention and Monitoring

### Health Monitoring
```bash
# Set up monitoring script
cat > monitor-health.sh << 'EOF'
#!/bin/bash
while true; do
  echo "$(date): Checking dependency monitor health..."

  # Check pod status
  POD_STATUS=$(kubectl get pods -l app=dependency-monitor -n monitoring-local -o jsonpath='{.items[0].status.phase}')
  echo "Pod status: $POD_STATUS"

  # Check API health
  if curl -s http://localhost:8080/ > /dev/null; then
    echo "API: Healthy"
  else
    echo "API: Unhealthy"
  fi

  # Check recent metrics
  METRIC_COUNT=$(curl -s http://localhost:8080/metrics | grep -c dependency_)
  echo "Metrics available: $METRIC_COUNT"

  echo "---"
  sleep 300  # Check every 5 minutes
done
EOF

chmod +x monitor-health.sh
./monitor-health.sh
```

### Automated Recovery
```bash
# Create recovery script
cat > auto-recovery.sh << 'EOF'
#!/bin/bash
NAMESPACE="monitoring-local"
DEPLOYMENT="dependency-monitor"

# Check if pod is running
if ! kubectl get pods -l app=$DEPLOYMENT -n $NAMESPACE | grep -q Running; then
  echo "Pod not running, restarting deployment..."
  kubectl rollout restart deployment/$DEPLOYMENT -n $NAMESPACE
fi

# Check if API is responding
if ! curl -s http://localhost:8080/ > /dev/null; then
  echo "API not responding, restarting pod..."
  kubectl delete pod -l app=$DEPLOYMENT -n $NAMESPACE
fi
EOF

chmod +x auto-recovery.sh

# Run periodically via cron
# */10 * * * * /path/to/auto-recovery.sh
```

This troubleshooting guide provides comprehensive solutions for most common issues with the dependency monitoring service. For complex issues not covered here, consider checking the application logs in detail and consulting the service documentation.
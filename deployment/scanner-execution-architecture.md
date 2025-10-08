# Scanner Execution Architecture - Kubernetes Jobs

## Overview

The Tool Integration service executes security scanners (Slither, Mythril, Aderyn, Solidity-Metrics) as isolated Kubernetes Jobs rather than embedded Python packages. This provides production-grade isolation, scalability, and security.

## Architecture Decision

**Decision**: Use Kubernetes Jobs for scanner execution instead of Docker-in-Docker or sidecar containers.

**Date**: October 2025

**Status**: Approved

## Rationale

### Why Kubernetes Jobs?

1. **Security Benefits**
   - No Docker socket mounting required (eliminates privileged access risk)
   - Each scan runs in isolated pod with its own network namespace
   - Pod security policies enforced per scanner
   - Network policies can isolate scanner traffic
   - Principle of least privilege - tool-integration only needs K8s Job creation permissions

2. **Operational Benefits**
   - Native K8s resource management and scheduling
   - Automatic cleanup with TTL (time-to-live) after completion
   - Resource limits per scan prevent resource exhaustion (CPU/memory quotas)
   - Built-in logging and monitoring through K8s
   - Easy to track scan history via Job resources
   - Horizontal scaling handled by K8s scheduler

3. **Scalability**
   - K8s scheduler distributes workload across nodes
   - Can run hundreds of concurrent scans
   - Auto-scaling based on queue depth
   - Resource quotas prevent system overload

### Why NOT alternatives?

**Docker-in-Docker (DinD)**:
- ❌ Security anti-pattern - requires privileged containers
- ❌ Pod has Docker daemon access
- ❌ Hard to track resource usage per scan
- ❌ Potential for resource leaks

**Sidecar Containers**:
- ❌ All scanners always running (waste resources)
- ❌ Doesn't scale with number of scanner types
- ❌ Pre-defined scanners only - no dynamic loading

## Problem Statement

### Dependency Hell

Each security scanner has conflicting Python dependencies:

**Mythril Requirements**:
```
eth-account>=0.8.0
eth-keyfile<0.9.0
hexbytes<0.3.0
ckzg<2
```

**Slither Requirements**:
```
web3<8,>=7.10
  which requires eth-account>=0.13.6
  which requires ckzg>=2.0.0
```

**Conflict**: `ckzg<2` vs `ckzg>=2.0.0`

### Solution

Run each scanner in its own Docker container with isolated dependencies:

```
Scanner Images:
- slither: trailofbits/eth-security-toolbox (includes slither, echidna, manticore)
- mythril: mythril/myth:latest
- aderyn: custom build (Rust-based)
- solidity-metrics: custom build (Node.js-based)
```

## Implementation

### High-Level Flow

```
User submits scan request
         ↓
API service creates scan record
         ↓
Tool-integration service receives scan task
         ↓
Creates Kubernetes Job for scanner
         ↓
K8s schedules Job pod
         ↓
Scanner executes in isolated pod
         ↓
Results written to shared storage or API
         ↓
Job completes and auto-deletes (TTL)
         ↓
Tool-integration aggregates results
```

### Tool-Integration Service Changes

**Add kubernetes Python library**:
```python
# requirements/base.txt
kubernetes>=28.1.0,<29.0.0
```

**Job Creation Logic**:
```python
from kubernetes import client, config

# Load in-cluster config
config.load_incluster_config()

# Create Job for scanner
def create_scanner_job(scan_id: str, scanner: str, contract_source: str):
    batch_v1 = client.BatchV1Api()

    # Create Job spec
    job = client.V1Job(
        metadata=client.V1ObjectMeta(
            name=f"scan-{scanner}-{scan_id}",
            namespace="tool-integration-local",
            labels={
                "app": "scanner",
                "scanner": scanner,
                "scan-id": scan_id
            }
        ),
        spec=client.V1JobSpec(
            ttl_seconds_after_finished=3600,  # Auto-delete after 1 hour
            backoff_limit=3,  # Retry failed jobs
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(
                    labels={
                        "app": "scanner",
                        "scanner": scanner
                    }
                ),
                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name=scanner,
                            image=get_scanner_image(scanner),
                            command=get_scanner_command(scanner, contract_source),
                            resources=client.V1ResourceRequirements(
                                limits={"memory": "2Gi", "cpu": "1000m"},
                                requests={"memory": "512Mi", "cpu": "250m"}
                            ),
                            volume_mounts=[
                                client.V1VolumeMount(
                                    name="contract-storage",
                                    mount_path="/contracts"
                                )
                            ]
                        )
                    ],
                    volumes=[
                        client.V1Volume(
                            name="contract-storage",
                            persistent_volume_claim=client.V1PersistentVolumeClaimVolumeSource(
                                claim_name="contract-pvc"
                            )
                        )
                    ],
                    restart_policy="Never",
                    service_account_name="tool-integration-scanner"
                )
            )
        )
    )

    # Create Job
    batch_v1.create_namespaced_job(
        namespace="tool-integration-local",
        body=job
    )

    return job.metadata.name
```

### Scanner-Specific Configuration

**Mythril**:
```python
def get_scanner_image(scanner):
    images = {
        "mythril": "mythril/myth:latest",
        "slither": "trailofbits/eth-security-toolbox:latest",
        "aderyn": "scanner-aderyn:0.1.0",
        "solidity-metrics": "scanner-solidity-metrics:0.1.0"
    }
    return images[scanner]

def get_scanner_command(scanner, contract_path):
    commands = {
        "mythril": [
            "myth", "analyze",
            f"/contracts/{contract_path}",
            "-o", "json",
            "--execution-timeout", "600"
        ],
        "slither": [
            "slither", f"/contracts/{contract_path}",
            "--json", "/results/slither-output.json"
        ],
        # ... other scanners
    }
    return commands[scanner]
```

### Result Collection

**Option 1: Shared Storage (PVC)**:
```python
# Scanner writes results to /results/output.json
# Tool-integration reads from PVC after Job completion
```

**Option 2: API Callback**:
```python
# Scanner POSTs results to tool-integration API
# No shared storage required
```

**Option 3: K8s ConfigMap/Secret**:
```python
# Scanner writes results to ConfigMap
# Tool-integration reads ConfigMap
# Limited to 1MB data
```

### RBAC Configuration

**ServiceAccount**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tool-integration
  namespace: tool-integration-local
```

**Role**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: job-creator
  namespace: tool-integration-local
rules:
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create", "get", "list", "watch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

**RoleBinding**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tool-integration-job-creator
  namespace: tool-integration-local
subjects:
- kind: ServiceAccount
  name: tool-integration
  namespace: tool-integration-local
roleRef:
  kind: Role
  name: job-creator
  apiGroup: rbac.authorization.k8s.io
```

### Monitoring Job Status

```python
def wait_for_job_completion(job_name: str, timeout: int = 600):
    batch_v1 = client.BatchV1Api()
    start_time = time.time()

    while time.time() - start_time < timeout:
        job = batch_v1.read_namespaced_job(
            name=job_name,
            namespace="tool-integration-local"
        )

        if job.status.succeeded:
            return "completed"
        elif job.status.failed:
            return "failed"

        time.sleep(5)

    return "timeout"
```

### Resource Limits

**Per-Scanner Limits**:
```yaml
Mythril:
  memory: 2Gi (symbolic execution memory-intensive)
  cpu: 1000m
  timeout: 600s

Slither:
  memory: 1Gi
  cpu: 500m
  timeout: 300s

Aderyn:
  memory: 512Mi (Rust efficient)
  cpu: 250m
  timeout: 180s

Solidity-Metrics:
  memory: 512Mi
  cpu: 250m
  timeout: 120s
```

## Production Deployment

### Namespace Organization

```
tool-integration-local (Minikube)
  ├── tool-integration (orchestrator)
  ├── Jobs (ephemeral scanner pods)
  └── PVC (shared contract storage)

tool-integration-staging
  └── Same structure

tool-integration-production
  └── Same structure
```

### Auto-Scaling

**Horizontal Pod Autoscaler for tool-integration**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tool-integration-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tool-integration
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: pending_scans
      target:
        type: AverageValue
        averageValue: "10"
```

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: scanner-quota
  namespace: tool-integration-local
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    count/jobs.batch: "100"
```

## Benefits Summary

### Security
✅ No privileged containers required
✅ Pod-level isolation with network policies
✅ Resource quotas prevent DoS
✅ Audit trail for all scanner executions

### Operations
✅ Native K8s monitoring and logging
✅ Automatic cleanup (TTL)
✅ Easy to debug (kubectl logs)
✅ Standard K8s tooling

### Scalability
✅ Hundreds of concurrent scans
✅ K8s handles scheduling
✅ Auto-scaling based on demand
✅ Resource limits prevent exhaustion

### Maintainability
✅ Scanner updates = image updates
✅ No dependency conflicts
✅ Easy to add new scanners
✅ Versioned scanner images

## Migration Path

### Phase 1: Add Kubernetes client
- Add `kubernetes` library to requirements
- Test in-cluster config loading
- Implement basic Job creation

### Phase 2: Implement scanner Jobs
- Create Job templates for each scanner
- Implement result collection
- Test with single scanner

### Phase 3: RBAC setup
- Create ServiceAccount
- Configure Role and RoleBinding
- Test permissions

### Phase 4: Production deployment
- Deploy to staging
- Load testing
- Deploy to production

## References

- **Kubernetes Jobs Documentation**: https://kubernetes.io/docs/concepts/workloads/controllers/job/
- **Python Kubernetes Client**: https://github.com/kubernetes-client/python
- **Scanner Images**:
  - Slither: https://hub.docker.com/r/trailofbits/eth-security-toolbox
  - Mythril: https://hub.docker.com/r/mythril/myth

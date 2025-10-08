# Monitoring Stack

This document describes the monitoring infrastructure for the Solidity Security Analysis Platform.

## Overview

The monitoring stack uses **kube-prometheus-stack** which includes:
- **Prometheus** - Metrics collection and storage
- **Grafana** - Metrics visualization and dashboards
- **AlertManager** - Alert routing and notification

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Monitoring Stack                         │
│                     (monitoring-local namespace)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐      ┌──────────────┐      ┌─────────────┐  │
│  │  Prometheus  │─────>│   Grafana    │      │ AlertManager│  │
│  │              │      │              │      │             │  │
│  │ Port: 9090   │      │ Port: 3000   │      │ Port: 9093  │  │
│  └──────────────┘      └──────────────┘      └─────────────┘  │
│         │                      │                     │         │
│         │                      │                     │         │
└─────────┼──────────────────────┼─────────────────────┼─────────┘
          │                      │                     │
          │                      │                     │
          v                      v                     v
    [ServiceMonitors]      [Dashboards]          [Alerts]
          │
          │
          v
┌─────────────────────────────────────────────────────────────────┐
│                    Application Services                         │
├─────────────────────────────────────────────────────────────────┤
│  api-service  │  contract-parser  │  data-service  │ ...        │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

The monitoring stack is deployed using Helm:

```bash
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring-local \
  --create-namespace \
  -f k8s/overlays/local/monitoring/values.yaml
```

**Configuration File**: `/Users/pwner/Git/ABS/k8s/overlays/local/monitoring/values.yaml`

## Components

### 1. Prometheus
**Purpose**: Collects and stores time-series metrics from Kubernetes and application services.

**Access**:
```bash
kubectl port-forward -n monitoring-local svc/kube-prometheus-stack-prometheus 9090:9090
```
**URL**: http://localhost:9090

**Features**:
- Automatic service discovery via ServiceMonitors
- 10-day retention period (configurable)
- Built-in query language (PromQL)
- Scrapes metrics from `/metrics` endpoints

### 2. Grafana
**Purpose**: Visualizes metrics with customizable dashboards.

**Access**:
```bash
kubectl port-forward -n monitoring-local svc/kube-prometheus-stack-grafana 3000:80
```
**URL**: http://localhost:3000
**Credentials**: `admin` / `admin`

**Pre-installed Dashboards**:
- Kubernetes cluster monitoring
- Pod resource usage
- Node metrics
- Service metrics

**Custom Dashboards**: Add application-specific dashboards via Grafana UI or ConfigMaps.

### 3. AlertManager
**Purpose**: Handles alerts sent by Prometheus and routes them to notification channels.

**Access**:
```bash
kubectl port-forward -n monitoring-local svc/kube-prometheus-stack-alertmanager 9093:9093
```
**URL**: http://localhost:9093

**Features**:
- Alert grouping and deduplication
- Routing to notification channels (email, Slack, PagerDuty, etc.)
- Silence and inhibit rules

## ServiceMonitors

ServiceMonitors tell Prometheus which services to scrape for metrics. Each microservice has a ServiceMonitor:

**Example**: API Service ServiceMonitor
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-service
  namespace: solidity-security
  labels:
    app: api-service
spec:
  selector:
    matchLabels:
      app: api-service
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

**Location**: `/Users/pwner/Git/ABS/k8s/base/{service}/servicemonitor.yaml`

## Metrics

### Application Metrics
All microservices expose metrics at `/metrics` endpoint in Prometheus format.

**Common Metrics**:
- `http_requests_total` - Total HTTP requests
- `http_request_duration_seconds` - Request latency
- `http_requests_in_progress` - Active requests
- `scanner_jobs_total` - Total scanner jobs created
- `scanner_job_duration_seconds` - Scanner job execution time
- `contract_upload_size_bytes` - Contract file sizes

### Kubernetes Metrics
Automatically collected from Kubernetes API:
- Pod CPU/memory usage
- Node resource utilization
- Container restarts
- PersistentVolume usage

## Alerts

Prometheus evaluates alerting rules and sends alerts to AlertManager.

**Example Alert**: High Pod Memory Usage
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-alerts
  namespace: monitoring-local
spec:
  groups:
  - name: pod.rules
    interval: 30s
    rules:
    - alert: HighPodMemory
      expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} high memory usage"
        description: "Pod is using {{ $value | humanizePercentage }} of memory limit"
```

**Alert Configuration**: `/Users/pwner/Git/ABS/k8s/overlays/local/monitoring/alerts.yaml`

## Quick Start

**Start all monitoring port forwards**:
```bash
cd /Users/pwner/Git/ABS
./scripts/start-monitoring.sh
```

This starts port forwards for:
- Grafana: http://localhost:3000
- Prometheus: http://localhost:9090
- AlertManager: http://localhost:9093

**Stop port forwards**:
```bash
lsof -ti:3000,9090,9093 | xargs kill
```

## Querying Metrics

### Prometheus Query Examples

**CPU usage by pod**:
```promql
rate(container_cpu_usage_seconds_total[5m])
```

**HTTP request rate**:
```promql
rate(http_requests_total[1m])
```

**P95 request latency**:
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Scanner job success rate**:
```promql
rate(scanner_jobs_total{status="completed"}[5m]) / rate(scanner_jobs_total[5m])
```

## Adding Custom Metrics

**Python (FastAPI) Example**:
```python
from prometheus_client import Counter, Histogram, generate_latest

# Define metrics
scan_counter = Counter('scans_total', 'Total scans', ['scanner', 'status'])
scan_duration = Histogram('scan_duration_seconds', 'Scan duration', ['scanner'])

# Instrument code
@app.post("/scans")
async def create_scan():
    with scan_duration.labels(scanner='aderyn').time():
        result = await run_scan()
    scan_counter.labels(scanner='aderyn', status=result.status).inc()
    return result

# Expose metrics endpoint
@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

## Troubleshooting

**Check Prometheus targets**:
```bash
kubectl port-forward -n monitoring-local svc/kube-prometheus-stack-prometheus 9090:9090
# Visit http://localhost:9090/targets
```

**Check ServiceMonitor discovery**:
```bash
kubectl get servicemonitors -A
```

**View Prometheus logs**:
```bash
kubectl logs -n monitoring-local -l app.kubernetes.io/name=prometheus
```

**View Grafana logs**:
```bash
kubectl logs -n monitoring-local -l app.kubernetes.io/name=grafana
```

## Production Recommendations

1. **Persistent Storage**: Configure PersistentVolumes for Prometheus and Grafana
2. **Retention**: Adjust retention period based on storage capacity
3. **Resource Limits**: Set appropriate CPU/memory limits for monitoring components
4. **High Availability**: Run multiple Prometheus and AlertManager replicas
5. **External Storage**: Consider long-term storage solutions (Thanos, Cortex)
6. **Alert Channels**: Configure AlertManager routing to production notification channels
7. **Authentication**: Enable OAuth or LDAP for Grafana in production
8. **TLS**: Enable TLS for all monitoring endpoints

## References

- **Helm Chart**: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- **Prometheus Docs**: https://prometheus.io/docs/
- **Grafana Docs**: https://grafana.com/docs/grafana/latest/
- **AlertManager Docs**: https://prometheus.io/docs/alerting/latest/alertmanager/
- **ServiceMonitor CRD**: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#servicemonitor

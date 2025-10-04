# Dependency Monitoring Service

## Overview

The Dependency Monitoring Service provides automated dependency tracking, security vulnerability scanning, and health monitoring across all SolidityOps platform services. This service supports Python, Node.js, and Rust projects with real-time metrics export to Prometheus and visualization through Grafana dashboards.

## Architecture

### Service Design

```
Dependency Monitor Service
├── API Layer (FastAPI)
│   ├── REST Endpoints
│   ├── Health Checks
│   └── Metrics Export
├── Collectors
│   ├── Python Collector (pip-audit)
│   ├── Node.js Collector (npm audit)
│   └── Rust Collector (cargo audit)
├── Metrics Engine
│   ├── Prometheus Exporter
│   ├── Health Aggregator
│   └── Alert Generator
└── Scheduling
    ├── CronJob Controller
    ├── Manual Triggers
    └── CI/CD Webhooks
```

### Repository Structure

```
solidity-security-monitoring/
├── dependency-monitor/
│   ├── src/
│   │   ├── api/                    # FastAPI REST API
│   │   │   ├── __init__.py
│   │   │   ├── main.py            # API application entry
│   │   │   ├── routes.py          # API route definitions
│   │   │   └── models.py          # Request/response models
│   │   ├── collectors/            # Language-specific scanners
│   │   │   ├── __init__.py
│   │   │   ├── base_collector.py  # Abstract base collector
│   │   │   ├── python_collector.py # Python dependency scanner
│   │   │   ├── nodejs_collector.py # Node.js dependency scanner
│   │   │   └── rust_collector.py   # Rust dependency scanner
│   │   ├── metrics/               # Prometheus metrics
│   │   │   ├── __init__.py
│   │   │   ├── exporter.py        # Metrics export functionality
│   │   │   └── registry.py        # Metric definitions
│   │   ├── config/                # Configuration management
│   │   │   ├── __init__.py
│   │   │   ├── settings.py        # Application settings
│   │   │   └── service_config.py  # Service discovery config
│   │   └── utils/                 # Utility functions
│   │       ├── __init__.py
│   │       ├── logger.py          # Logging configuration
│   │       └── helpers.py         # Helper functions
│   ├── dashboards/                # Grafana dashboards
│   │   ├── dependency-overview.json
│   │   └── security-alerts.json
│   ├── requirements.txt           # Python dependencies
│   ├── Dockerfile                 # Container build
│   └── README.md                  # Service documentation
└── k8s/                          # Kubernetes deployment
    ├── base/dependency-monitor/   # Base manifests
    └── overlays/                  # Environment overlays
```

## Features

### Multi-Language Support

#### Python Projects
- **Package Manager**: pip
- **Security Tools**: pip-audit, safety
- **Dependency Files**: requirements.txt, pyproject.toml, setup.py
- **Vulnerability Sources**: PyPA Advisory Database, OSV
- **Metrics Collected**:
  - Package versions (current vs latest)
  - Security vulnerabilities by severity
  - Dependency tree depth and complexity
  - Update recommendations

#### Node.js Projects
- **Package Manager**: npm
- **Security Tools**: npm audit
- **Dependency Files**: package.json, package-lock.json
- **Vulnerability Sources**: npm Security Advisories, GitHub Advisory Database
- **Metrics Collected**:
  - Package versions and update availability
  - Security vulnerabilities with CVSS scores
  - Dependency graph analysis
  - License compliance

#### Rust Projects
- **Package Manager**: cargo
- **Security Tools**: cargo audit
- **Dependency Files**: Cargo.toml, Cargo.lock
- **Vulnerability Sources**: RustSec Advisory Database
- **Metrics Collected**:
  - Crate versions and compatibility
  - Security advisories and patches
  - Feature flag usage
  - Build dependency tracking

### Security Scanning

#### Vulnerability Detection
- **Critical**: Immediate security threats requiring urgent action
- **High**: Important security updates with potential impact
- **Medium**: Security improvements with moderate risk
- **Low**: General security enhancements
- **CVSS Integration**: Common Vulnerability Scoring System scores

#### Advisory Sources
- **PyPA Advisory Database**: Python security advisories
- **npm Security Advisories**: Node.js vulnerability database
- **RustSec Advisory Database**: Rust security advisories
- **GitHub Advisory Database**: Cross-language vulnerability data
- **OSV (Open Source Vulnerabilities)**: Unified vulnerability database

### Monitoring Capabilities

#### Real-Time Metrics
```prometheus
# Version Information
dependency_current_version_info{service, language, package, version}
dependency_latest_version_info{service, language, package, latest_version}
dependency_versions_behind_total{service, language, package}
dependency_last_updated_days{service, language, package}

# Security Metrics
dependency_vulnerabilities_total{service, language, package, severity}
dependency_vulnerable_packages_total{service, language}
security_advisory_age_days{service, language, package, advisory_id}

# Health Metrics
service_dependency_count_total{service, language}
service_outdated_dependencies_total{service, language}
service_vulnerable_dependencies_total{service, language}

# Performance Metrics
dependency_scan_duration_seconds{service, language}
dependency_scan_success_total{service, language}
dependency_scan_errors_total{service, language, error_type}
```

#### Automated Scanning
- **Daily Full Scan**: Comprehensive dependency analysis (6:00 AM UTC)
- **Security Scan**: Vulnerability-focused scanning (every 4 hours)
- **Quick Scan**: Version check only (every 30 minutes, optional)
- **Manual Triggers**: On-demand scanning via API
- **CI/CD Integration**: Pipeline-triggered scans

## API Reference

### Health Endpoints

#### Service Health
```http
GET /
```
Returns service health status and basic information.

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "uptime": "2d 14h 32m",
  "last_scan": "2024-10-04T12:00:00Z",
  "services_monitored": 12
}
```

#### Prometheus Metrics
```http
GET /metrics
```
Returns Prometheus-formatted metrics for all monitored services.

#### Service Configuration
```http
GET /services
```
Lists all configured services and their monitoring settings.

**Response:**
```json
{
  "services": [
    {
      "name": "api-service",
      "language": "python",
      "path": "/app/repos/solidity-security-api-service",
      "last_scan": "2024-10-04T12:00:00Z",
      "status": "healthy"
    }
  ]
}
```

### Scanning Endpoints

#### Scan Specific Service
```http
POST /scan/{service_name}
```
Triggers dependency scan for a specific service.

**Response:**
```json
{
  "service": "api-service",
  "language": "python",
  "scan_time": "2024-10-04T12:00:00Z",
  "duration": "45.2s",
  "summary": {
    "total_packages": 45,
    "outdated_count": 8,
    "vulnerability_count": 2,
    "critical_vulnerabilities": 0,
    "high_vulnerabilities": 1,
    "medium_vulnerabilities": 1
  },
  "status": "completed"
}
```

#### Scan All Services
```http
POST /scan/all
```
Triggers dependency scan for all configured services.

#### Custom Project Scan
```http
POST /scan/custom
Content-Type: application/json

{
  "service_name": "custom-project",
  "language": "python",
  "project_path": "/path/to/project",
  "scan_options": {
    "include_dev_dependencies": true,
    "vulnerability_only": false
  }
}
```

### Results Endpoints

#### Scan Results
```http
GET /results/{service_name}
```
Retrieves latest scan results for a service.

#### Vulnerability Summary
```http
GET /vulnerabilities
GET /vulnerabilities?severity=critical
GET /vulnerabilities?service=api-service
```
Returns vulnerability summary with optional filtering.

**Response:**
```json
{
  "total_vulnerabilities": 15,
  "by_severity": {
    "critical": 2,
    "high": 5,
    "medium": 6,
    "low": 2
  },
  "by_service": {
    "api-service": 5,
    "ui-core": 3,
    "data-service": 7
  },
  "recent_vulnerabilities": [...]
}
```

## Configuration

### Service Discovery

Services are configured through Kubernetes ConfigMaps:

```yaml
# service_paths.json
{
  "api-service": "/app/repos/solidity-security-api-service",
  "ui-core": "/app/repos/solidity-security-ui-core",
  "dashboard": "/app/repos/solidity-security-dashboard",
  "findings": "/app/repos/solidity-security-findings",
  "analysis": "/app/repos/solidity-security-analysis",
  "tool-integration": "/app/repos/solidity-security-tool-integration",
  "intelligence-engine": "/app/repos/solidity-security-intelligence-engine",
  "orchestration": "/app/repos/solidity-security-orchestration",
  "data-service": "/app/repos/solidity-security-data-service",
  "notification": "/app/repos/solidity-security-notification",
  "contract-parser": "/app/repos/solidity-security-contract-parser",
  "shared": "/app/repos/solidity-security-shared"
}
```

```yaml
# service_languages.json
{
  "api-service": "python",
  "ui-core": "node",
  "dashboard": "node",
  "findings": "node",
  "analysis": "node",
  "tool-integration": "python",
  "intelligence-engine": "python",
  "orchestration": "python",
  "data-service": "python",
  "notification": "node",
  "contract-parser": "rust",
  "shared": "rust"
}
```

### Scan Scheduling

```yaml
# scan_schedule.yaml
daily_full_scan: "0 6 * * *"        # 6:00 AM UTC daily
security_scan: "0 */4 * * *"        # Every 4 hours
quick_scan: "*/30 * * * *"          # Every 30 minutes (optional)
maintenance_window: "0 2 * * 0"     # 2:00 AM UTC Sundays
```

### Application Settings

```python
# settings.py
class Settings:
    # API Configuration
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 8000
    API_PREFIX: str = "/api/v1"

    # Scanning Configuration
    SCAN_TIMEOUT: int = 300  # 5 minutes
    CONCURRENT_SCANS: int = 3
    RETRY_ATTEMPTS: int = 3

    # Metrics Configuration
    METRICS_PATH: str = "/metrics"
    METRICS_PORT: int = 8000

    # Logging Configuration
    LOG_LEVEL: str = "INFO"
    LOG_FORMAT: str = "json"

    # External Services
    PROMETHEUS_URL: str = "http://prometheus:9090"
    GRAFANA_URL: str = "http://grafana:3000"
```

## Deployment

### Kubernetes Deployment

#### Local Environment
```bash
# Deploy to local cluster
kubectl apply -k k8s/overlays/local/dependency-monitor/

# Verify deployment
kubectl get pods -n monitoring-local
kubectl get svc -n monitoring-local

# Check logs
kubectl logs -f deployment/dependency-monitor -n monitoring-local
```

#### Staging Environment
```bash
# Deploy to staging
kubectl apply -k k8s/overlays/staging/dependency-monitor/

# Verify with higher resource allocation
kubectl get deployment dependency-monitor -n monitoring-staging -o yaml
```

#### Production Environment
```bash
# Deploy to production with HA configuration
kubectl apply -k k8s/overlays/production/dependency-monitor/

# Verify production features
kubectl get hpa dependency-monitor -n monitoring-production
kubectl get pdb dependency-monitor -n monitoring-production
kubectl get networkpolicy -n monitoring-production
```

### Docker Deployment

#### Local Development
```bash
# Build the service
cd dependency-monitor
docker build -t dependency-monitor:latest .

# Run with local repositories
docker run -d \
  --name dependency-monitor \
  -p 8000:8000 \
  -v /Users/pwner/Git/ABS:/app/repos:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  dependency-monitor:latest
```

#### Development with Docker Compose
```yaml
# docker-compose.yml
version: '3.8'
services:
  dependency-monitor:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ../:/app/repos:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - LOG_LEVEL=DEBUG
      - SCAN_TIMEOUT=600
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## Monitoring and Alerting

### Grafana Dashboards

#### Dependency Overview Dashboard
- **Total Dependencies**: Count by service and language
- **Outdated Packages**: Services with outdated dependencies
- **Security Status**: Vulnerability counts by severity
- **Scan Performance**: Duration and success rates
- **Update Timeline**: Dependency update history

#### Security Alerts Dashboard
- **Critical Vulnerabilities**: Immediate action required
- **High Priority**: Important security updates
- **Vulnerability Trends**: Security status over time
- **Package Details**: Vulnerable package information
- **Advisory Timeline**: Security advisory publication dates

### Prometheus Alerts

#### Critical Security Alerts
```yaml
- alert: CriticalVulnerabilityDetected
  expr: dependency_vulnerabilities_total{severity="critical"} > 0
  for: 0m
  labels:
    severity: critical
    team: security
  annotations:
    summary: "Critical security vulnerability detected"
    description: "Service {{ $labels.service }} has {{ $value }} critical vulnerabilities"
    runbook_url: "https://docs.example.com/runbooks/critical-vulnerability"
```

#### Dependency Health Alerts
```yaml
- alert: ServiceVulnerabilityThreshold
  expr: dependency_vulnerabilities_total{severity=~"critical|high"} > 5
  for: 1h
  labels:
    severity: warning
    team: development
  annotations:
    summary: "Service has multiple high-severity vulnerabilities"
    description: "{{ $labels.service }} has {{ $value }} high/critical vulnerabilities"

- alert: DependencySignificantlyOutdated
  expr: dependency_versions_behind_total > 5
  for: 24h
  labels:
    severity: warning
    team: development
  annotations:
    summary: "Dependency significantly outdated"
    description: "{{ $labels.service }}/{{ $labels.package }} is {{ $value }} versions behind"
```

#### Operational Alerts
```yaml
- alert: DependencyMonitorDown
  expr: up{job="dependency-monitor"} == 0
  for: 5m
  labels:
    severity: critical
    team: platform
  annotations:
    summary: "Dependency monitor service is down"
    description: "Dependency monitoring service has been down for more than 5 minutes"

- alert: ScanFailureRate
  expr: rate(dependency_scan_errors_total[1h]) > 0.1
  for: 15m
  labels:
    severity: warning
    team: platform
  annotations:
    summary: "High dependency scan failure rate"
    description: "Dependency scan failure rate is {{ $value | humanizePercentage }}"
```

## Integration

### CI/CD Pipeline Integration

#### GitHub Actions
```yaml
name: Dependency Security Check
on: [push, pull_request]

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Trigger Dependency Scan
      run: |
        SCAN_RESULT=$(curl -s -X POST \
          "${{ secrets.DEPENDENCY_MONITOR_URL }}/scan/custom" \
          -H "Content-Type: application/json" \
          -d '{
            "service_name": "${{ github.repository }}",
            "language": "python",
            "project_path": ".",
            "scan_options": {"vulnerability_only": true}
          }')

        CRITICAL_COUNT=$(echo $SCAN_RESULT | jq '.summary.critical_vulnerabilities')
        if [ "$CRITICAL_COUNT" -gt 0 ]; then
          echo "Critical vulnerabilities found: $CRITICAL_COUNT"
          exit 1
        fi

    - name: Upload Scan Results
      uses: actions/upload-artifact@v3
      with:
        name: dependency-scan-results
        path: dependency-scan-results.json
```

#### Pre-commit Integration
```yaml
# .pre-commit-config.yaml
repos:
- repo: local
  hooks:
  - id: dependency-security-check
    name: Check for critical vulnerabilities
    entry: scripts/check-dependencies.sh
    language: script
    pass_filenames: false
    stages: [pre-commit, pre-push]
```

### External Integrations

#### Slack Notifications
```python
# Webhook integration for critical alerts
async def send_slack_alert(vulnerability_data):
    webhook_url = settings.SLACK_WEBHOOK_URL
    message = {
        "text": f"🚨 Critical Vulnerability Alert",
        "attachments": [{
            "color": "danger",
            "fields": [
                {"title": "Service", "value": vulnerability_data["service"], "short": True},
                {"title": "Package", "value": vulnerability_data["package"], "short": True},
                {"title": "Severity", "value": vulnerability_data["severity"], "short": True},
                {"title": "CVSS Score", "value": vulnerability_data["cvss_score"], "short": True}
            ]
        }]
    }
    await send_webhook(webhook_url, message)
```

#### JIRA Integration
```python
# Automatic ticket creation for critical vulnerabilities
async def create_jira_ticket(vulnerability_data):
    ticket_data = {
        "fields": {
            "project": {"key": "SEC"},
            "summary": f"Critical Vulnerability: {vulnerability_data['package']}",
            "description": f"Critical vulnerability detected in {vulnerability_data['service']}",
            "issuetype": {"name": "Bug"},
            "priority": {"name": "Critical"},
            "labels": ["security", "dependency", "critical"]
        }
    }
    await jira_client.create_issue(ticket_data)
```

## Performance and Optimization

### Scanning Performance

#### Optimization Strategies
- **Parallel Scanning**: Concurrent scans across services
- **Incremental Scans**: Only scan changed dependencies
- **Cache Utilization**: Cache dependency metadata
- **Batch Processing**: Group similar scans together

#### Performance Metrics
```prometheus
# Scan duration by service and language
histogram_quantile(0.95, dependency_scan_duration_seconds_bucket)

# Scan throughput
rate(dependency_scan_duration_seconds_count[5m])

# Cache hit rate
dependency_cache_hits_total / dependency_cache_requests_total
```

### Resource Management

#### Memory Optimization
- Stream large dependency files instead of loading in memory
- Implement garbage collection for scan results
- Use memory-efficient data structures

#### CPU Optimization
- Limit concurrent scans based on available CPU
- Implement scan queuing during high load
- Optimize regex patterns for dependency parsing

### Scaling Considerations

#### Horizontal Scaling
```yaml
# HPA configuration for production
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: dependency-monitor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dependency-monitor
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

#### Load Distribution
- Service-based load balancing
- Language-specific worker pools
- Priority queuing for critical services
- Circuit breaker patterns for external dependencies

This comprehensive dependency monitoring service provides the foundation for maintaining secure and up-to-date dependencies across the entire SolidityOps platform with real-time visibility and automated alerting.
# Dependency Monitoring Guide

## Overview

This guide covers the automated dependency monitoring and security scanning system for the SolidityOps platform. The dependency monitoring service provides real-time visibility into dependency health, security vulnerabilities, and update requirements across all platform services.

## Architecture

### Monitoring Service Location
The dependency monitoring service is located in the `solidity-security-monitoring` repository following the standard Kubernetes Kustomize structure:

```
solidity-security-monitoring/
├── dependency-monitor/          # Service source code
│   ├── src/                    # Service implementation
│   │   ├── collectors/         # Language-specific collectors
│   │   ├── exporters/         # Metrics exporters
│   │   └── api/               # REST API endpoints
│   ├── dashboards/            # Grafana dashboard configurations
│   ├── requirements.txt       # Python dependencies
│   ├── Dockerfile            # Container build configuration
│   └── README.md             # Service documentation
└── k8s/                       # Kubernetes infrastructure
    ├── base/
    │   └── dependency-monitor/ # Base Kubernetes manifests
    └── overlays/
        ├── local/
        │   └── dependency-monitor/ # Local development configuration
        ├── staging/
        │   └── dependency-monitor/ # Staging environment configuration
        └── production/
            └── dependency-monitor/ # Production environment configuration
```

### Service Integration
The dependency monitor integrates with the existing monitoring stack:

- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards
- **Kubernetes**: Container orchestration and scheduling
- **Docker**: Containerized service deployment

## Supported Languages and Tools

### Python Services
- **Package Manager**: pip
- **Security Scanner**: pip-audit, safety
- **Dependency Files**: requirements.txt, pyproject.toml
- **Vulnerability Database**: PyPA Advisory Database

**Monitored Services:**
- solidity-security-api-service
- solidity-security-tool-integration
- solidity-security-intelligence-engine
- solidity-security-orchestration
- solidity-security-data-service

### TypeScript/Node.js Services
- **Package Manager**: npm
- **Security Scanner**: npm audit
- **Dependency Files**: package.json, package-lock.json
- **Vulnerability Database**: npm Security Advisories

**Monitored Services:**
- solidity-security-ui-core
- solidity-security-dashboard
- solidity-security-findings
- solidity-security-analysis
- solidity-security-notification

### Rust Services
- **Package Manager**: cargo
- **Security Scanner**: cargo audit
- **Dependency Files**: Cargo.toml, Cargo.lock
- **Vulnerability Database**: RustSec Advisory Database

**Monitored Services:**
- solidity-security-contract-parser
- solidity-security-shared (Rust components)

## Monitoring Capabilities

### Dependency Tracking
The system monitors the following dependency aspects:

1. **Current Versions**: Track installed package versions
2. **Latest Versions**: Check for available updates
3. **Version Gap**: Calculate how many versions behind
4. **Security Vulnerabilities**: Scan for known security issues
5. **Update Age**: Track how long since last update

### Security Scanning
Comprehensive security vulnerability detection:

- **Critical Vulnerabilities**: Immediate security threats
- **High Severity**: Important security updates
- **Medium/Low Severity**: General security improvements
- **CVSS Scoring**: Common Vulnerability Scoring System integration
- **Advisory Integration**: Direct integration with security databases

### Performance Metrics
Service performance and reliability tracking:

- **Scan Duration**: Time taken for dependency scans
- **Scan Success Rate**: Reliability of scanning operations
- **Error Tracking**: Failed scans and error categorization
- **Resource Usage**: Memory and CPU usage during scans

## Prometheus Metrics

### Dependency Version Metrics
```prometheus
# Current version information
dependency_current_version_info{service, language, package_name, current_version}

# Latest available version
dependency_latest_version_info{service, language, package_name, latest_version}

# Versions behind latest
dependency_versions_behind{service, language, package_name}

# Days since last update
dependency_last_updated_days{service, language, package_name}
```

### Security Vulnerability Metrics
```prometheus
# Security vulnerabilities by severity
dependency_security_vulnerabilities_total{service, language, package_name, severity}

# Container base image age
container_base_image_age_days{service, base_image, current_tag}
```

### Service Summary Metrics
```prometheus
# Total dependencies per service
service_dependency_count_total{service, language}

# Outdated dependencies per service
service_outdated_dependencies_total{service, language}

# Total vulnerabilities per service
service_vulnerabilities_total{service, language}
```

### Operational Metrics
```prometheus
# Scan performance
dependency_scan_duration_seconds{service, language}

# Error tracking
dependency_scan_errors_total{service, language, error_type}
```

## Grafana Dashboards

### Dependency Overview Dashboard
**Purpose**: High-level dependency health overview
**Location**: `dependency-monitor/dashboards/grafana-dependency-overview.json`

**Panels Include:**
- Services with outdated dependencies (stat panel)
- Security vulnerabilities by severity (pie chart)
- Total dependencies by service (bar chart)
- Most outdated dependencies (table)
- Dependency update timeline (time series)
- Scan duration by service (time series)
- Scan errors by service (stat panel)

**Key Features:**
- Service and language filtering
- Real-time updates every 5 minutes
- 24-hour time range by default
- Drill-down capabilities to specific services

### Security Alerts Dashboard
**Purpose**: Security vulnerability tracking and alerting
**Location**: `dependency-monitor/dashboards/grafana-security-alerts.json`

**Panels Include:**
- Critical vulnerabilities requiring immediate action (alert list)
- Critical vulnerabilities by service (bar chart)
- High severity vulnerabilities by service (bar chart)
- Vulnerable packages detail (table)
- Container base image vulnerabilities (table)
- Security vulnerability trend (time series)

**Key Features:**
- Severity-based filtering
- Real-time alerts (1-minute refresh)
- 7-day time range for trend analysis
- Color-coded severity indicators

## API Endpoints

### Health and Status
```http
GET /
# Returns service health status

GET /metrics
# Prometheus metrics endpoint

GET /services
# List all configured services and languages
```

### Scanning Operations
```http
POST /scan/{service_name}
# Scan dependencies for specific service

POST /scan/all
# Scan dependencies for all services

POST /scan/custom
# Scan custom project
Content-Type: application/json
{
  "service_name": "custom-service",
  "language": "python",
  "project_path": "/path/to/project"
}
```

### Response Format
```json
{
  "service": "api-service",
  "language": "python",
  "scan_time": "2024-10-04T12:00:00Z",
  "summary": {
    "total_packages": 45,
    "outdated_count": 8,
    "vulnerability_count": 2
  },
  "outdated_packages": [...],
  "security_vulnerabilities": [...],
  "all_dependencies": [...]
}
```

## Automated Scanning

### Scheduled Scans
The system includes automated scanning via Kubernetes CronJobs:

#### Daily Full Scan
- **Schedule**: 6:00 AM daily (UTC)
- **Scope**: All services and dependencies
- **Purpose**: Comprehensive dependency health check
- **CronJob**: `dependency-scanner`

#### Security Vulnerability Scan
- **Schedule**: Every 4 hours
- **Scope**: Priority services (API, tool integration, intelligence engine, data service)
- **Purpose**: Rapid security vulnerability detection
- **CronJob**: `security-vulnerability-scanner`

### Manual Triggers
Scans can be triggered manually through:

1. **API Endpoints**: Direct REST API calls
2. **Kubernetes Jobs**: One-time job execution
3. **CI/CD Integration**: Automated pipeline triggers
4. **Grafana Annotations**: Manual scan initiation

## Alerting Rules

### Critical Security Alerts
```yaml
- alert: CriticalVulnerabilityDetected
  expr: dependency_security_vulnerabilities_total{severity="critical"} > 0
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Critical security vulnerability detected"
    description: "{{ $labels.service }} has {{ $value }} critical vulnerabilities in {{ $labels.package_name }}"
```

### Dependency Health Alerts
```yaml
- alert: DependencyMajorVersionsBehind
  expr: dependency_versions_behind > 2
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Dependency is multiple major versions behind"
    description: "{{ $labels.service }}/{{ $labels.package_name }} is {{ $value }} versions behind"

- alert: DependencyNotUpdatedInMonths
  expr: dependency_last_updated_days > 90
  for: 1d
  labels:
    severity: warning
  annotations:
    summary: "Dependency not updated in 3+ months"
    description: "{{ $labels.service }}/{{ $labels.package_name }} last updated {{ $value }} days ago"
```

### Operational Alerts
```yaml
- alert: DependencyScanFailure
  expr: increase(dependency_scan_errors_total[1h]) > 3
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Multiple dependency scan failures"
    description: "{{ $labels.service }} dependency scans failing: {{ $labels.error_type }}"
```

## Configuration

### Service Configuration
Services are configured in the Kubernetes ConfigMap:

```yaml
# k8s/base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dependency-monitor-config
data:
  service_paths.json: |
    {
      "api-service": "/app/repos/solidity-security-api-service",
      "ui-core": "/app/repos/solidity-security-ui-core",
      ...
    }
  service_languages.json: |
    {
      "api-service": "python",
      "ui-core": "node",
      "contract-parser": "rust",
      ...
    }
```

### Scan Schedules
Customize scanning frequency:

```yaml
scan_schedule.yaml: |
  daily_full_scan: "0 6 * * *"      # 6 AM daily
  security_scan: "0 */4 * * *"      # Every 4 hours
  quick_scan: "*/30 * * * *"        # Every 30 minutes (optional)
```

## Deployment

### Kubernetes Deployment
Deploy the dependency monitoring service:

```bash
# Navigate to monitoring repository
cd /Users/pwner/Git/ABS/solidity-security-monitoring

# Deploy core monitoring infrastructure
kubectl apply -k k8s/overlays/local/

# Deploy dependency monitor for local environment
kubectl apply -k k8s/overlays/local/dependency-monitor/

# For staging environment
kubectl apply -k k8s/overlays/staging/dependency-monitor/

# For production environment
kubectl apply -k k8s/overlays/production/dependency-monitor/
```

### Docker Deployment
For local development and testing:

```bash
# Build the service
cd dependency-monitor
docker build -t solidity-security-dependency-monitor:latest .

# Run with local repositories
docker run -p 8000:8000 \
  -v /Users/pwner/Git/ABS:/app/repos:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  solidity-security-dependency-monitor:latest
```

### Verification
Verify the deployment:

```bash
# Check service status
kubectl get pods -l app=dependency-monitor

# Check service logs
kubectl logs -l app=dependency-monitor

# Test API endpoint
curl http://localhost:8000/services

# Check metrics
curl http://localhost:8000/metrics
```

## Integration with Development Workflow

### CI/CD Integration
Integrate dependency monitoring into development pipelines:

```yaml
# .github/workflows/dependency-check.yml
name: Dependency Security Check
on: [push, pull_request]

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Trigger dependency scan
      run: |
        curl -X POST "${{ secrets.DEPENDENCY_MONITOR_URL }}/scan/custom" \
          -H "Content-Type: application/json" \
          -d '{"service_name": "${{ github.repository }}", "language": "python", "project_path": "."}'
```

### Pre-commit Hooks
Add dependency checking to pre-commit workflows:

```yaml
# .pre-commit-config.yaml
repos:
- repo: local
  hooks:
  - id: dependency-security-check
    name: Dependency Security Check
    entry: scripts/check-dependencies.sh
    language: script
    pass_filenames: false
```

### IDE Integration
Configure IDE alerts for dependency issues:

1. **VS Code Extension**: Custom extension for dependency alerts
2. **IntelliJ Plugin**: Integration with dependency scanning
3. **Vim/Neovim**: Custom functions for dependency checking

## Troubleshooting

### Common Issues

#### Scan Failures
**Symptoms**: High error rates in `dependency_scan_errors_total` metric

**Troubleshooting:**
1. Check service logs: `kubectl logs -l app=dependency-monitor`
2. Verify repository access: Check volume mounts and permissions
3. Validate tool installations: Ensure pip-audit, npm, cargo are available
4. Check network connectivity: Verify access to package registries

#### Missing Metrics
**Symptoms**: Empty Grafana dashboards or missing Prometheus metrics

**Troubleshooting:**
1. Verify Prometheus scraping: Check Prometheus targets page
2. Check service annotations: Ensure metrics endpoint is properly annotated
3. Validate metric names: Ensure metric names match dashboard queries
4. Check time ranges: Verify dashboard time ranges include recent data

#### Performance Issues
**Symptoms**: Long scan durations or timeouts

**Troubleshooting:**
1. Check resource limits: Increase CPU/memory limits in deployment
2. Optimize scan concurrency: Adjust parallel scanning configuration
3. Cache dependencies: Implement local caching for package registries
4. Split large scans: Break large repositories into smaller scan units

### Monitoring Health
Monitor the monitoring service itself:

```prometheus
# Service uptime
up{job="dependency-monitor"}

# Scan completion rate
rate(dependency_scan_duration_seconds_count[5m])

# Error rate
rate(dependency_scan_errors_total[5m])
```

## Security Considerations

### Access Control
- **RBAC**: Minimal Kubernetes permissions
- **Network Policies**: Restricted network access
- **Secret Management**: Secure handling of registry credentials
- **Audit Logging**: Comprehensive audit trails

### Data Security
- **Read-Only Access**: Repository volumes mounted read-only
- **Temporary Storage**: No persistent storage of sensitive data
- **Encrypted Transit**: TLS for all external communications
- **Regular Updates**: Keep scanning tools and base images updated

### Vulnerability Disclosure
- **Responsible Disclosure**: Follow security disclosure protocols
- **Alert Routing**: Route critical alerts to security team
- **Incident Response**: Integration with incident response procedures
- **Compliance**: Meet regulatory and compliance requirements

## Best Practices

### Dependency Management
1. **Regular Updates**: Schedule regular dependency updates
2. **Security First**: Prioritize security updates over feature updates
3. **Testing**: Test dependency updates in isolation before deployment
4. **Documentation**: Document breaking changes and migration paths

### Monitoring Configuration
1. **Baseline Metrics**: Establish baseline dependency health metrics
2. **Threshold Tuning**: Adjust alert thresholds based on service criticality
3. **Dashboard Maintenance**: Regular dashboard updates and improvements
4. **Alert Fatigue**: Balance coverage with alert noise reduction

### Operational Excellence
1. **Runbook Maintenance**: Keep troubleshooting guides updated
2. **Training**: Regular team training on dependency security
3. **Process Automation**: Automate routine dependency management tasks
4. **Continuous Improvement**: Regular review and optimization of monitoring

This comprehensive dependency monitoring system provides the visibility and automation needed to maintain secure and up-to-date dependencies across the entire SolidityOps platform.
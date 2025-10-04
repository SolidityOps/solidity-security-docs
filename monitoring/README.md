# Monitoring & Observability Documentation

This directory contains comprehensive documentation for monitoring and observability services in the SolidityOps platform.

## 📊 Available Documentation

### Core Monitoring Services
- **[Dependency Monitoring Service](dependency-monitoring-service.md)** - Complete dependency tracking and security scanning
- **[Prometheus Configuration](prometheus-configuration.md)** - Metrics collection and alerting setup
- **[Grafana Dashboards](grafana-dashboards.md)** - Visualization and dashboard management
- **[Alert Management](alert-management.md)** - Alerting rules and notification setup

### Deployment Guides
- **[Local Deployment](local-deployment.md)** - Deploy monitoring stack locally
- **[Production Deployment](production-deployment.md)** - Production deployment procedures
- **[Troubleshooting Guide](troubleshooting.md)** - Common issues and solutions

### Integration Guides
- **[CI/CD Integration](ci-cd-integration.md)** - Integrate monitoring with development workflows
- **[Service Integration](service-integration.md)** - Add monitoring to new services
- **[External Integrations](external-integrations.md)** - Slack, Teams, and other integrations

## 🏗️ Architecture Overview

The monitoring infrastructure consists of:

```
Monitoring Stack
├── Dependency Monitor       # Multi-language dependency scanning
├── Prometheus              # Metrics collection and storage
├── Grafana                 # Visualization and dashboards
├── AlertManager            # Alert routing and management
└── Service Discovery       # Automatic service detection
```

## 🚀 Quick Start

1. **Deploy Local Monitoring**:
   ```bash
   kubectl apply -k /Users/pwner/Git/ABS/solidity-security-monitoring/k8s/overlays/local/
   ```

2. **Access Dashboards**:
   - Grafana: `http://localhost:3000`
   - Prometheus: `http://localhost:9090`
   - Dependency Monitor API: `http://localhost:8080`

3. **Verify Health**:
   ```bash
   curl http://localhost:8080/health
   curl http://localhost:8080/metrics
   ```

## 📈 Key Metrics

### Dependency Health
- `dependency_current_version_info` - Current package versions
- `dependency_security_vulnerabilities_total` - Security vulnerabilities by severity
- `service_dependency_count_total` - Total dependencies per service

### Service Performance
- `dependency_scan_duration_seconds` - Scan execution time
- `dependency_scan_errors_total` - Scan failure tracking
- `service_vulnerabilities_total` - Total vulnerabilities per service

### Operational Metrics
- `up` - Service availability
- `process_cpu_seconds_total` - CPU usage
- `process_resident_memory_bytes` - Memory usage

## 🔧 Configuration

### Environment-Specific Settings

#### Local Development
- Minimal resource allocation
- Basic monitoring features
- Single-node deployment

#### Staging
- Full feature set
- Moderate resource allocation
- Production-like configuration

#### Production
- High availability setup
- Auto-scaling enabled
- Enhanced security policies

## 🛡️ Security

### Access Control
- **RBAC**: Role-based access control for all monitoring components
- **Network Policies**: Restricted network access between components
- **TLS**: Encrypted communication for all external connections

### Data Protection
- **Read-Only Access**: Repository scanning with read-only permissions
- **Secret Management**: Secure handling of credentials and API keys
- **Audit Logging**: Comprehensive audit trails for all operations

## 📋 Best Practices

### Monitoring
1. Set up baseline metrics for all services
2. Configure appropriate alert thresholds
3. Regular dashboard maintenance and updates
4. Balance monitoring coverage with alert noise

### Dependency Management
1. Prioritize security updates over feature updates
2. Test dependency updates in isolation
3. Maintain documentation for breaking changes
4. Schedule regular dependency review cycles

### Operations
1. Keep runbooks updated with current procedures
2. Regular team training on monitoring tools
3. Automate routine monitoring tasks
4. Continuous improvement through metrics review

## 🆘 Support

### Getting Help
- **Documentation**: Start with the specific service documentation
- **Troubleshooting**: Check the troubleshooting guide for common issues
- **Logs**: Review service logs for detailed error information
- **Metrics**: Use Prometheus queries to investigate performance issues

### Contributing
- **Documentation Updates**: Keep documentation current with implementation
- **Dashboard Improvements**: Enhance existing dashboards based on usage
- **Alert Tuning**: Adjust alert thresholds based on operational experience
- **Best Practices**: Share operational insights and lessons learned

This monitoring infrastructure provides comprehensive visibility into the health, security, and performance of the entire SolidityOps platform.
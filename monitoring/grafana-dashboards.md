# Grafana Dashboards Setup

This guide covers the setup and configuration of Grafana dashboards for monitoring the SolidityOps platform's dependency and security metrics.

## Overview

The monitoring stack includes pre-built Grafana dashboards that provide real-time visibility into:
- Dependency management across all services
- Security vulnerability tracking
- Service health and scan status

## Dashboard Architecture

### Available Dashboards

1. **Fixed Dependency Overview** - Primary operational dashboard
   - Services with outdated dependencies count
   - Total outdated dependencies across platform
   - Critical vulnerabilities summary
   - Per-service dependency and vulnerability metrics

2. **Test Dependency Dashboard** - Simple verification dashboard
   - Basic dependency metrics display
   - Used for testing data flow

### Data Sources

All dashboards connect to the local Prometheus instance deployed in the monitoring namespace:
- **URL**: `http://prometheus:9090`
- **Type**: Prometheus
- **Access**: Server (default)

## Local Development Setup

### Prerequisites

- Minikube cluster running
- Monitoring stack deployed (`prometheus-local.yaml`)
- Dependency monitor service active

### Quick Start

1. **Deploy monitoring stack**:
   ```bash
   kubectl apply -f prometheus-local.yaml
   ```

2. **Port forward to Grafana**:
   ```bash
   kubectl port-forward -n monitoring svc/grafana 3000:3000
   ```

3. **Access Grafana**:
   - URL: http://localhost:3000
   - Username: `admin`
   - Password: `admin`

4. **Verify data flow**:
   - Navigate to "Fixed Dependency Overview" dashboard
   - Confirm metrics are displaying for all services

### Dashboard Import Process

The dashboards are automatically imported via API during setup. Manual import can be done using:

```bash
# Import dependency overview dashboard
curl -X POST -H "Content-Type: application/json" \
  -d @/path/to/dashboard.json \
  http://admin:admin@localhost:3000/api/dashboards/db
```

## Metrics Overview

### Core Metrics

- `dependency_outdated_total{service="..."}` - Count of outdated dependencies per service
- `vulnerability_critical_total{service="..."}` - Critical vulnerabilities per service
- `dependency_scan_last_success{service="..."}` - Timestamp of last successful scan

### Monitored Services

- api-service
- ui-core
- contract-parser
- intelligence-engine
- notification

## Dashboard Configuration

### Panel Types

1. **Stat Panels** - Single value metrics with thresholds
2. **Bar Gauge** - Horizontal comparison across services
3. **Time Series** - Historical trends (when configured)

### Threshold Configuration

- **Green**: Normal operation (0 issues)
- **Yellow**: Warning levels (1-5 outdated dependencies, 1-3 vulnerabilities)
- **Red**: Critical levels (5+ outdated dependencies, 3+ vulnerabilities)

## Troubleshooting

### No Data Displayed

1. **Verify Prometheus connectivity**:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=dependency_outdated_total"
   ```

2. **Check data source configuration**:
   ```bash
   curl -s -X GET http://admin:admin@localhost:3000/api/datasources
   ```

3. **Test via Grafana proxy**:
   ```bash
   curl -s -X GET "http://admin:admin@localhost:3000/api/datasources/proxy/1/api/v1/query?query=dependency_outdated_total"
   ```

### Dashboard Import Issues

- Ensure dashboard JSON includes explicit datasource UIDs
- Verify Prometheus datasource is configured as default
- Check dashboard panel queries match actual metric names

### Common Fixes

1. **Restart dependency monitor**:
   ```bash
   kubectl rollout restart deployment/dependency-monitor -n monitoring
   ```

2. **Reload Prometheus configuration**:
   ```bash
   kubectl rollout restart deployment/prometheus -n monitoring
   ```

3. **Re-import dashboards** with corrected metric names

## Advanced Configuration

### Custom Dashboards

When creating custom dashboards:

1. Include explicit datasource configuration:
   ```json
   "datasource": {
     "type": "prometheus",
     "uid": "df02w2ynsui9sd"
   }
   ```

2. Use appropriate refresh intervals (30s-5m)
3. Set reasonable time ranges (1h-24h for development)

### Alerting Integration

Future enhancements will include:
- Webhook notifications for critical vulnerabilities
- Slack/Teams integration
- Email alerts for dependency scan failures

## Production Considerations

For production deployment:
- Configure persistent storage for Grafana
- Set up proper authentication (LDAP/OAuth)
- Implement dashboard backup/restore procedures
- Configure high availability for Prometheus/Grafana

## Related Documentation

- [Local Development Setup](local-deployment.md)
- [Dependency Monitoring Service](dependency-monitoring-service.md)
- [Troubleshooting Guide](troubleshooting.md)
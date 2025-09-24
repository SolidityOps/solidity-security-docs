# Service Mesh Dashboards User Guide

This guide covers how to access and use the Istio service mesh dashboards in your local development environment.

## 📊 Available Dashboards

### Kiali - Service Mesh Visualization
**URL**: `https://kiali.local.solidity-security.dev`

Kiali provides a comprehensive view of your service mesh topology, traffic flow, and configuration.

#### Key Features:
- **Service Graph**: Visual topology of all services and their connections
- **Traffic Metrics**: Request rates, success rates, and response times
- **Health Indicators**: Service health status with color-coded alerts
- **Configuration Validation**: Istio configuration errors and warnings
- **Security Visualization**: mTLS status and security policies

#### Navigation:
1. **Graph Tab**: Interactive service topology
   - Use filters to focus on specific namespaces or services
   - Toggle display options (traffic animation, security badges)
   - Click nodes for detailed service information

2. **Applications Tab**: Application-centric view
   - Group services by application labels
   - Monitor application health and performance

3. **Workloads Tab**: Pod-level details
   - Individual workload health and metrics
   - Configuration and deployment information

4. **Services Tab**: Service registry and configuration
   - Service discovery information
   - Port configurations and protocols

5. **Istio Config Tab**: Configuration validation
   - VirtualServices, DestinationRules, Gateways
   - Configuration errors and warnings

### Jaeger - Distributed Tracing
**URL**: `https://jaeger.local.solidity-security.dev`

Jaeger provides detailed request tracing across your microservices architecture.

#### Key Features:
- **Trace Search**: Find specific traces by service, operation, or tags
- **Request Timeline**: Detailed view of request flow through services
- **Performance Analysis**: Identify bottlenecks and latency issues
- **Error Tracking**: Trace failed requests and exceptions
- **Service Dependencies**: Discover service relationships

#### How to Use:
1. **Search Traces**:
   - Select service from dropdown
   - Choose operation (HTTP methods, endpoints)
   - Set time range and limits
   - Add tags for filtering

2. **Analyze Traces**:
   - Click on trace to see detailed timeline
   - Expand spans to see individual service calls
   - Look for red spans indicating errors
   - Check span duration for performance issues

3. **Service Dependencies**:
   - Navigate to Dependencies tab
   - View service call patterns
   - Identify critical service paths

## 🔧 Access Configuration

### Prerequisites
- Local minikube cluster running
- ArgoCD applications deployed
- Service mesh components installed

### DNS Setup
Add these entries to your `/etc/hosts` file:
```bash
127.0.0.1 kiali.local.solidity-security.dev
127.0.0.1 jaeger.local.solidity-security.dev
127.0.0.1 grafana.local.solidity-security.dev
127.0.0.1 prometheus.local.solidity-security.dev
127.0.0.1 argocd.local.solidity-security.dev
```

### Port Forwarding (Alternative Access)
If DNS resolution issues occur, use port forwarding:

```bash
# Kiali Dashboard
kubectl port-forward -n kiali-system svc/kiali 20001:20001

# Jaeger UI
kubectl port-forward -n jaeger-system svc/jaeger-query 16686:16686
```

Then access via:
- Kiali: `http://localhost:20001`
- Jaeger: `http://localhost:16686`

## 📈 Monitoring Service Mesh Health

### Using Kiali for Health Monitoring

1. **Service Health Overview**:
   - Green: Healthy services
   - Yellow: Degraded performance
   - Red: Service failures or high error rates

2. **Traffic Patterns**:
   - Thick arrows: High traffic volume
   - Dotted lines: Failed requests
   - Animation speed: Request frequency

3. **Configuration Issues**:
   - Orange warnings: Configuration problems
   - Red errors: Invalid configurations

### Using Jaeger for Performance Analysis

1. **Identify Slow Requests**:
   - Sort traces by duration
   - Look for traces with long total time
   - Drill down to find bottleneck services

2. **Error Analysis**:
   - Filter by error tags
   - Check error spans for exception details
   - Correlate errors with service deployments

3. **Service Performance**:
   - Compare response times across services
   - Identify services with high latency
   - Monitor performance trends over time

## 🚨 Common Issues and Solutions

### Kiali Access Issues
```bash
# Check Kiali pod status
kubectl get pods -n kiali-system

# Check service and ingress
kubectl get svc,ingress -n kiali-system

# View Kiali logs
kubectl logs -n kiali-system deployment/kiali
```

### Jaeger Tracing Issues
```bash
# Check Jaeger components
kubectl get pods -n jaeger-system

# Verify trace collection
kubectl logs -n jaeger-system deployment/jaeger

# Check Istio tracing configuration
kubectl get telemetry -n istio-system -o yaml
```

### No Traces Appearing
1. Verify Istio sidecar injection:
   ```bash
   kubectl get pods -o wide | grep "2/2"
   ```

2. Check telemetry configuration:
   ```bash
   kubectl get telemetry -A
   ```

3. Generate test traffic:
   ```bash
   curl -H "Host: grafana.local.solidity-security.dev" http://localhost/
   ```

### Missing Service Dependencies
1. Ensure services are in mesh:
   ```bash
   kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
   ```

2. Check namespace labels:
   ```bash
   kubectl get namespaces --show-labels
   ```

## 🔗 Integration with Other Tools

### Grafana Integration
- Kiali links to Grafana for detailed metrics
- Use Istio dashboards in Grafana for additional insights
- Correlate Kiali topology with Grafana time-series data

### Prometheus Integration
- Kiali uses Prometheus for service metrics
- Custom queries available in Kiali's metrics tab
- Service-level SLIs and alerts

### ArgoCD Integration
- Monitor application deployment status
- Correlate configuration changes with service issues
- Track service mesh configuration drift

## 📚 Additional Resources

- [Kiali Documentation](https://kiali.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Istio Observability](https://istio.io/latest/docs/tasks/observability/)
- [Service Mesh Troubleshooting Guide](./service-mesh-troubleshooting.md)
# Service Mesh Troubleshooting Guide

This guide helps diagnose and resolve common issues with the Istio service mesh, Jaeger tracing, and Kiali observability stack.

## 🚨 Common Issues and Solutions

### 1. Service Mesh Not Starting

#### Symptoms
- Pods stuck in `Pending` or `CrashLoopBackOff`
- ArgoCD applications showing `OutOfSync` or `Degraded`

#### Diagnosis
```bash
# Check ArgoCD application status
kubectl get applications -n argocd

# Check pod status across service mesh namespaces
kubectl get pods -n istio-system
kubectl get pods -n jaeger-system
kubectl get pods -n kiali-system

# Check events for issues
kubectl get events --sort-by='.lastTimestamp' -n istio-system
```

#### Solutions
1. **Resource Constraints**:
   ```bash
   # Check node resources
   kubectl describe nodes

   # Reduce resource requests if needed
   kubectl edit deployment istiod -n istio-system
   ```

2. **CRD Installation Issues**:
   ```bash
   # Verify CRDs are installed
   kubectl get crd | grep -E "(istio|jaeger|kiali)"

   # Reinstall CRDs if missing
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/1.20.1/manifests/charts/base/crds/crd-all.gen.yaml
   ```

3. **Webhook Issues**:
   ```bash
   # Check webhook configuration
   kubectl get validatingwebhookconfiguration
   kubectl get mutatingwebhookconfiguration

   # Delete problematic webhooks
   kubectl delete validatingwebhookconfiguration istio-validator-istio-system
   ```

### 2. Sidecar Injection Not Working

#### Symptoms
- Pods showing `1/1` instead of `2/2` containers
- No Envoy proxy in application pods

#### Diagnosis
```bash
# Check namespace labels
kubectl get namespaces --show-labels | grep istio-injection

# Check sidecar injector status
kubectl get pods -n istio-system -l app=istiod

# Verify webhook configuration
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml
```

#### Solutions
1. **Missing Namespace Label**:
   ```bash
   # Add injection label to namespace
   kubectl label namespace your-namespace istio-injection=enabled

   # Restart pods to trigger injection
   kubectl rollout restart deployment/your-app -n your-namespace
   ```

2. **Admission Controller Issues**:
   ```bash
   # Check admission controller logs
   kubectl logs -n istio-system -l app=istiod

   # Restart istiod
   kubectl rollout restart deployment/istiod -n istio-system
   ```

3. **Pod Annotations Override**:
   ```bash
   # Check for sidecar.istio.io/inject annotation
   kubectl get pod your-pod -o yaml | grep -A5 -B5 sidecar.istio.io

   # Remove problematic annotations
   kubectl annotate pod your-pod sidecar.istio.io/inject-
   ```

### 3. mTLS Connection Issues

#### Symptoms
- Services returning 503 errors
- Connection refused between services
- Certificate verification failures

#### Diagnosis
```bash
# Check PeerAuthentication policies
kubectl get peerauthentication -A

# Check DestinationRules
kubectl get destinationrule -A

# Test service connectivity from pod
kubectl exec -it your-pod -c your-container -- curl -v http://target-service
```

#### Solutions
1. **mTLS Mode Mismatch**:
   ```bash
   # Check current mTLS status in Kiali
   # Or verify with istioctl
   istioctl authn tls-check your-pod.your-namespace.svc.cluster.local

   # Fix DestinationRule mTLS mode
   kubectl edit destinationrule your-rule
   ```

2. **Certificate Issues**:
   ```bash
   # Check Istio CA certificates
   kubectl get secret -n istio-system | grep cacerts

   # Restart istiod to refresh certificates
   kubectl rollout restart deployment/istiod -n istio-system
   ```

3. **Policy Conflicts**:
   ```bash
   # List all security policies
   kubectl get peerauthentication,authorizationpolicy -A

   # Check for conflicting policies
   kubectl describe authorizationpolicy -A
   ```

### 4. Jaeger Traces Not Appearing

#### Symptoms
- Empty traces in Jaeger UI
- Services not appearing in dependency graph
- Spans missing from request traces

#### Diagnosis
```bash
# Check Jaeger components
kubectl get pods -n jaeger-system

# Verify telemetry configuration
kubectl get telemetry -n istio-system -o yaml

# Check Jaeger service endpoints
kubectl get svc -n jaeger-system
```

#### Solutions
1. **Telemetry Configuration**:
   ```bash
   # Check Istio tracing configuration
   kubectl get telemetry default-tracing -n istio-system -o yaml

   # Verify sampling rate
   kubectl get telemetry jaeger-sampling -n istio-system -o yaml
   ```

2. **Jaeger Connectivity**:
   ```bash
   # Test Jaeger collector endpoint
   kubectl exec -it -n istio-system deployment/istiod -- curl -v http://jaeger-collector.jaeger-system.svc.cluster.local:14250/api/traces

   # Check if traces are being generated
   kubectl logs -n jaeger-system deployment/jaeger -c jaeger
   ```

3. **Sidecar Configuration**:
   ```bash
   # Check Envoy proxy configuration
   kubectl exec your-pod -c istio-proxy -- pilot-agent request GET config_dump

   # Verify tracing configuration in Envoy
   kubectl exec your-pod -c istio-proxy -- curl localhost:15000/config_dump | grep -A10 -B10 tracing
   ```

### 5. Kiali Dashboard Issues

#### Symptoms
- Kiali UI not loading
- Empty service graph
- Missing metrics or connectivity info

#### Diagnosis
```bash
# Check Kiali pod status
kubectl get pods -n kiali-system

# Verify Kiali configuration
kubectl get kiali kiali -n kiali-system -o yaml

# Check external service connections
kubectl exec -n kiali-system deployment/kiali -- curl -v http://prometheus-server.prometheus-local.svc.cluster.local:9090/api/v1/query?query=up
```

#### Solutions
1. **Kiali Configuration**:
   ```bash
   # Check Kiali logs for configuration errors
   kubectl logs -n kiali-system deployment/kiali

   # Verify external services configuration
   kubectl get kiali kiali -n kiali-system -o jsonpath='{.spec.external_services}'
   ```

2. **Prometheus Connectivity**:
   ```bash
   # Test Prometheus connection from Kiali
   kubectl exec -n kiali-system deployment/kiali -- curl http://prometheus-server.prometheus-local.svc.cluster.local:9090/api/v1/label/__name__/values

   # Check Prometheus targets
   kubectl port-forward -n prometheus-local svc/prometheus-server 9090:9090
   # Visit http://localhost:9090/targets
   ```

3. **Grafana Integration**:
   ```bash
   # Test Grafana connection
   kubectl exec -n kiali-system deployment/kiali -- curl http://grafana.grafana-local.svc.cluster.local:3000/api/health

   # Check Grafana service
   kubectl get svc -n grafana-local
   ```

### 6. Gateway and Ingress Issues

#### Symptoms
- External URLs not accessible
- SSL/TLS certificate errors
- 404 errors for configured routes

#### Diagnosis
```bash
# Check gateway status
kubectl get gateway -A

# Check virtual services
kubectl get virtualservice -A

# Check ingress gateway pods
kubectl get pods -n istio-system -l app=istio-ingressgateway

# Test internal service connectivity
kubectl exec -it -n istio-system deployment/istio-ingressgateway -- curl -v http://kiali.kiali-system.svc.cluster.local:20001
```

#### Solutions
1. **Certificate Issues**:
   ```bash
   # Check certificate status
   kubectl get certificate -A
   kubectl describe certificate solidity-security-local-tls -n istio-system

   # Check cert-manager logs
   kubectl logs -n cert-manager deployment/cert-manager

   # Manually trigger certificate renewal
   kubectl delete certificate solidity-security-local-tls -n istio-system
   ```

2. **DNS Resolution**:
   ```bash
   # Add to /etc/hosts
   echo "127.0.0.1 kiali.local.solidity-security.dev" >> /etc/hosts
   echo "127.0.0.1 jaeger.local.solidity-security.dev" >> /etc/hosts

   # Test DNS resolution
   nslookup kiali.local.solidity-security.dev
   ```

3. **Gateway Configuration**:
   ```bash
   # Check gateway listeners
   kubectl get gateway solidity-security-gateway -n istio-system -o yaml

   # Verify virtual service routing
   kubectl get virtualservice -A -o yaml | grep -A10 -B10 hosts
   ```

## 🔧 Diagnostic Commands

### Service Mesh Health Check
```bash
# Overall cluster status
kubectl get pods -A | grep -E "(istio|jaeger|kiali)"

# Check service mesh configuration
kubectl get gateway,virtualservice,destinationrule,peerauthentication,authorizationpolicy -A

# Verify sidecar injection
kubectl get pods -A -o wide | awk '{print $1, $2, $3, $4}' | grep -E "2/2|1/1"
```

### Network Connectivity Test
```bash
# Test from application pod
kubectl exec -it your-pod -c your-container -- \
  curl -v -H "Host: target-service.target-namespace.svc.cluster.local" \
  http://target-service.target-namespace.svc.cluster.local:port/endpoint

# Test through Istio gateway
curl -v -H "Host: kiali.local.solidity-security.dev" https://localhost/
```

### Certificate Validation
```bash
# Check TLS certificate
openssl s_client -connect kiali.local.solidity-security.dev:443 -servername kiali.local.solidity-security.dev

# Verify cert-manager certificates
kubectl get certificate -A
kubectl describe certificate solidity-security-local-tls -n istio-system
```

## 🔍 Log Analysis

### Key Log Locations
```bash
# Istiod logs
kubectl logs -n istio-system deployment/istiod -f

# Envoy proxy logs
kubectl logs your-pod -c istio-proxy -f

# Jaeger logs
kubectl logs -n jaeger-system deployment/jaeger -c jaeger -f

# Kiali logs
kubectl logs -n kiali-system deployment/kiali -f

# Gateway logs
kubectl logs -n istio-system deployment/istio-ingressgateway -f
```

### Log Analysis Tips
1. **Look for Error Patterns**:
   - Certificate validation failures
   - Connection refused errors
   - Configuration validation errors
   - Webhook admission failures

2. **Common Error Messages**:
   - `upstream connect error or disconnect/reset before headers`
   - `no healthy upstream`
   - `certificate verify failed`
   - `webhook admission failed`

## 🚀 Performance Issues

### High Memory Usage
```bash
# Check resource usage
kubectl top pods -A | grep -E "(istio|jaeger|kiali)"

# Adjust resource limits
kubectl edit deployment istiod -n istio-system

# Monitor memory trends
kubectl logs -n istio-system deployment/istiod | grep -i "memory\|oom"
```

### Slow Response Times
```bash
# Check Envoy admin interface
kubectl exec your-pod -c istio-proxy -- curl localhost:15000/stats | grep -i timeout

# Analyze circuit breaker stats
kubectl exec your-pod -c istio-proxy -- curl localhost:15000/stats | grep -i outlier

# Check connection pool settings
kubectl get destinationrule -A -o yaml | grep -A20 connectionPool
```

## 📞 Emergency Procedures

### Complete Service Mesh Removal
```bash
# Remove all Istio resources
kubectl delete gateway,virtualservice,destinationrule,peerauthentication,authorizationpolicy -A --all

# Remove Istio control plane
kubectl delete namespace istio-system

# Remove sidecar injection labels
kubectl label namespace your-namespace istio-injection-

# Restart all application pods
kubectl rollout restart deployment -A
```

### Bypass Service Mesh
```bash
# Disable sidecar injection temporarily
kubectl label namespace your-namespace istio-injection=disabled --overwrite

# Remove existing sidecars
kubectl rollout restart deployment/your-app -n your-namespace

# Use direct service communication
kubectl patch service your-service -p '{"spec":{"type":"LoadBalancer"}}'
```

## 📚 Additional Resources

- [Istio Troubleshooting](https://istio.io/latest/docs/ops/common-problems/)
- [Jaeger Troubleshooting](https://www.jaegertracing.io/docs/1.52/troubleshooting/)
- [Kiali Troubleshooting](https://kiali.io/docs/troubleshooting/)
- [Service Mesh Architecture Documentation](../architecture/local/service-mesh-architecture.md)
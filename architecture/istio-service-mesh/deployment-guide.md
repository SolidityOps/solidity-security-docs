# Istio Service Mesh Deployment Guide
## Local Minikube Setup for Solidity Security Platform

This guide focuses specifically on setting up a local Kubernetes cluster with minikube and installing Istio service mesh for Sprint 1 development.

## Prerequisites

- Docker installed and running
- minikube installed
- kubectl installed
- Git repositories cloned locally

## Repository Structure Overview

```
Development Workspace/
├── solidity-security-infrastructure/    # Platform infrastructure
│   └── service-mesh/istio/              # Core Istio installation
└── solidity-security-platform/         # Application services (for future)
```

---

## Phase 1: Local Kubernetes Cluster Setup

### Step 1: Infrastructure Repository Structure

Create the following directory structure in `solidity-security-infrastructure` https://github.com/SolidityOps/solidity-security-infrastructure :

```
solidity-security-infrastructure/
├── service-mesh/
│   └── istio/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   └── istio-operator.yaml
│       ├── components/
│       │   ├── gateways/
│       │   │   ├── kustomization.yaml
│       │   │   ├── platform-gateway.yaml
│       │   │   └── monitoring-gateway.yaml
│       │   ├── security/
│       │   │   ├── kustomization.yaml
│       │   │   ├── mesh-policy.yaml
│       │   │   ├── authorization-policies.yaml
│       │   │   ├── peerauthentication.yaml
│       │   │   ├── request-authentication.yaml
│       │   │   ├── network-policies.yaml
│       │   │   └── security-standards.yaml
│       │   └── observability/
│       │       ├── kustomization.yaml
│       │       ├── telemetry.yaml
│       │       └── tracing.yaml
│       └── overlays/
│           └── local/
│               ├── kustomization.yaml
│               ├── istio-config-patch.yaml
│               └── gateway-patch.yaml
└── scripts/
    ├── validate-istio.sh
    └── comprehensive-validation.sh
```

### Step 2: Required Configuration Files

**Base Istio Files:**
- `service-mesh/istio/base/kustomization.yaml` - Base kustomization configuration
- `service-mesh/istio/base/namespace.yaml` - Istio system namespace
- `service-mesh/istio/base/istio-operator.yaml` - IstioOperator configuration

**Gateway Components:**
- `service-mesh/istio/components/gateways/kustomization.yaml` - Gateway component config
- `service-mesh/istio/components/gateways/platform-gateway.yaml` - Main platform gateway
- `service-mesh/istio/components/gateways/monitoring-gateway.yaml` - Monitoring services gateway

**Security Components:**
- `service-mesh/istio/components/security/kustomization.yaml` - Security component config
- `service-mesh/istio/components/security/mesh-policy.yaml` - Global mTLS policy
- `service-mesh/istio/components/security/authorization-policies.yaml` - Platform auth policies
- `service-mesh/istio/components/security/peerauthentication.yaml` - Workload-specific mTLS
- `service-mesh/istio/components/security/request-authentication.yaml` - JWT validation
- `service-mesh/istio/components/security/network-policies.yaml` - Kubernetes network policies
- `service-mesh/istio/components/security/security-standards.yaml` - Security baselines and policies

**Observability Components:**
- `service-mesh/istio/components/observability/kustomization.yaml` - Observability config
- `service-mesh/istio/components/observability/telemetry.yaml` - Metrics and logging
- `service-mesh/istio/components/observability/tracing.yaml` - Distributed tracing

**Local Environment Overlay:**
- `service-mesh/istio/overlays/local/kustomization.yaml` - Local environment composition
- `service-mesh/istio/overlays/local/istio-config-patch.yaml` - Local-specific Istio config
- `service-mesh/istio/overlays/local/gateway-patch.yaml` - Local gateway configuration

---

## Phase 2: Deployment Scripts

### Step 3: Required Scripts

**Validation Scripts:**
- `scripts/validate-istio.sh` - Basic Istio installation validation
- `scripts/comprehensive-validation.sh` - Complete validation with detailed checks

---

## Phase 3: Istio Deployment Process

### Step 4: Install Istio Operator

**Core Installation Verification:**
```bash
# Install Istio operator (manual command)
istioctl operator init
```

### Step 5: Deploy Istio Configuration with Kustomize

```bash
# Deploy Istio using kustomize
cd solidity-security-infrastructure
kustomize build service-mesh/istio/overlays/local | kubectl apply -f -

# Wait for Istio control plane to be ready
kubectl wait --for=condition=Ready pod -l app=istiod -n istio-system --timeout=300s

# Wait for ingress gateway to be ready
kubectl wait --for=condition=available --timeout=300s deployment/istio-ingressgateway -n istio-system
```

### Step 6: Validation and Verification

```bash
# Execute validation
chmod +x scripts/validate-istio.sh
chmod +x scripts/comprehensive-validation.sh

# Basic validation
./scripts/validate-istio.sh

# Comprehensive validation
./scripts/comprehensive-validation.sh
```

---

## Phase 4: Validation Framework

### Step 7: Validation Components

**Basic Validation Checks:**
- Istio operator status
- Control plane (istiod) readiness
- Ingress gateway availability
- Configuration analysis via istioctl
- Proxy status verification

**Comprehensive Validation Checks:**
- Prerequisites verification (tools, cluster connectivity)
- Istio installation status (operator, control plane, gateway)
- Configuration validation (istioctl analyze, proxy status)
- Resource validation (gateways, virtual services, destination rules)
- Security validation (mTLS, authorization policies)
- Namespace validation (injection labels, pod sidecars)
- Connectivity validation (ingress gateway, service discovery)
- DNS validation (local development DNS configuration)

### Step 8: Security Validation Commands

**Comprehensive Security Validation:**
```bash
# Verify mTLS is enabled mesh-wide
istioctl authn tls-check

# Check all peer authentication policies
kubectl get peerauthentication -A

# Verify authorization policies are applied
kubectl get authorizationpolicy -A

# Check request authentication (JWT validation)
kubectl get requestauthentication -A

# Verify network policies are enforcing traffic rules
kubectl get networkpolicy -A

# Check security standards compliance
kubectl get all -n istio-system -o yaml | grep -E "(securityContext|runAsNonRoot|readOnlyRootFilesystem)"

# Validate no workloads bypass security policies
istioctl proxy-config cluster <pod-name> -n istio-system | grep -E "(sds|tls)"
```

**Security Policy Testing:**
```bash
# Test mTLS enforcement between services
kubectl exec -n solidity-platform <source-pod> -- curl -k https://<target-service>:8443/health

# Verify unauthorized access is blocked
kubectl exec -n default <test-pod> -- curl <service-in-solidity-platform>

# Check certificate rotation
kubectl get secret -n istio-system | grep "istio.io/key-and-cert"

# Validate RBAC is working
kubectl auth can-i get secrets --as=system:serviceaccount:solidity-platform:default -n solidity-platform
```

### Step 9: Manual Validation Commands
```bash
# Verify cluster
kubectl cluster-info
kubectl get nodes

# Verify Istio operator
kubectl get pods -n istio-operator
kubectl get deployment istio-operator -n istio-operator

# Verify Istio control plane
kubectl get pods -n istio-system
kubectl wait --for=condition=Ready pod -l app=istiod -n istio-system --timeout=300s

# Check Istio version
istioctl version

# Verify ingress gateway
kubectl get service istio-ingressgateway -n istio-system
kubectl get endpoints istio-ingressgateway -n istio-system
```

**Configuration Validation:**
```bash
# Analyze Istio configuration
istioctl analyze -A

# Validate configurations
kustomize build service-mesh/istio/overlays/local | istioctl validate -f -

# Check proxy status
istioctl proxy-status

# Verify namespace injection
kubectl get namespace -L istio-injection
```

**Connectivity Testing:**
```bash
# Test DNS resolution
nslookup istiod.istio-system.svc.cluster.local

# Check ingress gateway access
kubectl get service istio-ingressgateway -n istio-system

# For minikube, get access URL
minikube service istio-ingressgateway -n istio-system --url
```

---

## Phase 5: Troubleshooting Guide

### Step 10: Security Standards Enforcement

**Default Security Policies Applied:**

1. **Mesh-wide mTLS (STRICT mode)** - All service-to-service communication encrypted
2. **Deny-by-default authorization** - Explicit allow rules required for access
3. **JWT validation** - Request authentication for external API access
4. **Network policies** - Kubernetes-level traffic isolation
5. **Security baselines** - Pod security standards and resource limits
6. **Certificate rotation** - Automatic certificate lifecycle management
7. **RBAC enforcement** - Role-based access control for Kubernetes resources

**Security Validation Framework:**
- Automated security policy validation in validation scripts
- mTLS enforcement verification
- Authorization policy testing
- Certificate validation checks
- Network isolation verification

### Step 11: Common Issues and Solutions

**Troubleshooting Guide:**

**Istio Installation Issues:**
- Operator not ready - check operator logs: `kubectl logs -n istio-operator deployment/istio-operator`
- Control plane startup - verify resource constraints and node capacity
- Gateway not accessible - check service type and port configuration

**Configuration Issues:**
- Failed validation - review istioctl analyze output
- Proxy not injecting - verify namespace injection labels
- Certificate issues - check CA certificate generation
- mTLS connection failures - verify peer authentication policies
- Authorization denied - check authorization policy rules
- JWT validation failures - verify request authentication configuration

### Step 12: Verification Checklist

**✅ Kubernetes Cluster:**
- [ ] Local Kubernetes cluster running (minikube or other)
- [ ] kubectl context set correctly
- [ ] Cluster has sufficient resources
- [ ] Required permissions for istio installation

**✅ Istio Installation:**
- [ ] Istio operator deployed and ready
- [ ] Istiod control plane running
- [ ] Ingress gateway deployed and accessible
- [ ] No configuration errors from istioctl analyze

**✅ Network Configuration:**
- [ ] Platform gateway created
- [ ] Monitoring gateway created
- [ ] Local DNS entries configured (if needed)
- [ ] Ingress gateway has external access

**✅ Security Configuration:**
- [ ] mTLS enabled mesh-wide (STRICT mode)
- [ ] Deny-by-default authorization policies applied
- [ ] JWT validation configured for external access
- [ ] Network policies enforcing traffic isolation
- [ ] Security baselines applied to all workloads
- [ ] Certificate rotation configured and working
- [ ] RBAC policies applied and tested

**✅ Observability:**
- [ ] Telemetry configuration applied
- [ ] Tracing configuration enabled
- [ ] Metrics collection configured

---

## Phase 6: Next Steps Preparation

### Step 11: Platform Repository Preparation

For future Sprint 1 development, prepare the platform repository structure https://github.com/SolidityOps/solidity-security-platform :

```
solidity-security-platform/
├── services/
│   ├── api-service/
│   │   ├── base/
│   │   ├── components/
│   │   │   └── istio/
│   │   └── overlays/
│   │       └── local/
│   ├── tool-integration-service/
│   │   ├── base/
│   │   ├── components/
│   │   │   └── istio/
│   │   └── overlays/
│   │       └── local/
│   ├── data-service/
│   │   ├── base/
│   │   ├── components/
│   │   │   └── istio/
│   │   └── overlays/
│   │       └── local/
│   ├── orchestration-service/
│   │   ├── base/
│   │   ├── components/
│   │   │   └── istio/
│   │   └── overlays/
│   │       └── local/
│   ├── intelligence-engine/
│   │   ├── base/
│   │   ├── components/
│   │   │   └── istio/
│   │   └── overlays/
│   │       └── local/
│   └── notification-service/
│       ├── base/
│       ├── components/
│       │   └── istio/
│       └── overlays/
│           └── local/
└── scripts/
    └── deploy-services.sh
```

**Each service will need:**
- `base/kustomization.yaml` - Base service configuration
- `base/deployment.yaml` - Kubernetes deployment
- `base/service.yaml` - Kubernetes service
- `base/configmap.yaml` - Service configuration
- `components/istio/kustomization.yaml` - Istio component config
- `components/istio/destinationrule.yaml` - Traffic policies
- `components/istio/virtualservice.yaml` - Routing rules
- `components/istio/peerauthentication.yaml` - Service-specific mTLS policies
- `components/istio/authorizationpolicy.yaml` - Service access control rules
- `components/istio/requestauthentication.yaml` - JWT validation for service
- `components/istio/networkpolicy.yaml` - Kubernetes network isolation
- `overlays/local/kustomization.yaml` - Local environment overrides

---

## Summary

This deployment guide provides the foundation for Sprint 1:

✅ **Local minikube cluster** ready for development  
✅ **Istio service mesh** installed and configured  
✅ **Kustomize-based IaC** organized for future scaling  
✅ **Validation framework** to ensure proper setup  
✅ **Troubleshooting guide** for common issues  
✅ **Repository structure** prepared for service deployment  

**Current State After Completion:**
- Local Kubernetes cluster with Istio service mesh
- Platform and monitoring gateways configured
- Security policies (mTLS) enabled
- Observability (telemetry, tracing) configured
- Ready for Sprint 1 service implementation

**Files Created:** 22 configuration files + 2 scripts = **24 total files**

**Security-First Approach:**
- All communication encrypted by default (mTLS STRICT)
- Zero-trust network model with deny-by-default policies
- Comprehensive security validation framework
- Production-ready security baselines applied

The infrastructure foundation is now ready for Sprint 1 service development!

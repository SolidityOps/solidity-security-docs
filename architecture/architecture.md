# Solidity Security Platform - Architecture Overview

## Overview

The Solidity Security Platform is a comprehensive, cloud-native security analysis platform designed to identify vulnerabilities in Solidity smart contracts. The platform leverages microservices architecture, containerization, and modern DevOps practices to provide scalable, secure, and maintainable security analysis capabilities.

## Domain & Environment

- **Domain**: `advancedblockchainsecurity.com`
- **Staging Environment**: `*.staging.advancedblockchainsecurity.com`
- **Production Environment**: `*.advancedblockchainsecurity.com`
- **Primary Region**: `us-west-2`
- **Kubernetes Version**: `1.28`

## AWS Resources

### Core Infrastructure

#### Amazon EKS (Elastic Kubernetes Service)
- **Cluster Name**: `solidity-security-eks`
- **Version**: `1.28`
- **Region**: `us-west-2`
- **Access Mode**: `private-public`
- **Cluster Autoscaler**: Enabled with priority expander
- **Configuration Files**:
  - `/solidity-security-aws-infrastructure/k8s/base/eks-cluster/cluster-config.yaml`
  - `/solidity-security-aws-infrastructure/k8s/base/node-groups/node-group.yaml`

#### Amazon VPC (Virtual Private Cloud)
- **Purpose**: Network isolation and security
- **Configuration**: Private and public subnets
- **Network Policies**: Kubernetes NetworkPolicy resources for service isolation

#### AWS Load Balancer Controller
- **Type**: Application Load Balancer (ALB)
- **Purpose**: Ingress traffic management and SSL termination
- **Configuration Files**:
  - `/solidity-security-aws-infrastructure/k8s/base/aws-load-balancer-controller/deployment.yaml`
  - `/solidity-security-aws-infrastructure/k8s/base/aws-load-balancer-controller/service-account.yaml`

#### Amazon ElastiCache
- **Purpose**: Redis caching layer for session management and data caching
- **Integration**: Used by multiple services for performance optimization
- **Configuration**: Referenced in Vault secrets and service configurations

#### AWS IAM (Identity and Access Management)
- **Service Accounts**: EKS service accounts with IAM roles for AWS resource access
- **Roles**: Cluster autoscaler role, load balancer controller role
- **Annotations**: `eks.amazonaws.com/role-arn` for service account role bindings

#### Amazon CloudWatch
- **Purpose**: Logging and monitoring
- **Integration**: Container insights and application logging
- **Status**: Cluster logging currently disabled (`cluster.logging.enabled: "false"`)

#### Amazon S3 (Simple Storage Service)
- **Purpose**: Artifact storage and data persistence (inferred from typical usage patterns)
- **Usage**: Likely used for analysis results and file storage

#### Amazon ECR (Elastic Container Registry)
- **Purpose**: Container image storage
- **Usage**: Hosting Docker images for all microservices

### Security & Secrets Management

#### AWS Secrets Manager
- **Integration**: Via External Secrets Operator
- **Purpose**: Secure storage and retrieval of sensitive configuration data
- **Configuration Files**:
  - `/solidity-security-aws-infrastructure/k8s/base/external-secrets/cluster-secret-store.yaml`
  - `/solidity-security-aws-infrastructure/k8s/base/external-secrets/operator.yaml`

## 3rd Party Services

### DNS & CDN

#### Cloudflare
- **Provider**: Cloudflare DNS and CDN
- **Configuration**: Terraform-managed DNS records
- **Features**:
  - DNS management for all subdomains
  - CDN and DDoS protection (proxied: true for most records)
  - SSL/TLS termination (strict SSL mode)
  - Security features: WAF, rate limiting, hotlink protection
  - Performance optimizations: Brotli compression, minification, caching
- **Configuration File**: `/solidity-security-aws-infrastructure/cloudflare/terraform/main.tf`
- **Subdomains**:
  - Production: `api`, `dashboard`, `vault`, `argocd`, `monitoring`, `mail`
  - Staging: `staging`, `api-staging`, `dashboard-staging`, `vault-staging`, `argocd-staging`, `monitoring-staging`

### Security & Secrets Management

#### HashiCorp Vault
- **Version**: `1.15.2`
- **Purpose**: Advanced secrets management and encryption
- **Deployment**: Kubernetes StatefulSet with persistent storage
- **Features**:
  - Secret rotation and lifecycle management
  - Integration with External Secrets Operator
  - TLS-secured communication
- **Configuration Files**:
  - `/solidity-security-monitoring/vault/manifests/vault-statefulset.yaml`
  - `/solidity-security-monitoring/vault/manifests/vault-service.yaml`
  - `/solidity-security-monitoring/vault/examples/vault-secret-*.yaml`

#### Cert-Manager
- **Purpose**: Automated SSL/TLS certificate management
- **Integration**: Let's Encrypt certificate automation
- **Configuration Files**:
  - `/solidity-security-aws-infrastructure/k8s/base/cert-manager/deployment.yaml`
  - `/solidity-security-aws-infrastructure/k8s/base/cert-manager/cluster-issuer.yaml`

### Database Systems

#### PostgreSQL
- **Purpose**: Primary relational database
- **Deployment**: Kubernetes StatefulSet with persistent volumes
- **Features**:
  - Automated backups via CronJob
  - Environment-specific configurations (staging/production)
  - Network isolation with NetworkPolicy
- **Configuration Files**:
  - `/solidity-security-monitoring/k8s/postgresql/*/postgresql-statefulset.yaml`
  - `/solidity-security-monitoring/k8s/postgresql/backup-cronjob.yaml`

#### Redis
- **Purpose**: Caching and session management
- **Deployment**: ElastiCache (AWS managed) + potential Kubernetes deployment
- **Integration**: Connected via Vault-managed secrets

#### MongoDB
- **Purpose**: Document storage for analysis results and metadata
- **Usage**: Inferred from application patterns and security tool integrations

#### Elasticsearch
- **Purpose**: Log aggregation and search capabilities
- **Usage**: Part of monitoring and observability stack

#### InfluxDB
- **Purpose**: Time-series data storage for metrics and monitoring
- **Integration**: Monitoring stack component

### Monitoring & Observability

#### Prometheus
- **Purpose**: Metrics collection and alerting
- **Deployment**: Kubernetes-native monitoring
- **Configuration Files**:
  - `/solidity-security-aws-infrastructure/k8s/base/monitoring/service-monitors.yaml`

#### Grafana
- **Purpose**: Monitoring dashboards and visualization
- **Access**: `monitoring.advancedblockchainsecurity.com`
- **Integration**: Prometheus data source

### DevOps & CI/CD

#### ArgoCD
- **Purpose**: GitOps continuous deployment
- **Access**: `argocd.advancedblockchainsecurity.com`
- **Features**:
  - Automated application deployment
  - Git-based configuration management
  - Multi-environment support (staging/production)

### Email Services

#### Email Infrastructure
- **MX Records**: Configured for `advancedblockchainsecurity.com`
- **Security**: SPF and DMARC records configured
- **Integration**: Google Workspace (`include:_spf.google.com`)
- **Purpose**: Notification system for security alerts and platform updates

## Microservices Architecture

### Core Services

1. **solidity-security-api-service**
   - **Technology**: Python FastAPI
   - **Purpose**: Main API gateway and service orchestration
   - **Container**: Multi-stage Docker build with security hardening

2. **solidity-security-dashboard**
   - **Technology**: React TypeScript with Vite
   - **Purpose**: Web interface for security analysis results
   - **Dependencies**: React Router, TanStack Query, Zustand, Tailwind CSS

3. **solidity-security-data-service**
   - **Purpose**: Data persistence and retrieval layer
   - **Container**: Dockerized service

4. **solidity-security-intelligence-engine**
   - **Purpose**: AI/ML-powered vulnerability detection
   - **Container**: Dockerized service

5. **solidity-security-analysis**
   - **Purpose**: Core security analysis logic
   - **Container**: Dockerized service

6. **solidity-security-contract-parser**
   - **Purpose**: Solidity code parsing and AST analysis
   - **Container**: Dockerized service

7. **solidity-security-findings**
   - **Purpose**: Vulnerability findings management
   - **Container**: Dockerized service

8. **solidity-security-notification**
   - **Purpose**: Alert and notification system
   - **Container**: Dockerized service

9. **solidity-security-orchestration**
   - **Purpose**: Workflow orchestration and job management
   - **Container**: Dockerized service

10. **solidity-security-tool-integration**
    - **Purpose**: Integration with external security tools
    - **Container**: Dockerized service

### Supporting Services

1. **solidity-security-monitoring**
   - **Purpose**: Platform monitoring and observability
   - **Components**: Prometheus, Grafana, logging infrastructure

2. **solidity-security-ui-core**
   - **Technology**: React TypeScript
   - **Purpose**: Shared UI components and libraries
   - **Container**: Dockerized service

3. **solidity-security-shared**
   - **Purpose**: Common libraries and utilities
   - **Type**: TypeScript shared library

## Infrastructure Management

### Infrastructure as Code

#### Terraform
- **Provider**: Cloudflare
- **Purpose**: DNS and CDN configuration management
- **Location**: `/solidity-security-aws-infrastructure/cloudflare/terraform/`

#### Kustomize
- **Purpose**: Kubernetes configuration management
- **Structure**: Base configurations with environment-specific overlays
- **Environments**: Staging and Production overlays

### Container Orchestration

#### Kubernetes
- **Platform**: Amazon EKS
- **Features**:
  - Namespace isolation
  - Resource quotas and limits
  - Network policies for security
  - Persistent volume management
  - Secret management integration

#### Docker
- **Build Strategy**: Multi-stage builds for security and efficiency
- **Security**: Non-root user execution, minimal base images
- **Features**: Health checks, proper labeling, build-time arguments

## Security Features

### Network Security
- **Network Policies**: Kubernetes NetworkPolicy for service isolation
- **TLS**: End-to-end encryption with automated certificate management
- **Firewall**: Cloudflare WAF and DDoS protection
- **VPC**: Private subnets for sensitive workloads

### Secrets Management
- **HashiCorp Vault**: Centralized secrets management
- **External Secrets Operator**: Kubernetes-native secrets synchronization
- **AWS Secrets Manager**: Cloud-native secrets storage
- **Encrypted Storage**: Persistent volumes with encryption

### Access Control
- **RBAC**: Kubernetes Role-Based Access Control
- **Service Accounts**: AWS IAM integration with EKS
- **Multi-factor Authentication**: Vault and ArgoCD authentication

### Container Security
- **Non-root Execution**: All containers run as non-root users
- **Read-only Filesystems**: Security-hardened container configurations
- **Vulnerability Scanning**: Container image security scanning
- **Minimal Images**: Slim base images to reduce attack surface

## Deployment Strategy

### Environments
- **Staging**: Full environment replica for testing (`*.staging.advancedblockchainsecurity.com`)
- **Production**: Production environment (`*.advancedblockchainsecurity.com`)

### GitOps
- **ArgoCD**: Automated deployment based on Git repository changes
- **Configuration Management**: Kustomize overlays for environment-specific configurations
- **Rollback Capability**: Git-based rollback and versioning

### Scaling
- **Horizontal Pod Autoscaling**: Kubernetes HPA for automatic scaling
- **Cluster Autoscaling**: AWS EKS cluster autoscaler for node management
- **Load Balancing**: AWS Application Load Balancer for traffic distribution

## Performance & Reliability

### Caching Strategy
- **Redis**: Application-level caching
- **ElastiCache**: Managed Redis for session and data caching
- **Cloudflare**: CDN caching for static assets

### Monitoring & Alerting
- **Prometheus**: Metrics collection and alerting rules
- **Grafana**: Dashboards and visualization
- **CloudWatch**: AWS-native monitoring integration
- **Structured Logging**: Centralized log management

### Backup & Recovery
- **Database Backups**: Automated PostgreSQL backups via CronJob
- **Persistent Volume Snapshots**: EBS volume snapshots
- **Git-based Configuration**: Infrastructure and application configuration versioning

## Technology Stack Summary

### Programming Languages
- **TypeScript/JavaScript**: Frontend applications (React)
- **Python**: Backend services (FastAPI)
- **Go**: Infrastructure components (likely)

### Frontend Technologies
- **React 18**: UI framework
- **TypeScript**: Type safety
- **Vite**: Build tool and development server
- **Tailwind CSS**: Styling framework
- **TanStack Query**: Data fetching and caching
- **Zustand**: State management
- **React Router**: Client-side routing

### Backend Technologies
- **FastAPI**: Python web framework
- **Uvicorn**: ASGI server
- **PostgreSQL**: Primary database
- **Redis**: Caching layer
- **MongoDB**: Document storage
- **Elasticsearch**: Search and analytics

### DevOps & Infrastructure
- **Amazon EKS**: Kubernetes management
- **Docker**: Containerization
- **ArgoCD**: GitOps deployment
- **Terraform**: Infrastructure as Code
- **Kustomize**: Kubernetes configuration management
- **HashiCorp Vault**: Secrets management

### Monitoring & Security
- **Prometheus**: Metrics collection
- **Grafana**: Monitoring dashboards
- **Cert-Manager**: SSL certificate automation
- **External Secrets Operator**: Secrets synchronization
- **Cloudflare**: DNS, CDN, and security

This architecture provides a robust, scalable, and secure foundation for the Solidity Security Platform, leveraging cloud-native technologies and best practices for enterprise-grade security analysis capabilities.
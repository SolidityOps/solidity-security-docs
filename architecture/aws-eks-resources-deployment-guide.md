# AWS and EKS Resources Deployment Guide

**Document Version**: 1.0
**Date**: 2025-09-28
**Scope**: Tasks 1.1 - 1.6 and Beyond
**Environment**: Staging and Production

## Executive Summary

This document provides a comprehensive inventory of all AWS and EKS resources that will be deployed across Sprint 1 tasks and subsequent sprints. The deployment follows the `[service]-[overlay]` namespace convention and implements Vault Community Edition for secret management.

## Task-by-Task Resource Deployment

### Task 1.1: AWS Infrastructure Foundation

#### AWS Core Infrastructure
| Resource Type | Resource Name | Purpose | Environment |
|---------------|---------------|---------|-------------|
| **VPC** | `staging-solidity-security-vpc` | Network isolation for staging | Staging |
| **VPC** | `production-solidity-security-vpc` | Network isolation for production | Production |
| **Internet Gateway** | `staging-igw` | Internet access for staging VPC | Staging |
| **Internet Gateway** | `production-igw` | Internet access for production VPC | Production |
| **NAT Gateway** | `staging-nat-gateway-1a` | Outbound internet for private subnets | Staging |
| **NAT Gateway** | `staging-nat-gateway-1b` | Outbound internet for private subnets | Staging |
| **NAT Gateway** | `production-nat-gateway-1a` | Outbound internet for private subnets | Production |
| **NAT Gateway** | `production-nat-gateway-1b` | Outbound internet for private subnets | Production |
| **NAT Gateway** | `production-nat-gateway-1c` | Outbound internet for private subnets | Production |

#### Subnets
| Subnet Type | CIDR | Availability Zone | Purpose | Environment |
|-------------|------|-------------------|---------|-------------|
| **Public** | `10.0.1.0/24` | us-west-2a | Load balancers, NAT | Staging |
| **Public** | `10.0.2.0/24` | us-west-2b | Load balancers, NAT | Staging |
| **Private** | `10.0.10.0/24` | us-west-2a | EKS worker nodes | Staging |
| **Private** | `10.0.11.0/24` | us-west-2b | EKS worker nodes | Staging |
| **Database** | `10.0.20.0/24` | us-west-2a | RDS instances | Staging |
| **Database** | `10.0.21.0/24` | us-west-2b | RDS instances | Staging |
| **Public** | `10.1.1.0/24` | us-west-2a | Load balancers, NAT | Production |
| **Public** | `10.1.2.0/24` | us-west-2b | Load balancers, NAT | Production |
| **Public** | `10.1.3.0/24` | us-west-2c | Load balancers, NAT | Production |
| **Private** | `10.1.10.0/24` | us-west-2a | EKS worker nodes | Production |
| **Private** | `10.1.11.0/24` | us-west-2b | EKS worker nodes | Production |
| **Private** | `10.1.12.0/24` | us-west-2c | EKS worker nodes | Production |
| **Database** | `10.1.20.0/24` | us-west-2a | RDS instances | Production |
| **Database** | `10.1.21.0/24` | us-west-2b | RDS instances | Production |
| **Database** | `10.1.22.0/24` | us-west-2c | RDS instances | Production |

#### Security Groups
| Security Group | Purpose | Ports | Environment |
|----------------|---------|-------|-------------|
| `staging-eks-cluster-sg` | EKS cluster communication | 443, 10250 | Staging |
| `staging-eks-worker-sg` | EKS worker node communication | 22, 1025-65535 | Staging |
| `staging-rds-sg` | Database access | 5432 | Staging |
| `staging-elasticache-sg` | Redis access | 6379 | Staging |
| `production-eks-cluster-sg` | EKS cluster communication | 443, 10250 | Production |
| `production-eks-worker-sg` | EKS worker node communication | 22, 1025-65535 | Production |
| `production-rds-sg` | Database access | 5432 | Production |
| `production-elasticache-sg` | Redis access | 6379 | Production |

### Task 1.2: EKS Cluster Setup

#### EKS Clusters
| Resource | Configuration | Environment |
|----------|---------------|-------------|
| **EKS Cluster** | `staging-solidity-security-cluster` | Staging |
| - Kubernetes Version | 1.28 | Staging |
| - Endpoint Access | Public + Private | Staging |
| - Logging | API, Audit, Authenticator, ControllerManager, Scheduler | Staging |
| **EKS Cluster** | `production-solidity-security-cluster` | Production |
| - Kubernetes Version | 1.28 | Production |
| - Endpoint Access | Private | Production |
| - Logging | API, Audit, Authenticator, ControllerManager, Scheduler | Production |

#### EKS Node Groups
| Node Group | Instance Type | Min/Max/Desired | Disk Size | Environment |
|------------|---------------|-----------------|-----------|-------------|
| `staging-general-nodes` | m5.large | 2/5/3 | 50GB | Staging |
| `staging-compute-nodes` | m5.xlarge | 1/3/2 | 100GB | Staging |
| `production-general-nodes` | m5.xlarge | 3/10/6 | 50GB | Production |
| `production-compute-nodes` | m5.2xlarge | 2/8/4 | 100GB | Production |
| `production-memory-nodes` | r5.xlarge | 1/5/2 | 100GB | Production |

#### EKS Add-ons
| Add-on | Version | Purpose | Environment |
|--------|---------|---------|-------------|
| `vpc-cni` | Latest | Pod networking | Both |
| `coredns` | Latest | DNS resolution | Both |
| `kube-proxy` | Latest | Network proxy | Both |
| `aws-ebs-csi-driver` | Latest | Persistent volumes | Both |

### Task 1.3: AWS Load Balancer Controller

#### IAM Resources
| Resource Type | Resource Name | Purpose | Environment |
|---------------|---------------|---------|-------------|
| **IAM Role** | `staging-eks-alb-controller-role` | ALB Controller permissions | Staging |
| **IAM Role** | `production-eks-alb-controller-role` | ALB Controller permissions | Production |
| **IAM Policy** | `AWSLoadBalancerControllerIAMPolicy` | ALB management permissions | Both |

#### Kubernetes Resources
| Resource Type | Namespace | Purpose | Environment |
|---------------|-----------|---------|-------------|
| **ServiceAccount** | `kube-system` | IRSA for ALB Controller | Both |
| **Deployment** | `kube-system` | ALB Controller pods | Both |
| **ClusterRole** | N/A | ALB Controller permissions | Both |
| **ClusterRoleBinding** | N/A | Bind role to service account | Both |

### Task 1.4: HashiCorp Vault Community Edition

#### Kubernetes Resources - Staging
| Resource Type | Namespace | Replicas | Purpose |
|---------------|-----------|----------|---------|
| **Namespace** | `vault-staging` | N/A | Vault server isolation |
| **StatefulSet** | `vault-staging` | 3 | Vault server pods |
| **Service** | `vault-staging` | N/A | Vault API access |
| **Service** | `vault-staging` | N/A | Vault internal cluster communication |
| **ConfigMap** | `vault-staging` | N/A | Vault configuration |
| **ServiceAccount** | `vault-staging` | N/A | Vault pod identity |
| **ClusterRole** | N/A | N/A | Vault cluster permissions |
| **ClusterRoleBinding** | N/A | N/A | Bind cluster role |
| **PersistentVolumeClaim** | `vault-staging` | 3 | Vault data storage (10GB each) |
| **Job** | `vault-staging` | N/A | Initialization helper |
| **Ingress** | `vault-staging` | N/A | External access |

#### Kubernetes Resources - Production
| Resource Type | Namespace | Replicas | Purpose |
|---------------|-----------|----------|---------|
| **Namespace** | `vault-production` | N/A | Vault server isolation |
| **StatefulSet** | `vault-production` | 3 | Vault server pods (HA) |
| **Service** | `vault-production` | N/A | Vault API access |
| **Service** | `vault-production` | N/A | Vault internal cluster communication |
| **ConfigMap** | `vault-production` | N/A | Vault configuration |
| **ServiceAccount** | `vault-production` | N/A | Vault pod identity |
| **ClusterRole** | N/A | N/A | Vault cluster permissions |
| **ClusterRoleBinding** | N/A | N/A | Bind cluster role |
| **PersistentVolumeClaim** | `vault-production` | 3 | Vault data storage (10GB each) |
| **Job** | `vault-production` | N/A | Initialization helper |
| **Ingress** | `vault-production` | N/A | External access |

#### Vault Configuration Details
| Feature | Staging | Production | Notes |
|---------|---------|------------|-------|
| **Storage Backend** | Raft Integrated | Raft Integrated | No external dependencies |
| **Unsealing** | Manual (3 of 5 keys) | Manual (3 of 5 keys) | Community Edition limitation |
| **High Availability** | 3-node cluster | 3-node cluster | Leader election via Raft |
| **UI Access** | Enabled | Enabled | Web interface available |
| **Audit Logging** | Enabled | Enabled | File-based audit logs |
| **Auth Methods** | Kubernetes | Kubernetes | For External Secrets integration |
| **Secrets Engines** | KV v2 | KV v2 | Key-value storage |

### Task 1.5: SSL/TLS Certificate Management

#### Kubernetes Resources - cert-manager
| Resource Type | Namespace | Purpose | Environment |
|---------------|-----------|---------|-------------|
| **Namespace** | `cert-manager-staging` | cert-manager isolation | Staging |
| **Namespace** | `cert-manager-production` | cert-manager isolation | Production |
| **Deployment** | `cert-manager-staging` | cert-manager controller | Staging |
| **Deployment** | `cert-manager-staging` | CA injector | Staging |
| **Deployment** | `cert-manager-staging` | Webhook | Staging |
| **Deployment** | `cert-manager-production` | cert-manager controller | Production |
| **Deployment** | `cert-manager-production` | CA injector | Production |
| **Deployment** | `cert-manager-production` | Webhook | Production |
| **Service** | `cert-manager-staging` | cert-manager API | Staging |
| **Service** | `cert-manager-staging` | Webhook service | Staging |
| **Service** | `cert-manager-production` | cert-manager API | Production |
| **Service** | `cert-manager-production` | Webhook service | Production |

#### Certificate Issuers
| Issuer Type | Name | ACME Server | DNS Validation |
|-------------|------|-------------|----------------|
| **ClusterIssuer** | `letsencrypt-staging` | Let's Encrypt Staging | Cloudflare DNS |
| **ClusterIssuer** | `letsencrypt-production` | Let's Encrypt Production | Cloudflare DNS |

#### SSL Certificates
| Certificate | Domain | Issuer | Environment |
|-------------|--------|--------|-------------|
| `vault-staging-tls` | `vault.staging.advancedblockchainsecurity.com` | letsencrypt-production | Staging |
| `vault-production-tls` | `vault.app.advancedblockchainsecurity.com` | letsencrypt-production | Production |
| `argocd-staging-tls` | `argocd.staging.advancedblockchainsecurity.com` | letsencrypt-production | Staging |
| `argocd-production-tls` | `argocd.app.advancedblockchainsecurity.com` | letsencrypt-production | Production |

### Task 1.6: External Secrets Operator

#### IAM Resources
| Resource Type | Resource Name | Purpose | Environment |
|---------------|---------------|---------|-------------|
| **IAM Role** | `staging-external-secrets-role` | External Secrets permissions | Staging |
| **IAM Role** | `production-external-secrets-role` | External Secrets permissions | Production |

#### Kubernetes Resources
| Resource Type | Namespace | Purpose | Environment |
|---------------|-----------|---------|-------------|
| **Namespace** | `external-secrets-staging` | External Secrets isolation | Staging |
| **Namespace** | `external-secrets-production` | External Secrets isolation | Production |
| **Deployment** | `external-secrets-staging` | External Secrets controller | Staging |
| **Deployment** | `external-secrets-staging` | External Secrets webhook | Staging |
| **Deployment** | `external-secrets-production` | External Secrets controller | Production |
| **Deployment** | `external-secrets-production` | External Secrets webhook | Production |
| **ServiceAccount** | `external-secrets-staging` | IRSA for Vault access | Staging |
| **ServiceAccount** | `external-secrets-production` | IRSA for Vault access | Production |
| **ClusterRole** | N/A | External Secrets permissions | Both |
| **ClusterRoleBinding** | N/A | Bind cluster role | Both |

#### Secret Store Configuration
| Resource Type | Name | Backend | Purpose |
|---------------|------|---------|---------|
| **ClusterSecretStore** | `vault-backend-staging` | Vault Staging | Access vault-staging |
| **ClusterSecretStore** | `vault-backend-production` | Vault Production | Access vault-production |

## Resources Deployed After Task 1.6

### Task 1.7: Monitoring and Observability

#### Kubernetes Resources - Monitoring Stack
| Resource Type | Namespace | Purpose | Environment |
|---------------|-----------|---------|-------------|
| **Namespace** | `monitoring-staging` | Monitoring isolation | Staging |
| **Namespace** | `monitoring-production` | Monitoring isolation | Production |
| **StatefulSet** | `monitoring-staging` | Prometheus server | Staging |
| **Deployment** | `monitoring-staging` | Grafana dashboard | Staging |
| **DaemonSet** | `monitoring-staging` | Fluent Bit log collector | Staging |
| **Deployment** | `monitoring-staging` | Loki log aggregation | Staging |
| **StatefulSet** | `monitoring-production` | Prometheus server (HA) | Production |
| **Deployment** | `monitoring-production` | Grafana dashboard (HA) | Production |
| **DaemonSet** | `monitoring-production` | Fluent Bit log collector | Production |
| **Deployment** | `monitoring-production` | Loki log aggregation (HA) | Production |

#### Monitoring Configuration
| Component | Storage | Retention | High Availability |
|-----------|---------|-----------|-------------------|
| **Prometheus Staging** | 50GB | 30 days | Single replica |
| **Prometheus Production** | 200GB | 90 days | 2 replicas |
| **Grafana Staging** | 10GB | N/A | Single replica |
| **Grafana Production** | 20GB | N/A | 2 replicas |
| **Loki Staging** | 100GB | 30 days | Single replica |
| **Loki Production** | 500GB | 90 days | 3 replicas |

### Task 1.8: ArgoCD GitOps

#### Kubernetes Resources - ArgoCD
| Resource Type | Namespace | Purpose | Environment |
|---------------|-----------|---------|-------------|
| **Namespace** | `argocd-staging` | ArgoCD isolation | Staging |
| **Namespace** | `argocd-production` | ArgoCD isolation | Production |
| **Deployment** | `argocd-staging` | ArgoCD server | Staging |
| **Deployment** | `argocd-staging` | ArgoCD repo server | Staging |
| **Deployment** | `argocd-staging` | ArgoCD application controller | Staging |
| **StatefulSet** | `argocd-production` | ArgoCD server (HA) | Production |
| **Deployment** | `argocd-production` | ArgoCD repo server (HA) | Production |
| **Deployment** | `argocd-production` | ArgoCD application controller (HA) | Production |
| **Service** | `argocd-staging` | ArgoCD server service | Staging |
| **Service** | `argocd-production` | ArgoCD server service | Production |
| **Ingress** | `argocd-staging` | External ArgoCD access | Staging |
| **Ingress** | `argocd-production` | External ArgoCD access | Production |

#### ArgoCD Configuration
| Feature | Staging | Production | Purpose |
|---------|---------|------------|---------|
| **GitHub Integration** | All 17 repositories | All 17 repositories | Source code access |
| **RBAC** | Basic team access | Advanced role-based access | Access control |
| **High Availability** | Single replica | 3 replicas | Reliability |
| **Persistent Storage** | 20GB | 50GB | Configuration and cache |
| **SSL Termination** | cert-manager | cert-manager | Secure access |

### Future Sprint Resources (Sprint 2+)

#### Database Infrastructure (Sprint 2)
| Resource Type | Configuration | Environment |
|---------------|---------------|-------------|
| **RDS PostgreSQL** | db.t3.medium, 100GB | Staging |
| **RDS PostgreSQL** | db.r5.xlarge, 500GB, Multi-AZ | Production |
| **ElastiCache Redis** | cache.t3.micro, 1 node | Staging |
| **ElastiCache Redis** | cache.r5.large, 3 nodes | Production |

#### Application Services (Sprint 3-6)
| Service | Namespace | Resources | Purpose |
|---------|-----------|-----------|---------|
| **API Service** | `api-service-staging/production` | 2-4 pods | REST API backend |
| **Data Service** | `data-service-staging/production` | 2-4 pods | Data processing |
| **Tool Integration** | `tool-integration-staging/production` | 2-6 pods | Security tool orchestration |
| **Orchestration** | `orchestration-staging/production` | 3-6 pods | Celery workers |
| **Intelligence Engine** | `intelligence-engine-staging/production` | 2-4 pods | AI/ML processing |
| **Notification** | `notification-staging/production` | 2-3 pods | Alert system |
| **Contract Parser** | `contract-parser-staging/production` | 2-4 pods | Solidity parsing |
| **UI Core** | `ui-core-staging/production` | 2-4 pods | React frontend |
| **Dashboard** | `dashboard-staging/production` | 2-4 pods | Analytics dashboard |
| **Findings** | `findings-staging/production` | 2-4 pods | Findings management |
| **Analysis** | `analysis-staging/production` | 2-4 pods | Analysis workflow |

## Resource Summary by Environment

### Staging Environment Total Resources
| Category | Count | Purpose |
|----------|-------|---------|
| **AWS Infrastructure** | 25+ resources | VPC, subnets, security groups, NAT gateways |
| **EKS Cluster** | 1 cluster | Kubernetes control plane |
| **EKS Node Groups** | 2 groups | Worker nodes (5-8 instances) |
| **Kubernetes Namespaces** | 5 namespaces | Service isolation |
| **Kubernetes Deployments** | 12+ deployments | Application workloads |
| **Kubernetes StatefulSets** | 2 statefulsets | Stateful services (Vault, Prometheus) |
| **Persistent Volumes** | 10+ volumes | Data persistence |
| **Load Balancers** | 3-5 ALBs | External access |
| **SSL Certificates** | 4+ certificates | Secure communications |

### Production Environment Total Resources
| Category | Count | Purpose |
|----------|-------|---------|
| **AWS Infrastructure** | 35+ resources | VPC, subnets, security groups, NAT gateways |
| **EKS Cluster** | 1 cluster | Kubernetes control plane |
| **EKS Node Groups** | 3 groups | Worker nodes (10-20 instances) |
| **Kubernetes Namespaces** | 5+ namespaces | Service isolation |
| **Kubernetes Deployments** | 15+ deployments | Application workloads (HA) |
| **Kubernetes StatefulSets** | 3+ statefulsets | Stateful services (HA) |
| **Persistent Volumes** | 20+ volumes | Data persistence |
| **Load Balancers** | 5-8 ALBs | External access |
| **SSL Certificates** | 6+ certificates | Secure communications |
| **Database Resources** | 2+ RDS instances | Data storage |
| **Cache Resources** | 1 Redis cluster | Performance optimization |

## Cost Estimation

### Monthly Cost Breakdown (USD)

#### Staging Environment
| Category | Monthly Cost | Notes |
|----------|-------------|-------|
| **EKS Cluster** | $75 | Control plane |
| **EC2 Instances** | $200-300 | Worker nodes (5-8 instances) |
| **Load Balancers** | $60-100 | ALB instances |
| **Storage** | $50-100 | EBS volumes |
| **NAT Gateways** | $90 | 2 NAT gateways |
| **Data Transfer** | $20-50 | Cross-AZ and internet |
| **Total Staging** | **$495-715** | Estimated monthly cost |

#### Production Environment
| Category | Monthly Cost | Notes |
|----------|-------------|-------|
| **EKS Cluster** | $75 | Control plane |
| **EC2 Instances** | $800-1200 | Worker nodes (10-20 instances) |
| **Load Balancers** | $150-250 | ALB instances |
| **Storage** | $200-400 | EBS volumes |
| **NAT Gateways** | $135 | 3 NAT gateways |
| **RDS PostgreSQL** | $300-500 | Multi-AZ database |
| **ElastiCache** | $100-200 | Redis cluster |
| **Data Transfer** | $50-150 | Cross-AZ and internet |
| **Total Production** | **$1,810-2,910** | Estimated monthly cost |

#### Combined Total
**Estimated Monthly Cost**: $2,305-3,625 USD

## Deployment Timeline

### Phase 1: Foundation (Tasks 1.1-1.3) - Week 1
- AWS infrastructure setup
- EKS cluster deployment
- AWS Load Balancer Controller installation

### Phase 2: Security & Secrets (Tasks 1.4-1.6) - Week 2
- Vault Community Edition deployment
- cert-manager installation
- External Secrets Operator setup

### Phase 3: Operations (Tasks 1.7-1.8) - Week 3
- Monitoring stack deployment
- ArgoCD GitOps setup
- Infrastructure validation

### Phase 4: Applications (Sprint 2+) - Week 4+
- Database infrastructure
- Application service deployment
- Full platform integration

## Security Considerations

### Network Security
- **VPC Isolation**: Separate VPCs for staging and production
- **Private Subnets**: Worker nodes in private subnets only
- **Security Groups**: Least-privilege access rules
- **NACLs**: Additional network-level protection

### Identity and Access Management
- **IRSA**: IAM Roles for Service Accounts
- **Least Privilege**: Minimal permissions for each service
- **Service Accounts**: Dedicated Kubernetes service accounts
- **RBAC**: Role-based access control

### Secret Management
- **Vault Community**: Centralized secret storage
- **Manual Unsealing**: Secure unseal procedures
- **External Secrets**: Automated secret injection
- **Encryption**: Secrets encrypted at rest and in transit

### SSL/TLS Security
- **cert-manager**: Automated certificate management
- **Let's Encrypt**: Free SSL certificates
- **DNS Validation**: Secure certificate issuance
- **Ingress TLS**: SSL termination at load balancer

## Operational Procedures

### Manual Operations Required
1. **Vault Unsealing**: Manual unsealing after pod restarts
2. **Certificate Renewal**: Monitor cert-manager automation
3. **Secret Rotation**: Manual rotation procedures for Vault Community
4. **Backup Procedures**: Manual Vault snapshots
5. **Monitoring**: Health checks and alerting validation

### Automation Capabilities
1. **GitOps Deployment**: ArgoCD automated deployments
2. **SSL Certificates**: Automatic renewal via cert-manager
3. **Secret Injection**: Automated via External Secrets Operator
4. **Scaling**: Kubernetes HPA and cluster autoscaler
5. **Monitoring**: Prometheus metrics collection

## Conclusion

This comprehensive resource inventory covers all AWS and EKS resources deployed from Task 1.1 through Task 1.8 and beyond. The architecture implements:

- **Scalable Infrastructure**: EKS with auto-scaling capabilities
- **Secure Secret Management**: Vault Community Edition with External Secrets
- **Automated Operations**: GitOps with ArgoCD and monitoring with Prometheus
- **Production-Ready**: HA configurations and proper security controls
- **Cost-Optimized**: Appropriate resource sizing for each environment

The deployment provides a solid foundation for the Solidity Security Platform with enterprise-grade reliability, security, and observability.
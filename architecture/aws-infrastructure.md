# AWS Infrastructure Architecture

## Overview

The Solidity Security Platform is built on AWS with a cloud-native, microservices architecture that provides enterprise-grade security, scalability, and reliability. The platform leverages AWS managed services and Kubernetes (EKS) to deliver a comprehensive smart contract security analysis solution.

## Architecture Flow

### Request Flow
1. **User Access**: Users access the platform via Route 53 DNS resolution
2. **CDN & Security**: CloudFront delivers cached content, WAF filters malicious requests
3. **Load Balancing**: ALB distributes traffic with SSL termination
4. **Service Mesh**: Istio manages internal traffic with mTLS and routing
5. **API Gateway**: API Service handles authentication and request routing
6. **Microservices**: Backend services process requests with data persistence
7. **Real-time Updates**: WebSocket notifications provide live status updates

### Analysis Workflow
1. **Contract Upload**: Users upload Solidity contracts via Analysis frontend
2. **Parsing**: Contract Parser (Rust) generates AST and dependency analysis
3. **Orchestration**: Celery workers schedule parallel tool execution
4. **Tool Integration**: Multiple security tools analyze contracts simultaneously
5. **Intelligence Engine**: AI/ML processes results for deduplication and scoring
6. **Data Storage**: Results stored in PostgreSQL with Redis caching
7. **Notification**: Real-time updates sent to users via WebSocket

## Infrastructure Components

### Core AWS Services

#### Compute & Container Services
- **Amazon EKS (Elastic Kubernetes Service)**
  - Multi-AZ deployment across 3 availability zones
  - Managed node groups with auto-scaling
  - Staging and production clusters
  - Worker nodes: m5.large to m5.2xlarge instances
  - Spot instances for cost optimization on non-critical workloads

- **Amazon ECR (Elastic Container Registry)**
  - Private Docker registries for all microservices
  - Image scanning and vulnerability assessment
  - Lifecycle policies for image retention
  - Cross-region replication for disaster recovery

#### Database Services
- **Amazon RDS PostgreSQL 15**
  - Multi-AZ deployment for high availability
  - Automated backups with 30-day retention
  - Encryption at rest using AWS KMS
  - Performance Insights for monitoring
  - Read replicas for read-heavy workloads
  - Instance types: db.r5.large to db.r5.2xlarge

- **Amazon ElastiCache Redis**
  - Cluster mode enabled with sharding
  - Multi-AZ with automatic failover
  - Encryption in transit and at rest
  - Backup and restore capabilities
  - Instance types: cache.r6g.large to cache.r6g.xlarge

#### Storage Services
- **Amazon S3**
  - Contract source code storage
  - Analysis results and reports
  - Static website hosting for documentation
  - Cross-region replication
  - Lifecycle policies for cost optimization
  - Server-side encryption (SSE-S3/KMS)

- **Amazon EFS (Elastic File System)**
  - Shared storage for Kubernetes pods
  - Analysis workspace and temporary files
  - Automatic scaling and provisioning

#### Networking & Security
- **Amazon VPC (Virtual Private Cloud)**
  - Custom VPC with public and private subnets
  - Multi-AZ deployment across 3 availability zones
  - NAT Gateways for outbound internet access
  - VPC Flow Logs for network monitoring
  - CIDR: 10.0.0.0/16

- **AWS Application Load Balancer (ALB)**
  - Layer 7 load balancing
  - SSL/TLS termination
  - Path-based routing
  - Health checks and auto-scaling integration
  - WAF integration for security

- **AWS WAF (Web Application Firewall)**
  - OWASP Top 10 protection
  - Rate limiting and DDoS protection
  - Custom rules for API protection
  - Geographic blocking capabilities

- **Amazon Route 53**
  - DNS management and routing
  - Health checks and failover
  - Subdomain management for different environments

#### Security & Identity
- **AWS IAM (Identity and Access Management)**
  - Service roles with least privilege access
  - IRSA (IAM Roles for Service Accounts) integration
  - Cross-account access for multi-environment setup
  - MFA enforcement for administrative access

- **AWS Secrets Manager**
  - Database credentials and API keys
  - Automatic rotation for RDS passwords
  - Integration with Kubernetes via External Secrets Operator
  - Encryption with customer-managed KMS keys

- **AWS KMS (Key Management Service)**
  - Customer-managed encryption keys
  - Separate keys for different data types
  - Key rotation and audit logging
  - Cross-service encryption

- **AWS Certificate Manager (ACM)**
  - SSL/TLS certificates for all endpoints
  - Automatic renewal and deployment
  - Wildcard certificates for subdomains

#### Monitoring & Logging
- **Amazon CloudWatch**
  - Application and infrastructure metrics
  - Custom metrics for business KPIs
  - Log aggregation and analysis
  - Alarms and automated responses

- **AWS CloudTrail**
  - API call logging and auditing
  - Data events for S3 and other services
  - Integration with CloudWatch for alerting

- **AWS Config**
  - Configuration compliance monitoring
  - Resource inventory and change tracking
  - Automated remediation rules

- **Amazon GuardDuty**
  - Threat detection and intelligence
  - Malware and anomaly detection
  - Integration with security response workflows

### Non-AWS Infrastructure Components

#### Secret Management
- **AWS Secrets Manager**
  - Centralized secret management and storage
  - Automatic rotation for RDS and other AWS services
  - Fine-grained access control with IAM policies
  - Encryption at rest using AWS KMS
  - Cross-service integration with AWS resources
  - Cost-effective pay-per-secret pricing model

#### Service Mesh
- **Istio Service Mesh**
  - Traffic management and routing
  - Security policies and mTLS
  - Observability and tracing
  - Load balancing and circuit breaking
  - Ingress and egress traffic control

#### Monitoring & Observability
- **Prometheus**
  - Metrics collection and storage
  - Custom metrics for security tools
  - Alerting rules and thresholds
  - Federation for multi-cluster monitoring
  - Deployed on EKS with persistent storage

- **Grafana**
  - Metrics visualization and dashboards
  - Business intelligence and KPI tracking
  - User authentication and RBAC
  - Alert management and notifications
  - Deployed on EKS with persistent storage

- **Jaeger**
  - Distributed tracing for microservices
  - Request flow visualization
  - Performance bottleneck identification
  - Integration with Istio service mesh

- **Kiali**
  - Service mesh visualization
  - Traffic flow and security policies
  - Configuration validation
  - Performance metrics and health

#### GitOps & Deployment
- **ArgoCD**
  - GitOps-based deployment automation
  - Multi-environment management (staging/production)
  - Application lifecycle management
  - RBAC and security integration
  - Deployed on EKS with HA configuration

#### Certificate Management
- **cert-manager**
  - Kubernetes-native certificate management
  - Let's Encrypt integration
  - Automatic certificate renewal
  - DNS-01 challenge with Route 53

#### Ingress & Load Balancing
- **AWS Load Balancer Controller**
  - Kubernetes-native ALB provisioning
  - Integration with EKS and VPC
  - Automatic target group management
  - SSL/TLS termination

## Network Architecture

### VPC Design
```
Production VPC (10.0.0.0/16)
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24)
│   ├── NAT Gateways
│   ├── Internet Gateway
│   └── Application Load Balancers
└── Private Subnets (10.0.4.0/24, 10.0.5.0/24, 10.0.6.0/24)
    ├── EKS Worker Nodes
    ├── RDS Database Instances
    ├── ElastiCache Redis Clusters
    └── Internal Services

Staging VPC (10.1.0.0/16)
├── Public Subnets (10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24)
└── Private Subnets (10.1.4.0/24, 10.1.5.0/24, 10.1.6.0/24)
```

### Security Groups
- **ALB Security Group**: HTTP/HTTPS from internet
- **EKS Node Security Group**: Internal cluster communication
- **RDS Security Group**: PostgreSQL from EKS nodes only
- **Redis Security Group**: Redis protocol from EKS nodes only
- **External Services Security Group**: External service APIs from EKS nodes only

## Deployment Architecture

### Kubernetes Namespaces
```
├── istio-system          # Istio control plane
├── monitoring-staging and monitoring-production    # Prometheus, Grafana, Jaeger
├── argocd-staging and argocd-production          # ArgoCD deployment
├── cert-manager-staging and cert-manager-production # Certificate management
├── external-secrets-staging and external-secrets-production # External secrets operator
├── external-secrets-staging and external-secrets-production # External Secrets Operator for HashiCorp Vault Community Edition
├── kube-system         # System components
├── security-platform   # Main application services
│   ├── api-service
│   ├── tool-integration
│   ├── intelligence-engine
│   ├── orchestration
│   ├── data-service
│   ├── notification
│   ├── contract-parser
│   ├── ui-core
│   ├── dashboard
│   ├── findings
│   └── analysis
└── security-tools      # Security scanning tools
```

### Container Images
All services are containerized and stored in Amazon ECR:
- Base images: Alpine Linux for security and size
- Multi-stage builds for optimization
- Image vulnerability scanning enabled
- Automated image promotion pipeline

## Scaling and Performance

### Auto-Scaling Configuration
- **Cluster Auto-scaler**: Automatic node provisioning/deprovisioning
- **Horizontal Pod Auto-scaler**: CPU/memory-based pod scaling
- **Vertical Pod Auto-scaler**: Resource request optimization
- **Custom metrics scaling**: Business metric-based scaling

### Performance Optimization
- **RDS Performance**: Read replicas, connection pooling, query optimization
- **Redis Caching**: Multi-tier caching strategy, cache warming
- **CDN**: CloudFront for static assets and API caching
- **Load Balancing**: ALB with multiple targets and health checks

## Security Architecture

### Defense in Depth
1. **Perimeter Security**: WAF, GuardDuty, VPC security groups
2. **Network Security**: Private subnets, NACLs, VPC Flow Logs
3. **Application Security**: mTLS, JWT authentication, RBAC
4. **Data Security**: Encryption at rest/transit, secrets management
5. **Container Security**: Image scanning, pod security policies
6. **Identity Security**: IAM, IRSA, MFA, least privilege

### Compliance and Governance
- **SOC 2 Type II**: Implemented security controls
- **ISO 27001**: Information security management
- **GDPR**: Data protection and privacy controls
- **Audit Logging**: Comprehensive audit trail
- **Data Residency**: Regional data storage controls

## Disaster Recovery

### Backup Strategy
- **RDS**: Automated daily backups, 30-day retention
- **Redis**: Backup to S3, point-in-time recovery
- **EKS**: etcd backups, application state snapshots
- **S3**: Cross-region replication, versioning enabled

### Recovery Procedures
- **RTO**: 15 minutes for critical services
- **RPO**: 5 minutes maximum data loss
- **Multi-AZ**: Automatic failover for databases
- **Cross-Region**: Disaster recovery in secondary region

## Cost Optimization

### Resource Management
- **Spot Instances**: Non-critical workloads (30-50% cost savings)
- **Reserved Instances**: Predictable workloads (40-60% savings)
- **S3 Lifecycle**: Automatic tier transitions
- **Resource Tagging**: Cost allocation and tracking
- **Right-sizing**: Regular instance optimization

### Monitoring and Alerting
- **Cost Alerts**: Budget thresholds and anomaly detection
- **Resource Utilization**: Unused resource identification
- **Optimization Recommendations**: AWS Trusted Advisor integration

## Multi-Region Architecture

### Global Infrastructure
- **Primary Region**: us-east-1 (US East - N. Virginia)
- **Secondary Region**: us-west-2 (US West - Oregon)
- **Data Replication**: Cross-region replication for critical data
- **Global Load Balancing**: Route 53 health-based routing

### Regional Components
Each region contains:
- Complete EKS cluster with all microservices
- RDS read replicas for disaster recovery
- S3 buckets with cross-region replication
- Regional monitoring and logging

This architecture provides a robust, scalable, and secure foundation for the Solidity Security Platform, ensuring enterprise-grade reliability while maintaining cost efficiency and operational excellence.
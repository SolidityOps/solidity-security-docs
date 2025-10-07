# Deployment Documentation

This directory contains comprehensive deployment guides for the Advanced Blockchain Security platform infrastructure components.

## Available Guides

### [DNS Infrastructure](./dns-infrastructure.md)
Complete guide for deploying DNS infrastructure using Cloudflare and Terraform.

**Covers:**
- Domain configuration for `advancedblockchainsecurity.com`
- Production and staging environment setup
- SSL/TLS certificate management
- Security configurations (WAF, DDoS protection, geo-blocking)
- Kubernetes integration with AWS Load Balancer Controller
- Troubleshooting and maintenance procedures

**Prerequisites:**
- Cloudflare account with domain zone
- AWS EKS clusters with Load Balancer Controller
- Terraform 1.0+

### [Orchestration Service Deployment](./orchestration-service-deployment.md)
Complete guide for deploying the Celery-based scan orchestration service.

**Covers:**
- Gevent worker pool configuration for high concurrency
- Dual database session support (sync/async)
- RedBeat scheduler for distributed environments
- Slither integration for security analysis
- Monitoring with Flower and Prometheus
- Troubleshooting event loop conflicts and resource issues

**Prerequisites:**
- Kubernetes cluster (Minikube for local)
- PostgreSQL database
- Redis instance
- Docker for container builds

## Coming Soon

### Infrastructure Deployment
- AWS EKS cluster setup and configuration
- VPC and networking infrastructure
- Security groups and IAM configurations
- Storage and database deployment

### Kubernetes Deployment
- ArgoCD GitOps setup
- Application deployment manifests
- Service mesh configuration
- Monitoring and logging stack

### Monitoring Setup
- Prometheus and Grafana deployment
- Alerting configuration
- Log aggregation with ELK stack
- Performance monitoring setup

### Security Configuration
- HashiCorp Vault deployment
- Secret management workflows
- Security scanning integration
- Compliance monitoring

## Quick Start

For first-time deployment, follow this sequence:

1. **[DNS Infrastructure](./dns-infrastructure.md)** - Set up domain and DNS records
2. **Infrastructure** - Deploy AWS resources and EKS clusters
3. **Kubernetes** - Configure GitOps and deploy applications
4. **Monitoring** - Set up observability stack
5. **Security** - Configure Vault and security policies

## Support

For deployment issues or questions:
- **Documentation Issues**: Create issue in `solidity-security-docs` repository
- **Infrastructure Support**: infrastructure@advancedblockchainsecurity.com
- **Emergency Escalation**: Follow company incident response procedures
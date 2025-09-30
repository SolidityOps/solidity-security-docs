# DNS Infrastructure Deployment Guide

## Overview

This guide provides comprehensive instructions for deploying and managing the DNS infrastructure for `advancedblockchainsecurity.com` using Cloudflare as the DNS provider and Terraform for infrastructure automation.

## Architecture

The DNS infrastructure supports two environments:

### Production Environment
- **Root Domain**: `advancedblockchainsecurity.com`
- **API**: `api.advancedblockchainsecurity.com`
- **Dashboard**: `dashboard.advancedblockchainsecurity.com`
- **Vault**: `vault.advancedblockchainsecurity.com`
- **ArgoCD**: `argocd.advancedblockchainsecurity.com`
- **Monitoring**: `monitoring.advancedblockchainsecurity.com`

### Staging Environment
- **Staging Root**: `staging.advancedblockchainsecurity.com`
- **API**: `api-staging.advancedblockchainsecurity.com`
- **Dashboard**: `dashboard-staging.advancedblockchainsecurity.com`
- **Vault**: `vault-staging.advancedblockchainsecurity.com`
- **ArgoCD**: `argocd-staging.advancedblockchainsecurity.com`
- **Monitoring**: `monitoring-staging.advancedblockchainsecurity.com`

## Prerequisites

### 1. Domain Setup
- Domain `advancedblockchainsecurity.com` registered and active
- Domain added to Cloudflare account
- Nameservers changed to Cloudflare nameservers

### 2. Tools Required
```bash
# Install required tools
brew install terraform
brew install jq
brew install dig
```

### 3. AWS Infrastructure
- AWS Load Balancer Controller deployed in EKS clusters
- Application Load Balancers (ALBs) provisioned for staging and production
- Security groups configured for HTTP/HTTPS traffic

### 4. Cloudflare Configuration
- Cloudflare account with domain zone
- API token with Zone:Edit permissions

## Step-by-Step Deployment

### Step 1: Environment Configuration

Create a `.env` file with your configuration:

```bash
# .env file
export CLOUDFLARE_API_TOKEN="your_cloudflare_api_token_here"
export TF_VAR_staging_lb_ip="staging_alb_ip_here"
export TF_VAR_production_lb_ip="production_alb_ip_here"
```

Load the environment:
```bash
source .env
```

### Step 2: Get Load Balancer IPs

Retrieve your AWS Application Load Balancer IPs:

```bash
# For staging environment
kubectl get ingress -n staging-namespace -o wide

# For production environment
kubectl get ingress -n production-namespace -o wide

# Or use AWS CLI
aws elbv2 describe-load-balancers --names staging-alb --query 'LoadBalancers[0].DNSName'
aws elbv2 describe-load-balancers --names production-alb --query 'LoadBalancers[0].DNSName'
```

### Step 3: Deploy DNS Infrastructure

Navigate to the Cloudflare directory:
```bash
cd /Users/pwner/Git/ABS/solidity-security-aws-infrastructure/cloudflare
```

#### Option A: Automated Deployment (Recommended)
```bash
# Run the automated deployment script
./scripts/deploy-dns.sh
```

#### Option B: Manual Terraform Deployment
```bash
cd terraform

# Initialize Terraform
terraform init

# Review the deployment plan
terraform plan

# Apply the configuration
terraform apply
```

### Step 4: Validation

Run the comprehensive DNS validation:
```bash
# Run validation script
./scripts/validation/dns-validation.sh

# Manual validation commands
dig @8.8.8.8 advancedblockchainsecurity.com A
dig @1.1.1.1 api.advancedblockchainsecurity.com A
dig @8.8.8.8 staging.advancedblockchainsecurity.com A
```

## Security Configuration

### SSL/TLS Configuration
- **Encryption Mode**: Full (Strict)
- **Minimum TLS Version**: 1.2
- **HSTS**: Enabled with 31,536,000 seconds (1 year)
- **Always Use HTTPS**: Enabled

### Web Application Firewall (WAF)
- **WAF Rules**: Enabled
- **DDoS Protection**: Cloudflare's automatic DDoS protection
- **Rate Limiting**:
  - Production: 500 requests/minute, 5,000 requests/hour
  - Staging: 1,000 requests/minute, 10,000 requests/hour

### Geo-blocking (Production Only)
Blocked countries:
- China (CN)
- Russia (RU)
- North Korea (KP)

### Security Headers
All responses include:
```
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## Integration with Kubernetes

### Ingress Configuration

Use these annotations in your Kubernetes Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.advancedblockchainsecurity.com
    secretName: api-tls-secret
  rules:
  - host: api.advancedblockchainsecurity.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### cert-manager Configuration

Ensure cert-manager is configured with the proper ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@advancedblockchainsecurity.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: alb
```

## Monitoring and Maintenance

### Health Checks

Regular health check commands:
```bash
# DNS resolution check
./scripts/validation/dns-validation.sh

# SSL certificate validation
echo | openssl s_client -servername advancedblockchainsecurity.com -connect advancedblockchainsecurity.com:443 2>/dev/null | openssl x509 -noout -dates

# HTTP to HTTPS redirect test
curl -I http://advancedblockchainsecurity.com
```

### Performance Monitoring

```bash
# Response time test
curl -w "@curl-format.txt" -o /dev/null -s "https://advancedblockchainsecurity.com"

# DNS resolution time
dig advancedblockchainsecurity.com | grep "Query time"
```

### Backup and Recovery

#### DNS Records Backup
```bash
# Automatic backup during deployment
./scripts/deploy-dns.sh

# Manual backup
curl -X GET "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
     -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
     -H "Content-Type: application/json" > dns-backup-$(date +%Y%m%d).json
```

#### Terraform State Backup
```bash
# Backup Terraform state
cp terraform/terraform.tfstate terraform/terraform.tfstate.backup-$(date +%Y%m%d)
```

## Troubleshooting

### Common Issues

#### 1. DNS Not Resolving
**Symptoms**: Domain doesn't resolve or returns wrong IP
**Solutions**:
```bash
# Check nameservers
dig NS advancedblockchainsecurity.com

# Verify zone is active in Cloudflare
curl -X GET "https://api.cloudflare.com/client/v4/zones?name=advancedblockchainsecurity.com" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Check DNS propagation globally
dig @8.8.8.8 advancedblockchainsecurity.com
dig @1.1.1.1 advancedblockchainsecurity.com
```

#### 2. SSL Certificate Issues
**Symptoms**: SSL errors or certificate not trusted
**Solutions**:
```bash
# Check certificate details
echo | openssl s_client -servername advancedblockchainsecurity.com -connect advancedblockchainsecurity.com:443 2>/dev/null | openssl x509 -noout -text

# Verify cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check certificate status
kubectl get certificates -A
```

#### 3. Load Balancer Not Responding
**Symptoms**: 504 Gateway Timeout or connection refused
**Solutions**:
```bash
# Check ALB status
aws elbv2 describe-load-balancers --names staging-alb
aws elbv2 describe-target-groups --load-balancer-arn <alb-arn>

# Verify target health
aws elbv2 describe-target-health --target-group-arn <target-group-arn>

# Check security groups
aws ec2 describe-security-groups --group-ids <security-group-id>
```

#### 4. Terraform State Issues
**Symptoms**: Terraform state conflicts or corruption
**Solutions**:
```bash
# Import existing resources
terraform import cloudflare_record.root <record-id>

# Refresh state
terraform refresh

# Force unlock (if locked)
terraform force-unlock <lock-id>
```

### Emergency Procedures

#### DNS Rollback
```bash
# Restore from backup
terraform state push terraform.tfstate.backup-YYYYMMDD

# Or manual record restoration via API
curl -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
     -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
     -H "Content-Type: application/json" \
     --data '{"type":"A","name":"@","content":"BACKUP_IP","ttl":300}'
```

#### Infrastructure Destruction
```bash
# Destroy all DNS infrastructure (CAUTION)
cd terraform
terraform destroy

# Selective resource removal
terraform destroy -target=cloudflare_record.staging
```

## Cost Optimization

### Cloudflare Usage
- **DNS Queries**: Unlimited on all plans
- **Bandwidth**: Unlimited for proxied traffic
- **SSL Certificates**: Free with Universal SSL

### Terraform State Management
- Use remote state backend for team collaboration:
```hcl
terraform {
  backend "s3" {
    bucket = "abs-terraform-state"
    key    = "dns/terraform.tfstate"
    region = "us-west-2"
  }
}
```

## Future Enhancements

### Planned Improvements
1. **ExternalDNS Integration**: Automatic DNS record management from Kubernetes
2. **Multi-region Load Balancing**: Global traffic distribution
3. **Advanced WAF Rules**: Custom security rules based on traffic patterns
4. **API Rate Limiting**: Per-endpoint rate limiting
5. **DNS Analytics**: Advanced traffic and performance analytics

### Automation Opportunities
1. **CI/CD Integration**: Automated DNS deployment in GitOps workflows
2. **Monitoring Alerts**: Proactive DNS and SSL monitoring
3. **Auto-scaling**: Dynamic DNS record management based on load
4. **Disaster Recovery**: Automated failover DNS configurations

## Support and Documentation

### Additional Resources
- [Cloudflare API Documentation](https://api.cloudflare.com/)
- [Terraform Cloudflare Provider](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [cert-manager Documentation](https://cert-manager.io/docs/)

### Contact Information
For issues or questions regarding DNS infrastructure:
- **Infrastructure Team**: infrastructure@advancedblockchainsecurity.com
- **Emergency Escalation**: Use company incident response procedures
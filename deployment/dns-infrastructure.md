# DNS Infrastructure Deployment Guide

## Overview

This guide provides step-by-step instructions for deploying the DNS infrastructure for the Advanced Blockchain Security platform using Cloudflare and Terraform. The infrastructure supports both staging and production environments with comprehensive security configurations.

## Domain Configuration

**Primary Domain**: `advancedblockchainsecurity.com`

### Production Subdomains
- `advancedblockchainsecurity.com` - Main application
- `api.advancedblockchainsecurity.com` - API services
- `dashboard.advancedblockchainsecurity.com` - Dashboard interface
- `vault.advancedblockchainsecurity.com` - HashiCorp Vault UI
- `argocd.advancedblockchainsecurity.com` - ArgoCD GitOps
- `monitoring.advancedblockchainsecurity.com` - Grafana/Prometheus

### Staging Subdomains
- `staging.advancedblockchainsecurity.com` - Staging environment
- `api-staging.advancedblockchainsecurity.com` - Staging API
- `dashboard-staging.advancedblockchainsecurity.com` - Staging dashboard
- `vault-staging.advancedblockchainsecurity.com` - Staging Vault
- `argocd-staging.advancedblockchainsecurity.com` - Staging ArgoCD
- `monitoring-staging.advancedblockchainsecurity.com` - Staging monitoring

## Prerequisites

### 1. Domain and DNS Provider Setup
- [ ] Domain `advancedblockchainsecurity.com` registered
- [ ] Domain added to Cloudflare account
- [ ] Nameservers changed to Cloudflare nameservers
- [ ] Domain status shows "Active" in Cloudflare dashboard

### 2. Required Tools
```bash
# macOS installation
brew install terraform
brew install jq
brew install dig
brew install curl

# Verify installations
terraform version  # Should be 1.0+
jq --version
dig -v
```

### 3. AWS Infrastructure Prerequisites
- [ ] EKS clusters deployed for staging and production
- [ ] AWS Load Balancer Controller installed in both clusters
- [ ] Application Load Balancers (ALBs) provisioned
- [ ] Security groups configured for HTTP/HTTPS traffic

### 4. Cloudflare API Configuration
- [ ] Cloudflare account with domain zone
- [ ] API token created with Zone:Edit permissions
- [ ] Token permissions verified for the target domain

## Pre-Deployment Checklist

### AWS Load Balancer IP Collection
```bash
# Get staging ALB DNS name
kubectl get ingress -n staging --context=staging-cluster -o wide

# Get production ALB DNS name
kubectl get ingress -n production --context=production-cluster -o wide

# Alternative: Use AWS CLI
aws elbv2 describe-load-balancers --names staging-alb --query 'LoadBalancers[0].DNSName'
aws elbv2 describe-load-balancers --names production-alb --query 'LoadBalancers[0].DNSName'
```

### Environment Variables Setup
Create a secure environment configuration:

```bash
# Create .env file (never commit this file)
cat > .env << EOF
export CLOUDFLARE_API_TOKEN="your_cloudflare_api_token_here"
export TF_VAR_staging_lb_ip="staging_alb_dns_name_here"
export TF_VAR_production_lb_ip="production_alb_dns_name_here"
EOF

# Load environment variables
source .env

# Verify variables are set
echo "Cloudflare token: ${CLOUDFLARE_API_TOKEN:0:10}..."
echo "Staging LB: $TF_VAR_staging_lb_ip"
echo "Production LB: $TF_VAR_production_lb_ip"
```

## Deployment Process

### Step 1: Repository Setup
```bash
# Navigate to DNS infrastructure directory
cd /Users/pwner/Git/ABS/solidity-security-aws-infrastructure/cloudflare

# Verify directory structure
ls -la
# Should show: terraform/, dns-records/, subdomain-configs/, scripts/
```

### Step 2: Validate Cloudflare Access
```bash
# Test API connectivity
curl -X GET "https://api.cloudflare.com/client/v4/zones?name=advancedblockchainsecurity.com" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     -H "Content-Type: application/json" | jq '.success'

# Expected output: true
```

### Step 3: Terraform Initialization
```bash
cd terraform

# Initialize Terraform with required providers
terraform init

# Validate configuration syntax
terraform validate

# Format Terraform files
terraform fmt
```

### Step 4: Infrastructure Planning
```bash
# Create deployment plan
terraform plan -out=dns-plan

# Review plan output carefully
# Look for:
# - DNS record creations
# - Security policy configurations
# - SSL certificate setups
```

### Step 5: Infrastructure Deployment
```bash
# Apply the infrastructure changes
terraform apply dns-plan

# Monitor output for any errors
# Deployment typically takes 2-5 minutes
```

### Step 6: Automated Deployment (Alternative)
For a fully automated deployment experience:

```bash
# Use the automated deployment script
./scripts/deploy-dns.sh

# With options
./scripts/deploy-dns.sh --dry-run  # Plan only
./scripts/deploy-dns.sh --force    # Skip confirmations
```

## Post-Deployment Validation

### Step 1: DNS Resolution Testing
```bash
# Run comprehensive validation
./scripts/validation/dns-validation.sh

# Manual checks for key domains
dig @8.8.8.8 advancedblockchainsecurity.com A
dig @1.1.1.1 api.advancedblockchainsecurity.com A
dig @8.8.8.8 staging.advancedblockchainsecurity.com A
```

### Step 2: SSL Certificate Verification
```bash
# Check SSL certificate status
echo | openssl s_client -servername advancedblockchainsecurity.com -connect advancedblockchainsecurity.com:443 2>/dev/null | openssl x509 -noout -dates

# Verify certificate chain
curl -I https://advancedblockchainsecurity.com
```

### Step 3: HTTP to HTTPS Redirect Testing
```bash
# Test automatic HTTPS redirect
curl -I http://advancedblockchainsecurity.com
# Should return 301/302 redirect to HTTPS

# Test staging redirect
curl -I http://staging.advancedblockchainsecurity.com
```

### Step 4: Global DNS Propagation Check
```bash
# Check propagation across multiple DNS servers
for dns in 8.8.8.8 1.1.1.1 208.67.222.222 9.9.9.9; do
  echo "Testing DNS server: $dns"
  dig @$dns advancedblockchainsecurity.com A +short
done
```

## Security Configuration

### Web Application Firewall (WAF)
The deployment automatically configures:
- **DDoS Protection**: Cloudflare's automatic protection
- **Rate Limiting**:
  - Production: 500 req/min, 5,000 req/hour
  - Staging: 1,000 req/min, 10,000 req/hour
- **Geo-blocking**: China, Russia, North Korea (production only)

### SSL/TLS Configuration
- **Encryption Mode**: Full (Strict)
- **Minimum TLS Version**: 1.2
- **HSTS**: Enabled with 1-year max-age
- **Certificate Authority**: Let's Encrypt
- **Auto-renewal**: Enabled

### Security Headers
All responses include enhanced security headers:
```
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## Kubernetes Integration

### Ingress Configuration Template
Use this template for your Kubernetes Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    cert-manager.io/cluster-issuer: letsencrypt-prod
    alb.ingress.kubernetes.io/tags: Environment=production,Project=ABS
spec:
  tls:
  - hosts:
    - api.advancedblockchainsecurity.com
    secretName: api-tls-secret
  rules:
  - host: api.advancedblockchainsecurity.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /api/v1/auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080
```

### cert-manager ClusterIssuer
Ensure cert-manager is configured:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-admin-email@your-domain.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: alb
```

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: DNS Records Not Resolving
**Symptoms**: `nslookup` or `dig` returns no results

**Diagnosis**:
```bash
# Check nameservers
dig NS advancedblockchainsecurity.com

# Verify zone status
curl -X GET "https://api.cloudflare.com/client/v4/zones?name=advancedblockchainsecurity.com" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | jq '.result[0].status'
```

**Solutions**:
1. Verify nameservers are set to Cloudflare
2. Wait for DNS propagation (up to 24 hours)
3. Check Cloudflare zone status is "active"

#### Issue 2: SSL Certificate Not Working
**Symptoms**: SSL/TLS errors, certificate warnings

**Diagnosis**:
```bash
# Check certificate details
echo | openssl s_client -servername api.advancedblockchainsecurity.com -connect api.advancedblockchainsecurity.com:443 2>/dev/null | openssl x509 -noout -text

# Check cert-manager status
kubectl get certificates -A
kubectl describe certificate api-tls-secret -n production
```

**Solutions**:
1. Verify cert-manager is running
2. Check ClusterIssuer configuration
3. Ensure domain validation is complete
4. Review Let's Encrypt rate limits

#### Issue 3: Load Balancer Not Responding
**Symptoms**: 504 Gateway Timeout, connection refused

**Diagnosis**:
```bash
# Check ALB status
aws elbv2 describe-load-balancers --names production-alb

# Verify target groups
aws elbv2 describe-target-groups --load-balancer-arn <alb-arn>

# Check target health
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

**Solutions**:
1. Verify ALB is provisioned and active
2. Check security group configurations
3. Ensure target groups have healthy targets
4. Review Kubernetes service configurations

#### Issue 4: Terraform State Issues
**Symptoms**: State lock errors, resource conflicts

**Diagnosis**:
```bash
# Check Terraform state
terraform show
terraform state list

# Verify remote state backend
terraform init -backend-config="check=true"
```

**Solutions**:
```bash
# Force unlock if necessary
terraform force-unlock <lock-id>

# Import existing resources
terraform import cloudflare_record.api <record-id>

# Refresh state
terraform refresh
```

### Emergency Procedures

#### DNS Rollback Process
```bash
# Option 1: Terraform rollback
cd terraform
terraform state push terraform.tfstate.backup-YYYYMMDD

# Option 2: Manual record restoration
curl -X POST "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
     -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
     -H "Content-Type: application/json" \
     --data '{"type":"A","name":"api","content":"BACKUP_IP","ttl":300}'
```

#### Complete Infrastructure Destruction
```bash
# CAUTION: This will destroy all DNS infrastructure
cd terraform
terraform destroy

# Confirm destruction
terraform state list  # Should be empty
```

## Monitoring and Maintenance

### Regular Health Checks
Create a monitoring script:

```bash
#!/bin/bash
# dns-health-check.sh

# Check key domains
DOMAINS=("advancedblockchainsecurity.com" "api.advancedblockchainsecurity.com" "staging.advancedblockchainsecurity.com")

for domain in "${DOMAINS[@]}"; do
  echo "Checking $domain..."

  # DNS resolution
  dig @8.8.8.8 "$domain" A +short

  # SSL certificate
  echo | timeout 5 openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null | openssl x509 -noout -dates

  # HTTP response
  curl -I "https://$domain" 2>/dev/null | head -1

  echo "---"
done
```

### Performance Monitoring
```bash
# DNS response time
dig advancedblockchainsecurity.com | grep "Query time"

# HTTP response time
curl -w "@curl-format.txt" -o /dev/null -s "https://advancedblockchainsecurity.com"
```

### Backup Strategy
```bash
# Automated backup script
#!/bin/bash
BACKUP_DIR="./backups/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup DNS records
curl -X GET "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
     -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" > "$BACKUP_DIR/dns-records.json"

# Backup Terraform state
cp terraform/terraform.tfstate "$BACKUP_DIR/"

echo "Backup completed: $BACKUP_DIR"
```

## Performance Optimization

### Caching Configuration
- **Browser Cache TTL**: 4 hours (14,400 seconds)
- **Edge Cache TTL**: 2 hours (7,200 seconds)
- **API Cache TTL**: 5 minutes (300 seconds)

### Compression Settings
- **Brotli Compression**: Enabled
- **Gzip Compression**: Enabled
- **Minification**: HTML, CSS, JavaScript

### Geographic Distribution
Cloudflare automatically distributes content across global edge locations for optimal performance.

## Cost Management

### Cloudflare Costs
- **DNS Queries**: Unlimited on all plans
- **Bandwidth**: Unlimited for proxied traffic
- **SSL Certificates**: Free with Universal SSL
- **Estimated Monthly Cost**: $0-20 depending on plan

### AWS Costs
- **Application Load Balancer**: ~$25/month per ALB
- **Data Transfer**: $0.09/GB for outbound traffic
- **Route 53**: $0.50 per hosted zone (if used)

## Security Best Practices

### API Token Management
- [ ] Use dedicated API tokens for infrastructure automation
- [ ] Implement least-privilege permissions
- [ ] Rotate tokens quarterly
- [ ] Monitor API token usage

### Access Control
- [ ] Enable two-factor authentication on Cloudflare account
- [ ] Use role-based access control for team members
- [ ] Regularly audit access permissions
- [ ] Implement IP restrictions where possible

### Monitoring and Alerting
- [ ] Set up Cloudflare Analytics monitoring
- [ ] Configure alerts for DNS failures
- [ ] Monitor SSL certificate expiration
- [ ] Track security events and anomalies

## Next Steps

After successful DNS deployment:

1. **Deploy Kubernetes Applications**: Configure Ingress resources with proper annotations
2. **Set Up Monitoring**: Deploy Grafana dashboards for DNS and SSL monitoring
3. **Configure CI/CD**: Integrate DNS management into GitOps workflows
4. **Implement ExternalDNS**: For automatic DNS record management from Kubernetes
5. **Set Up Disaster Recovery**: Configure backup DNS providers and failover procedures

## Support and Resources

### Documentation Links
- [Cloudflare API Documentation](https://api.cloudflare.com/)
- [Terraform Cloudflare Provider](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [cert-manager Documentation](https://cert-manager.io/docs/)

### Emergency Contacts
- **Infrastructure Team**: [Contact via internal ticketing system](https://intranet.advancedblockchainsecurity.com/support)
- **DevOps Support**: [Contact via internal ticketing system](https://intranet.advancedblockchainsecurity.com/support)
- **Security Team**: [Contact via internal ticketing system](https://intranet.advancedblockchainsecurity.com/support)

---

**Version**: 1.0
**Last Updated**: September 2025
**Next Review**: October 2025
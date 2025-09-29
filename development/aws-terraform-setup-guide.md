# AWS CLI and Terraform CLI Setup Guide for macOS with zsh

This guide provides comprehensive instructions for setting up AWS CLI and Terraform CLI on macOS with zsh shell, including troubleshooting steps and integration with Claude Code.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS CLI Installation](#aws-cli-installation)
3. [Terraform CLI Installation](#terraform-cli-installation)
4. [Configuration](#configuration)
5. [Troubleshooting](#troubleshooting)
6. [Claude Code Integration](#claude-code-integration)
7. [Common Issues and Solutions](#common-issues-and-solutions)

## Prerequisites

Before starting, ensure you have:
- macOS with zsh shell (default on macOS Catalina+)
- Homebrew package manager
- An AWS account with appropriate permissions
- Administrative access to your macOS system

### Install Homebrew (if not installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Verify installation:
```bash
brew --version
```

## AWS CLI Installation

### Method 1: Using Homebrew (Recommended)

```bash
# Install AWS CLI v2
brew install awscli

# Verify installation
aws --version
```

### Method 2: Using Official Installer

```bash
# Download and install
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify installation
aws --version
```

### Post-Installation Setup

Add AWS CLI to your PATH in `~/.zshrc`:

```bash
# Add to ~/.zshrc
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

## Terraform CLI Installation

### Method 1: Using Homebrew (Recommended)

```bash
# Install Terraform
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify installation
terraform version
```

### Method 2: Manual Installation

```bash
# Download latest version (replace with current version)
TERRAFORM_VERSION="1.6.0"
curl -O https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_darwin_amd64.zip

# Extract and install
unzip terraform_${TERRAFORM_VERSION}_darwin_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform version
```

### Enable Auto-completion for zsh

```bash
# Add to ~/.zshrc
echo 'autoload -U +X bashcompinit && bashcompinit' >> ~/.zshrc
echo 'complete -o nospace -C /usr/local/bin/terraform terraform' >> ~/.zshrc
source ~/.zshrc
```

## Configuration

### AWS CLI Configuration

#### Method 1: Interactive Configuration

```bash
aws configure
```

You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region name (e.g., `us-west-2`)
- Default output format (recommended: `json`)

#### Method 2: Using Environment Variables

Add to `~/.zshrc`:

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-west-2"
export AWS_DEFAULT_OUTPUT="json"
```

#### Method 3: Using AWS SSO

```bash
# Configure SSO
aws configure sso

# Login to SSO
aws sso login --profile your-profile-name
```

### Verify AWS Configuration

```bash
# Test AWS connectivity
aws sts get-caller-identity

# List S3 buckets (if you have permission)
aws s3 ls
```

### Terraform Configuration

Create a basic Terraform configuration file to test:

```bash
mkdir -p ~/terraform-test
cd ~/terraform-test

cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

# Test resource - S3 bucket
resource "aws_s3_bucket" "test_bucket" {
  bucket = "my-terraform-test-bucket-${random_string.suffix.result}"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

output "bucket_name" {
  value = aws_s3_bucket.test_bucket.id
}
EOF
```

Test Terraform:

```bash
# Initialize Terraform
terraform init

# Plan deployment
terraform plan

# Apply (optional - creates real resources)
# terraform apply

# Clean up (if you applied)
# terraform destroy
```

## Troubleshooting

### AWS CLI Issues

#### Issue: Command not found

```bash
# Check if AWS CLI is in PATH
which aws

# If not found, add to PATH
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

#### Issue: Permission denied

```bash
# Fix permissions
sudo chown -R $(whoami) /usr/local/bin/aws
chmod +x /usr/local/bin/aws
```

#### Issue: SSL Certificate errors

```bash
# Update certificates
brew install ca-certificates

# Or use specific certificate bundle
export AWS_CA_BUNDLE=/usr/local/etc/openssl/cert.pem
```

### Terraform Issues

#### Issue: Provider download failures

```bash
# Clear Terraform cache
rm -rf .terraform
rm .terraform.lock.hcl

# Reinitialize
terraform init
```

#### Issue: Version conflicts

```bash
# Check Terraform version
terraform version

# Upgrade if needed
brew upgrade hashicorp/tap/terraform
```

#### Issue: State file locks

```bash
# Force unlock (use carefully)
terraform force-unlock <LOCK_ID>
```

### Network and Connection Problems

#### Issue: Corporate firewall/proxy

```bash
# Configure proxy in ~/.zshrc
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,.local"

# For AWS CLI
aws configure set default.proxy_url http://proxy.company.com:8080
```

#### Issue: DNS resolution problems

```bash
# Test DNS resolution
nslookup amazonaws.com

# Use different DNS if needed
sudo networksetup -setdnsservers Wi-Fi 8.8.8.8 8.8.4.4
```

## Claude Code Integration

Claude Code can help you manage AWS infrastructure using Terraform. Here's how to set it up:

### 1. Project Structure

Organize your Terraform files:

```
project-root/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── scripts/
│   ├── deploy.sh
│   └── destroy.sh
└── README.md
```

### 2. Enable Claude Code for Terraform Commands

Create a deployment script that Claude Code can execute:

```bash
# Create scripts directory
mkdir -p scripts

# Create deploy script
cat > scripts/deploy.sh << 'EOF'
#!/bin/bash
set -e

echo "🚀 Starting Terraform deployment..."

# Navigate to terraform directory
cd terraform

# Initialize Terraform
echo "📦 Initializing Terraform..."
terraform init

# Validate configuration
echo "✅ Validating Terraform configuration..."
terraform validate

# Plan deployment
echo "📋 Planning deployment..."
terraform plan -out=tfplan

# Apply if plan succeeds
echo "🎯 Applying Terraform plan..."
terraform apply tfplan

echo "✅ Deployment completed successfully!"
EOF

chmod +x scripts/deploy.sh
```

### 3. Claude Code Usage Examples

When working with Claude Code, you can ask it to:

#### Deploy Infrastructure

```
Claude, please deploy the AWS infrastructure using the Terraform files in the terraform/ directory.
```

Claude Code will:
1. Navigate to the terraform directory
2. Run `terraform init`
3. Run `terraform plan`
4. Run `terraform apply` (with confirmation)

#### Update Infrastructure

```
Claude, please update the EC2 instance type in main.tf to t3.medium and apply the changes.
```

Claude Code will:
1. Edit the Terraform file
2. Run `terraform plan` to show changes
3. Apply the updates

#### Troubleshoot Issues

```
Claude, the Terraform apply is failing with a permission error. Please help debug this.
```

Claude Code will:
1. Check AWS credentials
2. Validate IAM permissions
3. Suggest fixes

### 4. Best Practices with Claude Code

1. **Always review plans**: Ask Claude Code to show the plan before applying
2. **Use version control**: Ensure Terraform state is properly managed
3. **Set up remote state**: Use S3 backend for team collaboration
4. **Implement proper IAM**: Use least privilege principle

Example remote state configuration:

```hcl
terraform {
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "infrastructure/terraform.tfstate"
    region = "us-west-2"

    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

## Common Issues and Solutions

### Issue 1: "Unable to locate credentials"

**Problem**: AWS CLI or Terraform can't find AWS credentials.

**Solutions**:
```bash
# Check current configuration
aws configure list

# Reconfigure
aws configure

# Or use environment variables
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"

# For temporary credentials
export AWS_SESSION_TOKEN="your-token"
```

### Issue 2: "Region not specified"

**Problem**: No default region configured.

**Solutions**:
```bash
# Set default region
aws configure set region us-west-2

# Or use environment variable
export AWS_DEFAULT_REGION="us-west-2"
```

### Issue 3: "Terraform backend initialization failed"

**Problem**: S3 backend bucket doesn't exist or insufficient permissions.

**Solutions**:
```bash
# Create S3 bucket for state
aws s3 mb s3://your-terraform-state-bucket

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket your-terraform-state-bucket \
    --versioning-configuration Status=Enabled

# Create DynamoDB table for locking
aws dynamodb create-table \
    --table-name terraform-state-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

### Issue 4: "Permission denied" errors

**Problem**: Insufficient IAM permissions.

**Solutions**:
1. Check your IAM user/role permissions
2. Ensure you have the necessary policies attached
3. Use AWS CloudTrail to see what permissions are being denied

```bash
# Check your identity
aws sts get-caller-identity

# List attached policies
aws iam list-attached-user-policies --user-name your-username
```

### Issue 5: "Terraform state locked"

**Problem**: Terraform state is locked by another operation.

**Solutions**:
```bash
# Check lock info
terraform show

# Force unlock (use with caution)
terraform force-unlock <LOCK_ID>
```

### Issue 6: macOS Gatekeeper blocking Terraform

**Problem**: macOS prevents Terraform from running due to security settings.

**Solutions**:
```bash
# Allow Terraform to run
sudo spctl --add /usr/local/bin/terraform
sudo spctl --enable --label "Terraform"

# Or disable Gatekeeper temporarily (not recommended)
sudo spctl --master-disable
```

### Issue 7: Homebrew installation issues

**Problem**: Homebrew can't install AWS CLI or Terraform.

**Solutions**:
```bash
# Update Homebrew
brew update

# Fix permissions
sudo chown -R $(whoami) /usr/local/var/homebrew

# Clear cache
brew cleanup

# Reinstall
brew uninstall awscli terraform
brew install awscli
brew install hashicorp/tap/terraform
```

## Additional Resources

- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- [Terraform Documentation](https://www.terraform.io/docs/)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Claude Code Documentation](https://claude.ai/code)

## Support

If you encounter issues not covered in this guide:

1. Check the official documentation
2. Search for solutions on Stack Overflow
3. Consult your team's internal documentation
4. Ask Claude Code for specific troubleshooting help

Remember to never commit sensitive information like AWS credentials to version control!
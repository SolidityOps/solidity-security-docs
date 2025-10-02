# AWS IAM Permissions Guide for Solidity Security Platform Infrastructure

## Overview

This guide provides the complete AWS IAM permissions required to deploy and manage the Solidity Security Platform infrastructure across Sprint 1-18. The infrastructure spans multiple AWS services and requires specific permissions for Terraform deployment, Kubernetes management, and ongoing operations.

## Quick Setup - Recommended IAM Policy

For rapid deployment, attach this comprehensive policy to your AWS CLI user:

### Option A: Admin Access (Development/Testing)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

**Use Case**: Development, testing, initial setup
**Security**: Highest access level - suitable for development environments

### Option B: Production-Ready Least Privilege Policy

For production deployments, use the detailed policy below that includes only required permissions.

## Infrastructure Components & Required Permissions

Based on analysis of Sprint tasks 1.1-1.18, the following AWS services are deployed:

### Core Infrastructure Services
- **VPC & Networking** (Task 1.2)
- **EKS Clusters** (Task 1.5)
- **ElastiCache Redis** (Task 1.3)
- **PostgreSQL StatefulSets** (Kubernetes-based)
- **Application Load Balancers** (Task 1.6)
- **CloudWatch Monitoring** (Task 1.7)
- **IAM Roles & Policies** (Task 1.6)
- **Secrets Manager** (Task 1.4)
- **ECR Container Registry** (Task 1.8)

## Complete IAM Policy - Production Ready

**Note**: Due to AWS IAM policy size limits (6144 characters), this has been split into two policies.

### Policy 1: Core Infrastructure (Attach to user/role)

Policy Name: `CoreInfrastructure`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VPCNetworkingPermissions",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateVpc",
                "ec2:CreateSubnet",
                "ec2:CreateInternetGateway",
                "ec2:CreateNatGateway",
                "ec2:CreateRouteTable",
                "ec2:CreateRoute",
                "ec2:CreateSecurityGroup",
                "ec2:CreateNetworkAcl",
                "ec2:CreateVpcEndpoint",
                "ec2:DeleteVpcEndpoints",
                "ec2:CreateFlowLogs",
                "ec2:AttachInternetGateway",
                "ec2:AssociateRouteTable",
                "ec2:AssociateSubnetCidrBlock",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:RevokeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupEgress",
                "ec2:ModifyVpcAttribute",
                "ec2:ModifySubnetAttribute",
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets",
                "ec2:DescribeInternetGateways",
                "ec2:DescribeNatGateways",
                "ec2:DescribeRouteTables",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeNetworkAcls",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeFlowLogs",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeRegions",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeKeyPairs",
                "ec2:DescribeAddresses",
                "ec2:AllocateAddress",
                "ec2:ReleaseAddress",
                "ec2:AssociateAddress",
                "ec2:DisassociateAddress",
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:DeleteVpc",
                "ec2:DeleteSubnet",
                "ec2:DeleteInternetGateway",
                "ec2:DeleteNatGateway",
                "ec2:DeleteRouteTable",
                "ec2:DeleteRoute",
                "ec2:DeleteSecurityGroup",
                "ec2:DeleteNetworkAcl",
                "ec2:DeleteFlowLogs",
                "ec2:DetachInternetGateway",
                "ec2:DisassociateRouteTable",
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "ec2:StartInstances",
                "ec2:StopInstances"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EKSClusterPermissions",
            "Effect": "Allow",
            "Action": [
                "eks:CreateCluster",
                "eks:CreateNodegroup",
                "eks:CreateAddon",
                "eks:DescribeCluster",
                "eks:DescribeNodegroup",
                "eks:DescribeAddon",
                "eks:ListClusters",
                "eks:ListNodegroups",
                "eks:ListAddons",
                "eks:ListUpdates",
                "eks:UpdateClusterConfig",
                "eks:UpdateClusterVersion",
                "eks:UpdateNodegroupConfig",
                "eks:UpdateNodegroupVersion",
                "eks:UpdateAddon",
                "eks:DeleteCluster",
                "eks:DeleteNodegroup",
                "eks:DeleteAddon",
                "eks:TagResource",
                "eks:UntagResource",
                "eks:ListTagsForResource"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ElastiCachePermissions",
            "Effect": "Allow",
            "Action": [
                "elasticache:CreateCacheCluster",
                "elasticache:CreateReplicationGroup",
                "elasticache:CreateCacheSubnetGroup",
                "elasticache:CreateCacheParameterGroup",
                "elasticache:CreateCacheSecurityGroup",
                "elasticache:DescribeCacheClusters",
                "elasticache:DescribeReplicationGroups",
                "elasticache:DescribeCacheSubnetGroups",
                "elasticache:DescribeCacheParameterGroups",
                "elasticache:DescribeCacheSecurityGroups",
                "elasticache:DescribeCacheParameters",
                "elasticache:DescribeEvents",
                "elasticache:DescribeSnapshots",
                "elasticache:ListTagsForResource",
                "elasticache:ModifyCacheCluster",
                "elasticache:ModifyReplicationGroup",
                "elasticache:ModifyCacheSubnetGroup",
                "elasticache:ModifyCacheParameterGroup",
                "elasticache:DeleteCacheCluster",
                "elasticache:DeleteReplicationGroup",
                "elasticache:DeleteCacheSubnetGroup",
                "elasticache:DeleteCacheParameterGroup",
                "elasticache:DeleteCacheSecurityGroup",
                "elasticache:AddTagsToResource",
                "elasticache:RemoveTagsFromResource",
                "elasticache:CreateSnapshot",
                "elasticache:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMPermissions",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:CreatePolicy",
                "iam:CreateInstanceProfile",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:AttachUserPolicy",
                "iam:DetachUserPolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRole",
                "iam:GetPolicy",
                "iam:GetInstanceProfile",
                "iam:GetRolePolicy",
                "iam:ListRoles",
                "iam:ListPolicies",
                "iam:ListInstanceProfiles",
                "iam:ListRolePolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListAttachedUserPolicies",
                "iam:UpdateRole",
                "iam:UpdateAssumeRolePolicy",
                "iam:DeleteRole",
                "iam:DeletePolicy",
                "iam:DeleteInstanceProfile",
                "iam:AddRoleToInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:TagRole",
                "iam:UntagRole",
                "iam:TagPolicy",
                "iam:UntagPolicy",
                "iam:TagInstanceProfile",
                "iam:UntagInstanceProfile",
                "iam:CreateOpenIDConnectProvider",
                "iam:DeleteOpenIDConnectProvider",
                "iam:GetOpenIDConnectProvider",
                "iam:ListOpenIDConnectProviders",
                "iam:TagOpenIDConnectProvider",
                "iam:UntagOpenIDConnectProvider"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMServiceLinkedRolePermissions",
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "eks.amazonaws.com",
                        "eks-nodegroup.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "autoscaling.amazonaws.com",
                        "elasticache.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Sid": "IAMPassRolePermissions",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": [
                        "eks.amazonaws.com",
                        "ec2.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "autoscaling.amazonaws.com",
                        "ecs.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Sid": "ApplicationLoadBalancerPermissions",
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:CreateLoadBalancer",
                "elasticloadbalancing:CreateTargetGroup",
                "elasticloadbalancing:CreateListener",
                "elasticloadbalancing:CreateRule",
                "elasticloadbalancing:DescribeLoadBalancers",
                "elasticloadbalancing:DescribeTargetGroups",
                "elasticloadbalancing:DescribeListeners",
                "elasticloadbalancing:DescribeRules",
                "elasticloadbalancing:DescribeTargetHealth",
                "elasticloadbalancing:DescribeLoadBalancerAttributes",
                "elasticloadbalancing:DescribeTargetGroupAttributes",
                "elasticloadbalancing:ModifyLoadBalancerAttributes",
                "elasticloadbalancing:ModifyTargetGroupAttributes",
                "elasticloadbalancing:ModifyListener",
                "elasticloadbalancing:ModifyRule",
                "elasticloadbalancing:RegisterTargets",
                "elasticloadbalancing:DeregisterTargets",
                "elasticloadbalancing:DeleteLoadBalancer",
                "elasticloadbalancing:DeleteTargetGroup",
                "elasticloadbalancing:DeleteListener",
                "elasticloadbalancing:DeleteRule",
                "elasticloadbalancing:AddTags",
                "elasticloadbalancing:RemoveTags",
                "elasticloadbalancing:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```

### Policy 2: Additional Services (Attach to same user/role)

Policy Name: `AdditionalServices`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SecretsManagerPermissions",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:CreateSecret",
                "secretsmanager:DescribeSecret",
                "secretsmanager:GetSecretValue",
                "secretsmanager:PutSecretValue",
                "secretsmanager:UpdateSecret",
                "secretsmanager:DeleteSecret",
                "secretsmanager:RestoreSecret",
                "secretsmanager:ListSecrets",
                "secretsmanager:TagResource",
                "secretsmanager:UntagResource",
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:PutResourcePolicy",
                "secretsmanager:DeleteResourcePolicy"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ECRPermissions",
            "Effect": "Allow",
            "Action": [
                "ecr:CreateRepository",
                "ecr:DescribeRepositories",
                "ecr:DescribeImages",
                "ecr:DescribeImageScanFindings",
                "ecr:ListImages",
                "ecr:GetRepositoryPolicy",
                "ecr:SetRepositoryPolicy",
                "ecr:DeleteRepositoryPolicy",
                "ecr:SetRepositoryPolicy",
                "ecr:GetLifecyclePolicy",
                "ecr:PutLifecyclePolicy",
                "ecr:DeleteLifecyclePolicy",
                "ecr:DeleteRepository",
                "ecr:TagResource",
                "ecr:UntagResource",
                "ecr:ListTagsForResource",
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3BucketPermissions",
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:DeleteBucket",
                "s3:GetBucketLocation",
                "s3:GetBucketVersioning",
                "s3:GetEncryptionConfiguration",
                "s3:GetBucketPublicAccessBlock",
                "s3:PutBucketVersioning",
                "s3:PutEncryptionConfiguration",
                "s3:PutBucketPublicAccessBlock",
                "s3:PutBucketPolicy",
                "s3:GetBucketPolicy",
                "s3:DeleteBucketPolicy",
                "s3:ListBucket",
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:GetObjectVersion",
                "s3:DeleteObjectVersion"
            ],
            "Resource": "*"
        },
        {
            "Sid": "CloudFormationPermissions",
            "Effect": "Allow",
            "Action": [
                "cloudformation:CreateStack",
                "cloudformation:UpdateStack",
                "cloudformation:DeleteStack",
                "cloudformation:DescribeStacks",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStackResources",
                "cloudformation:GetTemplate",
                "cloudformation:ListStacks",
                "cloudformation:ValidateTemplate"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AutoScalingPermissions",
            "Effect": "Allow",
            "Action": [
                "autoscaling:CreateAutoScalingGroup",
                "autoscaling:CreateLaunchConfiguration",
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:UpdateAutoScalingGroup",
                "autoscaling:DeleteAutoScalingGroup",
                "autoscaling:DeleteLaunchConfiguration",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*"
        },
        {
            "Sid": "STSPermissions",
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity",
                "sts:AssumeRole",
                "sts:AssumeRoleWithWebIdentity",
                "sts:GetAccessKeyInfo",
                "sts:GetSessionToken"
            ],
            "Resource": "*"
        }
    ]
}
```

## Service-Specific Permission Breakdown

### Task 1.2 - VPC & Networking
**Required Services**: EC2 (VPC, Subnets, Security Groups, NAT Gateways)
**Key Permissions**: `ec2:Create*`, `ec2:Describe*`, `ec2:Modify*`, `ec2:Delete*`

### Task 1.3 - Database & Cache
**Required Services**: ElastiCache, RDS (for subnet groups)
**Key Permissions**: `elasticache:*`, `rds:CreateDBSubnetGroup`

### Task 1.4 - Secret Management
**Required Services**: AWS Secrets Manager, IAM
**Key Permissions**: `secretsmanager:*`, `iam:CreateRole`

### Task 1.5 - EKS Clusters
**Required Services**: EKS, EC2 (for nodes), IAM
**Key Permissions**: `eks:*`, `iam:CreateRole`, `iam:AttachRolePolicy`

### Task 1.6 - Load Balancers
**Required Services**: Elastic Load Balancing v2, IAM
**Key Permissions**: `elasticloadbalancing:*`, `iam:CreateRole` (for IRSA)

### Task 1.7 - Monitoring
**Required Services**: CloudWatch Logs
**Key Permissions**: `logs:*`

### Task 1.8 - Container Registry
**Required Services**: ECR
**Key Permissions**: `ecr:*`

## Security Considerations

### Least Privilege Principles
1. **Resource-Specific Policies**: For production, scope policies to specific resource ARNs
2. **Condition-Based Access**: Add conditions for IP restrictions, time-based access
3. **Regular Policy Audits**: Review and remove unused permissions quarterly

### Example Resource-Scoped Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:*"
            ],
            "Resource": [
                "arn:aws:eks:us-west-2:*:cluster/solidity-security-*",
                "arn:aws:eks:us-west-2:*:nodegroup/solidity-security-*"
            ]
        }
    ]
}
```

## Setup Instructions

### Step 1: Create IAM User
```bash
# Create new IAM user for infrastructure deployment
aws iam create-user --user-name terraform-deploy-user
```

### Step 2: Attach Policy
```bash
# Option A: Attach admin policy (development)
aws iam attach-user-policy --user-name terraform-deploy-user --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Option B: Create and attach custom policies (production)
# Create Policy 1 (Core Infrastructure)
aws iam create-policy --policy-name SoliditySecurityInfrastructurePolicy1 --policy-document file://infrastructure-policy-1.json
aws iam attach-user-policy --user-name terraform-deploy-user --policy-arn arn:aws:iam::ACCOUNT-ID:policy/SoliditySecurityInfrastructurePolicy1

# Create Policy 2 (Additional Services)
aws iam create-policy --policy-name SoliditySecurityInfrastructurePolicy2 --policy-document file://infrastructure-policy-2.json
aws iam attach-user-policy --user-name terraform-deploy-user --policy-arn arn:aws:iam::ACCOUNT-ID:policy/SoliditySecurityInfrastructurePolicy2
```

### Step 3: Generate Access Keys
```bash
aws iam create-access-key --user-name terraform-deploy-user
```

### Step 4: Configure AWS CLI
```bash
aws configure --profile infrastructure
# Enter Access Key ID
# Enter Secret Access Key
# Enter Region: us-west-2
# Enter Output Format: json
```

### Step 5: Test Permissions
```bash
# Test basic access
aws sts get-caller-identity --profile infrastructure

# Test VPC creation (will not actually create)
aws ec2 describe-vpcs --profile infrastructure
```

## Common Permission Errors & Solutions

### Error: `AccessDenied: User is not authorized to perform: iam:CreateRole`
**Solution**: Add IAM permissions to the policy or use admin access for initial setup

### Error: `DBSubnetGroupDoesNotCoverEnoughAZs`
**Solution**: This is a configuration issue, not permissions. Ensure subnets span multiple AZs

### Error: `RequestLimitExceeded`
**Solution**: Add retry logic or implement exponential backoff in Terraform

## Terraform Backend Permissions

If using S3 backend for Terraform state:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::terraform-state-bucket"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::terraform-state-bucket/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/terraform-state-lock"
        }
    ]
}
```

## Monitoring & Troubleshooting

### CloudTrail Integration
Enable CloudTrail to monitor API calls made during infrastructure deployment:

```bash
aws cloudtrail create-trail --name infrastructure-audit-trail --s3-bucket-name audit-logs-bucket
```

### Permission Boundary (Optional)
For additional security, set permission boundaries:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": ["us-west-2", "us-east-1"]
                }
            }
        }
    ]
}
```

## Next Steps

1. **Create IAM user** with appropriate permissions
2. **Configure AWS CLI** with new credentials
3. **Test permissions** with basic AWS commands
4. **Deploy infrastructure** starting with Task 1.2 (VPC)
5. **Monitor usage** through CloudTrail and billing alerts

For questions or issues, refer to the specific Sprint task documentation in `/Users/pwner/Git/ABS/docs/Sprints/`.
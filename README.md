# 🚀 AWS Production Infrastructure — Terraform

> Production-grade 2-tier AWS infrastructure built with Terraform.  
> Covers VPC, ALB, Auto Scaling Group, NAT Gateway, Security Groups, S3, and EC2 (Nginx).

---

## 📁 Repository Structure

```
.
├── README.md
├── task.md
└── code/
    ├── DEMO.md
    ├── main.tf
    ├── vpc.tf
    ├── alb.tf
    ├── asg.tf
    ├── s3.tf
    ├── security_groups.tf
    ├── backend.tf
    ├── variables.tf
    ├── outputs.tf
    ├── dev.tfvars
    ├── test.tfvars
    ├── prod.tfvars
    └── scripts/
        └── user_data.sh
```

| File | Purpose |
|---|---|
| `main.tf` | Provider config and root module |
| `vpc.tf` | VPC, subnets, IGW, NAT Gateway, route tables |
| `alb.tf` | Application Load Balancer, target group, listener |
| `asg.tf` | Auto Scaling Group and launch template |
| `s3.tf` | S3 bucket for assets and ALB logs |
| `security_groups.tf` | Security group rules for ALB and EC2 |
| `backend.tf` | Remote state backend configuration |
| `variables.tf` | Input variable declarations |
| `outputs.tf` | Output values (ALB DNS, VPC ID, subnet IDs) |
| `dev.tfvars` | Variable values for dev environment |
| `test.tfvars` | Variable values for test environment |
| `prod.tfvars` | Variable values for prod environment |
| `scripts/user_data.sh` | EC2 bootstrap script — installs and starts Nginx |

---

## 🏗️ Architecture Overview

| Layer | Component | Detail |
|---|---|---|
| **Network** | VPC | `10.0.0.0/16` across 2 AZs |
| | Public Subnets | `10.0.1.0/24`, `10.0.2.0/24` |
| | Private Subnets | `10.0.11.0/24`, `10.0.12.0/24` |
| | NAT Gateway | Outbound access for private subnets |
| **Compute** | Auto Scaling Group | 2–6 instances, CPU-based scaling |
| | Launch Template | Ubuntu + Nginx (`t2.micro`) |
| **Traffic** | Application Load Balancer | HTTP :80, health checks |
| **Storage** | S3 Bucket | Assets and ALB logs |

---

## ✅ Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.0
- AWS CLI configured (`aws configure`)
- IAM permissions for: VPC, EC2, ASG, ALB, S3, EIP

---

## 🚦 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>/code
```

### 2. Initialise

```bash
terraform init
```

### 3. Validate

```bash
terraform validate
# Expected: Success! The configuration is valid.
```

### 4. Plan — target environment

```bash
# Dev
terraform plan -var-file="dev.tfvars" -out=tfplan

# Test
terraform plan -var-file="test.tfvars" -out=tfplan

# Prod
terraform plan -var-file="prod.tfvars" -out=tfplan
```

> Expect ~30–35 resources to be created.

### 5. Apply

```bash
terraform apply tfplan
# Duration: ~5–7 minutes
```

### 6. Test

```bash
ALB_DNS=$(terraform output -raw load_balancer_dns)
sleep 180  # wait for instances to become healthy
curl http://$ALB_DNS
```

Expected response:
```html
<h1>Welcome to the AWS Infrastructure Project - Proper 2-Tier Architecture (Ubuntu)</h1>
```

---

## 🔍 Verify Deployment

**Auto Scaling Group instances**
```bash
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(AutoScalingGroupName, 'app-asg')].Instances[*].[InstanceId,HealthStatus,AvailabilityZone]" \
  --output table
```

**Load Balancer target health**
```bash
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName, 'app-tg')].TargetGroupArn" \
  --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

**VPC and subnets**
```bash
VPC_ID=$(terraform output -raw vpc_id)
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]" \
  --output table
```

---

## 🧹 Teardown

```bash
terraform destroy -var-file="dev.tfvars"
# Type 'yes' when prompted — takes ~5–7 minutes
# Note: NAT Gateway deletion can take 3–5 minutes
```

Confirm cleanup:
```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=AWS-Production-Infrastructure"
```

---

## 💰 Estimated Cost

| Resource | Monthly |
|---|---|
| EC2 t2.micro × 2 | ~$15 |
| ALB | ~$20 |
| NAT Gateway | ~$35 |
| Data transfer + S3 | ~$11 |
| **Total** | **~$80/month** |

> **Demo cost** (destroyed within 1 hour): ~$0.15

---

## 🐛 Troubleshooting

**Instances not healthy**
```bash
aws ec2 describe-security-groups \
  --filters "Name=tag:Project,Values=AWS-Production-Infrastructure"

# Check Nginx on instance
sudo systemctl status nginx
sudo tail -f /var/log/cloud-init-output.log
```

**ALB returning 503** — wait 3–5 minutes, then check target health:
```bash
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[0].TargetGroupArn" --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

**Apply fails — verify credentials and quotas**
```bash
aws sts get-caller-identity
aws service-quotas get-service-quota \
  --service-code ec2 --quota-code L-1216C47A
```

---

## 🔑 Key Learning Points

- **Infrastructure as Code** — all resources declared, versioned, and reproducible
- **Multi-environment** — separate tfvars for dev / test / prod with identical code
- **High Availability** — multi-AZ deployment with automatic failover
- **Auto Scaling** — capacity adjusts dynamically to CPU demand
- **Network Isolation** — public/private subnet split with NAT Gateway
- **Load Balancing** — traffic distribution with built-in health monitoring
- **State Management** — remote backend tracks infrastructure state

---

## 📚 References

- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS VPC Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)

---

> **Demo duration:** 15–20 min &nbsp;|&nbsp; **Level:** Intermediate &nbsp;|&nbsp; **Updated:** May 2026

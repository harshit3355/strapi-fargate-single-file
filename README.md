# Strapi on ECS Fargate — Single-File Terraform Stack

The whole AWS environment for a containerised [Strapi](https://strapi.io/) CMS on **ECS Fargate**, expressed as one `main.tf`: VPC, subnets, security groups, ALB, ECS cluster, task definition, service, and a Route 53 record.

This is the compact version of the stack. If you want the same infrastructure split into readable per-concern files with separate IAM definitions, see [`strapi-ecs-fargate-terraform`](https://github.com/harshit3355/strapi-ecs-fargate-terraform).

## Why this exists

Deploying a container to Fargate touches six or seven AWS services that all have to agree with each other — the task needs a subnet, the subnet needs a route, the ALB needs two availability zones, the target group needs the right port and target type, DNS needs the ALB alias. Getting that combination right once and keeping it in version control is the entire value here.

Reach for this version when you want the smallest thing that stands up a working Fargate service and you would rather read one file than nine.

## What it builds

All sixteen resources live in `main.tf`:

| Layer | Resources |
| --- | --- |
| **Network** | VPC `10.0.0.0/16`, internet gateway, two public subnets (`ap-south-1a`, `ap-south-1b`), route table and associations |
| **Security** | Security groups for the ALB and for the ECS tasks |
| **Ingress** | Application Load Balancer, target group, listener |
| **Compute** | ECS cluster, Fargate task definition, ECS service |
| **DNS** | Route 53 record aliasing a subdomain to the ALB |

Output: `service_url`, the public HTTP address of the load balancer.

## Prerequisites

- Terraform 1.x with AWS credentials that can create VPC, ECS, ELB, and Route 53 resources
- A container image the task can pull (Docker Hub or ECR)
- A Route 53 hosted zone you control
- An S3 bucket for remote state

## Usage

**1. Point the backend at your own bucket**

`backend.tf` is checked in with a specific bucket name. Change it before the first `init`:

```hcl
terraform {
  backend "s3" {
    bucket = "your-state-bucket"
    region = "ap-south-1"
    key    = "terraform.tfstate"
  }
}
```

**2. Set your variables**

`terraform.tfvars` in this repository holds working example values. Replace them with your own:

```hcl
aws_region      = "ap-south-1"
route53_zone_id = "<your-hosted-zone-id>"
subdomain       = "strapi.example.com"
s3              = "<your-bucket>"
dockerimage     = "<your-registry>/strapi:latest"
```

**3. Apply**

```bash
terraform init
terraform plan
terraform apply
terraform output service_url
```

**4. Destroy when finished**

```bash
terraform destroy
```

The ALB and Fargate tasks bill hourly whether or not anyone visits.

## Notes and limits

- **HTTP only.** The listener is plain port 80 and the output URL is `http://`. Add an ACM certificate and an HTTPS listener before putting anything real behind it.
- **Tasks run in public subnets** with public IPs, so they can reach the registry without a NAT gateway. Cheap and simple; not what you want for production. Move to private subnets plus NAT or VPC endpoints when it matters.
- **No database.** Strapi falls back to SQLite inside the container, which is wiped whenever the task is replaced. Add RDS before storing real content.
- **Region and CIDRs are hardcoded** in `main.tf`.
- **No state locking.** Add a DynamoDB lock table to the backend before more than one person runs `apply`.
- The committed `terraform.tfvars` contains a real hosted zone ID and account number from the original deployment. They are not secrets, but replace them rather than inheriting them.

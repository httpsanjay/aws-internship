# Production-Grade AWS IAM Policies Matrix (3-Tier Web Application)

This repository contains a strictly decoupled, least-privilege **AWS Identity and Access Management (IAM)** policy configuration optimized for a small engineering team running a highly available 3-tier web application (Application Load Balancer → Amazon EC2 instances → Amazon RDS PostgreSQL → Amazon S3).

---

## 📁 Repository Structure

```text
aws-iam-policies-repo/
├── README.md                  # Comprehensive architectural note and execution steps
└── policies/
    ├── application-developer-role.json  # Read/Write app code deployment & S3 static assets
    ├── database-administrator-role.json # Stateful relational engine management & automated backups
    └── platform-engineer-role.json       # Core network topology, routing fabrics, and hardware capacity
```

---

## 🛠️ Role Definitions & Policy JSONs

### 1. Platform Engineer (`policies/platform-engineer-role.json`)
* **Core Mandate:** Manages the baseline infrastructural fabrics, network borders, routing matrices, and compute layers.
* **Deliberately Cannot Do:** This role is completely blind to data storage layers. It **cannot read, write, modify, or delete any data objects inside Amazon S3**, nor can it access, scale, or snapshot the **Amazon RDS databases**. 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCNetworkManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:CreateRouteTable",
        "ec2:AssociateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": [
        "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0123456789abcdef0",
        "arn:aws:ec2:us-east-1:123456789012:subnet/subnet-*",
        "arn:aws:ec2:us-east-1:123456789012:route-table/rtb-*",
        "arn:aws:ec2:us-east-1:123456789012:security-group/sg-*"
      ]
    },
    {
      "Sid": "ComputeCapacityManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "arn:aws:ec2:us-east-1:123456789012:instance/*"
    },
    {
      "Sid": "LoadBalancerManagement",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:ModifyLoadBalancerAttributes",
        "elasticloadbalancing:CreateListener",
        "elasticloadbalancing:DeleteListener"
      ],
      "Resource": "arn:aws:ec2:us-east-1:123456789012:loadbalancer/app/production-alb/*"
    }
  ]
}
```

---

### 2. Database Administrator (`policies/database-administrator-role.json`)
* **Core Mandate:** Orchestrates stateful database layers, handles replication configuration, cluster patches, tuning parameters, and backups.
* **Deliberately Cannot Do:** This role **cannot alter network security groups or route tables**, eliminating the risk of accidentally exposing database ports to the open internet. It possesses no permissions on the compute (EC2/ALB) or object storage (S3) tiers.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManagedDatabaseLifecycle",
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBInstance",
        "rds:DeleteDBInstance",
        "rds:ModifyDBInstance",
        "rds:StartDBInstance",
        "rds:StopDBInstance",
        "rds:RebootDBInstance"
      ],
      "Resource": "arn:aws:ec2:us-east-1:123456789012:db:production-postgres-db"
    },
    {
      "Sid": "DatabaseBackupManagement",
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:DeleteDBSnapshot",
        "rds:ModifyDBSnapshotAttribute"
      ],
      "Resource": "arn:aws:ec2:us-east-1:123456789012:snapshot:production-snapshot-*"
    }
  ]
}
```

---

### 3. Application Developer (`policies/application-developer-role.json`)
* **Core Mandate:** Facilitates continuous integration/deployment of application level packages, handles static file uploads, and reads basic performance observability metrics.
* **Deliberately Cannot Do:** This role **cannot provision, resize, or delete core compute/network nodes**, and has zero visibility into database systems. It also **cannot change high-level bucket configurations** (such as turning off AWS Block Public Access or modifying Bucket Policies).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StaticAssetObjectStoreAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::production-app-static-assets",
        "arn:aws:s3:::production-app-static-assets/*"
      ]
    },
    {
      "Sid": "DiagnosticReadPerformanceMonitoring",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics"
      ],
      "Resource": "arn:aws:cloudwatch:us-east-1:123456789012:metric/*"
    }
  ]
}
```

---

## 🎯 Architectural Reasoning & Security Posture

* **Zero Resource Wildcards (`"Resource": "*"`):** Broad resource wildcards are strictly prohibited. Every single statement targets granular, specific ARN strings (or patterns constrained by exact account, region, or prefix identifiers). This blocks cross-service entitlement creep out of the box.
* **Separation of Concerns:** By explicitly defining boundaries between Platform Engineers, DBAs, and Developers, a security breach or human error in one role cannot collapse the entire structural footprint. 
* **Data Layer Isolation:** Infrastructure provisioning actions (`ec2:RunInstances`) are completely isolated from analytical data reading actions (`s3:GetObject`), ensuring infrastructure operators have zero implicit access to client data payloads.

---

## 🚀 How to Validate & Run This Repository

To test these policies locally without deploying them live to an AWS environment, follow these standard steps:

### Prerequisites
* Install the AWS CLI.
* (Optional) Install `jq` to format shell outputs cleanly.

### Step 1: Clone the Repository
```bash
git clone <your-submitted-public-github-repo-url>
cd aws-iam-policies-repo
```

### Step 2: Validate IAM Policy Grammar via AWS CLI
Run the `aws iam validate-policy` utility to inspect your local JSON files for structural errors, syntactic compliance, and policy format alignment before trying to build them:

```bash
aws iam validate-policy --policy-document file://policies/platform-engineer-role.json
aws iam validate-policy --policy-document file://policies/database-administrator-role.json
aws iam validate-policy --policy-document file://policies/application-developer-role.json
```
*If a policy is valid, the CLI returns an empty JSON payload `{}`. If it finds errors, a detailed validation report is generated.*

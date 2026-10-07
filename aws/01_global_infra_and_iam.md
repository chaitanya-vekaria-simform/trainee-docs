# 🌐 AWS Notes — Stage 1: Global Infrastructure & IAM

---

## 1. What is AWS?

Amazon Web Services (AWS) is a cloud computing platform offering 200+ services
(compute, storage, databases, networking, AI, DevOps tools) on a pay-as-you-go basis.
You rent infrastructure instead of buying it.

```
Traditional (On-Premise)          AWS Cloud
┌──────────────────────┐          ┌──────────────────────┐
│ Buy servers          │          │ Rent on demand       │
│ Provision manually   │          │ Provision via API    │
│ Over-provision (idle)│    vs    │ Auto-scale           │
│ Fixed capacity       │          │ Elastic capacity     │
│ You manage hardware  │          │ AWS manages hardware │
│ CapEx heavy          │          │ OpEx (pay per use)   │
└──────────────────────┘          └──────────────────────┘
```

---

## 2. Global Infrastructure

### 2.1 Region

**Definition:** A physical geographic area where AWS has multiple data centers.

```
Example regions:
  us-east-1        → N. Virginia (US East)
  us-west-2        → Oregon (US West)
  eu-west-1        → Ireland (Europe)
  ap-south-1       → Mumbai (Asia Pacific)
  ap-southeast-1   → Singapore
```

**Why Regions matter:**
- Data residency: keep data in a specific country (GDPR compliance)
- Latency: serve users from the nearest region
- Service availability: not all services exist in all regions
- Pricing varies by region (us-east-1 is typically cheapest)

**Beginner example:**
> You're building an app for Indian users. Deploy in `ap-south-1` (Mumbai)
> instead of `us-east-1` (Virginia) to reduce latency from ~200ms to ~10ms.

**DevOps Gotcha:**
> ⚠️ **Region is not auto-selected globally.** Every AWS CLI command,
> Terraform resource, and Console action runs in the CURRENT region.
> Always explicitly set: `aws configure set region ap-south-1` or
> use `--region` flag. Forgetting this causes "Resource not found"
> errors when resources are in a different region.

**Best Practice:**
> Use `us-east-1` as your primary region for new projects — it gets
> new AWS features first and has the widest service coverage.

---

### 2.2 Availability Zone (AZ)

**Definition:** One or more discrete data centers within a Region, each with
independent power, cooling, and networking. Physically separated but connected
via high-bandwidth, low-latency links.

```
Region: ap-south-1 (Mumbai)
┌────────────────────────────────────────────────┐
│                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌────────┐ │
│  │  ap-south-1a│  │  ap-south-1b│  │...1c   │ │
│  │  Data Center│  │  Data Center│  │        │ │
│  │  A          │  │  B          │  │        │ │
│  └──────┬──────┘  └──────┬──────┘  └───┬────┘ │
│         └────────────────┴─────────────┘      │
│              High-speed private links          │
└────────────────────────────────────────────────┘
```

**Why AZs matter:**
- High Availability: if one AZ goes down, others keep running
- Deploy your app across 2+ AZs for production resilience

**Beginner example:**
> Run your EC2 web server in `ap-south-1a` AND `ap-south-1b`.
> If the Mumbai-A data center loses power, your app keeps running in Mumbai-B.

**DevOps Gotcha:**
> ⚠️ EBS volumes are AZ-scoped. A volume in `us-east-1a` cannot be
> attached to an EC2 instance in `us-east-1b`. If you need cross-AZ
> storage, use EFS (shared) or S3 (regional).

**Best Practice:**
> Always deploy production workloads across a minimum of 2 AZs.
> Use an Application Load Balancer (ALB) to distribute traffic between AZs.

---

### 2.3 Edge Location / Points of Presence (PoP)

**Definition:** Data centers used by CloudFront (CDN) and Route 53 (DNS)
to serve content closer to end users. There are 400+ edge locations worldwide
vs ~35 regions.

```
User in Chennai
      │
      ▼
CloudFront Edge (Chennai PoP)     ← cached here
      │ (cache miss only)
      ▼
Origin (S3 or EC2 in Mumbai)
```

**Why Edge Locations matter:**
- Static content (images, JS, CSS) is cached at the nearest edge
- Reduces latency from hundreds of ms to single digits
- Shields origin from traffic spikes

---

### 2.4 Local Zone

**Definition:** An extension of an AWS Region placed closer to a specific city.
Useful when you need <10ms latency to a city that doesn't have a full region.

```
Example: AWS Local Zone in Delhi (extension of Mumbai region)
Users in Delhi → Local Zone (Delhi) → <5ms
vs
Users in Delhi → Mumbai region → ~20ms
```

**Use case:** Real-time gaming, live video streaming, industrial IoT in specific cities.

---

### 2.5 Wavelength Zone

**Definition:** AWS infrastructure embedded within telecom providers' 5G networks.
Allows apps to reach mobile devices with <10ms latency.

**Use case:** Autonomous vehicles, AR/VR on 5G, real-time bidding.

---

### 2.6 AWS Outposts

**Definition:** AWS-managed hardware rack installed in YOUR data center.
Brings AWS services (EC2, RDS, S3, etc.) on-premises.

```
Your Data Center
┌──────────────────────────────────┐
│  ┌────────────────────────────┐  │
│  │   AWS Outpost Rack         │  │
│  │   (managed by AWS)         │  │
│  │   Runs: EC2, RDS, EKS, S3 │  │
│  └────────────────────────────┘  │
│                                  │
│  Connected to AWS Region via     │
│  Direct Connect or internet      │
└──────────────────────────────────┘
```

**Use case:** Ultra-low latency requirements, data residency laws preventing cloud.

---

## 3. IAM — Identity and Access Management

### 3.1 What is IAM?

**Definition:** The AWS service that controls WHO (identity) can do WHAT (action)
on WHICH (resource) AWS resource. Everything in AWS requires IAM authorization.

```
Every AWS API call:
  WHO    → IAM Identity (User / Role / Service)
  WHAT   → Action (s3:GetObject, ec2:StartInstances)
  WHERE  → Resource (arn:aws:s3:::my-bucket/*)
  RESULT → Allow or Deny
```

---

### 3.2 IAM User

**Definition:** A permanent identity for a person or application with
long-term credentials (password + access key).

```
IAM User: alice
  ├── Password (for Console login)
  ├── Access Key ID: AKIAIOSFODNN7EXAMPLE
  ├── Secret Access Key: wJalrXUtnFEMI...
  └── Attached policies → what alice can do
```

**Beginner example:**
> Create an IAM User "alice" for your developer. Give her the
> `AmazonS3ReadOnlyAccess` policy so she can list and download
> S3 objects but NOT upload or delete them.

**DevOps Gotcha:**
> ⚠️ Never use the ROOT account for day-to-day operations.
> Root has unrestricted access to everything including billing.
> Enable MFA on root immediately and lock it away.
> Create an IAM user with Admin access for daily use instead.

**Best Practice:**
> Rotate access keys every 90 days.
> Delete unused access keys immediately.
> Never commit access keys to Git (use AWS Secrets Manager or IAM Roles instead).

---

### 3.3 IAM Group

**Definition:** A collection of IAM users that share the same permissions.
Policies attached to a group apply to all its members.

```
Group: developers
├── Policy: AmazonEC2FullAccess
├── Policy: AmazonS3ReadOnlyAccess
└── Members: alice, bob, charlie
    All three can manage EC2 and read S3.
```

**Why Groups matter:**
- Change one group policy → affects all members instantly
- Easier than attaching policies to each user individually
- Users can belong to multiple groups

---

### 3.4 IAM Role

**Definition:** A temporary identity with permissions that can be ASSUMED
by AWS services, EC2 instances, Lambda functions, or other accounts.
No long-term credentials — AWS issues short-lived tokens automatically.

```
Without IAM Role (BAD):
  EC2 instance → hardcoded AWS_ACCESS_KEY in code → S3
  Risk: key exposed in GitHub, logs, AMI

With IAM Role (GOOD):
  EC2 instance → assumes role automatically → gets temp token → S3
  No secrets anywhere in code
```

**Common role use cases:**
```
EC2 Role       → EC2 instance can read from S3
Lambda Role    → Lambda can write to DynamoDB
ECS Task Role  → Container can access Secrets Manager
Cross-Account  → Account A can deploy to Account B's S3
OIDC Role      → GitHub Actions can deploy to AWS without stored keys
```

**Beginner example:**
> Your Lambda function needs to read from DynamoDB.
> Instead of hardcoding access keys in Lambda code,
> attach an IAM Role with `AmazonDynamoDBReadOnlyAccess` to the Lambda.
> Lambda automatically gets temporary credentials.

**DevOps Gotcha:**
> ⚠️ IAM Role assumption can fail if the **trust policy** is misconfigured.
> Trust policy = who CAN assume this role.
> Permissions policy = what the role CAN DO once assumed.
> Both must be correct. Common mistake: attaching only permissions
> but forgetting the trust relationship.

```json
Trust Policy example (allows EC2 to assume this role):
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Best Practice:**
> Always use IAM Roles instead of IAM Users for:
> - EC2 instances
> - Lambda functions
> - ECS/EKS workloads
> - CI/CD pipelines (use OIDC federation)

---

### 3.5 IAM Policy

**Definition:** A JSON document that defines permissions.
Each statement says: Allow/Deny + Action(s) + Resource(s).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Policy evaluation logic:**
```
Is there an explicit DENY?  → YES → DENY (always wins)
                            ↓ NO
Is there an explicit ALLOW? → YES → ALLOW
                            ↓ NO
                              DENY (implicit deny — default)
```

**Types of policies:**
```
AWS Managed     → Created and maintained by AWS (e.g., AmazonS3FullAccess)
Customer Managed→ Created by you, reusable across identities
Inline          → Embedded directly in one user/role (not reusable)
Resource-Based  → Attached to a resource (e.g., S3 bucket policy)
SCP             → Service Control Policy (via AWS Organizations)
```

**DevOps Gotcha:**
> ⚠️ **Explicit Deny always overrides Allow.**
> If an SCP denies `ec2:*` at the org level, no IAM policy can override it.
> Debug permission issues from outermost (SCP) to innermost (inline policy).

**Best Practice:**
> Use the **Principle of Least Privilege**: grant only the exact permissions needed.
> Use IAM Access Analyzer to detect overly permissive policies.
> Prefer AWS Managed policies for standard roles; write Customer Managed
> for fine-grained control.

---

### 3.6 IAM Policy — Conditions

**Definition:** Conditions add context-based rules to policies.

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": "203.0.113.0/24"
    },
    "StringEquals": {
      "aws:RequestedRegion": "ap-south-1"
    },
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

**Common condition keys:**
```
aws:SourceIp              → Client IP must match
aws:RequestedRegion       → Action must be in this region
aws:MultiFactorAuthPresent→ MFA must be active
aws:CurrentTime           → Time-based access
aws:TagKeys               → Resource must have specific tags
s3:prefix                 → Restrict S3 path prefix
ec2:Region                → EC2 in specific region
```

---

### 3.7 IAM Identity Center (formerly SSO)

**Definition:** Centralized single sign-on for all AWS accounts and business apps.
One login → access multiple AWS accounts with different roles.

```
Employee logs into Identity Center portal
  ↓
Selects account: "Production" → assumes PowerUserRole
Selects account: "Dev"        → assumes AdminRole
Selects account: "Audit"      → assumes ReadOnlyRole

All from one portal, one login.
```

**Why it matters for DevOps:**
- Developers don't need individual IAM Users in each account
- Integrates with Active Directory / Okta / Google Workspace
- Replaces manual IAM User creation across 50+ accounts

---

### 3.8 ARN — Amazon Resource Name

**Definition:** A unique identifier for every AWS resource.

```
Format:
arn:aws:<service>:<region>:<account-id>:<resource>

Examples:
arn:aws:s3:::my-bucket                              → S3 bucket (no region, no account)
arn:aws:s3:::my-bucket/path/to/file.txt             → S3 object
arn:aws:ec2:us-east-1:123456789012:instance/i-12345 → EC2 instance
arn:aws:iam::123456789012:role/MyLambdaRole         → IAM Role (no region for IAM)
arn:aws:lambda:us-east-1:123456789012:function:myFn → Lambda function
arn:aws:rds:ap-south-1:123456789012:db:mydb         → RDS database

Wildcards:
arn:aws:s3:::my-bucket/*    → all objects in bucket
arn:aws:ec2:us-east-1:*:*   → all EC2 resources in us-east-1 across all accounts
```

**DevOps Gotcha:**
> ⚠️ IAM global services (IAM, STS, Route 53) have NO region in ARN.
> Getting the ARN format wrong in policies causes silent permission failures.

---

## 4. AWS Organizations

### 4.1 What is AWS Organizations?

**Definition:** A service to manage multiple AWS accounts as a single organization.
Enables consolidated billing, centralized governance, and account vending.

```
Root (Management Account)
├── OU: Security
│   └── Account: Audit (CloudTrail logs, GuardDuty)
├── OU: Infrastructure
│   ├── Account: Networking (VPCs, Transit Gateway)
│   └── Account: Shared Services (CI/CD, artifact repos)
├── OU: Workloads
│   ├── OU: Production
│   │   ├── Account: Prod-App-1
│   │   └── Account: Prod-App-2
│   └── OU: Development
│       ├── Account: Dev-App-1
│       └── Account: Dev-App-2
└── Account: Billing (consolidated billing only)
```

---

### 4.2 OU — Organizational Unit

**Definition:** A folder for grouping AWS accounts within an Organization.
Policies (SCPs) applied to an OU affect ALL accounts within it.

---

### 4.3 SCP — Service Control Policy

**Definition:** A guardrail policy that restricts what IAM identities CAN DO
even if they have IAM Admin access. Applied at OU or account level.

```
SCP on Production OU:
{
  "Effect": "Deny",
  "Action": [
    "ec2:TerminateInstances",    ← devs can't terminate prod instances
    "s3:DeleteBucket",           ← devs can't delete prod buckets
    "iam:CreateUser"             ← no local IAM users in prod
  ],
  "Resource": "*"
}
```

**Key insight:** SCPs do NOT grant permissions. They only restrict what the
maximum possible permissions can be.

```
Max permissions possible = SCP allows ∩ IAM Policy allows

SCP allows: EC2, S3, RDS
IAM Policy allows: EC2, S3, Lambda
Result: EC2, S3 only (Lambda is blocked by SCP)
```

**DevOps Best Practice:**
> Use SCPs to enforce:
> - Deny resource creation outside approved regions
> - Deny disabling CloudTrail
> - Deny root account API calls
> - Require tags on all resources
> - Prevent IAM User creation (force SSO usage)

---

### 4.4 AWS Control Tower

**Definition:** A service that automates setting up a multi-account environment
following AWS best practices. Sits on top of AWS Organizations.

```
Control Tower sets up:
├── Landing Zone (multi-account structure)
├── Guardrails (mandatory SCPs + Config rules)
├── Account Factory (self-service account vending)
└── Log Archive + Audit accounts automatically
```

**Beginner example:**
> Instead of manually creating 20 AWS accounts, setting up SCPs,
> enabling CloudTrail, and configuring billing alerts — Control Tower
> does all of that automatically with a few clicks.

---

## 5. AWS STS — Security Token Service

**Definition:** Issues temporary, short-lived security credentials.
Every role assumption, cross-account access, and federated login uses STS.

```
Operation: AssumeRole

  Your Identity                    Target Role
  ┌──────────┐   AssumeRole    ┌─────────────────┐
  │ IAM User │ ─────────────→  │ IAM Role        │
  │ alice    │                 │ MyProductionRole │
  └──────────┘                 └────────┬────────┘
                                        │
                                        ▼
                               Temporary credentials:
                               AccessKeyId      (expires in 1hr)
                               SecretAccessKey  (expires in 1hr)
                               SessionToken     (expires in 1hr)
```

**Common STS operations:**
```
AssumeRole           → Assume a role (cross-account, same account)
AssumeRoleWithWebIdentity → OIDC federation (GitHub Actions, K8s IRSA)
AssumeRoleWithSAML   → SAML federation (Active Directory)
GetCallerIdentity    → Who am I? (debugging)
GetSessionToken      → Get temp credentials for current user
```

**CLI: Who am I?**
```bash
aws sts get-caller-identity
# Output:
# {
#   "UserId": "AIDAIOSFODNN7EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/alice"
# }
```

**DevOps Gotcha:**
> ⚠️ Temporary credentials have a session duration (default 1hr, max 12hr for roles).
> Long-running CI/CD pipelines can fail mid-way when credentials expire.
> Use `--duration-seconds` to extend, or refresh credentials via re-assume.

---

## 6. AWS Account vs AWS Region vs Resource — Summary

```
┌──────────────────────────────────────────────────────────────┐
│  AWS ACCOUNT (123456789012)                                  │
│                                                              │
│  ┌────────────────────────┐  ┌─────────────────────────┐   │
│  │  Region: us-east-1     │  │  Region: ap-south-1     │   │
│  │                        │  │                         │   │
│  │  AZ: us-east-1a  ─┐   │  │  AZ: ap-south-1a ─┐    │   │
│  │  AZ: us-east-1b  ─┤   │  │  AZ: ap-south-1b ─┤    │   │
│  │  AZ: us-east-1c  ─┘   │  │                   │    │   │
│  │                        │  │  EC2, RDS, ECS    │    │   │
│  │  EC2, RDS, S3, VPC    │  └─────────────────────────┘   │
│  └────────────────────────┘                                │
│                                                              │
│  GLOBAL resources (no region):                              │
│  IAM, Route 53, CloudFront, AWS Organizations              │
│                                                              │
│  REGIONAL resources:                                         │
│  S3 (namespace is global, data is regional)                 │
│  VPC, EC2, RDS, Lambda, ECS, SQS, SNS...                   │
│                                                              │
│  AZ-scoped resources:                                        │
│  EBS volumes, EC2 instances, RDS instances                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Shared Responsibility Model

**Definition:** Defines what AWS is responsible for vs what YOU are responsible for.

```
                    AWS RESPONSIBILITY
┌────────────────────────────────────────────────┐
│  Physical security of data centers             │
│  Hardware (servers, networking, storage)        │
│  Hypervisor / virtualization layer             │
│  Managed service software (RDS engine, etc.)   │
└────────────────────────────────────────────────┘
                   ↑  The "Cloud" itself

                    YOUR RESPONSIBILITY
┌────────────────────────────────────────────────┐
│  IaaS (EC2):  OS, patches, app, data, network  │
│  PaaS (RDS):  DB data, access control, backups │
│  SaaS (S3):   Data, bucket policy, encryption  │
│                                                │
│  ALWAYS YOU:                                   │
│  ├── IAM (users, roles, policies)             │
│  ├── Data encryption (at rest + in transit)   │
│  ├── Security groups and NACLs                │
│  ├── Application security                     │
│  └── Compliance of your data                  │
└────────────────────────────────────────────────┘
```

**Beginner example:**
> AWS ensures the physical server your EC2 runs on is secure.
> But if you leave port 22 open to 0.0.0.0/0 on your Security Group
> and someone brute-forces your SSH — that's YOUR responsibility.

---

## 8. Billing & Cost Concepts

### 8.1 Free Tier

AWS offers a Free Tier with three types:
```
Always Free     → Never expires (Lambda: 1M requests/month, DynamoDB: 25GB)
12-Month Free   → First 12 months after account creation (EC2 t2.micro 750hr/mo)
Trial           → Short-term free trial for specific services
```

**DevOps Gotcha:**
> ⚠️ Free tier limits apply PER ACCOUNT, not per resource.
> Running two t2.micro instances = 1,500hrs/month = 750hrs over free tier.
> Set up a Billing Alert immediately after creating an account.

### 8.2 Pricing Models

```
On-Demand       → Pay per second/hour, no commitment, most expensive
Reserved (RI)   → 1yr or 3yr commitment, 30-60% cheaper than On-Demand
Savings Plans   → Like RIs but more flexible (applies to any EC2 size/region)
Spot            → Spare AWS capacity, up to 90% cheaper, can be interrupted
Dedicated Host  → Physical server reserved for you (compliance/licensing)
```

### 8.3 AWS Cost Explorer

**Definition:** Tool to visualize, analyze, and forecast AWS spending.

**Best Practice:**
> Set billing alerts:
> `CloudWatch → Billing → Create alarm → $10 threshold`
> Create budgets in AWS Budgets for each team/project.

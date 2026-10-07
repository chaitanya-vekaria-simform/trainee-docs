# ⚙️ AWS Notes — Stage 2: Compute Services

---

## 1. EC2 — Elastic Compute Cloud

### 1.1 What is EC2?

**Definition:** Virtual machines (instances) running in AWS. The most fundamental
compute service — you get a server you control completely (OS, software, networking).

```
Traditional Server          EC2 Instance
┌──────────────┐            ┌──────────────────────────┐
│ Physical box │            │ Virtual Machine           │
│ Fixed specs  │   vs       │ Choose CPU/RAM/Storage    │
│ Buy upfront  │            │ Pay per second            │
│ Weeks to get │            │ Running in ~60 seconds    │
└──────────────┘            └──────────────────────────┘
```

---

### 1.2 EC2 Instance Types

**Definition:** Predefined combinations of CPU, memory, storage, and network capacity.

```
Naming convention: <Family><Generation>.<Size>

  t3.micro        m5.xlarge       c6g.2xlarge
  │ │ └─ size     │ │ └─ size     │ │  └─ size
  │ └─── gen      │ └─── gen      │ └──── gen
  └───── family   └───── family   └────── family (Graviton/ARM)

Sizes (roughly):
  nano < micro < small < medium < large < xlarge < 2xlarge < 4xlarge ...
```

**Instance families:**
```
Family  │ Optimized for          │ Use case
────────┼────────────────────────┼────────────────────────────────
t       │ Burstable (CPU credits)│ Dev/test, low-traffic web apps
m       │ General purpose        │ Most workloads (balanced)
c       │ Compute optimized      │ High CPU: gaming, HPC, encoding
r       │ Memory optimized       │ In-memory DBs, caches, big data
x       │ Extra memory           │ SAP HANA, massive in-memory DBs
i       │ Storage optimized      │ High IOPS: databases, analytics
p/g     │ GPU accelerated        │ ML training, graphics rendering
a/t4g   │ ARM (Graviton)         │ 20% cheaper, great perf for apps
```

**Beginner example:**
> Running a small WordPress blog? Use `t3.micro` (burstable, cheap).
> Running MySQL DB in production? Use `r6g.large` (memory optimized).
> Training a machine learning model? Use `p3.2xlarge` (GPU).

**DevOps Gotcha:**
> ⚠️ **t-family instances use CPU credits.** When credits run out, CPU is throttled
> to baseline (as low as 10%). Your app appears to "slow down randomly."
> Check CloudWatch `CPUCreditBalance` metric. If it hits 0, upgrade to m-family.

---

### 1.3 EC2 Instance Lifecycle

```
         ┌──────────┐
         │  Start   │  ← Launch new instance (from AMI)
         └────┬─────┘
              │
         ┌────▼─────┐
    ┌────│  Running │────┐
    │    └────┬─────┘    │
    │         │           │
    │    Stop │           │ Terminate
    │         │           │
    │    ┌────▼──────┐   │    ┌────────────┐
    │    │  Stopped  │   └───→│ Terminated │ (deleted forever)
    │    │(no billing│        └────────────┘
    │    │ for CPU)  │
    │    └────┬──────┘
    │         │ Start
    └─────────┘

State           │ CPU billed │ Storage billed │ Data preserved
────────────────┼────────────┼────────────────┼────────────────
Running         │ YES        │ YES            │ YES
Stopped         │ NO         │ YES (EBS)      │ YES (EBS)
Terminated      │ NO         │ NO (deleted)   │ NO (lost!)
```

**DevOps Gotcha:**
> ⚠️ **Stopped ≠ Terminated.** Stopping saves your data and EBS volumes.
> Terminating DELETES the instance and any non-persistent storage permanently.
> Also: instance store (ephemeral) data is LOST even on Stop. EBS data survives.

---

### 1.4 AMI — Amazon Machine Image

**Definition:** A pre-built template containing the OS, software, and configuration
needed to launch an EC2 instance. The "blueprint" for an instance.

```
AMI contains:
  ├── OS (Ubuntu 22.04, Amazon Linux 2023, Windows Server 2022)
  ├── Root EBS snapshot
  ├── Block device mapping (which volumes to attach)
  └── Launch permissions (public, private, or shared)

AMI → Launch → EC2 Instance
AMI → Launch → EC2 Instance   (all identical)
AMI → Launch → EC2 Instance
```

**Types:**
```
AWS Official  → Amazon Linux 2023, Ubuntu from Canonical (trusted, maintained)
AWS Marketplace→ Third-party AMIs (Nginx, Palo Alto, etc.) — may have licensing cost
Community     → Anyone can share (USE WITH CAUTION — verify before use)
Custom        → Your own: bake OS + app + config into a reusable image
```

**Best Practice:**
> Create **Golden AMIs** — pre-baked with all software, agents, and hardening.
> New EC2 instances start from golden AMI → no provisioning scripts at boot.
> Faster launch times, consistent configuration across fleet.

**DevOps Gotcha:**
> ⚠️ AMIs are region-specific. An AMI in us-east-1 cannot launch in ap-south-1.
> Copy the AMI to the target region before use: `aws ec2 copy-image --source-region`

---

### 1.5 EC2 Pricing Models

```
                        Cost    Flexibility  Interruption Risk
──────────────────────────────────────────────────────────────
On-Demand               $$$$    Maximum      None
Reserved (1yr)          $$      Low           None
Reserved (3yr)          $       Lowest        None
Savings Plans           $$$     Medium        None
Spot Instances          $       Medium        HIGH (2-min warning)
Dedicated Instances     $$$$$   Medium        None
Dedicated Hosts         $$$$$$  Low           None
──────────────────────────────────────────────────────────────
```

**Spot Instances — deep dive:**
```
AWS has spare EC2 capacity it sells at up to 90% discount.
When AWS needs that capacity back: 2-minute termination notice.

Good for:  batch jobs, ML training, stateless workers, CI/CD runners
Bad for:   web servers, databases, anything that needs to be always-on

Spot Fleet = collection of Spot instances across multiple types/AZs
             to reduce interruption risk
```

**Best Practice:**
> Use a mix: Reserved/Savings Plans for baseline + Spot for burst.
> This can reduce bill by 60-70% vs pure On-Demand.

---

### 1.6 Security Groups

**Definition:** A virtual stateful firewall for EC2 instances.
Controls inbound and outbound traffic at the instance level.

```
Security Group: sg-web-servers
┌────────────────────────────────────────────────────────┐
│  INBOUND RULES                                         │
│  Port 80  (HTTP)   │ Source: 0.0.0.0/0  (anywhere)    │
│  Port 443 (HTTPS)  │ Source: 0.0.0.0/0  (anywhere)    │
│  Port 22  (SSH)    │ Source: 10.0.0.0/8 (internal VPN)│
│                                                        │
│  OUTBOUND RULES                                        │
│  All traffic       │ Destination: 0.0.0.0/0            │
└────────────────────────────────────────────────────────┘
```

**Key properties:**
```
Stateful     → If inbound request allowed, response automatically allowed
Allow-only   → You can ALLOW traffic; there is no explicit Deny rule
Default      → All inbound DENIED, all outbound ALLOWED
Multiple SGs → One EC2 instance can have multiple Security Groups (rules union)
```

**Reference another SG as source:**
```
DB Security Group inbound rule:
Port 5432 from source: sg-web-servers  ← only instances in that SG can connect

This is better than IP-based rules because:
- Auto-scales with your fleet (new instances get same SG)
- No hardcoded IPs to maintain
```

**DevOps Gotcha:**
> ⚠️ Security Groups are stateful, but NACLs are stateless.
> Common mistake: add an inbound NACL rule for port 443 but forget the
> outbound rule for the ephemeral ports (1024-65535) needed for the response.

---

### 1.7 Key Pairs

**Definition:** SSH key pairs for connecting to EC2 Linux instances.
AWS stores the public key; you keep the private key (.pem file).

```bash
# Connect to EC2
ssh -i /path/to/key.pem ec2-user@<public-ip>

# Common usernames by OS:
# Amazon Linux  → ec2-user
# Ubuntu        → ubuntu
# Debian        → admin
# RHEL/CentOS   → ec2-user or centos
```

**DevOps Gotcha:**
> ⚠️ AWS does NOT store your private key. If you lose the .pem file,
> you cannot SSH in. You must attach the root volume to another instance
> and manually fix authorized_keys. Use EC2 Instance Connect or
> AWS Systems Manager Session Manager instead of SSH keys in production.

---

### 1.8 User Data (Bootstrap Script)

**Definition:** A script run ONE TIME when an EC2 instance first launches.
Used to install software, configure the OS, pull application code.

```bash
#!/bin/bash
# This runs at FIRST BOOT as root
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
echo "<h1>Hello from EC2</h1>" > /var/www/html/index.html
```

**DevOps Gotcha:**
> ⚠️ User data runs only on FIRST launch, not on reboot or start-after-stop.
> User data logs are at: `/var/log/cloud-init-output.log` on Amazon Linux/Ubuntu.
> If your script fails silently, check this log first.

---

### 1.9 EC2 Instance Metadata Service (IMDS)

**Definition:** An HTTP endpoint running on every EC2 instance that provides
information about the instance itself (instance ID, region, attached IAM role credentials).

```bash
# From inside an EC2 instance:
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/public-hostname
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
```

**IMDSv2 (current standard):**
```bash
# Must get a token first (prevents SSRF attacks)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token"   -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN"   http://169.254.169.254/latest/meta-data/instance-id
```

**DevOps Gotcha:**
> ⚠️ IMDSv1 is vulnerable to SSRF attacks (attackers can steal IAM credentials
> via web app that makes HTTP requests). Always enforce IMDSv2.
> Set `HttpTokens: required` on all EC2 instances.

---

### 1.10 Elastic IP (EIP)

**Definition:** A static public IPv4 address that persists across instance
stop/start cycles. Without EIP, public IP changes every time you stop and restart.

```
Without EIP:           With EIP:
Start: 54.23.11.5      Always: 54.23.11.5 (yours until released)
Stop
Start: 54.89.7.222     54.23.11.5 (same)
Stop
Start: 54.12.44.3      54.23.11.5 (same)
```

**DevOps Gotcha:**
> ⚠️ **Unused EIPs cost money** (~$3.60/month if allocated but not attached).
> AWS charges for EIPs NOT associated with a running instance.
> Release EIPs when decommissioning instances.

**Best Practice:**
> Prefer DNS names over EIPs. Use Route 53 to map a friendly domain to your
> instance and update the DNS record if the IP changes, rather than paying for EIP.

---

### 1.11 EC2 Auto Scaling

**Definition:** Automatically adjust the number of EC2 instances based on demand.

```
Traffic low:     2 instances running
Traffic spikes:  Auto Scaling adds instances → 6 instances
Traffic drops:   Auto Scaling removes → back to 2 instances

Policies:
├── Target Tracking  → "Keep CPU at 50%" (AWS manages scaling math)
├── Step Scaling     → "+2 instances when CPU > 70%, +4 when CPU > 90%"
├── Scheduled        → "Add 5 instances every Friday at 5PM"
└── Predictive       → ML-based forecasting (AWS handles it)
```

**Key components:**
```
Launch Template    → What to launch (instance type, AMI, SG, user data)
Auto Scaling Group → How many (min/max/desired), which AZs, which subnets
Scaling Policy     → When to scale
```

**DevOps Gotcha:**
> ⚠️ **Cooldown period.** After scaling out, Auto Scaling waits (default 300s)
> before scaling again. During a rapid spike, you may run low while waiting.
> Tune cooldown + use Target Tracking (it handles cooldowns better).

---

## 2. Lambda

### 2.1 What is Lambda?

**Definition:** Serverless compute — run code without managing servers.
You upload a function; AWS runs it in response to events and charges only
for the time it actually executes (per millisecond).

```
Traditional server approach:
  Server running 24/7 → pay for idle time

Lambda approach:
  Event occurs → Lambda runs → finishes → you pay only for execution
  No events → no cost

Cost: $0.20 per 1 million requests
      $0.0000166667 per GB-second of compute
      First 1M requests + 400,000 GB-seconds FREE per month (always free)
```

---

### 2.2 Lambda Triggers (Event Sources)

```
API Gateway / ALB     → HTTP requests (build REST/GraphQL APIs)
S3 Events             → file uploaded, deleted (image processing)
DynamoDB Streams      → DB changes (audit logging, CDC)
SQS                   → process messages from queue
SNS                   → react to notifications
EventBridge           → scheduled events (cron), AWS service events
Kinesis               → real-time stream processing
IoT Core              → device messages
CloudWatch Events     → scheduled tasks, AWS API events
Step Functions        → workflow orchestration
Cognito               → user pool triggers (pre-signup, post-auth)
```

**Beginner example:**
> User uploads a profile photo to S3 →
> S3 triggers a Lambda function →
> Lambda resizes the image to thumbnail →
> Lambda saves thumbnail back to S3.
> Zero servers to manage.

---

### 2.3 Lambda Execution Model

```
Cold Start:
  New request arrives → AWS provisions container → load runtime → run code
  Time: 100ms - 3s (varies by runtime and package size)

Warm Start:
  Container already running (reused from previous invocation)
  Time: <10ms overhead

Cold Start impact by runtime (rough order):
  Fastest: Node.js, Python
  Medium:  Go, Ruby
  Slowest: Java, .NET (JVM startup overhead)
```

**DevOps Gotcha:**
> ⚠️ **Cold starts hurt latency-sensitive APIs.** Mitigation options:
> 1. Provisioned Concurrency → AWS keeps N containers warm (costs money)
> 2. Use Snapstart (Java only) → pre-initialized snapshots
> 3. Use lighter runtimes (Node.js/Python) for API functions
> 4. Keep package size small (use Lambda Layers for shared dependencies)

---

### 2.4 Lambda Concurrency

```
Concurrency = number of simultaneous function executions

Account default limit: 1,000 concurrent executions per region
(Soft limit — request increase via support)

Reserved Concurrency:
  "This function gets max 100 concurrent executions"
  Protects other functions from one function consuming all concurrency
  Also acts as a cap (throttles at 100)

Provisioned Concurrency:
  "Keep 10 containers warm at all times"
  Eliminates cold starts
  Cost: charged even when idle
```

**DevOps Gotcha:**
> ⚠️ **Concurrency limit is shared.** One runaway Lambda consuming all 1,000
> concurrent slots will throttle ALL other Lambdas in the region.
> Set Reserved Concurrency on critical functions as a protective ceiling.

---

### 2.5 Lambda Layers

**Definition:** A ZIP archive that contains libraries, dependencies, or custom runtimes
shared across multiple Lambda functions. Uploaded once, reused many times.

```
Without Layers:                    With Layers:
Function A: 50MB (app + pandas)    Function A: 2MB (app only)
Function B: 50MB (app + pandas)    Function B: 2MB (app only)
Function C: 50MB (app + pandas)    Function C: 2MB (app only)
                                   Layer: 48MB (pandas, shared)
Total: 150MB deployed              Total: 54MB deployed
```

---

### 2.6 Lambda Best Practices

```
✅ Keep functions small and single-purpose
✅ Store secrets in Secrets Manager / SSM Parameter Store (not env vars)
✅ Set memory based on profiling — more memory = more CPU = faster + sometimes cheaper
✅ Set timeout appropriately (default 3s, max 15min)
✅ Use /tmp (512MB-10GB) for temporary files, not /
✅ Handle errors explicitly — use Dead Letter Queue (DLQ) for failed events
✅ Enable X-Ray tracing for distributed debugging
✅ Use Lambda Destinations for async success/failure routing

❌ Don't store state in Lambda (stateless!)
❌ Don't exceed 15 minute timeout (use Step Functions for long workflows)
❌ Don't put Lambda inside VPC unless needed (adds cold start time)
❌ Don't use Lambda for sustained high-throughput (EC2/containers are cheaper)
```

---

## 3. ECS — Elastic Container Service

### 3.1 What is ECS?

**Definition:** AWS-managed container orchestration service. Run Docker containers
without managing Kubernetes. AWS handles the orchestration layer.

```
You provide:   Docker image + task definition
ECS provides:  scheduling, health checks, service discovery, load balancing
```

---

### 3.2 ECS Key Concepts

```
ECS Cluster
├── Infrastructure (where containers run):
│   ├── EC2 Launch Type  → YOU manage EC2 instances
│   └── Fargate          → AWS manages servers (serverless containers)
│
├── Task Definition
│   ├── Docker image to use
│   ├── CPU and memory allocation
│   ├── Port mappings
│   ├── Environment variables
│   ├── IAM Task Role
│   └── Log configuration
│
├── Task → A running instance of a Task Definition (like a Pod in K8s)
│
└── Service
    ├── Runs N tasks (desired count)
    ├── Replaces failed tasks
    ├── Integrates with ALB for load balancing
    └── Supports rolling updates, blue/green deployments
```

**ECS vs EKS — when to choose:**
```
ECS                                EKS (Kubernetes)
───────────────────────────────    ────────────────────────────────
Simpler to operate                 More complex, steeper learning curve
AWS-specific (less portable)       Kubernetes standard (portable)
Less configuration overhead        Full K8s ecosystem (Helm, operators)
Tighter AWS integration            Use when: team knows K8s, need CRDs
Good for most AWS-first teams      Good for multi-cloud or K8s experts
```

---

### 3.3 ECS Task Definition (Example)

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/myapp-task-role",
  "containerDefinitions": [{
    "name": "myapp",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "environment": [{"name": "ENV", "value": "production"}],
    "secrets": [{
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-pass"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

**DevOps Gotcha:**
> ⚠️ Two IAM roles for ECS:
> `executionRoleArn` → ECS agent pulls image from ECR, writes logs to CloudWatch
> `taskRoleArn`      → your application code accesses AWS services (S3, DynamoDB)
> Confusing them is the #1 ECS IAM mistake.

---

## 4. Fargate

### 4.1 What is Fargate?

**Definition:** Serverless compute engine for containers. Run ECS or EKS
containers without provisioning or managing EC2 instances.

```
ECS + EC2:                         ECS + Fargate:
  You pick instance type             You pick vCPU + memory only
  You patch the OS                   AWS patches everything
  You manage capacity                AWS scales underlying infra
  You pay for idle capacity          Pay only for container runtime
  Cheaper at sustained load          More expensive per unit, less ops
```

**When to use Fargate:**
```
✅ Bursty or unpredictable workloads
✅ Small teams without ops capacity
✅ Dev/test environments
✅ Batch jobs that run occasionally

❌ Not ideal for sustained, high-traffic production (EC2 is cheaper)
❌ No GPU support
❌ No host-level networking (awsvpc mode only)
```

---

## 5. EKS — Elastic Kubernetes Service

### 5.1 What is EKS?

**Definition:** Managed Kubernetes control plane. AWS manages the K8s API server,
etcd, and control plane HA. You manage worker nodes (or use Fargate for serverless).

```
EKS Architecture:
┌──────────────────────────────────────────────────────────┐
│  AWS Managed Control Plane                               │
│  ├── API Server (ha, multi-AZ)                          │
│  ├── etcd (managed, backed up)                          │
│  └── Controller Manager, Scheduler                      │
└───────────────────────┬──────────────────────────────────┘
                        │ kubectl / API calls
┌───────────────────────▼──────────────────────────────────┐
│  Worker Nodes (you manage)                               │
│  ├── EC2 Self-managed                                   │
│  ├── EC2 Managed Node Groups (AWS handles OS patches)   │
│  └── Fargate Profiles (serverless pods)                 │
└──────────────────────────────────────────────────────────┘
```

---

### 5.2 EKS Add-ons

```
Core add-ons (AWS managed, kept updated):
├── CoreDNS                → cluster DNS
├── kube-proxy             → network rules on nodes
├── Amazon VPC CNI         → pod networking (pods get VPC IPs)
└── aws-ebs-csi-driver     → EBS persistent volumes

Community/optional:
├── AWS Load Balancer Controller  → ALB/NLB from K8s Ingress/Service
├── Cluster Autoscaler            → scale nodes based on pod demand
│   (or Karpenter — faster, more efficient)
└── ADOT (OpenTelemetry)          → metrics/traces to CloudWatch/X-Ray
```

**DevOps Gotcha:**
> ⚠️ **Amazon VPC CNI gives each pod a VPC IP address.**
> A /24 subnet = 256 IPs. With 250 pods, you exhaust the subnet.
> Plan subnets carefully: use /22 or larger for EKS node subnets.
> Or enable prefix delegation to get more IPs per ENI.

---

### 5.3 IRSA — IAM Roles for Service Accounts

**Definition:** The mechanism that allows Kubernetes pods to assume IAM Roles
using OIDC without storing credentials anywhere.

```
Without IRSA (bad):
  EC2 Node IAM Role → ALL pods on that node share the same AWS permissions
  One compromised pod → access to all AWS resources the node can reach

With IRSA (good):
  Pod A → ServiceAccount A → IAM Role A → only S3:GetObject on bucket-a
  Pod B → ServiceAccount B → IAM Role B → only DynamoDB on table-b
  Least privilege per pod.
```

**Setup:**
```bash
# 1. Create OIDC provider for cluster
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# 2. Create IAM Role with trust policy for ServiceAccount
eksctl create iamserviceaccount \
  --name my-app-sa \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

---

## 6. Elastic Beanstalk

**Definition:** Platform-as-a-Service (PaaS) for deploying web applications.
You provide code/container; Beanstalk handles EC2, Auto Scaling, ALB, and monitoring.

```
You upload:  application code (ZIP, WAR, Docker image)
Beanstalk:   provisions EC2 + ALB + RDS + Auto Scaling
             deploys your code
             manages health checks and restarts

Supported platforms:
  Node.js, Python, Java, Ruby, PHP, Go, .NET, Docker, Multi-container Docker
```

**When to use Beanstalk:**
```
✅ Quick deployment of standard web apps
✅ Teams that don't want to manage infrastructure
✅ Prototypes and internal tools

❌ Complex microservices architectures (use ECS/EKS)
❌ When you need fine-grained infrastructure control
❌ High-scale production (limited configuration flexibility)
```

**DevOps Gotcha:**
> ⚠️ Beanstalk uses EC2 under the hood. The underlying instances, security groups,
> and Load Balancers are visible in the console. If you manually modify them,
> Beanstalk may override your changes on the next deployment.
> Use `.ebextensions` or `Platform Hooks` for customization.

---

## 7. AWS Batch

**Definition:** Fully managed batch computing service. Runs batch jobs at any scale
by provisioning the right EC2 (or Fargate) instances automatically.

```
Use cases:
  Video transcoding       → process 10,000 videos overnight
  Genomics analysis       → run thousands of DNA processing jobs
  Financial simulations   → Monte Carlo simulations at scale
  ML training             → distributed training jobs
  ETL pipelines           → transform large datasets

How it works:
  1. Submit job (Docker image + command + resource requirements)
  2. Batch queues job in a Job Queue
  3. Batch scheduler picks optimal Compute Environment
  4. Spot instances provisioned, jobs run, instances terminated
```

**Key concepts:**
```
Job            → A unit of work (a Docker container to run)
Job Definition → Template for a job (image, vCPU, memory, command)
Job Queue      → Where jobs wait until resources are available
Compute Env    → Pool of EC2/Fargate resources (Spot, On-Demand, Fargate)
```

---

## 8. Lightsail

**Definition:** Simplified cloud platform with fixed monthly pricing.
Bundles compute + storage + networking into easy-to-manage "instances."

```
Target audience: developers who find EC2 too complex

Lightsail $5/month plan:
  1 vCPU + 512MB RAM + 20GB SSD + 1TB transfer

Comes with:
  - Pre-configured blueprints (WordPress, LAMP, Node.js, etc.)
  - Simple DNS management
  - Built-in load balancer
  - Managed databases (MySQL, PostgreSQL)
```

**When to use Lightsail:**
```
✅ Simple websites, blogs (WordPress)
✅ Development environments
✅ Small business apps with predictable traffic
✅ Users new to cloud who want simplicity

❌ Production workloads needing auto-scaling
❌ Microservices or containerized apps
❌ When you need VPC, IAM fine-grained control
```

---

## 9. EC2 Image Builder

**Definition:** Automated pipeline to build, test, and distribute AMIs (and
container images). Replaces manual AMI baking scripts.

```
Pipeline:
  Base AMI → Install components → Run tests → Distribute to regions

Components:
  AWS managed: apply OS patches, install CloudWatch agent, CIS hardening
  Custom: install your app dependencies, configure settings

Schedule: daily, weekly, or on new base AMI release
```

**Best Practice:**
> Automate AMI creation with Image Builder.
> Set Auto Scaling Groups to use the latest AMI version.
> Old AMIs are automatically deprecated after new ones are built.

---

## 10. Compute — Quick Decision Guide

```
I need to...                                     → Use
─────────────────────────────────────────────────────────────────────
Run a custom app on a VM, full OS control        → EC2
Run a function triggered by an event             → Lambda
Run containers, don't know Kubernetes           → ECS + Fargate
Run containers, team knows Kubernetes            → EKS
Deploy a web app without managing infra          → Elastic Beanstalk
Run batch/overnight processing jobs              → AWS Batch
Simple website/blog with fixed pricing           → Lightsail
Run a GPU workload for ML                        → EC2 p3/g5 instances
Run high-performance computing (HPC)             → EC2 HPC cluster + FSx
Build and distribute custom AMIs                 → EC2 Image Builder
Serverless containers without orchestration      → App Runner
Run code at CloudFront edge (ultra-low latency)  → Lambda@Edge / CloudFront Functions
```

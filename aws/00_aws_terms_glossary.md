# AWS Terms Glossary — with Azure Equivalents

> Quick-reference vocabulary for the AWS notes series (`01_global_infra_and_iam.md`, `02_compute.md`, ...).
> Each term: what it is in plain English, the closest Azure equivalent, and a DevOps note.
> Mappings are "closest", not identical — the gotchas column is where they differ.

---

## 1. How AWS is organized vs Azure

```
 AWS                                           AZURE
 ───                                           ─────
 AWS Organization (management account)         Entra ID tenant + root Management Group
   └── Organizational Units (OUs)                └── Management Groups
         └── AWS Accounts  ◄── billing +               └── Subscriptions  ◄── billing +
               │               security boundary              │              policy boundary
               └── (no RG equivalent;                         └── Resource Groups
                    use tags / CloudFormation stacks)               └── Resources
 SCPs (guardrails on accounts)                 Azure Policy (deny/audit) on MG/sub/RG
 IAM (per account)                             Entra ID (identities) + Azure RBAC (permissions)
```

```
 Region (e.g. ap-south-1 Mumbai)               Region (e.g. Central India, Pune)
   ├── AZ ap-south-1a                            ├── Availability Zone 1
   ├── AZ ap-south-1b                            ├── Availability Zone 2
   └── AZ ap-south-1c                            └── Availability Zone 3
 Local Zones / Wavelength / Outposts           Edge Zones / Azure Stack / Azure Local
 Edge locations (CloudFront POPs)              Front Door / CDN POPs
```

Biggest mental difference: **in AWS a VPC/subnet is per-AZ for subnets** (a subnet lives in exactly one AZ); **in Azure a subnet spans all zones** in the region and you pick the zone per resource.

---

## 2. Global infrastructure & account

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| Region | Geographic area with multiple AZs | Region | Prices/services vary per region |
| Availability Zone (AZ) | Isolated datacenter(s) in a region | Availability Zone | AZ names (1a) are mapped per account; use AZ IDs (aps1-az1) to compare across accounts |
| Edge location | CDN/DNS POP | Front Door/CDN POP | |
| AWS account | Billing + isolation boundary | Subscription | Multi-account strategy = multi-subscription strategy |
| Root user | Account owner email login | Global Admin / Account owner | Lock it with MFA, never use day-to-day |
| AWS Organizations | Manage many accounts | Management groups + EA/MCA billing | |
| OU | Group of accounts | Management group | |
| SCP | Max-permission guardrail for accounts | Azure Policy (deny) | SCPs never grant permissions |
| Control Tower | Landing zone automation | Azure Landing Zones (ALZ) | |
| ARN | Amazon Resource Name: `arn:aws:s3:::bucket` | Resource ID: `/subscriptions/.../resourceGroups/...` | |
| Tags | Key/value metadata | Tags | Cost allocation tags must be activated in billing |
| Service quotas | Per-account/region limits | Quotas/limits | Request increases early (vCPU, EIPs) |

## 3. Identity & security

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| IAM user | Long-lived identity with keys/password | Entra ID user | Avoid IAM users for humans; use Identity Center |
| IAM group | Collection of users | Entra group | |
| IAM role | Identity assumed temporarily (no long-term keys) | Service principal / managed identity / PIM role | Everything in AWS should assume roles |
| IAM policy | JSON permissions (Effect/Action/Resource/Condition) | RBAC role definition + assignment | AWS policies attach to identity OR resource |
| Resource-based policy | Policy on the resource (S3 bucket policy) | (Some) resource-level RBAC / access policies | |
| Trust policy | Who may assume a role | Federated credential on app registration | |
| STS | Issues temporary credentials | Entra token service | |
| Instance profile | Role attached to EC2 | Managed identity on VM | |
| IRSA / EKS Pod Identity | Pod assumes IAM role | AKS Workload Identity | |
| IAM Identity Center (SSO) | Workforce login to many accounts | Entra ID SSO + RBAC | |
| Cognito | App user sign-up/sign-in | Entra External ID (formerly B2C) | |
| Permission boundary | Max permissions for a role/user | (No direct) Policy + custom roles | |
| KMS / CMK | Key management | Key Vault (keys) / Managed HSM | |
| Secrets Manager | Secrets with rotation | Key Vault (secrets) | |
| SSM Parameter Store | Config + secure strings | App Configuration / Key Vault | |
| ACM | TLS certificates | Key Vault certificates / App Service managed certs | |
| GuardDuty | Threat detection | Defender for Cloud | |
| Security Hub | Security posture aggregation | Defender for Cloud (secure score) | |
| Inspector | Vulnerability scanning | Defender for Servers/Containers | |
| Macie | Find PII in S3 | Microsoft Purview | |
| WAF / Shield | Web firewall / DDoS | WAF (Front Door/App GW) / DDoS Protection | |
| CloudTrail | API audit log | Activity Log + Entra audit logs | Enable org trail for all regions |
| AWS Config | Resource config history + rules | Azure Policy + Resource Graph change history | |

## 4. Compute

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| EC2 | Virtual machines | Virtual Machines | |
| AMI | Machine image | Managed image / Compute Gallery image | AMIs are regional — copy across regions |
| Instance type (t3.medium) | VM size | VM size (Standard_B2s) | |
| EBS | Block disk for EC2 | Managed Disks | EBS is AZ-bound |
| Instance store | Ephemeral local disk | Temp disk | Data lost on stop |
| Key pair | SSH key | SSH key resource | |
| User data | Boot script | Custom data / cloud-init | |
| Auto Scaling Group (ASG) | Scale EC2 fleet | VM Scale Sets | |
| Launch template | VM blueprint for ASG | VMSS model | |
| Spot instances | Cheap interruptible VMs | Spot VMs | |
| Reserved Instances / Savings Plans | Commit for discount | Reservations / Savings plan | |
| Elastic IP | Static public IP | Static Public IP | Charged when unused (and now for all public IPv4) |
| Lambda | Serverless functions | Azure Functions | 15-min max runtime |
| ECS | AWS container orchestrator | Container Apps (closest) | |
| Fargate | Serverless container compute | Container Apps / ACI | |
| EKS | Managed Kubernetes | AKS | EKS control plane billed hourly; AKS free tier exists |
| ECR | Container registry | ACR | |
| Elastic Beanstalk | PaaS app hosting | App Service | |
| App Runner | Simple container web apps | Container Apps / App Service | |
| Lightsail | Simple VPS bundles | (B-series VMs) | |
| AWS Batch | Batch jobs | Azure Batch | |

## 5. Networking

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| VPC | Private network | VNet | |
| Subnet (public/private) | IP range in ONE AZ | Subnet (zone-spanning) | "Public" = route table has IGW route |
| Route table | Routes for subnets | Route table (UDR) | |
| Internet Gateway (IGW) | VPC ↔ internet | Built-in system route (default outbound being retired) | |
| NAT Gateway | Private subnets → internet | NAT Gateway | AWS NAT GW is per-AZ; deploy one per AZ for HA |
| Security Group | Stateful firewall on ENI/instance, allow-only | NSG (stateful, allow+deny) / ASG | SGs can reference other SGs |
| NACL | Stateless subnet firewall | NSG on subnet (but NSG is stateful) | Must allow ephemeral return ports |
| ENI | Network interface | NIC | |
| VPC Peering | Connect 2 VPCs (non-transitive) | VNet peering | |
| Transit Gateway | Hub router for many VPCs/VPNs | Virtual WAN / hub-spoke with Azure Firewall/NVA | |
| VPC Endpoint – Gateway | Private route to S3/DynamoDB | Service endpoints | Free |
| VPC Endpoint – Interface (PrivateLink) | Private IP for a service | Private Endpoint / Private Link | Needs private DNS |
| Site-to-Site VPN | IPsec to on-prem | VPN Gateway | |
| Direct Connect | Dedicated private line | ExpressRoute | |
| Route 53 | DNS + health checks + routing policies | Azure DNS + Traffic Manager | |
| Route 53 Resolver | Hybrid DNS forwarding | DNS Private Resolver | |
| ELB – ALB | L7 HTTP load balancer | Application Gateway | |
| ELB – NLB | L4 TCP/UDP load balancer | Load Balancer (Standard) | |
| GWLB | Insert firewalls/NVAs | Gateway Load Balancer | |
| CloudFront | CDN | Front Door / Azure CDN | |
| Global Accelerator | Anycast static IPs | Front Door / cross-region LB | |
| API Gateway | Managed API front door | API Management | |
| Network Firewall | Managed firewall | Azure Firewall | |
| VPC Flow Logs | Network traffic logs | VNet flow logs | |

## 6. Storage & databases

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| S3 | Object storage (buckets) | Blob Storage (containers in storage account) | Bucket names global |
| S3 storage classes | Standard, IA, Glacier, Intelligent-Tiering | Hot, Cool, Cold, Archive tiers | |
| S3 lifecycle rules | Auto tier/delete | Lifecycle management policies | |
| S3 versioning / Object Lock | Versions / WORM | Blob versioning / immutability | |
| Pre-signed URL | Temporary object access | SAS token (user delegation SAS) | |
| EFS | Managed NFS | Azure Files (NFS) / Azure NetApp Files | |
| FSx | Managed Windows/Lustre/NetApp FS | Azure Files (SMB) / Managed Lustre / ANF | |
| Storage Gateway | Hybrid storage | Azure File Sync / Data Box Gateway | |
| AWS Backup | Central backup | Azure Backup / Backup Center | |
| RDS | Managed SQL DBs (MySQL, PG, SQL Server, Oracle) | Azure Database for MySQL/PostgreSQL, Azure SQL | |
| Aurora | AWS cloud-native MySQL/PG | Azure SQL Hyperscale / PG Flexible (closest) | |
| Multi-AZ | Synchronous standby | Zone-redundant HA | |
| Read replica | Async read copy | Read replicas | |
| DynamoDB | Key-value NoSQL, serverless | Cosmos DB (NoSQL API) / Table Storage | |
| ElastiCache | Redis/Memcached | Azure Managed Redis / Azure Cache for Redis | |
| Redshift | Data warehouse | Synapse / Microsoft Fabric warehouse | |
| DocumentDB | Mongo-compatible | Cosmos DB for MongoDB | |
| Neptune | Graph DB | Cosmos DB for Apache Gremlin | |
| Athena | SQL over S3 | Synapse serverless / Fabric | |
| Glue | ETL + data catalog | Data Factory + Purview | |

## 7. Messaging & integration

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| SQS (Standard / FIFO) | Queue | Service Bus queue (sessions for FIFO) / Storage Queue | SQS visibility timeout ≈ SB lock duration |
| SQS DLQ + redrive | Poison message queue | Service Bus DLQ | |
| SNS | Pub/sub push notifications | Event Grid / Service Bus topics | |
| EventBridge | Event bus, rules, schedules | Event Grid (+ Logic Apps schedules) | |
| EventBridge Scheduler | Cron at scale | Logic Apps / Functions timer | |
| Kinesis Data Streams | Streaming | Event Hubs | |
| MSK | Managed Kafka | Event Hubs (Kafka API) / HDInsight Kafka | |
| Step Functions | Workflow state machine | Logic Apps / Durable Functions | |
| SES | Email sending | Azure Communication Services Email | |
| AppSync | Managed GraphQL | API Management (GraphQL) | |
| MQ | Managed ActiveMQ/RabbitMQ | Service Bus (AMQP) | |

## 8. DevOps, IaC & monitoring

| AWS term | Meaning | Azure equivalent | DevOps note |
|----------|---------|------------------|-------------|
| CloudFormation | Native IaC (JSON/YAML), stacks | ARM templates / **Bicep** + deployment stacks | CFN stack ≈ Bicep deployment stack |
| CDK | IaC in code (TS/Python) → CFN | (Bicep / Pulumi / Terraform CDK) | |
| SAM | Serverless IaC | Bicep / Functions tooling | |
| Change set | Preview stack changes | what-if | |
| StackSets | Deploy across accounts/regions | Deployment at MG scope / pipelines | |
| CodeCommit | Git hosting (closed to new customers) | Azure Repos | |
| CodeBuild | Build service | Azure Pipelines agents / GitHub Actions | |
| CodeDeploy | Deployment orchestration | Azure Pipelines release / deployment slots | |
| CodePipeline | CI/CD pipeline | Azure Pipelines / GitHub Actions | |
| Systems Manager (SSM) | Run commands, patch, session manager | Azure Arc, Update Manager, Run Command, Bastion | Session Manager = SSH without open ports |
| OpsWorks | Chef/Puppet (retired) | Automation State Configuration (retired) | |
| CloudWatch Metrics/Logs/Alarms | Monitoring | Azure Monitor metrics / Log Analytics / Alerts | |
| CloudWatch Logs Insights | Log query language | KQL in Log Analytics | |
| X-Ray | Distributed tracing | Application Insights | |
| Trusted Advisor | Best practice checks | Azure Advisor | |
| Cost Explorer / Budgets | Cost analysis | Cost Management + Budgets | |
| Well-Architected Framework | Design pillars | Azure Well-Architected Framework | Same 5–6 pillars idea |

## 9. AI / ML on AWS (for comparison with the Azure AI notes)

| AWS term | Meaning | Azure equivalent |
|----------|---------|------------------|
| Amazon Bedrock | Managed access to foundation models (Claude, Llama, Titan, Nova...) | Microsoft Foundry / Azure OpenAI |
| Bedrock Agents / AgentCore | Build & run agents | Foundry Agent Service |
| Bedrock Knowledge Bases | Managed RAG | Azure AI Search + Foundry "on your data" / knowledge bases |
| Bedrock Guardrails | Safety filters | Azure AI Content Safety / content filters |
| Provisioned Throughput (Bedrock) | Reserved model capacity | PTU |
| SageMaker | Build/train/host ML models | Azure Machine Learning |
| Amazon Q | AI assistant for devs/business | GitHub Copilot / Microsoft 365 Copilot |
| Kendra | Enterprise search | Azure AI Search |
| OpenSearch Service | Search + vector DB | Azure AI Search / Elastic on Azure |
| Textract | Document text/form extraction | Document Intelligence |
| Rekognition | Image/video analysis | Azure AI Vision |
| Transcribe / Polly | Speech-to-text / text-to-speech | Azure AI Speech |
| Comprehend | NLP (entities, sentiment) | Azure AI Language |
| Translate | Translation | Azure AI Translator |

---

## 10. Frequently confused pairs (interview favourites)

| Pair | Difference |
|------|------------|
| Security Group vs NACL | SG: stateful, instance-level, allow only. NACL: stateless, subnet-level, allow + deny, ordered rules. |
| IAM user vs role | User: permanent credentials. Role: temporary credentials via STS, assumed by users/services. |
| IAM policy vs SCP | Policy grants permissions. SCP only limits the maximum for an account; grants nothing. |
| ALB vs NLB | ALB: L7, path/host routing, WAF. NLB: L4, static IP, ultra-low latency, TCP/UDP. |
| SQS vs SNS vs EventBridge | SQS: queue (pull). SNS: fan-out push. EventBridge: event bus with rules, SaaS/AWS events, schedules. |
| EBS vs EFS vs S3 | EBS: block, one AZ, one instance (mostly). EFS: shared NFS across AZs. S3: object store via API. |
| RDS Multi-AZ vs Read Replica | Multi-AZ: HA standby, not readable (classic). Read replica: scaling reads, async. |
| ECS vs EKS | ECS: AWS-native simpler orchestrator. EKS: Kubernetes. |
| Fargate vs EC2 launch type | Fargate: no nodes to manage. EC2: you manage the instances. |
| CloudWatch vs CloudTrail | CloudWatch: performance metrics/logs. CloudTrail: who did what (API audit). |
| Gateway vs Interface endpoint | Gateway: route-table based, S3/DynamoDB only, free. Interface: ENI with private IP (PrivateLink), many services, paid. |
| Spot vs Reserved vs Savings Plan | Spot: cheapest, can be interrupted. RI: commit to instance type. SP: commit to $/hour spend, flexible. |

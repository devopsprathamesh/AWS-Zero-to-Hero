### AWS-Zero-to-Hero

## Identity & Access

AWS IAM [Tier-0]

AWS STS [Tier-0]

IAM Identity Center (SSO) [Tier-0]

## Governance

AWS Organizations [Tier-0]

AWS Control Tower [Tier-1]

## Compliance & Audit

AWS CloudTrail [Tier-0]

AWS Config [Tier-1]

AWS Security Hub [Tier-1]

## Security, Keys & Secrets

AWS KMS [Tier-0]

AWS Secrets Manager [Tier-0]

AWS Certificate Manager (ACM) [Tier-1]

AWS WAF [Tier-1]

Amazon GuardDuty [Tier-1]

## Networking

Amazon VPC [Tier-0]

VPC Endpoints / PrivateLink [Tier-0]

AWS Transit Gateway [Tier-1]

## Edge

Amazon CloudFront [Tier-1]

## Load Balancing

Elastic Load Balancing (ALB/NLB/GWLB) [Tier-0]

## Traffic Management

Amazon Route 53 [Tier-0]

## Kubernetes

Amazon EKS [Tier-0]

## Containers

Amazon ECR [Tier-0]

AWS Fargate [Tier-1]

Amazon ECS [Tier-2]

## Compute

Amazon EC2 [Tier-0]

EC2 Auto Scaling [Tier-0]

AWS Lambda [Tier-0]

## Storage

Amazon S3 [Tier-0]

Amazon EBS [Tier-1]

Amazon EFS [Tier-1]

## Backup & DR

AWS Backup [Tier-1]

## Databases

Amazon RDS [Tier-1]

Amazon Aurora [Tier-1]

Amazon DynamoDB [Tier-1]

## Cache

Amazon ElastiCache (Redis/Memcached) [Tier-2]

## Search

Amazon OpenSearch Service [Tier-1]

## Observability

Amazon CloudWatch [Tier-0]

## Operations

AWS Systems Manager (SSM) [Tier-0]

## Automation & Orchestration

Amazon EventBridge [Tier-1]

AWS Step Functions [Tier-1]

## Messaging

Amazon SQS [Tier-1]

Amazon SNS [Tier-1]

## Infrastructure as Code

AWS CloudFormation [Tier-0]

AWS CDK [Tier-0]

## CI/CD

AWS CodePipeline [Tier-2]

AWS CodeBuild [Tier-2]

## Data & Analytics

Amazon Athena [Tier-2]

AWS Glue [Tier-2]

## FinOps / Cost Management

AWS Cost Explorer [Tier-1]

AWS Budgets [Tier-2]

## Migration & Transfer

AWS Database Migration Service (DMS) [Tier-2]


=-=-=-=-=-=-=-


Beginner learning order (recommended)
1) Identity & Access

IAM, STS, IAM Identity Center (SSO)
Why first: you can’t do anything safely without permissions; most “it doesn’t work” issues are IAM.

2) Core Networking Foundations

VPC, Security Groups, NACLs, VPC DNS basics
Why: every workload (EKS included) depends on correct subnetting/routing/security boundaries.

3) Compute Fundamentals

EC2, Auto Scaling (and basic AMI/user-data concepts)
Why: even in EKS-heavy orgs, node groups often run on EC2; you must understand instances and scaling.

4) Storage Fundamentals

S3, EBS, EFS
Why: apps/logs/artifacts/backups live here; EKS storage and many AWS services depend on S3/KMS.

5) Observability Basics

CloudWatch, CloudTrail
Why: you need logs/metrics/audit trails early so you can debug everything you build later.

6) Infrastructure as Code

CloudFormation, CDK
Why: production AWS is provisioned via IaC; also forces clean, repeatable networking/IAM patterns.

7) Operations & Day-2 Management

Systems Manager (SSM)
Why: patching, remote access without bastions, automation runbooks—core on-call tooling.

8) Load Balancing

ELB (ALB/NLB/GWLB)
Why: almost every real service needs ingress/load balancing; critical for EKS Ingress and service exposure.

9) Traffic Management (DNS)

Route 53
Why: blue/green, failover, and multi-environment routing all start with DNS patterns.

10) Containers Fundamentals

ECR, (optionally ECS + Fargate basics)
Why: understand images/registries/task roles first; then Kubernetes becomes much easier.

11) Kubernetes on AWS

EKS
Why now: you’ll be effective faster once IAM/VPC/ELB/observability/container basics are in place.

12) Security Hardening (Platform Security)

KMS, Secrets Manager, ACM, WAF, GuardDuty
Why: once workloads run, you must lock down encryption, secrets, TLS, and threat detection.

13) Messaging (Decoupling)

SQS, SNS
Why: common production patterns for resilience, retries, async workflows.

14) Automation & Orchestration

EventBridge, Step Functions
Why: event-driven ops, workflows, and reliable automation become natural after messaging basics.

15) Databases (Core Options)

RDS, Aurora, DynamoDB
Why: most platforms run on one of these; you’ll need backups, HA, scaling, auth patterns.

16) Backup & DR

AWS Backup
Why: once data exists, you need recovery guarantees and restore testing.

17) Governance (Multi-account at scale)

Organizations, Control Tower
Why: best learned after you’ve built a few environments; then you’ll “feel” why guardrails matter.

18) Compliance & Posture Management

Config, Security Hub
Why: makes sense after governance + security basics; used for audits and continuous controls.

19) Advanced Networking (Scale / Multi-account)

Transit Gateway
Why: becomes necessary in multi-VPC, multi-account EKS platforms; too heavy for day-1 learning.

20) Edge & Acceleration

CloudFront
Why: important for global apps, caching, security posture—but not a prerequisite for EKS fundamentals.

21) FinOps / Cost Management

Cost Explorer, Budgets
Why: should start early as habits, but deep optimization is more meaningful once you run workloads.

22) Data & Analytics

Athena, Glue
Why: often owned by data/platform teams; useful for logs/CUR analytics later.

23) Search & Cache (Specialized but common)

OpenSearch, ElastiCache
Why: great differentiators, but easier after you understand databases + observability.

24) Migration & Transfer

DMS
Why: mostly project-driven; learn when you hit a migration/modernization initiative.
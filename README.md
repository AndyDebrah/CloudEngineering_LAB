# AWS Well-Architected & Cloud Adoption Framework Assessment

> **Cloud Engineering Fundamentals Lab** — Design and Evaluate an AWS Solution  
> Prepared by: **Andy Kwasi Debrah**

---

## Overview

This repository documents a full AWS architecture assessment and redesign exercise based on the **AWS Well-Architected Framework (WAF)** and the **AWS Cloud Adoption Framework (CAF)**. The project begins with a two-tier on-premises web application, evaluates its weaknesses across all five WAF pillars, assesses organisational readiness using the six CAF perspectives, and delivers a production-ready revised architecture diagram with detailed justification.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Task 1 — Existing Architecture Review](#task-1--existing-architecture-review)
- [Task 2 — Well-Architected Framework Assessment](#task-2--well-architected-framework-assessment)
- [Task 3 — Cloud Adoption Framework Readiness Assessment](#task-3--cloud-adoption-framework-readiness-assessment)
- [Task 4 — Revised Architecture Design](#task-4--revised-architecture-design)
- [Architecture Diagram](#architecture-diagram)
- [AWS Services Reference](#aws-services-reference)
- [Key Design Decisions](#key-design-decisions)
- [Reflection](#reflection)

---

## Project Structure

```
.
├── README.md                         # This file
├── diagrams/
│   └── revised-architecture.png     # Revised three-tier AWS architecture diagram
└── report/
    └── andy_debrah_aws_lab.pdf       # Full WAF & CAF assessment report
```

---

## Task 1 — Existing Architecture Review

### Workload Description

The baseline workload is a two-tier web application originally running on-premises. It was migrated to AWS via a **lift-and-shift** without redesign, meaning all on-premises weaknesses were carried directly into the cloud environment.

| Tier | Component |
|------|-----------|
| Frontend | Single EC2 instance running Apache/Nginx, serving HTTP/HTTPS and static assets |
| Backend | Single RDS-equivalent database (MySQL/PostgreSQL), no replication, no failover |
| Network | Flat VPC with no subnet segmentation; security groups open to `0.0.0.0/0` |

### Identified Risks

| # | Risk | Impact |
|---|------|--------|
| 1 | Single point of failure across both tiers | Full outage on any instance failure |
| 2 | No Availability Zone redundancy | Location-level failure takes down the entire app |
| 3 | Overly permissive security groups (`0.0.0.0/0`) | Database reachable from the public internet |
| 4 | No backup or disaster recovery strategy | Data loss risk; unknown RTO/RPO |
| 5 | No encryption in transit or at rest | Data readable if network or disk is compromised |
| 6 | Shared and unmanaged SSH credentials | No audit trail; impossible to enforce least-privilege |
| 7 | No monitoring or alerting | Incidents detected only via user complaints |
| 8 | Manual SSH-based deployments | No rollback path; no deployment audit trail |
| 9 | Static assets served from app server | Wasted compute; high latency for distant users |
| 10 | No auto scaling mechanism | Traffic spikes degrade or crash the single instance |

---

## Task 2 — Well-Architected Framework Assessment

The five WAF pillars were applied to evaluate the current state and define targeted improvements.

### Summary Table

| Pillar | Current State | Improvement | AWS Services |
|--------|--------------|-------------|--------------|
| **Operational Excellence** | Manual SSH deployments, no IaC, no centralised logging | CI/CD pipeline, CloudFormation IaC, CloudWatch observability | CloudFormation, CodePipeline, CodeDeploy, CloudWatch |
| **Security** | Open security groups, shared credentials, no encryption | Least-privilege IAM, KMS encryption, TLS enforcement, WAF | IAM, KMS, ACM, WAF, CloudTrail, Secrets Manager |
| **Reliability** | Single-instance, single-AZ, no failover, no backups | Multi-AZ deployment, Auto Scaling, RDS Multi-AZ, automated backups | RDS Multi-AZ, Auto Scaling, ALB, AWS Backup |
| **Performance Efficiency** | No caching, no CDN, no utilisation-based right-sizing | ElastiCache for read offload, CloudFront CDN, Auto Scaling | ElastiCache, S3, CloudFront, Compute Optimizer |
| **Cost Optimization** | Fixed capacity regardless of demand, no cost visibility | Auto Scaling, right-sizing, Savings Plans, budget alerts | Cost Explorer, Budgets, Compute Optimizer, Savings Plans |

### Pillar Highlights

**Operational Excellence** — All infrastructure is defined in CloudFormation for version-controlled, peer-reviewed, reproducible environments. CodePipeline orchestrates build, test, and deploy stages. CloudWatch Logs aggregates all application and infrastructure logs with metric-based alarms.

**Security** — The database is isolated in a private subnet with no internet route. Security group rules follow a strict chain: ALB accepts HTTPS from `0.0.0.0/0`; EC2 accepts traffic only from the ALB security group; RDS accepts traffic only from the EC2 security group on the database port. AWS Systems Manager Session Manager replaces all SSH access. AWS Secrets Manager handles credential rotation automatically.

**Reliability** — The web tier runs as an Auto Scaling Group across two AZs, with the ALB performing continuous health checks. The database uses RDS Multi-AZ with automatic failover to a synchronous standby replica in under two minutes. AWS Backup manages all snapshot retention centrally.

**Performance Efficiency** — Amazon ElastiCache (Redis) sits between the application tier and the database, serving frequent read queries from memory. Static assets are moved to S3 and distributed globally via CloudFront edge locations. Auto Scaling policies add capacity when CPU exceeds 70% and remove it when it drops below 30%.

**Cost Optimization** — Auto Scaling eliminates idle capacity during off-peak periods. S3 and CloudFront reduce EC2 bandwidth costs. AWS Compute Optimizer provides utilisation-based right-sizing recommendations after two weeks of live data. Reserved Instances or Savings Plans are applied to predictable baseline capacity for up to 72% cost reduction.

---

## Task 3 — Cloud Adoption Framework Readiness Assessment

The CAF was applied across six perspectives to assess organisational readiness for the AWS migration.

| Perspective | Gap Identified | Recommended Action |
|-------------|---------------|-------------------|
| **Business** | No documented migration business case or success metrics | Quantify on-premises vs. AWS costs; define measurable outcomes (uptime %, cost reduction); secure stakeholder buy-in from finance and operations |
| **People** | Team experienced in on-premises administration, limited AWS exposure | Map skill gaps to AWS training paths (Skill Builder, Cloud Practitioner, Solutions Architect Associate); define cloud-native roles; establish a Cloud Centre of Excellence |
| **Governance** | No tagging standards, no cost allocation, no compliance controls | Enforce resource tagging policy (environment, owner, cost centre); use AWS Organizations for account separation; deploy AWS Config for drift detection; set day-one budget controls |
| **Platform** | Application config hardcoded; no IaC; self-managed database | Externalise config to Parameter Store/Secrets Manager; define environment in CloudFormation; migrate database to Amazon RDS; separate static assets to S3 |
| **Security** | No formal access policies, no encryption, no audit logging | Enable CloudTrail across all regions; activate GuardDuty; enforce MFA on all IAM users including root; apply KMS encryption at rest; enforce TLS with ACM |
| **Operations** | No runbooks, no dashboards, no alerting, no defined SLAs | Define baseline metrics (CPU, latency, error rate, DB connections); configure CloudWatch alarms; write and test runbooks before go-live; document RTO and RPO targets |

---

## Task 4 — Revised Architecture Design

### Architecture Overview

The revised architecture addresses every weakness from Task 1 and maps directly to all five WAF pillars. The environment runs inside a custom Amazon VPC spanning **two Availability Zones** in `eu-west-1`, providing geographic redundancy at every tier.

### VPC Subnet Design

| Subnet | CIDR | Purpose |
|--------|------|---------|
| Public Subnet AZ-1 | `10.0.1.0/24` | Application Load Balancer, NAT Gateway |
| Public Subnet AZ-2 | `10.0.2.0/24` | Application Load Balancer, NAT Gateway |
| Private Subnet AZ-1 | `10.0.3.0/24` | EC2 Web / App Tier |
| Private Subnet AZ-2 | `10.0.4.0/24` | EC2 Web / App Tier |
| Private Subnet AZ-1 | `10.0.5.0/24` | RDS Primary |
| Private Subnet AZ-2 | `10.0.6.0/24` | RDS Standby (Multi-AZ) |

### Layer-by-Layer Description

**1. DNS & Content Delivery**
- **Route 53** resolves the application domain and routes traffic to the ALB with health-check-based failover
- **CloudFront** caches static assets at global edge locations, reducing latency and offloading origin traffic
- **S3** stores all static assets (images, CSS, JavaScript bundles), decoupling them from EC2

**2. Network Layer**
- Internet Gateway attached to public subnets only
- Strict security group chain: `ALB-sg → app-sg → db-sg`
- No inbound SSH port open on any instance

**3. Web & Application Tier**
- **Application Load Balancer** terminates HTTPS (via ACM certificate), performs health checks, and distributes traffic across both AZs
- **Auto Scaling Group** maintains a minimum of two EC2 instances (one per AZ), scales out at 70% CPU, scales in at 30% CPU
- **EC2 instances** use IAM Instance Roles — no hardcoded credentials anywhere
- **Systems Manager Session Manager** provides auditable shell access without port 22
- **Secrets Manager** stores and auto-rotates database credentials

**4. Caching Layer**
- **ElastiCache (Redis)** deployed in cluster mode across both AZs, serving frequent database reads from memory and reducing query load on RDS

**5. Database Tier**
- **RDS Multi-AZ** (MySQL or PostgreSQL) with a synchronous standby replica in AZ-2; automatic failover in 1–2 minutes
- **KMS encryption at rest** applied to all RDS data and snapshots
- No internet route; accepts connections only from the EC2 security group

**6. Monitoring & Observability**
- **CloudWatch** collects metrics across EC2, ALB, RDS, and ElastiCache; alarms trigger SNS notifications
- **CloudTrail** logs every API call to S3, forwarded to CloudWatch Logs for querying and alerting
- **GuardDuty** continuously analyses CloudTrail, VPC Flow Logs, and DNS logs for threat indicators

---


## AWS Services Reference

| Category | Service | Role in Architecture |
|----------|---------|---------------------|
| DNS | Route 53 | Domain resolution with health-check routing |
| CDN | CloudFront | Global static asset delivery from edge |
| Storage | S3 | Static asset origin; CloudTrail log archive |
| Networking | VPC, Subnets, IGW, NAT | Network isolation and egress control |
| Load Balancing | Application Load Balancer | HTTPS termination, health checks, traffic distribution |
| Compute | EC2, Auto Scaling | Application hosting with demand-based scaling |
| Database | RDS Multi-AZ | Managed relational DB with automatic failover |
| Caching | ElastiCache (Redis) | In-memory read caching; session storage |
| Security | IAM, KMS, ACM, WAF, GuardDuty, CloudTrail | Identity, encryption, threat detection, audit logging |
| Secrets | Secrets Manager | Credential storage and automatic rotation |
| Operations | Systems Manager, CloudWatch, SNS | Shell access, observability, alerting |
| IaC & CI/CD | CloudFormation, CodePipeline, CodeDeploy | Repeatable infrastructure and automated deployments |
| Cost | Cost Explorer, Budgets, Compute Optimizer | Spend visibility, alerts, right-sizing |
| Backup | AWS Backup | Centralised backup policies and restore testing |

---

## Key Design Decisions

- **Private subnets for all compute and data** — EC2 instances and the RDS cluster have no direct internet exposure; all inbound traffic flows through the ALB
- **Security group chaining** — each tier only accepts traffic from the tier directly above it, on the minimum required port
- **No SSH port open** — all operator access is via Systems Manager Session Manager, logged to CloudWatch
- **Secrets Manager over config files** — database credentials are never stored in code, environment variables, or config files
- **Multi-AZ by default** — both the Auto Scaling Group and RDS are spread across two Availability Zones from day one, not as an afterthought
- **CloudFront as the static asset layer** — removes static file serving responsibility from EC2 entirely, reducing compute load and improving global response times

---

## Reflection

A direct lift-and-shift to AWS would have reproduced every on-premises weakness — single points of failure, open network access, no backups, no visibility — hosted on someone else's hardware. Working through the Well-Architected Framework pillar by pillar made that gap concrete.

The CAF assessment added a dimension that purely technical frameworks miss: an organisation can design a sound architecture and still fail the migration because staff lack the skills to operate it, governance policies are absent, or leadership has no way to measure whether the migration delivered value.

The most practical takeaway is that security, reliability, and operational visibility are not features to be added after the architecture is built. Designing them in from the start — through private subnets, IAM roles, Multi-AZ deployments, and automated pipelines — costs less effort than retrofitting them later.

---

## Author

**Andy Kwasi Debrah**  
Cloud Engineering Fundamentals — AWS WAF & CAF Assessment Lab


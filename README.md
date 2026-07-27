# Project 1: Scalable Web Application with ALB and Auto Scaling

**Architecture type:** EC2-based
**Pattern:** Highly available, multi-AZ web application on AWS

## Overview

Deploy a production-grade web application on AWS using EC2 instances inside a properly architected VPC with public and private subnets across two Availability Zones. High availability and scalability are achieved with an Application Load Balancer (ALB), an Auto Scaling Group (ASG), and a CloudFront distribution for caching static assets. A Multi-AZ RDS instance serves as the database backend, with all compute placed in private subnets.

## Key AWS Services

- **VPC** — public and private subnets, NAT Gateway, Security Groups, NACLs
- **EC2 + ASG** — Launch Template, scaling policies (target tracking)
- **ALB + WAF** — Layer 7 routing, WAF rules for OWASP Top 10
- **CloudFront** — cache static assets, reduce latency
- **RDS Multi-AZ** — MySQL/PostgreSQL with automated failover
- **Route 53** — alias record pointing to ALB, health checks
- **Systems Manager** — Session Manager for secure instance access
- **CloudWatch + SNS** — dashboards, alarms, and notifications

## Learning Outcomes

- Design VPCs with correct subnet, route table, and NAT Gateway configurations
- Build highly available architectures across multiple Availability Zones
- Configure ALB listener rules and target group health checks
- Implement Auto Scaling with target tracking and step scaling policies
- Secure applications with WAF, Security Groups, and private subnets
- Use Systems Manager Session Manager as a bastion-free access alternative

---

## Architecture Diagram

![Project Architecture] (mantra-project-architecture.png)

---

## 1. Networking (VPC)

### CIDR Plan

VPC: `10.0.0.0/16`, split across two Availability Zones:

| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| Public A | 10.0.1.0/24 | AZ-A | ALB, NAT Gateway A |
| Public B | 10.0.2.0/24 | AZ-B | ALB, NAT Gateway B |
| Private App A | 10.0.11.0/24 | AZ-A | EC2 (ASG) |
| Private App B | 10.0.12.0/24 | AZ-B | EC2 (ASG) |
| Private DB A | 10.0.21.0/24 | AZ-A | RDS primary |
| Private DB B | 10.0.22.0/24 | AZ-B | RDS standby |

### Route Tables

- One **public route table** (`0.0.0.0/0` → Internet Gateway), shared by both public subnets.
- One **private route table per AZ** (`0.0.0.0/0` → that AZ's NAT Gateway). This avoids cross-AZ NAT traffic charges and removes a single point of failure.
- **Private DB route tables** have no default route to the internet at all — `local` only.

### NACLs

Stateless, subnet-level controls. Allow ephemeral return ports (1024–65535) explicitly, since NACLs don't track connection state the way Security Groups do. Use NACLs for coarse-grained subnet isolation (e.g., deny the DB subnet from all internet-bound traffic), not for fine-grained application logic.

### Security Groups (the real enforcement layer, stateful)

- **ALB-SG**: inbound 443/80 from `0.0.0.0/0`
- **App-SG**: inbound 80/443 from ALB-SG only
- **DB-SG**: inbound 3306/5432 from App-SG only

No Security Group should ever allow direct internet inbound to App-SG or DB-SG.

### Design Principle

Each AZ is a fully self-contained failure-isolation unit — its own public, app, and DB subnet, its own NAT Gateway, and its own route table. Losing one AZ never strands outbound internet access or routing for the other AZ.

---

## 2. Compute — EC2 + Auto Scaling Group

- **Launch Template** (not the deprecated Launch Configuration): defines AMI, instance type, IAM instance profile, user data (bootstraps the app, installs the CloudWatch agent and SSM agent), and the App-SG.
- **ASG** spans both private app subnets, e.g. `min=2, max=6` (tune to expected load), at least one instance per AZ for HA.
- **Health checks**: ASG uses the **ELB health check type** (not just EC2 status checks), so instances that are running but failing application health checks still get replaced.

### Scaling Policies

- **Target tracking** — e.g., keep average CPU at 50%, or use `ALBRequestCountPerTarget`. Simplest option, self-adjusting, and closes the loop automatically as load rises and falls.
- **Step scaling** — for spiky or predictable load, define scaling steps against CloudWatch alarm breach sizes (e.g., +2 instances if CPU > 80%, +4 if CPU > 95%).

### Feedback Loop

CloudWatch continuously collects metrics from EC2 targets → evaluates them against alarms tied to the chosen scaling policy → triggers the ASG to add or remove instances from the Launch Template → new instances immediately begin emitting their own metrics back into CloudWatch. This is a closed loop, which is what allows the system to scale back down once load drops, not just scale up.

---

## 3. ALB + WAF

- ALB deployed in the public subnets, listener on 443 (ACM certificate), with 80 → 443 redirect.
- **Listener rules** support path- or host-based routing if multiple services sit behind one ALB.
- **Target group health checks**: HTTP path (e.g. `/health`), tuned interval and healthy/unhealthy thresholds to avoid flapping.
- **WAF Web ACL** attached to the ALB (and ideally CloudFront too): AWS Managed Rules — Core Rule Set (OWASP Top 10), SQL injection rule set, plus a rate-based rule to blunt Layer 7 DDoS and brute-force attempts.

---

## 4. CloudFront

- Origin = the ALB.
- **Cache behavior**: static paths (`/static/*`, `/assets/*`) get long TTLs; dynamic/API paths get `TTL=0` or minimal caching with correct cache-key and header forwarding.
- WAF can be attached here as well, filtering malicious traffic at the edge before it ever reaches the ALB.

---

## 5. RDS Multi-AZ

- Primary and synchronous standby placed in separate private DB subnets/AZs, inside a **DB Subnet Group**.
- **Automated failover** (typically 60–120 seconds) on primary failure. The application never needs to change anything, since it always connects to the same **RDS endpoint DNS name**, which repoints to the promoted standby.
- Enable automated backups, encryption at rest (KMS), and Enhanced Monitoring.
- Note: Multi-AZ is for **high availability and failover**, not read scaling — read replicas are a separate, additional concern.

---

## 6. Route 53

- **Alias record** (apex-compatible, no extra DNS lookup cost) pointing to the ALB.
- **Health checks** against the ALB or an application health endpoint, enabling a failover routing policy if a second region is added later.

---

## 7. Systems Manager — Session Manager

- Attach an IAM instance profile with the `AmazonSSMManagedInstanceCore` policy to the Launch Template.
- No bastion host and no open port 22. Access is via `aws ssm start-session`, fully audited through CloudTrail, and works even though instances have no public IP — the SSM agent establishes an outbound-only connection to the SSM service endpoint.

---

## 8. CloudWatch + SNS

**Dashboards** should cover:
- ALB request count, latency, 5xx rate
- ASG instance count
- EC2 CPU/memory (custom metric via CloudWatch agent)
- RDS CPU, connections, replica lag

**Alarms → SNS topic → email/Slack/PagerDuty**, for example:
- ALB 5xx rate above threshold
- RDS free storage low
- ASG stuck at max capacity

These alarms double as the trigger source for step-scaling policies.

**Two independent safety nets**: Session Manager gives *reactive* access (go in and investigate when something's wrong), while CloudWatch → SNS gives *proactive* notice before a human would otherwise notice. Together they replace both a bastion host and someone watching dashboards all day.

---

## 9. IAM (cross-cutting)

- **EC2 instance profile**: least-privilege — SSM core policy plus only the specific S3/Secrets Manager/CloudWatch permissions the application needs (e.g., pull DB credentials from Secrets Manager rather than hardcoding them in user data).
- **Separate roles** for CI/CD deployment versus the runtime instance role.

---

## Suggested Build Order

1. VPC, subnets, route tables, Internet Gateway, NAT Gateways
2. Security Groups + NACLs
3. RDS Multi-AZ (provisioning takes longest — start early)
4. Launch Template + ASG + target group
5. ALB + listener rules + WAF
6. CloudFront distribution
7. Route 53 alias record
8. CloudWatch alarms + SNS + dashboards
9. Verify Systems Manager Session Manager access; confirm no bastion/SSH path exists

---

## Request Flow Summary

1. Client resolves DNS via Route 53 alias record.
2. Request hits CloudFront edge — cache hit returns instantly; cache miss forwards to origin.
3. WAF inspects the request for malicious patterns.
4. ALB listener routes by path/host rule to a healthy target.
5. A healthy EC2 instance in the ASG (AZ-A or AZ-B) receives and processes the request.
6. The instance queries the RDS primary, with the DB Security Group enforcing that only App-SG can connect.
7. The response returns along the same path in reverse.

Every hop in this chain is independently scalable and independently observable — a failure at any single step does not take down the whole path. If all targets in one AZ fail their health check, the ALB simply stops routing there and sends all traffic to the healthy AZ, with no DNS change or manual intervention required.

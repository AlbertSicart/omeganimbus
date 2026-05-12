# OmegaNimbus Day 11: Security Monitoring Stack — ALB, Private Subnet Migration & CloudWatch Observability

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** EC2 · Ubuntu 24.04 · Prometheus · Node Exporter · Grafana · ALB · ACM · Route 53 · CloudWatch · CloudTrail · GuardDuty · EventBridge · Lambda · VPC · SSM Endpoints  
**Date:** May 2026

---

## Overview

Day 11 completed the OmegaNimbus Security Monitoring Stack in two phases.

The first phase (previous session) built the core observability layer: Prometheus + Node Exporter + Grafana on EC2, with an on-demand activation system via Lambda + EventBridge Scheduler that keeps costs near zero when not in use.

This session completed three remaining objectives:

1. **Private Subnet Migration** — moved the EC2 instance from a public subnet to a private subnet using AMI-based recreation, establishing correct network architecture
2. **HTTPS Exposure via ALB** — Application Load Balancer in public subnets routing to Grafana in the private subnet, with ACM certificate at `siem.omeganimbus.com`
3. **CloudWatch Security Observability** — CloudTrail → CloudWatch Logs, GuardDuty → CloudWatch Logs via EventBridge, and a Grafana Security Operations dashboard querying all sources

---

## Architecture

```
Internet
    │
    ▼
siem.omeganimbus.com (Route 53 A alias)
    │
    ▼
omeganimbus-alb (ALB, Internet-facing)
Subnets: subnet-0d2a93f4bf4ebbe19 (public 1a)
         subnet-016faa724fdfd09b5 (public 1b)
Security group: omeganimbus-alb-sg (inbound 443+80, outbound all)
    │
    ├── Listener HTTP:80  → Redirect HTTPS 443 (301)
    └── Listener HTTPS:443 → Forward → omeganimbus-siem-tg
            │
            ▼
        Target Group: omeganimbus-siem-tg (HTTP:3000)
            │
            ▼
EC2: omeganimbus-siem (i-098fc9967d9d6d4af)
VPC: vpc-03291e8312144dcd3
Subnet: subnet-0523c357184d1e7f9 (PRIVATE 1a)
IP: 10.0.128.190 — no public IP
    ├── Prometheus :9090
    ├── Node Exporter :9100
    └── Grafana :3000
            │
            ├── Datasource: Prometheus (system metrics)
            └── Datasource: CloudWatch (security logs + metrics)
                    │
                    ├── /aws/lambda/omeganimbus-start-siem
                    ├── /aws/cloudtrail/omeganimbus
                    └── /aws/events/guardduty/omeganimbus

CloudWatch VPC Endpoints:
    ├── com.amazonaws.eu-north-1.monitoring (vpce-0661afb6fffae6629)
    └── com.amazonaws.eu-north-1.logs (vpce-01190aaff97c9654e)

Security Monitoring Sources:
    ├── CloudTrail → omeganimbus-trail → /aws/cloudtrail/omeganimbus
    └── GuardDuty → EventBridge rule → /aws/events/guardduty/omeganimbus

On-Demand Activation:
API Gateway POST /siem → Lambda omeganimbus-start-siem
    ├── EC2 StartInstances
    ├── EventBridge Scheduler: at(T+2h) → omeganimbus-stop-siem
    └── SNS: omeganimbus-security-alerts
```

---

## Part 1 — Private Subnet Migration

The original instance was in a public subnet (`subnet-0d2a93f4bf4ebbe19`) without a public IP — acceptable for initial testing but architecturally incorrect for a monitoring instance that should never be directly internet-reachable.

AWS does not support changing the subnet of an existing instance. Migration via AMI:

1. Created AMI `omeganimbus-siem-ami` from the existing instance (with reboot for data consistency)
2. Launched new instance from the AMI into `subnet-0523c357184d1e7f9` (private 1a) with auto-assign public IP disabled
3. Verified Grafana, Prometheus, and Node Exporter running via SSM session
4. Terminated the original instance
5. Updated `INSTANCE_ID` in both Lambda functions

### New Instance

| Parameter | Value |
|---|---|
| Instance ID | i-098fc9967d9d6d4af |
| AMI | omeganimbus-siem-ami |
| Type | t3.micro |
| VPC | vpc-03291e8312144dcd3 |
| Subnet | subnet-0523c357184d1e7f9 (private 1a) |
| Private IP | 10.0.128.190 |
| Public IP | None |
| Security group | omeganimbus-siem-sg |
| IAM role | omeganimbus-siem-role |

---

## Part 2 — ALB + HTTPS

### ACM Certificate

Requested via Certificate Manager (eu-north-1) for `siem.omeganimbus.com`. DNS validation via Route 53 — issued in under 2 minutes.

- **Certificate ARN:** `arn:aws:acm:eu-north-1:247906201225:certificate/0ff10ddf-0a47-44fc-946b-b3317b1ab093`

### Security Groups

**omeganimbus-alb-sg** (new):
- Inbound: HTTPS 443 from `0.0.0.0/0`
- Inbound: HTTP 80 from `0.0.0.0/0`
- Outbound: All traffic

**omeganimbus-siem-sg** (updated):
- Added: TCP 3000 from `omeganimbus-alb-sg`
- Removed: SSH from `0.0.0.0/0` (was incorrectly open)

### Target Group

| Parameter | Value |
|---|---|
| Name | omeganimbus-siem-tg |
| Target type | Instance |
| Protocol:Port | HTTP:3000 |
| Health check path | /api/health |
| VPC | omeganimbus-vpc |

Target registered manually — instance stays registered permanently. ALB marks it healthy/unhealthy based on health checks as the instance starts and stops.

### ALB

| Parameter | Value |
|---|---|
| Name | omeganimbus-alb |
| Scheme | Internet-facing |
| Subnets | subnet-0d2a93f4bf4ebbe19 (public 1a), subnet-016faa724fdfd09b5 (public 1b) |
| Security group | omeganimbus-alb-sg |
| DNS name | omeganimbus-alb-335383902.eu-north-1.elb.amazonaws.com |
| Hosted zone | Z23TAZ6LKFMNIO |
| Idle timeout | 300s |

**Listeners:**
- HTTP:80 → Redirect to HTTPS:443 (301 Permanently moved)
- HTTPS:443 → Forward to `omeganimbus-siem-tg` · Certificate: `siem.omeganimbus.com`

### Route 53

Added A alias record: `siem.omeganimbus.com` → `omeganimbus-alb` (eu-north-1).

---

## Part 3 — CloudWatch Security Observability

### CloudTrail → CloudWatch Logs

Enabled CloudWatch Logs delivery on `omeganimbus-trail`:

- **Log group:** `/aws/cloudtrail/omeganimbus`
- **IAM role:** `omeganimbus-cloudtrail-cw-role` (created automatically)

All API activity in the account now streams to CloudWatch in near real-time.

### GuardDuty → CloudWatch Logs

Created EventBridge rule `omeganimbus-guardduty-to-cw-logs`:

- **Event pattern:** `source: aws.guardduty`, `detail-type: GuardDuty Finding`
- **Target:** CloudWatch log group `/aws/events/guardduty/omeganimbus`

### CloudWatch VPC Endpoints

The EC2 instance has no internet access (private subnet, no NAT Gateway). Two Interface endpoints added to allow Grafana to reach CloudWatch APIs:

| Endpoint ID | Service | Private DNS |
|---|---|---|
| vpce-0661afb6fffae6629 | com.amazonaws.eu-north-1.monitoring | Yes |
| vpce-01190aaff97c9654e | com.amazonaws.eu-north-1.logs | Yes |

Both endpoints use `omeganimbus-ssm-endpoints-sg` (TCP 443 from `10.0.0.0/16`).

### Grafana CloudWatch Datasource

- **Authentication:** AWS SDK Default (uses EC2 instance role)
- **Default region:** eu-north-1
- **IAM policy added to omeganimbus-siem-role:** CloudWatchReadOnlyAccess

### OmegaNimbus Security Operations Dashboard

Four panels:

| Panel | Type | Source | Query |
|---|---|---|---|
| Lambda — SIEM Activations | Logs | /aws/lambda/omeganimbus-start-siem | fields @timestamp, @message \| sort desc \| limit 20 |
| CloudTrail — Recent API Activity | Logs | /aws/cloudtrail/omeganimbus | fields @timestamp, eventName, userIdentity.type, sourceIPAddress \| sort desc \| limit 50 |
| Lambda Invocations | Time series | CloudWatch Metrics AWS/Lambda | Invocations, Sum, all functions |
| GuardDuty — Findings | Bar chart | /aws/events/guardduty/omeganimbus | fields detail.type \| stats count() by detail.type |

---

## Debugging Notes

**EC2 subnet change not possible**
AWS does not allow changing the subnet of an existing instance. The only option is AMI-based recreation. The original instance had been in a public subnet without a public IP — technically safe but architecturally wrong. Corrected via AMI migration.

**ALB health check returning unhealthy**
Grafana's root path `/` returns a 302 redirect to `/login`. Changed health check path to `/api/health`, which returns HTTP 200 directly. Target became healthy immediately.

**CloudWatch Logs query returning 504**
Grafana was proxying CloudWatch Logs Insights queries through the ALB. Default ALB idle timeout is 60 seconds — CloudWatch Logs Insights queries can take longer. Increased ALB idle timeout to 300 seconds.

**Lambda register_targets failing on stopped instance**
Initial Lambda design called `register_targets` before `start_instances`. AWS rejects registering instances not in running state. Simplified architecture: register the target once manually and leave it permanently registered. ALB health checks handle the healthy/unhealthy state automatically.

**Grafana CloudWatch returning no data despite correct config**
Root cause: ALB idle timeout (60s) was cutting off the CloudWatch Logs Insights queries before they completed. Fixed by increasing ALB idle timeout to 300s. After fix, all log groups returned data correctly.

**GuardDuty finding: Policy:IAMUser/RootCredentialUsage**
GuardDuty detected root credential usage during today's session (EC2 API calls `DescribeInstanceStatus` and `GetInstanceProfile`). Expected — root account was used for console work. Action item: create an IAM admin user and stop using root for daily operations.

---

## Cost Summary

| Service | Usage | Monthly Cost |
|---|---|---|
| EC2 t3.micro | ~2h/day on-demand | ~$0.65 |
| EBS 8 GiB gp3 | Always | ~$0.64 |
| ALB | Always active | ~$16.00 |
| ACM Certificate | Free with ALB | $0 |
| CloudWatch VPC Endpoints (x2) | Always active | ~$0.21 each = ~$0.42 |
| CloudWatch Logs ingestion | CloudTrail + GuardDuty | ~$0.50 |
| SSM VPC Endpoints (x3) | Always active | ~$0.63 |
| Lambda (start + stop) | <100K/month | $0 |
| EventBridge Scheduler | Per schedule | ~$0 |
| Route 53 | Existing hosted zone | ~$0 |

**Total added cost this session: ~$18/month** (ALB dominates)

---

## Key Concepts Practiced

- AMI-based instance migration between subnets
- ALB architecture: listeners, rules, target groups, health checks
- HTTPS termination at the load balancer with ACM certificates
- ALB → private subnet routing (internet-facing ALB, private targets)
- Target group health check tuning (`/api/health` vs `/`)
- ALB idle timeout and its effect on long-running queries
- CloudTrail → CloudWatch Logs integration
- EventBridge rules for routing GuardDuty findings to CloudWatch
- VPC Interface Endpoints for CloudWatch (monitoring + logs services)
- Grafana CloudWatch datasource with EC2 instance role authentication
- CloudWatch Logs Insights query language
- GuardDuty finding types: `Policy:IAMUser/RootCredentialUsage`
- Security monitoring vs SIEM: the distinction matters

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| security.html button | "Request SIEM Access" with dynamic status | High |
| IAM admin user | Stop using root for daily operations | High |
| Day 12: RDS + DynamoDB Streams | Data layer expansion | Next session |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

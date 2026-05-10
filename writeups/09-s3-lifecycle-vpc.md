# OmegaNimbus Days 9 & 10: S3 Versioning & Lifecycle + VPC Network Architecture

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** S3 · VPC · Subnets · Internet Gateway · Route Tables · Security Groups · Lambda  
**Date:** May 2026

---

## Overview

Two sessions combined into one writeup:

**Day 9 — S3 Versioning & Lifecycle**  
S3 versioning enabled on the main bucket with a lifecycle rule to automatically delete noncurrent object versions after 30 days — balancing data recovery capability with cost control.

**Day 10 — VPC Network Architecture**  
Custom VPC created with public and private subnets across two Availability Zones, Internet Gateway, route tables, S3 Gateway endpoint, and Security Group. Three Lambda functions migrated into the VPC. One Lambda (`omeganimbus-guardduty-dashboard`) kept outside the VPC due to its dependency on public AWS APIs without a NAT Gateway.

Additionally, the GuardDuty security dashboard was updated with a **Resolved Findings** section — a static, curated table of the three real security findings detected and remediated during the project.

---

## Architecture

```
Internet
    │
    ▼
Internet Gateway (igw-0c74818366c2c0272)
    │
    ▼
omeganimbus-vpc (10.0.0.0/16) — vpc-03291e8312144dcd3
    │
    ├── Public Subnets
    │     ├── omeganimbus-subnet-public1-eu-north-1a  (10.0.0.0/20)
    │     └── omeganimbus-subnet-public2-eu-north-1b  (10.0.16.0/20)
    │
    └── Private Subnets
          ├── omeganimbus-subnet-private1-eu-north-1a (10.0.128.0/20)
          │     └── Lambda: omeganimbus-visitor-counter-cfn
          │     └── Lambda: omeganimbus-contact-form-cfn
          │     └── Lambda: omeganimbus-guardduty-dashboard (removed — needs public API access)
          └── omeganimbus-subnet-private2-eu-north-1b (10.0.144.0/20)

S3 Gateway Endpoint (vpce-09fd35150eb6a7557)
    └── Attached to private subnet route tables
    └── Direct S3 access from VPC without internet

Security Group: omeganimbus-lambda-sg (sg-017126f8346b0a316)
    └── Inbound: none
    └── Outbound: all traffic allowed
```

---

## Part 1 — S3 Versioning & Lifecycle (Day 9)

**Bucket:** `omeganimbus.com-cfn`

### Versioning

Enabled via S3 → Properties → Bucket Versioning → Enable. With versioning active, every object modification creates a new version rather than overwriting. Previous versions are retained and recoverable — useful for accidental overwrites during CI/CD deploys.

### Lifecycle Rule

**Rule name:** `omeganimbus-lifecycle`  
**Scope:** All objects in the bucket  
**Action:** Permanently delete noncurrent versions of objects  
**Retention:** 30 days after object becomes noncurrent

This means: when CodePipeline deploys a new version of `index.html`, the previous version is retained for 30 days before being automatically deleted. Provides a recovery window without unbounded storage accumulation.

| Setting | Value |
|---|---|
| Versioning | Enabled |
| Noncurrent version retention | 30 days |
| Storage class transitions | None (Standard only) |
| Estimated cost impact | ~$0 at portfolio scale |

---

## Part 2 — VPC Network Architecture (Day 10)

### VPC Configuration

**VPC ID:** `vpc-03291e8312144dcd3`  
**CIDR:** `10.0.0.0/16` (65,536 IPs)  
**Region:** eu-north-1  
**AZs:** eu-north-1a, eu-north-1b  
**DNS hostnames:** Enabled  
**DNS resolution:** Enabled  

### Resources created automatically

| Resource | ID | Details |
|---|---|---|
| VPC | vpc-03291e8312144dcd3 | 10.0.0.0/16 |
| Public subnet 1a | subnet-0d2a93f4bf4ebbe19 | 10.0.0.0/20 |
| Public subnet 1b | subnet-016faa724fdfd09b5 | 10.0.16.0/20 |
| Private subnet 1a | subnet-0523c357184d1e7f9 | 10.0.128.0/20 |
| Private subnet 1b | subnet-09fdb4faee8424162 | 10.0.144.0/20 |
| Internet Gateway | igw-0c74818366c2c0272 | Attached to VPC |
| Public route table | rtb-0768eb12c425b206e | 0.0.0.0/0 → IGW |
| Private route table 1 | rtb-030db354d33875f65 | Local only + S3 endpoint |
| Private route table 2 | rtb-0e0eb53f2c3ce2028 | Local only + S3 endpoint |
| S3 Gateway Endpoint | vpce-09fd35150eb6a7557 | Free — direct S3 access |

**NAT Gateway:** Not deployed — cost (~$32/month) not justified for serverless architecture. Lambdas in private subnets that need internet access use VPC endpoints instead.

### Security Group

**Name:** `omeganimbus-lambda-sg`  
**ID:** `sg-017126f8346b0a316`  
**VPC:** omeganimbus-vpc  
**Inbound:** No rules — Lambdas don't receive direct traffic  
**Outbound:** All traffic allowed — Lambdas need to call AWS APIs

### Lambda VPC Integration

Three Lambdas moved into the VPC, deployed in both private subnets for multi-AZ availability:

| Lambda | VPC | Subnets | Reason |
|---|---|---|---|
| omeganimbus-visitor-counter-cfn | omeganimbus-vpc | private1a + private2b | DynamoDB access via S3 endpoint |
| omeganimbus-contact-form-cfn | omeganimbus-vpc | private1a + private2b | SES access via outbound |
| omeganimbus-guardduty-dashboard | No VPC | — | Requires public GuardDuty API — no NAT |

**IAM permission added to each Lambda role:** `AWSLambdaVPCAccessExecutionRole` — grants permission to create Elastic Network Interfaces (ENIs) in the VPC, required for Lambda VPC attachment.

---

## Debugging Notes

**Lambda VPC attachment failed — missing IAM permission**  
Error: `The provided execution role does not have permissions to call CreateNetworkInterface on EC2`. Lambda needs to create an ENI to attach to the VPC. Fixed by attaching `AWSLambdaVPCAccessExecutionRole` to each Lambda's execution role before saving the VPC configuration.

**GuardDuty dashboard stopped loading after VPC migration**  
`omeganimbus-guardduty-dashboard` Lambda moved to private subnets. Private subnets have no internet access without a NAT Gateway. GuardDuty API is a public AWS endpoint — unreachable from private subnets without NAT or a VPC endpoint. Solution: removed the Lambda from the VPC entirely. It only reads from GuardDuty and doesn't access any private VPC resources, so there's no architectural benefit to placing it inside the VPC.

**VPC endpoint vs NAT Gateway decision**  
A GuardDuty VPC endpoint would have solved the connectivity issue but costs ~$7/month per AZ. A NAT Gateway costs ~$32/month. For a portfolio running near zero cost, neither is justified. The correct decision is to keep public-API-only Lambdas outside the VPC.

---

## Part 3 — GuardDuty Dashboard: Resolved Findings Section

Added a static **Resolved Findings** table to `security.html` documenting the three real security findings detected and remediated during the OmegaNimbus build. This replaces the idea of pulling archived findings from GuardDuty dynamically — the archived findings pool contains ~400 sample findings generated during Day 4 testing, which would pollute a dynamic feed.

### Resolved findings documented

| Severity | Type | Detected | Resolution |
|---|---|---|---|
| HIGH 8.0 | Policy:S3/BucketAnonymousAccessGranted | 2026-05-06 04:23 UTC | CloudFront OAC migration. Block Public Access enabled. |
| LOW 2.0 | Policy:S3/BucketBlockPublicAccessDisabled | 2026-05-06 04:23 UTC | All four Block Public Access settings enabled. |
| LOW 2.0 | Stealth:IAMUser/CloudTrailLoggingDisabled | 2026-05-06 04:05 UTC | CloudTrail re-enabled with KMS. Root usage discontinued. |

The result is a security dashboard that tells a complete story: active findings at the top, resolved findings below — demonstrating both detection capability and remediation discipline.

---

## Cost Summary

| Service | Usage | Monthly Cost |
|---|---|---|
| S3 Versioning | Noncurrent versions retained 30 days | ~$0 at portfolio scale |
| VPC | VPC, subnets, IGW, route tables | $0 |
| S3 Gateway Endpoint | Included with VPC | $0 |
| NAT Gateway | Not deployed | $0 |
| Lambda ENIs | Auto-managed by AWS | $0 |

**Total added cost: $0/month**

---

## Key Concepts Practiced

- S3 versioning: object version lifecycle, noncurrent version management
- S3 lifecycle rules: automated cost control for versioned buckets
- VPC design: CIDR planning, public/private subnet architecture, multi-AZ
- Internet Gateway: public subnet internet access
- Route tables: public (0.0.0.0/0 → IGW) vs private (local only)
- S3 Gateway Endpoint: free VPC-native S3 access without NAT
- Security groups: stateful firewall, Lambda inbound/outbound rules
- Lambda VPC integration: ENI creation, AWSLambdaVPCAccessExecutionRole
- NAT Gateway trade-offs: cost vs connectivity for private subnets
- VPC endpoint vs NAT Gateway: decision framework for public API access
- Security dashboard design: active vs resolved findings narrative

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| WAF + Shield | DDoS protection and rate limiting on CloudFront | Medium |
| GuardDuty dashboard improvements | Sample findings generator | Low |
| Auto Scaling + ELB | Load balancing and auto scaling groups | Low |
| SIEM Stack | Wazuh + OpenSearch + Grafana on EC2 inside VPC | Low |
| RDS + DynamoDB Streams | Relational database layer, event-driven Lambda | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

---

## Appendix — Frontend & Portfolio Updates

Several improvements to the web portfolio were made during this session alongside the infrastructure work.

### writeups.html — New dedicated page

Created `writeups.html` as a standalone page with two tabs:

- **OmegaNimbus Build** — loads writeups dynamically from the GitHub API (`/repos/AlbertSicart/omeganimbus/contents/writeups`). Cards display day number, category, title, and description. Each links to the raw markdown on GitHub.
- **Security Research** — static placeholder for upcoming HTB writeups and AWS security research.

### Nav — Dropdown for Notes

Replaced the single "Notes" link in the nav with a dropdown containing two options:

- **Study Notes** → `notes.html`
- **Writeups** → `writeups.html`

Applied across all pages: `index.html`, `security.html`, `notes.html`, `rekognition.html`, `writeups.html`.

CSS fix required: `.nav-links li { display: flex; align-items: center; }` to prevent the dropdown item from rendering misaligned vertically relative to other nav items.

### About section — Updated copy

Replaced the generic About description with the LinkedIn-aligned copy:

> History and Archaeology graduate turned cybersecurity consultant. That shift wasn't accidental — it reflects how I think: systems, context, patterns, risk. Currently at Azertium IT, focused on GRC and security strategy. Building toward Cloud Security through AWS certifications and hands-on infrastructure at omeganimbus.com.

Skills updated to match LinkedIn top skills: Cloud Security, GRC, AWS, Cybersecurity, DevOps.

### Hero section — Removed Japanese characters

Removed the `鬼 雲 · Oni + Nimbus` subtitle from the hero section — inconsistent with the technical portfolio branding.

### security.html — Resolved Findings section

Added a static **Resolved Findings** table below the active GuardDuty feed documenting the three real security findings detected and remediated during the build:

| Severity | Type | Detected | Resolution |
|---|---|---|---|
| HIGH 8.0 | Policy:S3/BucketAnonymousAccessGranted | 2026-05-06 04:23 UTC | CloudFront OAC migration |
| LOW 2.0 | Policy:S3/BucketBlockPublicAccessDisabled | 2026-05-06 04:23 UTC | Block Public Access enabled |
| LOW 2.0 | Stealth:IAMUser/CloudTrailLoggingDisabled | 2026-05-06 04:05 UTC | CloudTrail re-enabled with KMS |

The archived GuardDuty pool was not used dynamically due to ~400 sample findings from Day 4 testing polluting the feed.

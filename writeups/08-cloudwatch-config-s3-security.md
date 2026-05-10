# OmegaNimbus Day 8: CloudWatch Observability, AWS Config Compliance & S3 Security Hardening

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** CloudWatch · AWS Config · S3 · CloudFront · Origin Access Control (OAC)  
**Date:** May 2026

---

## Overview

This session covered three objectives:

1. **CloudWatch dashboard** — real-time observability metrics for API Gateway, Lambda, and DynamoDB
2. **CloudWatch alarm** — automated alerting on Lambda errors via SNS
3. **AWS Config** — compliance rules evaluating resource configuration continuously
4. **S3 security hardening** — remediation of a HIGH severity GuardDuty finding by implementing CloudFront Origin Access Control (OAC)

The Config evaluation surfaced a real security issue: two S3 buckets with public access enabled, confirmed by an active GuardDuty HIGH finding (`Policy:S3/BucketAnonymousAccessGranted`). The finding was remediated by migrating from public S3 website hosting to private S3 + OAC — the recommended AWS architecture for static site delivery.

---

## Architecture

```
Browser → Route 53 → CloudFront (E2Y1NNQD999CRW)
                          │
                          │ OAC (signed requests)
                          ▼
                    S3 (omeganimbus.com-cfn)
                    Block Public Access: ON
                    Bucket Policy: CloudFront only

Observability Pipeline:
API Gateway → CloudWatch Metrics → Dashboard (omeganimbus-dashboard)
Lambda      → CloudWatch Metrics → Dashboard + Alarm → SNS → Email
DynamoDB    → CloudWatch Metrics → Dashboard

Compliance Pipeline:
AWS Config → Continuous recording → Rules evaluation
          └── s3-bucket-public-read-prohibited     → NONCOMPLIANT (remediated)
          └── lambda-function-public-access-prohibited → COMPLIANT
          └── cloudtrail-enabled                   → COMPLIANT
```

---

## Part 1 — CloudWatch Dashboard

**Dashboard name:** `omeganimbus-dashboard`  
**Region:** eu-north-1

### Widgets

| Widget | Metrics | Source |
|---|---|---|
| API Gateway | Latency, Count | ApiGateway → ApiId `judzwkiy9h`, Stage `$default` |
| Lambda | Errors, Invocations | omeganimbus-visitor-counter-cfn, omeganimbus-contact-form-cfn, omeganimbus-guardduty-dashboard, omeganimbus-rekognition |
| DynamoDB | ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits | omeganimbus-visitors-cfn |

All widgets use a 1-week time range with line chart visualization.

---

## Part 2 — CloudWatch Alarm

**Alarm name:** `omeganimbus-lambda-errors`  
**Metric:** Lambda → Errors → `omeganimbus-visitor-counter-cfn`  
**Condition:** Errors > 1  
**Action:** SNS topic `omeganimbus-lambda-errors` → email to `sicart@protonmail.ch`  
**State:** Insufficient data (normal on creation — requires data accumulation)

SNS subscription confirmed via email after creation.

---

## Part 3 — AWS Config

**Recording strategy:** All resource types with customizable overrides  
**Recording frequency:** Continuous  
**Delivery:** S3 bucket `config-bucket-247906201225`  
**IAM:** AWS Config service-linked role (auto-created)

### Rules configured

| Rule | Purpose | Result |
|---|---|---|
| `s3-bucket-public-read-prohibited` | No S3 bucket should allow public read access | ⚠️ 2 Noncompliant (remediated) |
| `lambda-function-public-access-prohibited` | No Lambda should have public resource policy | ✅ Compliant |
| `cloudtrail-enabled` | CloudTrail must be active | ✅ Compliant |

### Config finding → GuardDuty correlation

The `s3-bucket-public-read-prohibited` rule flagged `omeganimbus.com-cfn` and `omeganimbus.com` as noncompliant. Cross-referencing with the GuardDuty dashboard confirmed an active **HIGH 8.0** finding:

- `Policy:S3/BucketAnonymousAccessGranted` — public anonymous access granted on `omeganimbus.com-cfn`
- `Policy:S3/BucketBlockPublicAccessDisabled` — Block Public Access disabled

This was not a false positive. The bucket was genuinely public — a legacy of the initial S3 static website hosting setup from Day 1.

---

## Part 4 — S3 Security Hardening (OAC Migration)

### Problem

S3 static website hosting requires public bucket access by design. The original architecture used the S3 website endpoint as the CloudFront origin, which necessitated `BlockPublicAccess: OFF` and a public bucket policy. This triggered GuardDuty HIGH findings and AWS Config noncompliance.

### Solution: CloudFront Origin Access Control (OAC)

OAC allows CloudFront to access a **private** S3 bucket using signed requests — no public access required. CloudFront authenticates to S3 using AWS SigV4 signing, and the bucket policy restricts access to only the specific CloudFront distribution.

### Steps

**1. Enable Block Public Access on the bucket**  
S3 → `omeganimbus.com-cfn` → Permissions → Block public access → Enable all 4 settings.

**2. Change CloudFront origin from website endpoint to REST endpoint**  
Old origin: `omeganimbus.com-cfn.s3-website.eu-north-1.amazonaws.com`  
New origin: `omeganimbus.com-cfn.s3.eu-north-1.amazonaws.com`

**3. Create OAC**  
CloudFront → Edit origin → Origin access → Origin access control settings → Create new OAC. Signing behavior: Sign requests (recommended). Origin type: S3.

**4. Update bucket policy**  
Replaced public `s3:GetObject` policy with OAC-scoped policy:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::omeganimbus.com-cfn/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::247906201225:distribution/E2Y1NNQD999CRW"
        }
      }
    }
  ]
}
```

**5. Invalidate CloudFront cache**  
Created invalidation `/*` to force immediate propagation.

**Result:** Site loads correctly. Block Public Access enabled. GuardDuty finding pending resolution (up to 24h evaluation cycle).

---

## Debugging Notes

**CloudWatch widget lost after first save**  
Lambda widget disappeared after saving the dashboard. Root cause: dashboard was not explicitly saved after adding the widget. Fixed by recreating and saving immediately after each widget.

**AWS Config shows 0 resources on first load**  
Normal behavior — Config needs 5-10 minutes to discover and inventory all resources after initial setup. Rules evaluate after discovery completes.

**S3 website endpoint incompatible with OAC**  
CloudFront origin was configured as the S3 website endpoint (`s3-website.eu-north-1`), which doesn't support OAC. OAC requires the S3 REST endpoint (`s3.eu-north-1`). Changed origin domain accordingly — CloudFront displayed a warning suggesting the website endpoint, which was intentionally ignored.

**GuardDuty HIGH finding still active after remediation**  
GuardDuty evaluates findings on a periodic cycle (up to 24 hours). The finding will be automatically resolved in the next evaluation cycle. Configuration is correct.

---

## Security Architecture: Before vs After

| | Before (Day 1) | After (Day 8) |
|---|---|---|
| S3 Block Public Access | OFF | ON |
| Bucket access | Public (anyone) | Private (CloudFront only) |
| CloudFront → S3 auth | None (public URL) | OAC signed requests |
| GuardDuty finding | HIGH 8.0 active | Pending resolution |
| Config compliance | Noncompliant | Compliant |

---

## Cost Summary

| Service | Usage | Monthly Cost |
|---|---|---|
| CloudWatch Dashboard | 1 dashboard, 3 widgets | $3.00 |
| CloudWatch Alarm | 1 alarm | $0.10 |
| CloudWatch Metrics | Standard resolution | $0 (free tier) |
| AWS Config | ~50 resources recorded | ~$0.05 |
| OAC | Included with CloudFront | $0 |

**Total added cost: ~$3.15/month**  
Note: CloudWatch dashboards cost $3/month per dashboard after the free tier.

---

## Key Concepts Practiced

- CloudWatch dashboards: metric widgets, time ranges, multi-metric graphs
- CloudWatch alarms: threshold conditions, SNS integration, subscription confirmation
- AWS Config: continuous recording, managed rules, compliance evaluation
- Config rule correlation with GuardDuty findings
- S3 Block Public Access: four independent settings
- CloudFront Origin Access Control (OAC) vs Legacy Origin Access Identity (OAI)
- S3 REST endpoint vs S3 website endpoint — architectural differences
- SigV4 signed requests for CloudFront → S3 authentication
- Bucket policy scoping to specific CloudFront distribution ARN
- Security finding remediation workflow: detect → analyze → fix → verify

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| Study Notes polish | CCP notes improvements, SAA-C03 structure prep | Medium |
| S3 versioning + lifecycle | Versioning and lifecycle policies on main bucket | Medium |
| VPC | Custom VPC with public/private subnets, security groups | Medium |
| WAF + Shield | DDoS protection on CloudFront | Low |
| Auto Scaling + ELB | Load balancing and auto scaling groups | Low |
| SIEM Stack | Wazuh + OpenSearch + Grafana on EC2 | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

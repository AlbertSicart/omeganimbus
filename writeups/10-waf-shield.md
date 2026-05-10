# OmegaNimbus Day 10: WAF + Shield — Layer 7 Protection on CloudFront

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** AWS WAF · AWS Shield Standard · Lambda · API Gateway · CloudWatch · CloudFront  
**Date:** May 2026

---

## Overview

This session added a full web application firewall layer to the OmegaNimbus infrastructure, completing the security stack with L7 threat protection on top of the existing GuardDuty + CloudTrail + Security Hub monitoring pipeline.

Three objectives:

1. **AWS WAF** — Web ACL on CloudFront with managed rule groups (OWASP Top 10, known bad inputs, IP reputation) and a custom rate-limiting rule
2. **AWS Shield Standard** — review and documentation of the always-on DDoS protection already active on the account
3. **WAF Metrics Dashboard** — real-time allowed/blocked request counts surfaced in `security.html` via a new Lambda + API Gateway endpoint

The result is a production-grade L7 security layer protecting the CloudFront distribution, with live traffic visibility in the SecOps dashboard.

---

## Architecture

```
Internet
    │
    ▼
AWS Shield Standard (L3/L4 DDoS — automatic, always on)
    │
    ▼
CloudFront (E2Y1NNQD999CRW)
    │
    ▼
AWS WAF — omeganimbus-waf (Web ACL)
    ├── AWS Core Rule Set (CRS) — OWASP Top 10         [700 WCU]
    ├── Known Bad Inputs — exploit pattern matching     [200 WCU]
    ├── Amazon IP Reputation List — threat intel IPs   [ 25 WCU]
    └── Custom Rate Limit — 1000 req / 5 min / IP     [  2 WCU]
    │
    ▼
S3 Origin (omeganimbus.com-cfn) via OAC

WAF Metrics Pipeline:
CloudFront → WAF → CloudWatch (AWS/WAFV2, us-east-1)
                        │
                        ▼
            Lambda (omeganimbus-waf-metrics, eu-north-1)
            boto3 client → region_name='us-east-1'
                        │
                        ▼
            API Gateway GET /waf
                        │
                        ▼
            security.html — WAF + Shield dashboard section
```

---

## Part 1 — AWS WAF

### Web ACL

**Name:** `omeganimbus-waf`  
**Scope:** CloudFront (Global — managed from us-east-1)  
**Associated resource:** `arn:aws:cloudfront::247906201225:distribution/E2Y1NNQD999CRW`

WAF for CloudFront must be created in us-east-1 regardless of the CloudFront origin region. This is an AWS architectural constraint — all CloudFront-scoped Web ACLs live in the global us-east-1 WAF namespace.

### Rule Groups

| Rule | Type | WCU | Action |
|---|---|---|---|
| AWS-AWSManagedRulesCommonRuleSet | AWS Managed | 700 | Block (default) |
| AWS-AWSManagedRulesKnownBadInputsRuleSet | AWS Managed | 200 | Block (default) |
| AWS-AWSManagedRulesAmazonIpReputationList | AWS Managed | 25 | Block (default) |
| omeganimbus-rate-limit | Custom (Rate-based) | 2 | Block |

**Total WCU:** 927 / 5000 maximum

### Rate Limiting Rule

```
Rule type:        Rate-based
Evaluation window: 5 minutes
Threshold:        1000 requests per IP
Action:           Block (HTTP 403)
Scope:            All requests to the Web ACL
```

1000 requests per 5 minutes is a conservative threshold for a portfolio site. Legitimate users will never approach it; automated scanners and brute-force tools will be blocked after the first burst.

### Rule Evaluation Order

WAF evaluates rules in priority order — the managed rule groups run first (CRS → Known Bad Inputs → IP Reputation), then the custom rate limit rule. This means a request from a known-malicious IP is blocked by the reputation list before the rate limit even fires.

---

## Part 2 — AWS Shield Standard

Shield Standard is automatically active on all AWS accounts at no additional cost. It provides:

- **L3/L4 DDoS mitigation** — volumetric and protocol attacks (UDP floods, SYN floods, reflection attacks)
- **Automatic inline mitigation** — no manual intervention required
- **Protected resources** — CloudFront distributions, Route 53 hosted zones, API Gateway endpoints

Shield Standard does not expose per-account CloudWatch metrics — those are only available with Shield Advanced ($3,000/month). For OmegaNimbus, Shield Standard provides the necessary baseline protection for a portfolio workload.

**Shield Advanced** was evaluated and intentionally not activated. The cost is disproportionate for a personal portfolio, and the additional features (DRT access, attack forensics, cost protection) are not relevant at this scale.

---

## Part 3 — WAF Metrics Lambda

### Problem

WAF metrics for CloudFront distributions are published to CloudWatch in **us-east-1**, not in the distribution's origin region. The existing Lambda functions live in eu-north-1 (main stack) or eu-west-1 (Rekognition/Lex). A cross-region boto3 client is needed.

### Solution

New Lambda in eu-north-1 with a CloudWatch client explicitly targeting us-east-1:

```python
import json
import boto3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    client = boto3.client('cloudwatch', region_name='us-east-1')
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=24)
    
    web_acl_name = 'omeganimbus-waf'

    def get_metric(metric_name):
        response = client.get_metric_statistics(
            Namespace='AWS/WAFV2',
            MetricName=metric_name,
            Dimensions=[
                {'Name': 'Rule', 'Value': 'ALL'},
                {'Name': 'WebACL', 'Value': web_acl_name},
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,
            Statistics=['Sum']
        )
        datapoints = response.get('Datapoints', [])
        return int(sum(d['Sum'] for d in datapoints)) if datapoints else 0

    allowed = get_metric('AllowedRequests')
    blocked = get_metric('BlockedRequests')

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*',
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'allowed': allowed,
            'blocked': blocked,
            'period': '24h',
            'timestamp': datetime.utcnow().isoformat()
        })
    }
```

**Lambda config:**
- Name: `omeganimbus-waf-metrics`
- Region: eu-north-1
- Runtime: Python 3.12
- Architecture: arm64
- IAM: `CloudWatchReadOnlyAccess` + inline policy for `cloudwatch:GetMetricStatistics`

**API Gateway:**
- Route: `GET /waf`
- API: `omeganimbus-api-cfn` (judzwkiy9h)
- Integration: `omeganimbus-waf-metrics`

**Sample response:**
```json
{
  "allowed": 55,
  "blocked": 0,
  "period": "24h",
  "timestamp": "2026-05-10T10:33:15.123456"
}
```

---

## Part 4 — security.html Dashboard Update

New section added to `security.html` between the Resolved Findings table and the Info Cards:

- **Allowed requests (24h)** — green counter, dynamic from `/waf` endpoint
- **Blocked requests (24h)** — red counter, dynamic from `/waf` endpoint
- **Shield Standard status** — static card, always shows ACTIVE with protected resources
- **Active rules** — static tags showing the 4 active WAF rules

The WAF section loads independently of the GuardDuty section, so a failure in one does not affect the other.

---

## Debugging Notes

**WAF created in us-east-1, Lambda in eu-north-1**  
Initial Lambda was created in us-east-1 to be "close" to the WAF. This made API Gateway integration impossible since the API lives in eu-north-1. Solution: Lambda in eu-north-1 with boto3 cross-region client targeting us-east-1 for CloudWatch calls. No performance impact — the CloudWatch call is internal AWS traffic.

**CloudWatchReadOnlyAccess insufficient**  
The managed policy `CloudWatchReadOnlyAccess` was attached but `GetMetricStatistics` was still denied when invoked via API Gateway. Root cause: the managed policy's effective permissions depend on the execution context. Fixed by adding an explicit inline policy granting `cloudwatch:GetMetricStatistics` on `*`.

**Wrong CloudWatch dimensions**  
Initial query used `{'Name': 'Region', 'Value': 'CloudFront'}` as a dimension. This dimension does not exist in the `AWS/WAFV2` namespace. Correct dimensions for `AllowedRequests`/`BlockedRequests` are `Rule` and `WebACL` only. Verified via CloudWatch Metrics browser in us-east-1.

**Period=86400 returns no datapoints for new WAF**  
First implementation used a 24h period (86400s) expecting a single datapoint. CloudWatch only returns a datapoint for a period when that period has fully elapsed — a WAF active for less than 24h has no complete 86400s bucket. Fixed by using `Period=300` (5 minutes) and summing all datapoints in the 24h window.

**WAF metrics lag**  
CloudWatch WAFV2 metrics have an approximate 5-10 minute publication delay. Live traffic visible in the WAF dashboard (22 requests) was not yet reflected in CloudWatch during initial testing. After waiting ~10 minutes, metrics appeared correctly.

---

## Cost Summary

| Service | Usage | Monthly Cost |
|---|---|---|
| AWS WAF Web ACL | 1 Web ACL | $5.00 |
| WAF Rule Groups | 3 managed + 1 custom | ~$4.00 |
| WAF Requests | ~$0.60 per 1M requests | ~$0 |
| Lambda (waf-metrics) | <100K invocations/month | $0 |
| CloudWatch GetMetricStatistics | Free tier | $0 |
| Shield Standard | Always included | $0 |

**Total added cost: ~$9/month**

---

## Key Concepts Practiced

- AWS WAF Web ACL scope: CloudFront (Global) vs Regional
- WAF rule evaluation order and WCU (Web ACL Capacity Units)
- AWS Managed Rule Groups: CRS, Known Bad Inputs, IP Reputation List
- Rate-based rules: per-IP request throttling
- Shield Standard vs Shield Advanced: capabilities and cost tradeoffs
- DDoS protection layers: L3/L4 (Shield) vs L7 (WAF)
- CloudWatch namespace `AWS/WAFV2`: dimensions, metric publication regions
- Cross-region boto3 clients in Lambda
- IAM: managed policy vs inline policy for fine-grained permissions
- API Gateway HTTP API: adding routes to existing APIs

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| SIEM Stack | Wazuh + OpenSearch + Grafana on EC2 inside VPC | High |
| GuardDuty sample findings | Integrated generator in security.html dashboard | Medium |
| RDS + DynamoDB Streams | Relational database layer, event-driven architecture | Medium |
| Auto Scaling + ELB | Load balancing and auto scaling groups | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

# OmegaNimbus Day 4: Automated Security Alerting & Real-Time Threat Dashboard

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** GuardDuty · EventBridge · SNS · Lambda · API Gateway · S3 · CloudFront  
**Date:** May 2026

---

## Overview

Building on the security monitoring stack activated in Day 3 (GuardDuty, CloudTrail, Security Hub), this session focused on two objectives:

1. **Automated Security Alerting** — real-time email notifications for GuardDuty findings via EventBridge + SNS
2. **Threat Dashboard** — a live Security Operations page on omeganimbus.com showing active GuardDuty findings via Lambda + API Gateway

---

## Part 1 — Automated Security Alerting (EventBridge + SNS)

### Architecture

```
GuardDuty Finding (severity >= 4)
         │
         ▼
Amazon EventBridge (event pattern rule)
         │
         ▼
Amazon SNS Topic (omeganimbus-security-alerts)
         │
         ▼
Email → sicart@protonmail.ch
```

### SNS Topic

Created a Standard SNS topic `omeganimbus-security-alerts` with an email subscription to `sicart@protonmail.ch`. Subscription confirmed via AWS verification email.

**Why Standard (not FIFO):** Standard topics support email subscriptions and have no ordering requirements — FIFO topics are for SQS queues where message ordering matters.

### EventBridge Rule

Created rule `omeganimbus-guardduty-alerts` on the default event bus with a custom JSON event pattern:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{"numeric": [">=", 4]}]
  }
}
```

**Severity scale:**
- Low: 1–3.9
- Medium: 4–6.9  
- High: 7–8.9  
- Critical: 9+

The filter `>= 4` captures Medium, High, and Critical findings — enough signal for a personal account without noise from purely informational events.

**Target:** SNS topic `omeganimbus-security-alerts`. EventBridge automatically creates the IAM execution role (`Amazon_EventBridge_Invoke_Sns_180158985`) with permissions to publish to the topic.

### Verification

Generated 389 sample findings via GuardDuty → Settings → Generate sample findings. Email notifications arrived within 60 seconds, each containing the full GuardDuty finding JSON including finding type, severity, affected resource, and detection timestamp.

**Note:** GuardDuty sample findings do trigger EventBridge rules despite some documentation suggesting otherwise. There is a delay of ~60 seconds before the events propagate.

### Key concepts practiced

- SNS topic creation and email subscription confirmation
- EventBridge rule with custom JSON event pattern
- Numeric filter syntax for severity thresholds
- GuardDuty → EventBridge → SNS pipeline
- IAM execution roles auto-created by EventBridge

---

## Part 2 — Real-Time Threat Dashboard

### Architecture

```
Browser (GET /security on page load)
         │
         ▼
API Gateway HTTP API (omeganimbus-api-cfn)
         │  GET /security
         ▼
Lambda (omeganimbus-guardduty-dashboard)
         │
         ▼
GuardDuty API (list_findings + get_findings)
         │
         ▼
Returns JSON → rendered in security.html
```

### Lambda Function

**Function name:** `omeganimbus-guardduty-dashboard`  
**Runtime:** Python 3.12 / arm64  
**IAM policy:** `AmazonGuardDutyReadOnlyAccess` attached to execution role

```python
import json
import boto3

def lambda_handler(event, context):
    
    client = boto3.client('guardduty', region_name='eu-north-1')
    
    detectors = client.list_detectors()
    detector_id = detectors['DetectorIds'][0]
    
    response = client.list_findings(
        DetectorId=detector_id,
        FindingCriteria={
            'Criterion': {
                'service.archived': {
                    'Eq': ['false']
                }
            }
        },
        SortCriteria={
            'AttributeName': 'severity',
            'OrderBy': 'DESC'
        },
        MaxResults=50
    )
    
    finding_ids = response.get('FindingIds', [])
    
    findings = []
    if finding_ids:
        details = client.get_findings(
            DetectorId=detector_id,
            FindingIds=finding_ids
        )
        for f in details['Findings']:
            findings.append({
                'id': f['Id'],
                'title': f['Title'],
                'type': f['Type'],
                'severity': f['Severity'],
                'region': f['Region'],
                'createdAt': f['CreatedAt'],
                'updatedAt': f['UpdatedAt'],
                'description': f.get('Description', '')
            })
    
    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': 'https://omeganimbus.com',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Methods': 'GET, OPTIONS'
        },
        'body': json.dumps({
            'findings': findings,
            'total': len(findings)
        })
    }
```

**Key design decisions:**
- `service.archived: Eq: ['false']` — only active findings, excluding archived/suppressed ones
- No severity filter — all findings shown for maximum visibility
- `MaxResults: 50` — enough for a personal account
- `SortCriteria: severity DESC` — most critical findings appear first

### API Gateway

Added route `GET /security` to the existing `omeganimbus-api-cfn` HTTP API. Created Lambda proxy integration and added resource-based policy to the Lambda function:

```
Source ARN: arn:aws:execute-api:eu-north-1:247906201225:judzwkiy9h/*/*/security
```

**Why resource-based policy matters:** API Gateway needs explicit permission to invoke a Lambda function. Without the `lambda:InvokeFunction` resource-based policy statement scoped to the specific API ARN, requests return 403 even if the integration is correctly configured.

### Frontend — security.html

New page at `omeganimbus.com/security.html` built with the same design language as the main site.

**Features:**
- Stats bar showing finding counts by severity (Critical / High / Medium / Low/Info) — updates on every load
- Full findings table with severity badge, finding title, finding type, detection timestamp, and region
- Manual refresh button
- Last updated timestamp
- Info cards explaining the underlying stack (GuardDuty, EventBridge+SNS, CloudTrail)

**Severity color coding:**
- Critical (9+) → red `#e74c3c`
- High (7–8.9) → orange `#e67e22`
- Medium (4–6.9) → yellow `#f1c40f`
- Low (<4) → green `#2ecc71`

**Deployed via:** GitHub → CodePipeline → S3 (`omeganimbus.com-cfn`) → CloudFront

### index.html updates

- Added **SecOps** link to main navigation pointing to `security.html`
- Updated **AWS Security Monitoring Stack** project card from `Planned` to `Live` with direct link to the dashboard

---

## Active Findings (Day 4 baseline)

| Severity | Finding | Type |
|---|---|---|
| High 8.0 | Amazon S3 Public Anonymous Access was granted for the S3 bucket omeganimbus.com-cfn | Policy:S3/BucketAnonymousAccessGranted |
| Low 2.0 | The API GetAccountSettings was invoked using root credentials | Policy:IAMUser/RootCredentialUsage |
| Low 2.0 | The API GetRole was invoked using root credentials | Policy:IAMUser/RootCredentialUsage |
| Low 2.0 | Amazon S3 Block Public Access was disabled for the S3 bucket omeganimbus.com-cfn | Policy:S3/BucketBlockPublicAccessDisabled |
| Low 2.0 | An AWS CloudTrail trail was disabled | Stealth:IAMUser/CloudTrailLoggingDisabled |

**Notes on findings:**
- The S3 High finding is expected — the bucket has public access enabled for website hosting. In a production environment, this would be mitigated by using an Origin Access Control (OAC) in CloudFront instead of a public bucket policy.
- The RootCredentialUsage findings are from early account setup actions performed before IAM users/roles were configured.
- The CloudTrail finding reflects the brief period between account creation and trail activation.

---

## Cost Summary

| Service | Usage | Cost |
|---|---|---|
| SNS | <1M publishes/month | ~$0 |
| EventBridge | <1M events/month | ~$0 |
| Lambda (dashboard) | <1M requests/month | ~$0 |
| API Gateway (GET /security) | <1M calls/month | ~$0 |
| GuardDuty | 30-day trial active | ~$0 |

**Total added cost: ~$0/month**

---

## Key Concepts Practiced

- SNS topic and email subscription management
- EventBridge event pattern rules with numeric filters
- GuardDuty API: `list_detectors`, `list_findings`, `get_findings`
- Lambda resource-based policies for API Gateway invocation
- Real-time data fetching from AWS APIs via frontend JavaScript
- Severity-based color coding and UI state management
- CI/CD deployment of new pages via GitHub → CodePipeline

---

## What's Next

| Feature | AWS Services | Priority |
|---|---|---|
| Study Notes section | S3 + CloudFront | Medium |
| Chatbot | Amazon Lex | Medium |
| Image recognition demo | Amazon Rekognition | Medium |
| RDS + ElastiCache | Relational database layer | Low |
| WAF integration | AWS WAF + CloudFront | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

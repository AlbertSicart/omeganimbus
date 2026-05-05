# OmegaNimbus Day 2: IAM Least Privilege, Serverless Visitor Counter & CI/CD Pipeline

**Author:** Albert Sicart  
**Domain:** [omeganimbus.com](https://omeganimbus.com)  
**Stack:** IAM · Lambda · DynamoDB · API Gateway · CodePipeline · GitHub  
**Date:** May 2026

---

## Overview

Building on the infrastructure deployed in Day 1 (S3, CloudFront, ACM, Lambda, SES), this session focused on three objectives:

1. **IAM Least Privilege** — replacing overly permissive managed policies with custom scoped policies
2. **Serverless Visitor Counter** — real-time visit tracking using Lambda + DynamoDB
3. **CI/CD Pipeline** — automated deployments from GitHub to S3 via CodePipeline

---

## Part 1 — IAM Least Privilege Refactor

### Problem
The Lambda execution roles were using AWS managed policies (`AmazonSESFullAccess`, `AmazonDynamoDBFullAccess`) which grant far more permissions than necessary. This violates the principle of least privilege — a core security requirement for any production environment.

### Solution
Created custom IAM policies scoped to the exact actions and resources required.

#### SES Policy — `omeganimbus-ses-send-only`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSESSendEmailOnly",
      "Effect": "Allow",
      "Action": "ses:SendEmail",
      "Resource": [
        "arn:aws:ses:eu-north-1:247906201225:identity/omeganimbus.com",
        "arn:aws:ses:eu-north-1:247906201225:identity/sicart@protonmail.ch"
      ]
    }
  ]
}
```

**Key insight:** SES requires permissions on both the sender identity (`omeganimbus.com`) and the recipient identity (`sicart@protonmail.ch`) when operating in sandbox mode. Scoping to only the sender identity causes silent failures.

#### DynamoDB Policy — `omeganimbus-dynamodb-visitors-only`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDynamoDBVisitorCounter",
      "Effect": "Allow",
      "Action": [
        "dynamodb:UpdateItem",
        "dynamodb:GetItem"
      ],
      "Resource": "arn:aws:dynamodb:eu-north-1:247906201225:table/omeganimbus-visitors"
    }
  ]
}
```

Only `UpdateItem` (to increment the counter) and `GetItem` (to read it) are allowed — on the specific table ARN only.

### Result
Both Lambda functions operate with the minimum permissions required. Full managed policies removed.

### Key concepts practiced
- IAM custom policy creation (JSON editor)
- Resource-level permissions using ARNs
- Principle of least privilege
- Difference between identity-based and resource-based policies

---

## Part 2 — Serverless Visitor Counter (Lambda + DynamoDB)

### Architecture

```
Browser (GET request on page load)
     │
     ▼
API Gateway (HTTP API)
     │
     ▼
Lambda (Python 3.12) — increments counter
     │
     ▼
DynamoDB table: omeganimbus-visitors
     │
     ▼
Returns { visits: N } → displayed in hero section
```

### DynamoDB Table
- **Table name:** `omeganimbus-visitors`
- **Partition key:** `id` (String)
- **Item:** `{ id: "total", visits: 0 }`

### Lambda Function

```python
import json
import boto3

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb', region_name='eu-north-1')
    table = dynamodb.Table('omeganimbus-visitors')

    response = table.update_item(
        Key={'id': 'total'},
        UpdateExpression='ADD visits :inc',
        ExpressionAttributeValues={':inc': 1},
        ReturnValues='UPDATED_NEW'
    )

    count = int(response['Attributes']['visits'])

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': 'https://omeganimbus.com',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Methods': 'GET, OPTIONS'
        },
        'body': json.dumps({'visits': count})
    }
```

**Note:** Python 3.12 on Lambda has a known issue with importing `Decimal` from the `decimal` module. Using native integer types with DynamoDB's `ADD` expression resolves this without workarounds.

### Frontend Integration

```javascript
const COUNTER_ENDPOINT = 'https://5p1tgv1b09.execute-api.eu-north-1.amazonaws.com/default/omeganimbus-visitor-counter';

fetch(COUNTER_ENDPOINT)
  .then(r => r.json())
  .then(data => {
    document.getElementById('visit-count').textContent = data.visits.toLocaleString();
  })
  .catch(() => {
    document.getElementById('visitor-counter').style.display = 'none';
  });
```

The counter is displayed in the hero section as `// N VISITORS` and increments on every page load.

### Debugging Notes
- Initial Lambda used `Decimal` from Python's `decimal` module → `ImportModuleError` on Python 3.12
- DynamoDB table was empty on first invocation → `UpdateItem` with `ADD` on a non-existent item requires the item to exist first, or use `if_not_exists` — resolved by seeding the initial item manually
- CORS required explicit configuration on the API Gateway for `GET` and `OPTIONS` methods

### Key concepts practiced
- DynamoDB table design (single-item counter pattern)
- Atomic increment operations with `UpdateExpression`
- Lambda + API Gateway + DynamoDB serverless pattern
- CORS configuration for GET requests
- Error handling and graceful degradation in frontend

---

## Part 3 — CI/CD Pipeline (CodePipeline + GitHub → S3)

### What I built
An automated deployment pipeline that triggers on every push to the `main` branch of the GitHub repository and deploys the updated files to the S3 bucket — replacing the manual upload workflow.

### Architecture

```
Local machine
     │
     git push
     │
     ▼
GitHub (AlbertSicart/omeganimbus — main branch)
     │
     Webhook trigger
     │
     ▼
AWS CodePipeline
     │
     ├── Source stage: GitHub via GitHub App
     │
     └── Deploy stage: Amazon S3 (omeganimbus.com bucket)
                            │
                            ▼
                    CloudFront serves updated content
```

### Pipeline Configuration
- **Pipeline name:** `omeganimbus-pipeline`
- **Pipeline type:** V2
- **Execution mode:** Superseded (new execution cancels in-progress one)
- **Source:** GitHub via GitHub App (AWS Connector for GitHub)
- **Repository:** `AlbertSicart/omeganimbus`
- **Branch:** `main`
- **Trigger:** Webhook on push events
- **Build stage:** Skipped (static site, no compilation needed)
- **Deploy stage:** Amazon S3, `omeganimbus.com` bucket, extract files before deploy ✓

### GitHub App Connection
Connected AWS CodePipeline to GitHub using the **AWS Connector for GitHub** app, scoped to the `omeganimbus` repository only — following least privilege principles at the source level as well.

### Result
First automated deployment completed in **7 seconds** after commit. The pipeline execution log shows:

```
Trigger: Commit 80e7b20a pushed in AlbertSicart/omeganimbus/main
Status: Succeeded
Duration: 7 seconds
```

### Workflow going forward
```
Edit code locally → git push → Pipeline deploys automatically in ~7s
```
No more manual S3 uploads.

### Key concepts practiced
- CI/CD pipeline design and implementation
- CodePipeline stages (Source → Build → Deploy)
- GitHub App OAuth integration with AWS
- Webhook-based pipeline triggers
- Artifact handling between pipeline stages

---

## Cost Summary

All services remain within the AWS Free Tier:

| Service | Usage | Cost |
|---------|-------|------|
| DynamoDB | 25 GB storage, 25 WCU/RCU | ~$0 |
| CodePipeline | 1 active pipeline (free tier: 1) | ~$0 |
| Lambda (counter) | <1M requests/month | ~$0 |
| API Gateway (counter) | <1M calls/month | ~$0 |

**Total added cost: $0/month**

---

## Security Posture Summary

| Resource | Policy | Permissions |
|----------|--------|-------------|
| Contact form Lambda | `omeganimbus-ses-send-only` | `ses:SendEmail` on 2 specific identities |
| Visitor counter Lambda | `omeganimbus-dynamodb-visitors-only` | `dynamodb:UpdateItem`, `dynamodb:GetItem` on 1 specific table |
| CodePipeline | AWS managed (auto-created) | S3 deploy permissions scoped to pipeline |
| GitHub App | AWS Connector | Read/write on `omeganimbus` repo only |

All Lambda functions operate under custom least-privilege policies. No AWS managed full-access policies remain attached.

---

## What's Next

| Feature | AWS Services | Priority |
|---------|-------------|----------|
| Security monitoring | GuardDuty + CloudTrail + Security Hub | High |
| Infrastructure as Code | CloudFormation / SAM | High |
| Chatbot | Amazon Lex | Medium |
| Image recognition demo | Amazon Rekognition | Medium |
| RDS + ElastiCache | Relational database layer | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

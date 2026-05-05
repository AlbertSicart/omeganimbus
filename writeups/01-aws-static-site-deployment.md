# Deploying OmegaNimbus: A Serverless Cloud Security Portfolio on AWS

**Author:** Albert Sicart  
**Domain:** [omeganimbus.com](https://omeganimbus.com)  
**Stack:** S3 · CloudFront · ACM · Route 53 (DNS) · Lambda · API Gateway · SES · IAM · CloudWatch  
**Date:** May 2026

---

## Overview

This writeup documents the full deployment of **OmegaNimbus**, a personal cloud security portfolio website, built entirely on AWS using serverless and managed services. The project covers static site hosting, CDN configuration, HTTPS with a custom domain, and a fully serverless contact form pipeline — all within the AWS Free Tier.

The goal was twofold: build a production-grade personal brand site, and use the process as hands-on practice for the AWS Solutions Architect Associate (SAA-C03) and Security Specialty (SCS-C02) certification paths.

---

## Architecture

```
User Browser
     │
     ▼
omeganimbus.com (DNS via Don Dominio)
     │
     ▼
Amazon CloudFront (CDN + HTTPS)
     │
     ├──► S3 Bucket (static website hosting)
     │
     └──► API Gateway (HTTP API)
               │
               ▼
           AWS Lambda (Python 3.12)
               │
               ▼
           Amazon SES (email delivery)
               │
               ▼
         sicart@protonmail.ch
```

---

## Phase 1 — Static Website Hosting on S3

### What I built
A static HTML/CSS/JS website hosted on Amazon S3 with public read access, served as a static website.

### Steps
1. Created an S3 bucket named `omeganimbus.com` in `eu-north-1` (Stockholm).
2. Disabled "Block all public access" to allow public read.
3. Enabled **Static Website Hosting** under bucket Properties, setting `index.html` as both the index and error document.
4. Attached a **bucket policy** granting `s3:GetObject` to all principals (`"Principal": "*"`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::omeganimbus.com/*"
    }
  ]
}
```

5. Uploaded `index.html` to the bucket.

### Result
The site was accessible via the S3 website endpoint:
```
http://omeganimbus.com.s3-website.eu-north-1.amazonaws.com
```

### Key concepts practiced
- S3 bucket policies
- Static website hosting
- Public access configuration
- IAM resource-based policies

---

## Phase 2 — HTTPS and CDN with CloudFront + ACM

### What I built
A CloudFront distribution serving the S3 bucket over HTTPS, with a custom domain and a TLS certificate issued by AWS Certificate Manager.

### Steps

#### 2.1 — Request SSL Certificate (ACM)
1. Switched region to **us-east-1 (N. Virginia)** — required for CloudFront certificates.
2. Requested a **public certificate** via AWS Certificate Manager for:
   - `omeganimbus.com`
   - `www.omeganimbus.com`
3. Chose **DNS validation** method.
4. Added the two CNAME validation records to the domain's DNS zone in Don Dominio.
5. Waited for both domains to reach **Success** status → certificate issued.

#### 2.2 — Create CloudFront Distribution
1. Created a new CloudFront distribution with origin:
   ```
   omeganimbus.com.s3-website.eu-north-1.amazonaws.com
   ```
   > **Note:** Used the S3 static website endpoint, not the S3 REST endpoint, to correctly serve the index document.

2. Added alternate domain names (CNAMEs):
   - `omeganimbus.com`
   - `www.omeganimbus.com`

3. Attached the ACM certificate issued in step 2.1.

#### 2.3 — DNS Configuration
Added two records in Don Dominio DNS zone:

| Type  | Host | Value                              |
|-------|------|------------------------------------|
| ANAME | @    | `d33nj3prjmvy0x.cloudfront.net`   |
| CNAME | www  | `d33nj3prjmvy0x.cloudfront.net`   |

#### 2.4 — Cache Invalidation
After uploading updated files to S3, invalidated the CloudFront cache using path `/*` to force immediate propagation.

### Result
The site became accessible at:
```
https://omeganimbus.com
https://www.omeganimbus.com
```
With a valid TLS certificate and global CDN distribution.

### Key concepts practiced
- CloudFront distributions and origin configuration
- ACM certificate provisioning and DNS validation
- HTTPS termination at the edge
- Cache invalidation strategies
- DNS record types (ANAME, CNAME)

---

## Phase 3 — Serverless Contact Form (Lambda + API Gateway + SES)

### What I built
A fully serverless contact form pipeline: the website sends a POST request to an API Gateway endpoint, which triggers a Lambda function that sends a formatted email via Amazon SES.

### Architecture
```
Browser (fetch POST)
     │
     ▼
API Gateway (HTTP API, Open auth)
     │
     ▼
Lambda Function (Python 3.12)
     │
     ▼
SES → contact@omeganimbus.com → sicart@protonmail.ch
```

### Steps

#### 3.1 — Verify SES Identity
1. Verified `sicart@protonmail.ch` as a sending identity in SES (eu-north-1).
2. Later verified the full domain `omeganimbus.com` via DKIM — added 3 CNAME records to DNS:
   - `_domainkey` CNAME records pointing to `dkim.amazonses.com`
3. Sending from `contact@omeganimbus.com` significantly improved email deliverability.

#### 3.2 — Create Lambda Function
Created a Python 3.12 Lambda function `omeganimbus-contact-form`:

```python
import json
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    # Handle CORS preflight
    if event.get('requestContext', {}).get('http', {}).get('method') == 'OPTIONS':
        return response(200, 'OK')

    try:
        body = json.loads(event.get('body', '{}'))
    except Exception:
        return response(400, 'Invalid request body')

    name    = body.get('name', '').strip()
    email   = body.get('email', '').strip()
    subject = body.get('subject', '').strip()
    message = body.get('message', '').strip()

    if not all([name, email, subject, message]):
        return response(400, 'All fields are required')

    ses = boto3.client('ses', region_name='eu-north-1')
    try:
        ses.send_email(
            Source='contact@omeganimbus.com',
            Destination={'ToAddresses': ['sicart@protonmail.ch']},
            Message={
                'Subject': {'Data': f'[OmegaNimbus] {subject}'},
                'Body': {'Text': {'Data': f'From: {name}\nEmail: <{email}>\n\n{message}'}}
            }
        )
    except ClientError as e:
        return response(500, str(e))

    return response(200, 'Email sent successfully')


def response(status_code, message):
    return {
        'statusCode': status_code,
        'headers': {
            'Access-Control-Allow-Origin': 'https://omeganimbus.com',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Methods': 'POST, OPTIONS'
        },
        'body': json.dumps({'message': message})
    }
```

#### 3.3 — IAM Permissions
Attached `AmazonSESFullAccess` to the Lambda execution role.

> **Security note:** This is overly permissive for production. The correct approach (to be implemented) is a custom IAM policy scoped to `ses:SendEmail` on the specific identity ARN only — following the principle of least privilege.

#### 3.4 — API Gateway
Added an HTTP API trigger to the Lambda function:
- Type: HTTP API
- Auth: Open
- CORS configured:
  - Allow-Origin: `https://omeganimbus.com`
  - Allow-Headers: `content-type`
  - Allow-Methods: `POST, OPTIONS`
  - Auto-deploy: enabled

#### 3.5 — Frontend Integration
Updated `index.html` to send a `fetch` POST request to the API endpoint:

```javascript
const API_ENDPOINT = 'https://06w1j6z7kg.execute-api.eu-north-1.amazonaws.com/default/omeganimbus-contact-form';

form.addEventListener('submit', async (e) => {
  e.preventDefault();
  const payload = {
    name:    document.getElementById('name').value,
    email:   document.getElementById('email').value,
    subject: document.getElementById('subject').value,
    message: document.getElementById('message').value
  };
  const res = await fetch(API_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  });
  // handle response...
});
```

### Debugging with CloudWatch
Used CloudWatch Logs to debug the Lambda function in real time. Key issues identified and resolved:
- **CORS preflight (OPTIONS) not handled** — Lambda was processing OPTIONS requests as form submissions, returning empty body errors.
- **Body arriving empty** — identified via `print("BODY:", body)` log statements.
- **SES deliverability** — initial sends from `sicart@protonmail.ch` via SES were silently dropped by ProtonMail. Resolved by verifying the custom domain `omeganimbus.com` with DKIM and sending from `contact@omeganimbus.com`.

### Result
Contact form fully functional. Emails arrive from `contact@omeganimbus.com` to `sicart@protonmail.ch`.

### Key concepts practiced
- AWS Lambda (serverless compute)
- API Gateway (HTTP API, CORS configuration)
- Amazon SES (email identity verification, DKIM, sandbox mode)
- IAM execution roles
- CloudWatch Logs for debugging
- CORS handling in serverless architectures

---

## What's Next

The following improvements and extensions are planned:

| Feature | AWS Services | Certification relevance |
|--------|-------------|------------------------|
| IAM least privilege refactor | IAM custom policies | SAA-C03, SCS-C02 |
| Visit counter | Lambda + DynamoDB | SAA-C03 |
| CI/CD pipeline | CodePipeline + CodeBuild | SAA-C03 |
| Security monitoring | GuardDuty + Security Hub + CloudTrail | SCS-C02 |
| Infrastructure as Code | CloudFormation / SAM | SAA-C03 |
| Chatbot | Amazon Lex | ML/AI |
| Image recognition demo | Amazon Rekognition | ML/AI |

---

## Cost

All services used are within the **AWS Free Tier**:

| Service | Free Tier | Estimated cost |
|---------|-----------|---------------|
| S3 | 5 GB storage, 20k GET requests | ~$0 |
| CloudFront | 1 TB transfer, 10M requests | ~$0 |
| ACM | Free for public certificates | $0 |
| Lambda | 1M requests/month | ~$0 |
| API Gateway | 1M HTTP calls/month | ~$0 |
| SES | 62k emails/month (from EC2/Lambda) | ~$0 |
| CloudWatch | 5 GB logs | ~$0 |

**Total: ~$0/month** (first 12 months)

---

## Takeaways

This project covered a significant portion of the AWS SAA-C03 exam domains in a practical, production context:

- **Design Resilient Architectures** — CDN with origin failover, serverless scaling
- **Design High-Performing Architectures** — CloudFront caching, Lambda cold start optimization
- **Design Secure Applications** — bucket policies, IAM roles, HTTPS enforcement, CORS
- **Design Cost-Optimized Architectures** — serverless over EC2, Free Tier utilization

More importantly, it produced a live, publicly accessible result — not just a lab exercise.

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*

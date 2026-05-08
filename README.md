# OmegaNimbus — Project Roadmap

**Domain:** omeganimbus.com  
**Author:** Albert Sicart  
**Goal:** Production-grade AWS cloud security portfolio, built from scratch, documented day by day.

---

## Completed

### Day 1 — Static Hosting + CDN + TLS ✅
*May 2026, 1st*

S3 static website hosting, CloudFront distribution with custom domain, ACM certificate (us-east-1), HTTPS redirect, SES identity verification, Lambda + API Gateway contact form.

### Day 2 — IAM, DynamoDB & CI/CD Pipeline ✅
*May 2026, 2nd*

IAM least privilege roles for each Lambda, DynamoDB visitor counter, CodePipeline connected to GitHub via GitHub App. Deploy time: ~7 seconds per push.

### Day 3 — Security Monitoring & Infrastructure as Code ✅
*May 2026, 3rd*

GuardDuty enabled, CloudTrail with KMS encryption, Security Hub with AWS Foundational Security Best Practices + CIS AWS Foundations. Full stack migrated to CloudFormation (15 resources managed as code).

### Day 4 — Automated Alerting & Threat Dashboard ✅
*May 2026, 4th*

EventBridge rule triggering SNS email alerts on GuardDuty findings (severity ≥ 4). Real-time GuardDuty dashboard on `security.html` — Lambda + API Gateway querying findings live.

### Day 5 — Computer Vision & AI Image Analysis ✅
*May 2026, 5th*

Rekognition pipeline (detect_labels, detect_text, detect_faces) + Amazon Bedrock Claude Haiku 4.5 for AI-generated image descriptions. Live at `omeganimbus.com/rekognition.html`.

### Day 6 — Conversational AI Assistant ✅
*May 2026, 6th*

Amazon Lex V2 bot with 6 custom intents covering skills, projects, certifications, and contact. Lambda handler invoking Lex runtime, `POST /chat` endpoint on existing API Gateway. Floating chat widget integrated across all portfolio pages.

---

## Upcoming

### Day 7 — Route 53 + AWS Budgets
DNS migration from DonDominio to Route 53. Hosted Zone, A/CNAME records, nameserver cutover. AWS Budgets alarm for monthly spend control.

### Day 8 — CloudWatch + AWS Config
CloudWatch dashboards with real metrics: API Gateway latency, Lambda errors, DynamoDB read/write capacity. Alarms configured. AWS Config enabled with baseline compliance rules.

### Day 9 — Study Notes Polish + S3 Lifecycle
CCP study notes section improvements. SAA-C03 structure preparation. S3 versioning and lifecycle policies on the main bucket.

### Day 10 — VPC
Custom VPC with public and private subnets, security groups, route tables, and NAT Gateway. Lambda functions moved inside the VPC. Core SAA-C03 networking concepts applied to production infrastructure.

### Day 11 — WAF + Shield + GuardDuty Improvements
AWS WAF on CloudFront with rate limiting and geo-blocking rules. Shield Standard review. GuardDuty sample findings generator integrated into the security dashboard.

### Day 12 — RDS + DynamoDB Streams
Relational database layer with RDS. DynamoDB Streams with reactive Lambda — event-driven architecture pattern.

---

## Stack Reference

| Layer | Service | Region |
|---|---|---|
| CDN | CloudFront | Global |
| DNS | Route 53 (Day 7) | Global |
| Storage | S3 | eu-north-1 |
| Compute | Lambda | eu-north-1 / eu-west-1 |
| API | API Gateway HTTP API | eu-north-1 |
| Database | DynamoDB | eu-north-1 |
| Email | SES | eu-north-1 |
| IaC | CloudFormation | eu-north-1 |
| CI/CD | CodePipeline | eu-north-1 |
| Threat Detection | GuardDuty | eu-north-1 |
| Audit | CloudTrail + KMS | eu-north-1 |
| Compliance | Security Hub | eu-north-1 |
| Alerting | EventBridge + SNS | eu-north-1 |
| Computer Vision | Rekognition | eu-west-1 |
| AI | Bedrock (Claude Haiku 4.5) | eu-west-1 |
| Conversational AI | Amazon Lex V2 | eu-west-1 |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*


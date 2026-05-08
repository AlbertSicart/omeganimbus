# OmegaNimbus Day 7: DNS Migration to Route 53 & FinOps with AWS Budgets

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** Route 53 · AWS Budgets · Cost Anomaly Detection · CloudFront  
**Date:** May 2026

---

## Overview

This session focused on two objectives:

1. **Route 53** — full DNS migration from DonDominio to AWS, completing the stack entirely within the AWS ecosystem
2. **AWS Budgets + Cost Anomaly Detection** — FinOps controls to monitor and alert on unexpected spend

The result is a fully AWS-managed domain with zero downtime during cutover, and an automated cost monitoring pipeline covering both threshold-based and anomaly-based alerting.

---

## Architecture

```
Browser (DNS query: omeganimbus.com)
         │
         ▼
Route 53 Public Hosted Zone (omeganimbus.com)
         │
         ├── A record (apex) → Alias → CloudFront (d2l9mcdh2l7kk2.cloudfront.net)
         ├── CNAME www       → d2l9mcdh2l7kk2.cloudfront.net
         ├── CNAME x3        → SES DKIM verification
         └── CNAME x2        → ACM certificate validation
                  │
                  ▼
         CloudFront Distribution (E2Y1NNQD999CRW)
                  │
                  ▼
         S3 Origin (omeganimbus.com-cfn)

Cost Monitoring Pipeline:
AWS Budgets ($10/month) → Email alert at 85% ($8.50)
Cost Anomaly Detection  → Email alert on anomalous spend (>$100 or >40%)
```

---

## Part 1 — Route 53 Hosted Zone

**Hosted Zone:** `omeganimbus.com` (Public)  
**Nameservers assigned by AWS:**
- `ns-873.awsdns-45.net`
- `ns-1089.awsdns-08.org`
- `ns-1813.awsdns-34.co.uk`
- `ns-404.awsdns-50.com`

### Records created

| Record | Type | Value | Notes |
|---|---|---|---|
| `omeganimbus.com` | A (Alias) | `d2l9mcdh2l7kk2.cloudfront.net` | Apex → CloudFront |
| `www.omeganimbus.com` | CNAME | `d2l9mcdh2l7kk2.cloudfront.net` | www → CloudFront |
| `vi3gbtjcslim4xprqmzevqles6slmu5j._domainkey` | CNAME | `vi3gbtjcslim4xprqmzevqles6slmu5j.dkim.amazonses.com` | SES DKIM |
| `wa65zivwkrvi553xsaeesg2p3wlnmp2n._domainkey` | CNAME | `wa65zivwkrvi553xsaeesg2p3wlnmp2n.dkim.amazonses.com` | SES DKIM |
| `zid7ygqrs3g3koagojaitppr5xujbu76._domainkey` | CNAME | `zid7ygqrs3g3koagojaitppr5xujbu76.dkim.amazonses.com` | SES DKIM |
| `_1741f0ee3d6bfd8794f8697d035137f9` | CNAME | `_3febde60f1894cfa663fc9423edf6f7c.jkddzztszm.acm-validations.aws` | ACM validation |
| `_1ea2743fcdfe285454e36ed4af6e17e8` | CNAME | `_4a6ab75006a2e7906ca73ddd0a5f76b7.jkddzztszm.acm-validations.aws` | ACM validation |

**Records intentionally omitted:** DonDominio parking, FTP, mail, webmail, BBDD — all pointed to DonDominio hosting services that were never active.

### Apex record: A Alias vs CNAME

The root domain (`omeganimbus.com`) cannot use a CNAME record per DNS specification — only subdomains can. Route 53 solves this with **Alias records**: an AWS-specific extension that behaves like a CNAME but is valid at the apex. It resolves directly to the CloudFront IPs with no extra DNS hop, and has no additional cost.

### Nameserver cutover

Nameservers changed in DonDominio from `ns1/ns2.dondominio.com` to the 4 AWS nameservers above. DNS propagation verified with `nslookup omeganimbus.com` — resolving to CloudFront IPs (`108.157.128.x`) within minutes. Zero downtime.

---

## Part 2 — AWS Budgets

**Budget name:** `omeganimbus-monthly-budget`  
**Type:** Monthly cost budget  
**Amount:** $10.00  
**Alert threshold:** 85% actual cost ($8.50) → email to `sicart@protonmail.ch`  
**Period:** Monthly, recurring

### Budget Actions

Budget Actions (automated responses to threshold breaches) were evaluated but not configured. The available actions — applying IAM deny policies and stopping EC2/RDS instances — do not apply to the current stack, which runs entirely on serverless services (Lambda, S3, CloudFront, API Gateway, DynamoDB). The email alert at 85% is sufficient for this architecture.

---

## Part 3 — Cost Anomaly Detection

Automatically configured by AWS upon first access to the Billing console. Uses ML models to detect anomalous spend patterns across all services.

**Default configuration:**
- Monitor: All AWS services
- Alert threshold: spend exceeding $100 **or** 40% above expected
- Delivery: Daily summary email + immediate alert on detection

This complements the fixed-threshold Budget by catching unexpected spikes that might still be under $10 but are anomalous relative to baseline (e.g. a runaway Lambda invocation loop).

---

## Debugging Notes

**Apex domain cannot use CNAME**  
DNS specification prohibits CNAME records at the zone apex. Attempted to create a CNAME for `omeganimbus.com` → CloudFront. Fixed by using Route 53 Alias A record instead, which achieves the same result without violating DNS standards.

**DonDominio auto-populated AWS nameservers**  
DonDominio had the 4 AWS nameservers pre-saved as preferred DNS in the account settings — they appeared automatically when switching to custom nameservers. This was a result of a previous configuration attempt on the account.

**Budget Actions not applicable to serverless stack**  
Budget Actions support IAM policy attachment and EC2/RDS instance termination. Neither applies to a Lambda + S3 + CloudFront architecture. The feature is noted for future use when RDS is added (Day 12).

---

## Cost Summary

| Service | Usage | Monthly Cost |
|---|---|---|
| Route 53 Hosted Zone | 1 zone | $0.50 |
| Route 53 DNS Queries | <1M queries/month | ~$0 |
| AWS Budgets | 1 budget (free tier: 2 budgets) | $0 |
| Cost Anomaly Detection | Included with Cost Explorer | $0 |

**Total added cost: $0.50/month** (Route 53 hosted zone flat fee)

---

## Key Concepts Practiced

- Route 53 public hosted zones and DNS record types (A, CNAME, Alias)
- Apex domain routing via Route 53 Alias records
- DNS nameserver delegation and propagation
- SES DKIM verification via DNS CNAME records
- ACM certificate validation via DNS CNAME records
- AWS Budgets: threshold-based cost alerting
- Cost Anomaly Detection: ML-based spend monitoring
- FinOps principles: budget governance in serverless architectures

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| CloudWatch dashboards | Real metrics: API Gateway latency, Lambda errors, DynamoDB capacity | High |
| AWS Config | Compliance rules and resource change tracking | High |
| S3 versioning + lifecycle | Versioning and cost-optimized lifecycle policies on main bucket | Medium |
| VPC | Custom VPC with public/private subnets, security groups | Medium |
| WAF + Shield | DDoS protection on CloudFront | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*
